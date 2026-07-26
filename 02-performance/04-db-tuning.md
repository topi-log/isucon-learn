# 2-4. DBチューニング — スキーマ改善とMySQL設定

> **このドキュメントで学ぶこと**
> - クエリ改善の次の一手: 書き込みのまとめ方（バルクINSERT）
> - 履歴テーブル・巨大カラムなど「遅いスキーマ」の改善パターン
> - MySQL 設定値のチューニング

インデックス（2-2）と N+1 解消（2-3）が終わってもまだ DB がボトルネックなら、この章の出番です。

## バルクINSERT — 書き込みをまとめる

1件ずつの INSERT はネットワーク往復・トランザクションのオーバーヘッドが大きい。まとめて入れると数十倍速くなります。

```typescript
// ❌ Before: ループでN回INSERT
for (const loc of locations) {
  await pool.query(
    "INSERT INTO chair_locations (chair_id, latitude, longitude) VALUES (?, ?, ?)",
    [loc.chairId, loc.lat, loc.lon],
  );
}

// ✅ After: 1回のバルクINSERT
await pool.query(
  "INSERT INTO chair_locations (chair_id, latitude, longitude) VALUES ?",
  [locations.map((l) => [l.chairId, l.lat, l.lon])],
);
```

さらに進んだ形として「**リクエスト時は配列に積むだけ → 定期的にまとめて flush**」する非同期バルク化があります（実装は 2-5、ISUCON14 での適用例は 3-3）。

## スキーマ改善パターン

### パターンA: 履歴テーブル → 最新値テーブル

「全履歴を INSERT で積み、読むときに `ORDER BY created_at DESC LIMIT 1`」という設計は ISUCON 頻出の罠です。

```sql
-- ❌ 読むたびに履歴から最新を探す
SELECT * FROM chair_locations WHERE chair_id = ? ORDER BY created_at DESC LIMIT 1;
```

改善: **最新値だけ持つテーブル（またはカラム）を追加**し、書き込み時に更新する。

```sql
CREATE TABLE chair_latest_locations (
  chair_id VARCHAR(26) PRIMARY KEY,
  latitude INT NOT NULL,
  longitude INT NOT NULL,
  total_distance INT NOT NULL DEFAULT 0,  -- 累積値もここで持つと集計クエリも消える
  updated_at DATETIME(6) NOT NULL
);

-- 書き込み時: UPSERT
INSERT INTO chair_latest_locations (chair_id, latitude, longitude, total_distance, updated_at)
VALUES (?, ?, ?, ?, NOW(6))
ON DUPLICATE KEY UPDATE
  total_distance = total_distance + VALUES(total_distance),
  latitude = VALUES(latitude), longitude = VALUES(longitude), updated_at = VALUES(updated_at);
```

「累積移動距離」のような**集計値も書き込み時に差分更新**しておくと、読み取り時の重い SUM クエリが消えます（読み取り時集計 → 書き込み時集計への転換）。

**注意**: 初期データにも履歴が入っているので、**初期化処理で最新値テーブルを履歴から構築するSQL**を仕込む必要があります。

### パターンB: 巨大カラムの追い出し

画像バイナリ（LONGBLOB）などが DB に入っていると、バッファプールを圧迫し、その行を触るすべてのクエリが遅くなります。

改善: 初期化時に画像をファイルとして書き出し、以後は **nginx が静的ファイルとして直接配信**（→ 2-6）。`SELECT *` が巨大カラムを巻き込んでいるだけなら、まず**必要な列だけ SELECT する**のも手軽で効きます。

### パターンC: 非正規化

JOIN が重い場合、参照頻度の高い値を持たせてしまう（例: `rides.latest_status` カラムを追加してステータステーブルへの JOIN を消す）。書き込み側の全経路で更新する必要があるので、整合性チェックに注意しながら。

## MySQL 設定チューニング

`/etc/mysql/mysql.conf.d/mysqld.cnf` の `[mysqld]` に追記して `sudo systemctl restart mysql`。

```ini
[mysqld]
# ① バッファプール: 最重要。DB専用サーバーならメモリの6〜7割
innodb_buffer_pool_size = 2G

# ② ログのディスク同期を緩める（クラッシュ時に最大1秒分失う代わりに書き込みが速い）
#    ISUCONでは定番。再起動試験ではデータは残るので通常問題ない
innodb_flush_log_at_trx_commit = 2
sync_binlog = 0

# ③ バイナリログを無効化（レプリケーション不要なら）
disable-log-bin

# ④ 接続数上限（アプリのプール合計より大きく）
max_connections = 1024
```

効果の目安: ① はデータがメモリに収まっていなかった場合に絶大。②③ は書き込み heavy な問題で効く。**設定変更もベンチ前後の計測で効果確認**を忘れずに。

現在値の確認:

```sql
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
-- バッファプールのヒット率的な状況
SHOW ENGINE INNODB STATUS\G
```

## DB 接続を unix socket にする

アプリと MySQL が同居している間は、TCP (127.0.0.1:3306) より unix socket の方が速い:

```typescript
const pool = mysql.createPool({
  socketPath: "/var/run/mysqld/mysqld.sock",
  ...
});
```

ただし DB を別サーバーに分離（2-7）したら TCP に戻すので、環境変数で切り替えられるようにしておくと楽です。

---

## 用語集

| 用語 | 意味 |
|------|------|
| バルクINSERT | 複数行を1回のINSERT文でまとめて挿入すること。往復コストが激減する |
| flush（フラッシュ） | 溜めておいたデータをまとめて書き出すこと。「配列に積んで定期的にflush」が非同期バルク化の型 |
| UPSERT | 「なければINSERT、あればUPDATE」を1文で行う操作。MySQLでは`INSERT ... ON DUPLICATE KEY UPDATE` |
| 履歴テーブル | 変化のたびにINSERTで積み上げるテーブル。「最新値の取得」「集計」が重くなりがち |
| 最新値テーブル | 履歴とは別に「現在の値」だけを1行で持つテーブル。読み取りが激軽になる |
| 差分更新 | 集計値（累積距離など）を読むときに計算するのではなく、書き込みのたびに加算して持っておくこと |
| 非正規化 | 本来別テーブルにある値を、参照高速化のために重複して持たせること。更新時に全箇所を揃える責任が生じる |
| 正規化 | データの重複をなくすテーブル設計の原則。非正規化はこれを意図的に崩すこと |
| innodb_buffer_pool_size | バッファプール（1-4参照）のサイズ設定。DB専用サーバーならメモリの6〜7割が目安 |
| innodb_flush_log_at_trx_commit | コミットごとにログをディスク同期するかの設定。`2`にすると耐障害性と引き換えに書き込みが速くなる |
| sync_binlog / バイナリログ | 更新記録ログ（レプリケーションや復旧に使う）とその同期設定。ISUCONでは無効化が定番 |
| レプリケーション | DBの内容を別サーバーに複製し続ける仕組み。ISUCONでは通常使わない |
| unix domain socket | 同一マシン内のプロセス間通信の仕組み。TCP（127.0.0.1経由）よりオーバーヘッドが小さい |
| `SELECT *` | 全列を取得する書き方。巨大な列を巻き込むと遅いので、必要な列だけ指定するのが改善パターン |

---

**自分用メモ →** [分からなかった単語を書き足す](04-db-tuning-terms.md)
**次に読む →** [2-5. アプリケーション最適化](05-app-optimization.md)
