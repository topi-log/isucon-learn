# 1-3. Node.jsバックエンドの基礎

> **このドキュメントで学ぶこと**
> - Node.js ランタイムの特性（イベントループ）と ISUCON での意味
> - ISUCON 参考実装（Express / Fastify + mysql2）の典型的な構造と読み方
> - Node.js 実装で気をつけるべき性能ポイント

## Node.js の実行モデル: イベントループ

Node.js は **シングルスレッド + ノンブロッキング I/O** で動きます。

```
        ┌──────────── イベントループ（1本のスレッド）────────────┐
        │                                                    │
リクエストA → ハンドラ実行 → SQL発行(待つ間、手放す) ─┐            │
リクエストB → ハンドラ実行 → SQL発行(待つ間、手放す) ─┤ DB応答待ち  │
リクエストC → ハンドラ実行 ...                      │            │
        │     ← A の DB 応答が返ってきたら続きを実行 ┘            │
        └────────────────────────────────────────────────────┘
```

これが意味すること:

1. **I/O 待ち（DBクエリ・HTTP呼び出し）は並行に大量に捌ける**
   - `await pool.query(...)` で待っている間、スレッドは他のリクエストを処理できる
2. **CPU を使う同期処理はすべてを止める**
   - 重いループ、巨大 JSON の `JSON.stringify`、同期的なパスワードハッシュ計算（`bcrypt` の sync 版など）が走っている間、**全リクエストが止まる**
   - ISUCON で「CPU 重い処理」を見つけたら、結果のキャッシュ（→ 2-5）が特に効く
3. **1プロセスでは CPU 1コアしか使えない**
   - サーバーが 2 コアあっても素の Node.js は 1 コアしか使わない
   - `cluster` モジュールや systemd で複数プロセスを立ててコアを使い切るのが定石（→ 2-7）

## ISUCON 参考実装の典型構造

Node.js 参考実装は近年 **Fastify または Express + mysql2** の構成が多いです。典型的なファイル構造:

```
webapp/nodejs/
├── src/
│   ├── main.ts          # エントリポイント: サーバー起動、DB接続、ルーティング登録
│   ├── app_handlers.ts  # ユーザー向けAPIのハンドラ群
│   ├── owner_handlers.ts # 管理者向けAPIのハンドラ群
│   └── ...
├── package.json
└── tsconfig.json
```

ハンドラの典型形（ISUCON14 Node.js 実装風）:

```typescript
import mysql from "mysql2/promise";

// コネクションプール（アプリ起動時に1度だけ作る）
const pool = mysql.createPool({
  host: process.env.DB_HOST ?? "127.0.0.1",
  user: "isucon",
  password: "isucon",
  database: "isuride",
  connectionLimit: 10,
});

app.get("/api/app/rides/:id", async (req, reply) => {
  const [[ride]] = await pool.query<RowDataPacket[]>(
    "SELECT * FROM rides WHERE id = ?",
    [req.params.id],
  );
  if (!ride) {
    return reply.status(404).send({ message: "ride not found" });
  }
  return reply.send(ride);
});
```

### 競技開始後、最初にコードを読むときのチェックポイント

1. **ルーティング一覧を眺める** — どんなエンドポイントがあるか全体像を掴む（main.ts のルート登録部分）
2. **`for` ループの中に `await pool.query` がないか** — これが N+1 の典型的な見た目（→ 2-3）
3. **トランザクションの範囲** — `beginTransaction` 〜 `commit` の間が長いとロック待ちが起きる
4. **外部への `fetch`** — 決済API等の外部呼び出しは自分では速くできない。呼び出し回数を減らす方向で考える

## Node.js 実装での性能上の注意点

### コネクションプールのサイズ

```typescript
const pool = mysql.createPool({ connectionLimit: 10, ... });
```

`connectionLimit`（デフォルト10）が小さすぎるとクエリが**プールの空き待ち**で詰まります。逆に大きすぎると MySQL 側が苦しむ。まずは 10〜50 程度で、MySQL 側の `max_connections`（→ 2-4）とセットで調整します。

### `await` の直列と並列

依存関係のない複数クエリを直列に `await` するのは無駄:

```typescript
// 遅い: 合計 = クエリ1 + クエリ2 の時間
const [users] = await pool.query("SELECT ...");
const [chairs] = await pool.query("SELECT ...");

// 速い: 合計 = 遅い方の時間だけ
const [[users], [chairs]] = await Promise.all([
  pool.query("SELECT ..."),
  pool.query("SELECT ..."),
]);
```

