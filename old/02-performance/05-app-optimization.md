# 2-5. アプリケーション最適化 — キャッシュと非同期化

> **このドキュメントで学ぶこと**
> - オンメモリキャッシュの設計と「整合性を壊さない」ための考え方
> - 重い処理の非同期化・後回し化
> - 外部API呼び出しの扱い方

DB を叩く回数を減らす・CPU を使う処理を減らすのがこの章のテーマです。

## オンメモリキャッシュ

### 基本形

Node.js ならプロセス内の `Map` が最速のキャッシュです。Redis 等を立てるより先にまず検討します。

```typescript
const chairCache = new Map<string, Chair>();

async function getChair(id: string): Promise<Chair | undefined> {
  const hit = chairCache.get(id);
  if (hit) return hit;
  const [[row]] = await pool.query<RowDataPacket[]>(
    "SELECT * FROM chairs WHERE id = ?", [id],
  );
  if (row) chairCache.set(id, row as Chair);
  return row as Chair | undefined;
}
```

### 整合性を壊さない3原則

キャッシュの怖さは「古い値を返してベンチの整合性チェックに落ちる」こと。次の3つを守ります。

1. **更新経路を全部洗い出す**: そのテーブルを UPDATE/INSERT/DELETE している箇所を grep し、全箇所でキャッシュも更新（または削除）する

   ```typescript
   await pool.query("UPDATE chairs SET is_active = ? WHERE id = ?", [active, id]);
   chairCache.delete(id);  // 次回読み込みでDBから取り直させる（削除の方が安全）
   ```

2. **POST /initialize でキャッシュを必ずクリア**: ベンチのたびに DB が初期化されるので、キャッシュに前回の残骸があると即死する

   ```typescript
   app.post("/api/initialize", async (req, reply) => {
     await execInitSql();
     chairCache.clear();
     ...
   });
   ```

3. **迷ったらキャッシュ対象を絞る**: 「更新されないデータ」（マスタ、作成後不変のレコード）から始める。更新が激しいデータのキャッシュは上級技

### 何をキャッシュするか（優先順）

| 対象 | 例 | 難易度 |
|------|-----|--------|
| 不変マスタデータ | 料金表、設定値 | 低。全部載せてよい |
| 作成後に変わらないレコード | ユーザー、チェア基本情報 | 低〜中 |
| 認証トークン → ユーザーの対応 | 毎リクエストの認証ミドルウェアのSELECTを消す。**効果大** | 中 |
| 更新のある集計値・最新値 | 最新ステータス | 中〜高。更新経路の管理が必要 |
| レスポンス丸ごと | 重い一覧APIのJSON | 高。無効化条件の設計が難しい |

**認証ミドルウェアのクエリは全エンドポイントで走る**ので、キャッシュ効果が全体に波及します。alp で気づきにくい定番ポイント。

## 非同期化・後回し処理

「レスポンスを返すのに必須でない処理」はレスポンス後に回せます。

```typescript
// ❌ Before: ログ的なINSERTを待ってからレスポンス
await pool.query("INSERT INTO access_logs ...", [...]);
return reply.send(result);

// ✅ After: 待たずに返す（エラーは握りつぶさずログには出す）
pool.query("INSERT INTO access_logs ...", [...]).catch((e) => console.error(e));
return reply.send(result);
```

さらに進めて**バッファリング + 定期flush**（2-4 のバルクINSERTと組み合わせ）:

```typescript
const buffer: LocationRow[] = [];

setInterval(async () => {
  if (buffer.length === 0) return;
  const rows = buffer.splice(0);  // 取り出して空にする
  try {
    await pool.query(
      "INSERT INTO chair_locations (chair_id, latitude, longitude) VALUES ?",
      [rows.map((r) => [r.chairId, r.lat, r.lon])],
    );
  } catch (e) {
    console.error(e);
  }
}, 100);  // flush間隔は仕様の許容遅延（マニュアル記載）内で設定

app.post("/api/chair/coordinate", async (req, reply) => {
  buffer.push(parse(req.body));
  return reply.send({ recorded_at: Date.now() });  // DBを待たず即応答
});
```

