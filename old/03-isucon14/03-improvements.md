# 3-3. ISUCON14 改善方法案

> **このドキュメントで学ぶこと**
> - 3-2 の各ボトルネックに対する具体的な改善策（公式講評ベース）
> - 実施する順番の考え方

3-2 のボトルネック番号（①〜⑥）に対応させて改善案を示します。コード例は TypeScript 風の擬似コードです。

## ① インデックス追加（最初の30分でやる）

公式解説で挙げられた追加候補:

```sql
-- 認証（毎リクエスト実行される。最優先）
ALTER TABLE users  ADD INDEX idx_access_token (access_token);
ALTER TABLE chairs ADD INDEX idx_access_token (access_token);

-- 招待コード・クーポン
ALTER TABLE users   ADD INDEX idx_invitation_code (invitation_code);
ALTER TABLE coupons ADD INDEX idx_user_id_code (user_id, code);

-- 「最新1件」系（WHERE等値 + ORDER BY を1本で解決 → 2-2 ルール2）
ALTER TABLE rides           ADD INDEX idx_chair_id_updated (chair_id, updated_at DESC);
ALTER TABLE rides           ADD INDEX idx_user_id_created (user_id, created_at DESC);
ALTER TABLE ride_statuses   ADD INDEX idx_ride_id_created (ride_id, created_at DESC);
ALTER TABLE chair_locations ADD INDEX idx_chair_id_created (chair_id, created_at DESC);

-- オーナー画面
ALTER TABLE chairs ADD INDEX idx_owner_id (owner_id);
```

**初期化SQLに組み込むのを忘れずに**（2-2 の実戦手順参照）。

## ② 通知エンドポイントの軽量化

段階的に3レベルの解法があります。公式講評は「**SSE 化は必須ではなく、JSON API + 適切なキャッシュでも10万点以上に到達できた**」としています。

**レベル1: クエリ改善**。①のインデックスと⑤の N+1 解消だけでもかなり軽くなる。

**レベル2: ライド状態のオンメモリキャッシュ**。通知の本質は「そのユーザー/椅子の最新ライドと最新ステータス」。これをメモリに持ち、状態が変わるとき（ライド作成・ステータス更新・マッチング時）にキャッシュを更新すれば、通知はほぼ DB を触らずに返せる。加えて `retry_after_ms` を調整して「負荷 vs 満足度」のバランスを取る。

```typescript
// ライドの最新状態キャッシュ（更新経路は3箇所: 作成/status遷移/マッチ）
const latestRideByUser = new Map<string, RideWithStatus>();
const latestRideByChair = new Map<string, RideWithStatus>();
```

**レベル3: SSE (Server-Sent Events) 化**。ポーリング自体をやめ、サーバー側から状態変化をプッシュする。リクエスト数が激減するが実装コストは高い。上位チームの一部が採用。

## ③ マッチングアルゴリズムの改善（この問題の華）

初期実装の「500msに1件・ランダム」を段階的に改善します:

1. **全件処理**: 1回の実行で、待機中の**全ライド × 空き椅子**をまとめてマッチングする
2. **実行間隔の短縮**: 500ms → 数十ms（例: 参加記では 10ms まで縮めた例も）。待ち時間 = 満足度に直結
3. **距離ベースの割り当て（貪欲法）**: 各ライドに対して**乗車位置に最も近い空き椅子**を割り当てる。距離はマンハッタン距離:

   ```typescript
   const distance = (a: Coord, b: Coord) =>
     Math.abs(a.lat - b.lat) + Math.abs(a.lon - b.lon);

   // 貪欲法: 待機ライドごとに最近傍の空き椅子を選ぶ
   for (const ride of waitingRides) {
     let best: Chair | null = null;
     for (const chair of availableChairs) {
       if (!best || distance(chair.pos, ride.pickup) < distance(best.pos, ride.pickup)) {
         best = chair;
       }
     }
     if (best && distance(best.pos, ride.pickup) <= 400) {  // 4. 距離しきい値
       assign(ride, best);
       availableChairs.delete(best.id);
     }
   }
   ```

4. **距離しきい値**: 遠すぎる組み合わせ（目安 ~400）はあえてマッチさせず次回に回す。ISURIDE の世界は2つの地域に分かれており、ユーザーは地域内しか移動しないため、**地域をまたぐマッチングは空車距離の無駄**になる
5. （発展）2部グラフの最適マッチング等のアルゴリズムも適用可能だが、公式講評いわく**貪欲法でも十分上位スコアに到達できた**

これらの高速化の前提として、「空き椅子の最新位置」が高速に引けること（④⑤の改善）が必要です。改善が連鎖する構造がよくわかります。

## ④ 位置情報の最新値テーブル化 + 累積距離の差分更新

2-4 パターンAの教科書どおり:

```sql
CREATE TABLE chair_latest_locations (
  chair_id VARCHAR(26) PRIMARY KEY,
  latitude INT NOT NULL,
  longitude INT NOT NULL,
  total_distance INT NOT NULL DEFAULT 0,
  updated_at DATETIME(6) NOT NULL
);
```

- 座標受信時: 前回位置とのマンハッタン距離を計算して `total_distance` に**加算**、最新座標で UPDATE（UPSERT）
- オーナー画面の総移動距離: 重い履歴集計クエリ → `total_distance` を読むだけ
- 初期化処理で、既存の `chair_locations` 履歴からこのテーブルを構築するSQLを流す
- 履歴 INSERT 自体が整合性チェックに不要なら削減も検討（マニュアル確認）