ただし**根本対策は「そもそもクエリ数を減らす」**（JOIN や一括取得、→ 2-3）。`Promise.all` は最後の仕上げです。

### 開発モードで動かさない

`ts-node` や `nodemon`、`NODE_ENV=development` のまま動いていないか確認。ビルドして `node dist/main.js` を systemd から起動するのが基本です。参考実装は最初から本番相当になっていることが多いですが、自分でいじった後に戻し忘れがち。

### ログ出力もコスト

毎リクエスト `console.log` する処理は、量が多いと無視できない負荷になります。計測が終わったら消す・無効化する（競技終盤の定石）。

## ISUCON 用に覚えておく mysql2 のイディオム

```typescript
import type { RowDataPacket, ResultSetHeader } from "mysql2/promise";

// SELECT: 結果は [rows, fields] のタプル
const [rows] = await pool.query<RowDataPacket[]>(
  "SELECT * FROM users WHERE id IN (?)", [ids],  // IN句は配列をそのまま渡せる
);

// 1件だけ欲しいとき（分割代入で先頭を取る）
const [[user]] = await pool.query<RowDataPacket[]>(
  "SELECT * FROM users WHERE id = ?", [id],
);

// INSERT: 挿入IDは ResultSetHeader から
const [result] = await pool.query<ResultSetHeader>(
  "INSERT INTO rides (user_id) VALUES (?)", [userId],
);
console.log(result.insertId);

// バルクINSERT: VALUES ? に二次元配列
await pool.query(
  "INSERT INTO locations (chair_id, lat, lon) VALUES ?",
  [rows.map((r) => [r.chairId, r.lat, r.lon])],
);

// トランザクション（コネクションを専有する必要がある）
const conn = await pool.getConnection();
try {
  await conn.beginTransaction();
  await conn.query("UPDATE ...");
  await conn.query("INSERT ...");
  await conn.commit();
} catch (e) {
  await conn.rollback();
  throw e;
} finally {
  conn.release();  // 忘れるとプールが枯渇する！
}
```

---

## 用語集

| 用語 | 意味 |
|------|------|
| イベントループ | Node.jsの実行モデル。1本のスレッドが「待ち時間のある処理は手放し、完了したら続きを実行」を繰り返す仕組み |
| シングルスレッド | 同時に1つの処理しかCPUで実行しないこと。Node.jsのJavaScript実行部分はこれ |
| ノンブロッキングI/O | 入出力（DB・ネットワーク等）の完了を待たずに他の処理を進められる方式 |
| I/O | Input/Output。DBアクセス、ファイル読み書き、ネットワーク通信など「CPU以外とのやりとり」の総称 |
| async / await | 非同期処理を同期処理のような見た目で書けるJavaScript構文。`await`は「完了を待つが、その間スレッドは他の仕事ができる」 |
| Promise | 非同期処理の「将来の結果」を表すオブジェクト。`Promise.all`で複数を並列に待てる |
| Fastify / Express | Node.jsの代表的なWebフレームワーク。ルーティングとハンドラ登録の仕組みを提供する |
| mysql2 | Node.jsで最もよく使われるMySQLクライアントライブラリ。`mysql2/promise`でasync/await対応 |
| コネクションプール | DB接続をあらかじめ複数本作って使い回す仕組み。接続の作り直しコストを避ける |
| connectionLimit | mysql2のプールが保持する最大接続数。小さすぎると空き待ちで詰まる |
| トランザクション | 複数のDB操作を「全部成功か全部取り消しか」にまとめる仕組み。詳細は 1-4 |
| RowDataPacket / ResultSetHeader | mysql2の型。SELECT結果の行 / INSERT等の実行結果（insertIdなど）を表す |
| プレースホルダ（`?`） | SQL中の値の差し込み位置。`query("... WHERE id = ?", [id])` の形。SQLインジェクション対策にもなる |
| cluster | Node.jsで複数プロセスを立ててCPUコアを使い切るための標準モジュール |
| ts-node / nodemon | TypeScriptを直接実行する/変更を検知して再起動する開発用ツール。本番では使わない（遅い） |

---

**自分用メモ →** [分からなかった単語を書き足す](03-nodejs-basics-terms.md)
**次に読む →** [1-4. MySQLの基礎](04-mysql-basics.md)