**必ずマニュアルを確認**: 「位置情報の反映は3秒以内なら遅延してよい」のような許容条件が書かれています（ISUCON14 が実際そうでした → 3-3）。許容がない処理を後回しにすると整合性エラーになります。

## 外部API呼び出し

決済APIなど外部サービスは自分では速くできません。打ち手は:

1. **呼び出し回数を減らす**: 結果が変わらない呼び出しはキャッシュ。リトライ時は `Idempotency-Key` 等の仕組みを使って安全に再送
2. **並列化**: 複数件の呼び出しは `Promise.all` で同時に
3. **タイムアウトとコネクション再利用**: Node.js の `fetch` (undici) はデフォルトで keep-alive が効くが、古い実装で毎回接続していないか確認

## CPU 処理の削減

1-3 で述べたとおり、Node.js は同期 CPU 処理が全リクエストを止めます。

- **パスワードハッシュ (bcrypt)**: ISUCON 頻出。`bcrypt.compareSync` が重い。ラウンド数は変えられない（既存ハッシュと互換が要る）が、「認証成功したトークン→ユーザー」をキャッシュして**比較の回数自体を減らす**
- **巨大 JSON の生成**: レスポンスに不要なフィールドを含めていないか。`SELECT *` をやめて必要な列だけ取るとシリアライズも軽くなる
- **正規表現・画像処理**: 結果をキャッシュ、または初期化時に前計算

---

## 用語集

| 用語 | 意味 |
|------|------|
| オンメモリキャッシュ | アプリのプロセス内メモリ（Mapなど）に結果を保存して使い回すこと。最速のキャッシュ |
| キャッシュヒット / ミス | キャッシュに目的の値があった / なかった。ミス時だけDBに取りに行く |
| 整合性 | キャッシュとDBの内容が食い違っていないこと。ISUCONでは食い違うとベンチのチェックで失格になり得る |
| キャッシュの無効化（invalidation） | 元データが更新されたときにキャッシュを消す/更新すること。「更新経路の全洗い出し」が鉄則 |
| 認証ミドルウェア | 全リクエストの最初に走る認証処理（Webフレームワークの機能としてのミドルウェア）。ここのクエリは全エンドポイントに効くのでキャッシュ効果大 |
| 非同期化 | 処理の完了を待たずにレスポンスを返し、処理は裏で行うこと |
| 後回し処理（バッファリング + 定期flush） | リクエスト時は配列に積むだけにして、別のタイマーでまとめてDBに書く方式 |
| setInterval | 一定間隔で関数を実行するJavaScriptのタイマー。定期flushの実装に使う |
| 許容遅延 | 「この情報の反映はX秒まで遅れてよい」という競技マニュアル上のルール。非同期化してよい根拠になる |
| Idempotency-Key | 同じリクエストを再送しても二重処理されないようにする識別キー（HTTPヘッダ）。安全なリトライに使う |
| 冪等（べきとう） | 同じ操作を何度実行しても結果が変わらない性質 |
| keep-alive | TCP接続を使い回す仕組み。毎回の接続確立コストを省く |
| undici | Node.js標準の`fetch`の内部実装であるHTTPクライアント。デフォルトでkeep-aliveが効く |
| bcrypt | パスワードを安全にハッシュ化するアルゴリズム。意図的に計算が重く作られており、ISUCONではCPUボトルネックの定番 |
| ハッシュ化 | 元に戻せない形にデータを変換すること。パスワード保存の基本 |
| シリアライズ | オブジェクトをJSON文字列などに変換すること。巨大なレスポンスでは無視できないCPUコスト |

---

**自分用メモ →** [分からなかった単語を書き足す](05-app-optimization-terms.md)
**次に読む →** [2-6. ミドルウェア (nginx)](06-middleware-nginx.md)