## ⑤ N+1 の解消

**`GET /api/app/nearby-chairs`**: 「全椅子 → 1台ずつ確認」の 2N+1 を、④の最新位置テーブル + 状態管理を使って1クエリ（または全部オンメモリ）にする。

- 公式解説の方針: ウィンドウ関数・CTE を使い「アクティブかつ空きの椅子と現在位置」を1クエリで取る
- さらに進めるなら `chairs` に `is_available` カラム（または Map）を追加して即時更新する。**位置は3秒までの遅延が許されるが、空き状態は即時反映が必要**という仕様の非対称性に注意（レスポンス丸ごとキャッシュはできない理由）

**`GET /api/owner/chairs`**: 「全椅子の距離を計算 → オーナーで絞る」を「**オーナーの椅子に絞ってから**必要な分だけ集計」に順序を入れ替える。④が入っていれば `total_distance` を読むだけになる。

## ⑥ 座標送信の非同期化

仕様の「**3秒までの反映遅延が許容**」を根拠に:

```typescript
app.post("/api/chair/coordinate", async (req, reply) => {
  locationBuffer.push({ chairId, lat, lon, at: new Date() });
  updateChairLatestOnMemory(chairId, lat, lon);  // メモリ上の最新値は即時更新
  return reply.send({ recorded_at: ... });        // DBを待たずに即応答
});

// 別途 setInterval で locationBuffer をバルクINSERT（2-5参照）
```

椅子は応答が返り次第次の移動をするので、**この応答が速いほど椅子が速く動き、実車距離が伸びてスコアが直接増える**。

## その他

- **決済APIの Idempotency-Key**: 決済リクエスト失敗時、`Idempotency-Key` ヘッダを付ければ安全にリトライできる仕様があった。クリティカルではないが把握しておくべき仕様
- **複数台構成**: 定石どおり DB 分離 + マッチング処理の別サーバー分離など（2-7）。参加記では「2台目を MySQL 専用、3台目に matcher 分離」といった構成例がある

## 実施順序の提案（練習で再現する場合）

1. 計測環境構築（alp, pt-query-digest）→ 初回ベンチ
2. ① インデックス一式（30分・効果絶大）
3. ⑤ nearby-chairs / owner系の N+1 解消
4. ④ 最新位置テーブル化 + 距離差分更新
5. ③ マッチング改善（全件・間隔短縮・最近傍・しきい値）
6. ⑥ 座標送信の非同期バルク化
7. ② 通知のキャッシュ化・retry_after_ms 調整
8. 複数台構成・MySQL設定・ログ無効化・再起動試験

## 参考リンク

- [ISUCON14 問題の解説と講評（公式）](https://isucon.net/archives/58869617.html)
- [isucon/isucon14 リポジトリ](https://github.com/isucon/isucon14)
- 参加記の例: [ISUCON14振り返り (Zenn)](https://zenn.dev/kanzen_rikai/articles/a099344df19751)、[ISUCON14に参加しました (note)](https://note.com/tagty/n/nfc6cc72c0e68)、[感想戦でRubyで6万点 (toshimaru/blog)](https://blog.toshimaru.net/isucon14/)

---

## 用語集

| 用語 | 意味 |
|------|------|
| 貪欲法（グリーディ法） | 各ステップで目先の最善（最も近い椅子）を選ぶアルゴリズム。最適解の保証はないが単純で速く、ISUCON14では十分だった |
| 最近傍 | 最も距離が近いもの。「最近傍の空き椅子を割り当てる」がマッチング改善の核 |
| しきい値（threshold） | 「これ以上遠い組はマッチさせない」の境界値。地域をまたぐ無駄なマッチを防ぐ |
| 2部グラフマッチング | 「ライド集合」と「椅子集合」の間の最適な組み合わせを求めるグラフアルゴリズム。貪欲法の上位互換だが実装コストが高い |
| SSE (Server-Sent Events) | サーバーからクライアントへ一方向にイベントを流し続けるHTTPの仕組み。ポーリング自体をなくせる |
| プッシュ型 / プル型 | サーバーから通知を送る（SSE）/ クライアントが取りに来る（ポーリング）方式の対比 |
| CTE (WITH句) | クエリ内で名前付きの一時結果を定義するSQL構文。複雑なクエリを段階的に書ける |
| ウィンドウ関数 | グループごとの順位付けなどを行うSQL機能。「各椅子の最新位置」の一括取得に使う（2-3参照） |
| UPSERT | 「なければINSERT、あればUPDATE」（2-4参照）。最新位置テーブルの更新に使う |
| 差分更新（累積距離） | 総移動距離を毎回計算せず、座標受信のたびに前回との距離を加算して持つこと |
| is_available | 椅子が空いているかを即時に引けるようにする状態カラム/フラグ。位置（3秒遅延OK）と違い即時反映が必要 |
| Idempotency-Key | 決済APIを安全にリトライするためのキー（2-5参照）。ISUCON14の決済APIにこの仕様があった |
| 感想戦 | 競技終了後も環境が残り、引き続き改善を試せる期間・モードのこと（将棋用語から） |

---

**自分用メモ →** [分からなかった単語を書き足す](03-improvements-terms.md)
**次に読む →** [3-4. 練習ガイド](04-practice-guide.md)
