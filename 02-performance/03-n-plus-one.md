# 2-3. N+1問題 — 見つけ方と3つの解消パターン

> **このドキュメントで学ぶこと**
> - N+1 クエリとは何か、なぜ致命的か
> - コードと計測結果からの見つけ方
> - 3つの解消パターン（JOIN / IN句一括取得 / 事前Map化）の TypeScript 実装

## N+1 とは

「一覧を1クエリで取り（**1**）、各要素の関連データをループで1件ずつ取る（**N**）」パターン。

```typescript
// ❌ Before: ライド一覧 + 各ライドのユーザー情報
const [rides] = await pool.query<RowDataPacket[]>(
  "SELECT * FROM rides ORDER BY created_at DESC LIMIT 100",
);
for (const ride of rides) {
  const [[user]] = await pool.query<RowDataPacket[]>(
    "SELECT * FROM users WHERE id = ?", [ride.user_id],
  );  // ← 100回実行される
  ride.user = user;
}
```

1クエリが 1ms でも、101 回のクエリ = 101ms + 往復のオーバーヘッド。一覧系エンドポイントはアクセスも多いので、**N+1 はスコアを最も削る要因の1つ**です。しかもネストしていることがある（ライド→ユーザー→ユーザーのクーポン、で N×M+N+1）。

## 見つけ方

1. **pt-query-digest で「Calls が異常に多い」クエリ**（2-1）。`WHERE id = ?` 形の単純クエリが数万回呼ばれていたらほぼ N+1
2. **コードの見た目**: `for` / `map` の中に `await pool.query` があったら疑う
3. alp で遅い一覧系エンドポイントのハンドラを読む

## 解消パターン1: JOIN で1クエリにする

```typescript
// ✅ After: JOIN
const [rows] = await pool.query<RowDataPacket[]>(
  `SELECT r.*, u.name AS user_name, u.token AS user_token
   FROM rides r
   JOIN users u ON u.id = r.user_id
   ORDER BY r.created_at DESC LIMIT 100`,
);
```

- 最もシンプル。1対1・多対1 の関連に向く
- 1対多を JOIN すると行が増殖してアプリ側の整形が面倒になる → パターン2へ

## 解消パターン2: IN 句で一括取得して Map で結合

ISUCON で最も汎用性が高いパターン。**「まとめて取って、アプリでくっつける」**。

```typescript
// ✅ After: IN句 + Map
const [rides] = await pool.query<RowDataPacket[]>(
  "SELECT * FROM rides ORDER BY created_at DESC LIMIT 100",
);

const userIds = [...new Set(rides.map((r) => r.user_id))]; // 重複除去
const [users] = userIds.length
  ? await pool.query<RowDataPacket[]>(
      "SELECT * FROM users WHERE id IN (?)", [userIds],
    )
  : [[]];

const userById = new Map(users.map((u) => [u.id, u]));
for (const ride of rides) {
  ride.user = userById.get(ride.user_id);
}
```

ポイント:

- クエリ数は **N+1 → 2** になる
- `IN (?)` に mysql2 は配列をそのまま展開してくれる
- **空配列ガード必須**: `IN ()` は SQL エラーになる
- 結合は必ず `Map` で。`users.find(...)` をループ内でやると O(N×M) でアプリ側が遅くなる

### 「各グループの最新1件」パターン

N+1 でよくある難物が「各ライドの**最新の**ステータスを取る」形:

```sql
-- ループ内でこれを N 回やっている
SELECT * FROM ride_statuses WHERE ride_id = ? ORDER BY created_at DESC LIMIT 1;
```

一括化はウィンドウ関数（MySQL 8.0+）で:

```sql
SELECT * FROM (
  SELECT rs.*,
         ROW_NUMBER() OVER (PARTITION BY ride_id ORDER BY created_at DESC) AS rn
  FROM ride_statuses rs
  WHERE ride_id IN (?)
) t WHERE rn = 1;
```

さらに根本的には「最新値を別テーブル/別カラムに持つ」設計変更が効く（→ 2-4。ISUCON14 の chair_locations がまさにこれ）。

## 解消パターン3: 事前に全件読んでオンメモリ化

マスタデータ（更新されない・小さいテーブル）なら、**起動時 or 初期化時に全部メモリに載せてしまう**のが最速です。クエリ数は 0 になります。

```typescript
// アプリ起動時に1度だけ
const userById = new Map<string, User>();

async function loadMasters() {
  const [users] = await pool.query<RowDataPacket[]>("SELECT * FROM users");
  userById.clear();
  for (const u of users) userById.set(u.id, u as User);
}

// POST /initialize（ベンチ開始時の初期化）でも必ず再ロードする
app.post("/api/initialize", async (req, reply) => {
  await execInitSql();
  await loadMasters();  // ← 忘れると古いデータで整合性エラー
  ...
});
```

注意点:

- **更新のあるデータをキャッシュするなら、更新経路すべてでキャッシュも更新する**必要がある（詳細は 2-5）
- 複数プロセス・複数サーバー構成にすると各プロセスが別々のメモリを持つ点に注意（→ 2-7）

## どのパターンを選ぶか

| 状況 | 推奨 |
|------|------|
| 多対1 の関連を数個くっつけるだけ | パターン1 (JOIN) |
| 関連が多い・1対多・ネストした N+1 | パターン2 (IN + Map) |
| 更新されない/ほぼ更新されないマスタ | パターン3 (オンメモリ) |
| 「各グループの最新1件」 | ウィンドウ関数 or 最新値テーブル (2-4) |

迷ったらパターン2。安全（整合性を壊しにくい）で適用範囲が広いです。

---

## 用語集

| 用語 | 意味 |
|------|------|
| N+1問題 | 一覧を1クエリで取り（1）、各要素の関連データをループで1件ずつ取る（N回）パターン。クエリ回数が爆発する |
| JOIN | 複数テーブルを結合して1クエリで取得するSQL構文 |
| IN句 | `WHERE id IN (1, 2, 3)`のように複数の値のどれかに一致する行をまとめて取る構文。N+1解消の主役 |
| Map | JavaScriptのキー値辞書。キーによる検索がO(1)（データ量によらず一定時間）でできる |
| Set | JavaScriptの重複なし集合。`[...new Set(array)]`で重複除去に使う |
| O(1) / O(N×M) | 計算量の記法。処理時間がデータ量に対してどう増えるかを表す。ループ内で`find`するとO(N×M)になり遅い |
| 多対1 / 1対多 | テーブル間の関連の形。「ライド→ユーザー」は多対1、「ライド→ステータス履歴」は1対多 |
| ウィンドウ関数 | 行のグループごとに順位付けなどができるSQL機能（MySQL 8.0以降）。「各グループの最新1件」の一括取得に使う |
| ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...) | ウィンドウ関数の代表。PARTITION BYのグループごとにORDER BY順で連番を振る。`rn = 1`で各グループの先頭が取れる |
| サブクエリ | SQLの中に入れ子で書くSELECT。ウィンドウ関数の結果を絞り込むときなどに使う |
| マスタデータ | ほとんど更新されない基準データ（料金表・設定値など）。丸ごとメモリに載せやすい |
| オンメモリ化 | データをアプリのメモリ上（変数）に持ち、DBに問い合わせずに済ませること |
| 空配列ガード | `IN (?)`に空配列を渡すとSQLエラーになるため、事前に空チェックすること |

---

**自分用メモ →** [分からなかった単語を書き足す](03-n-plus-one-terms.md)
**次に読む →** [2-4. DBチューニング](04-db-tuning.md)
