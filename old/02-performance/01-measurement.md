# 2-1. 計測ファースト — 推測するな、計測せよ

> **このドキュメントで学ぶこと**
> - 計測 → 改善 → 再計測のループの回し方
> - アクセスログ解析ツール **alp** の使い方（どのエンドポイントが遅いか）
> - スロークエリ解析ツール **pt-query-digest** の使い方（どのSQLが遅いか）

## なぜ計測が最初なのか

1-2 で見たとおり、**ボトルネック以外を改善してもスコアは上がりません**。そして「どこがボトルネックか」は人間の直感とよくズレます。だから ISUCON の基本ループはこれです:

```
┌────────────────────────────────────────────┐
│ ① ベンチマークを実行                          │
│ ② 計測結果を見る（alp / pt-query-digest / top）│
│ ③ 一番効きそうな改善を1つ実施                  │
│ ④ ベンチを再実行してスコアと計測結果を比較        │──┐
└────────────────────────────────────────────┘  │
        ▲___________________________________________│
```

ポイント: **改善は1つずつ**。まとめて変えるとどれが効いたか（あるいはどれが壊したか）わからなくなります。

## 計測の3点セット

| ツール | 何がわかるか | レイヤー |
|--------|------------|---------|
| **alp** | どの**エンドポイント**が合計で遅いか | nginx アクセスログ |
| **pt-query-digest** | どの**SQL**が合計で遅いか | MySQL スロークエリログ |
| **top / dstat** | どの**プロセス/リソース**が限界か | OS |

## alp — アクセスログ解析

### セットアップ

nginx のログを alp が読める JSON 形式にします。`/etc/nginx/nginx.conf` の `http` ブロックに:

```nginx
log_format json escape=json '{"time":"$time_iso8601",'
  '"host":"$remote_addr",'
  '"method":"$request_method",'
  '"uri":"$request_uri",'
  '"status":"$status",'
  '"body_bytes":"$body_bytes_sent",'
  '"request_time":"$request_time"}';

access_log /var/log/nginx/access.log json;
```

```bash
sudo nginx -t && sudo systemctl reload nginx

# alp 本体のインストール（GitHub Releases からバイナリを取得）
wget https://github.com/tkuchiki/alp/releases/latest/download/alp_linux_amd64.tar.gz
tar xzf alp_linux_amd64.tar.gz && sudo mv alp /usr/local/bin/
```

### 使い方

```bash
# ベンチ実行後に集計（sum 降順 = 合計時間を食っているエンドポイント順）
sudo alp json --file /var/log/nginx/access.log \
  --sort sum --reverse \
  -m '/api/users/[0-9]+,/api/rides/[a-zA-Z0-9-]+'
```

- `-m` は **URI のグルーピング**。`/api/users/1`, `/api/users/2`... を1行にまとめるための正規表現。**これをやらないと集計が意味をなさない**ので、マニュアルとルーティング定義を見て序盤に書く
- ベンチのたびにログをリセットする: `sudo truncate -s 0 /var/log/nginx/access.log`

### 結果の読み方

```
+-------+--------+---------------------+-------+--------+------+------+
| COUNT | METHOD | URI                 | MIN   | AVG    | MAX  | SUM  |
+-------+--------+---------------------+-------+--------+------+------+
| 12000 | GET    | /api/notification   | 0.004 | 0.041  | 0.9  | 492  | ← 塵積型
|   300 | POST   | /api/estimate       | 0.900 | 1.400  | 3.2  | 420  | ← 単発重量型
```

**見るのは SUM（合計時間）**。改善候補は2タイプあります:

- **塵積型**: 1回は速いが回数が膨大。キャッシュや呼び出し削減が効く
- **単発重量型**: 1回が重い。SQL改善・N+1解消が効く

## pt-query-digest — スロークエリ解析

### セットアップ

MySQL で全クエリをログに出します（`/etc/mysql/mysql.conf.d/mysqld.cnf` の `[mysqld]`）:

```ini
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 0   # 0 = 全クエリを記録（競技中の計測用）
```

```bash
sudo systemctl restart mysql

# percona-toolkit のインストール
sudo apt install -y percona-toolkit
```

### 使い方と読み方

```bash
# ベンチ前にログをリセット
sudo truncate -s 0 /var/log/mysql/slow.log

# ベンチ後に集計
sudo pt-query-digest /var/log/mysql/slow.log | less
```

出力の先頭に「合計時間を食っているクエリランキング」が出ます:

```
# Profile
# Rank Query ID           Response time  Calls  R/Call  Item
# ==== ================== ============== ====== ======= ===============
#    1 0xABCD...          311.2s  45.1%  90000  0.0035  SELECT chair_locations
#    2 0x1234...          201.8s  29.3%    450  0.4485  SELECT rides statuses
```

- **Rank 1 から順に潰す**。`R/Call`（1回あたり時間）× `Calls`（回数）の内訳を見て、
  - 回数が異常に多い → **N+1 の疑い**（→ 2-3）
  - 1回が遅い → **インデックス不足の疑い**（→ 2-2）。下に出る実クエリ例を `EXPLAIN` にかける

### 注意

- スロークエリログ自体が負荷になる。**最終ベンチ前には `slow_query_log = 0` に戻す**（alp 用の nginx ログも同様に検討）

## 計測ループを速くする仕込み

計測→改善のサイクルタイムがスコアに直結します。序盤に作っておくと良いもの:

```bash
# ~/bin/bench-prepare.sh — ベンチ前に毎回叩くスクリプト
#!/bin/bash
sudo truncate -s 0 /var/log/nginx/access.log
sudo truncate -s 0 /var/log/mysql/slow.log
sudo systemctl restart isuride-node.service

# ~/bin/analyze.sh — ベンチ後に毎回叩くスクリプト
#!/bin/bash
sudo alp json --file /var/log/nginx/access.log --sort sum --reverse -m '<パターン>' | head -30
sudo pt-query-digest /var/log/mysql/slow.log | head -60
```

---

## 用語集

| 用語 | 意味 |
|------|------|
| alp | nginxのアクセスログを集計し、エンドポイントごとの回数・合計時間を出すツール。「どのAPIが遅いか」を特定する |
| アクセスログ | nginxが記録する「いつ・どのURLに・何秒で応答したか」のログ |
| log_format | nginxのログ出力形式の定義。alpに読ませるためJSON形式に変える |
| スロークエリログ | MySQLが記録する「実行に時間がかかったクエリ」のログ。`long_query_time = 0`にすると全クエリを記録できる |
| pt-query-digest | スロークエリログを集計し、合計時間を食っているSQLのランキングを出すツール（percona-toolkitに含まれる） |
| percona-toolkit | MySQL運用ツール集。ISUCONではpt-query-digest目当てでインストールする |
| SUM / AVG / MAX | 集計結果の合計 / 平均 / 最大。ボトルネック探しでは「合計時間（SUM）」を見るのが鉄則 |
| R/Call | pt-query-digestの「1回あたりの応答時間」。Calls（回数）との掛け算で合計時間が決まる |
| グルーピング（alpの`-m`） | `/api/users/1`と`/api/users/2`を同じエンドポイントとしてまとめる設定。正規表現で指定する |
| 正規表現 | 文字列のパターンを表す記法。`[0-9]+`は「1文字以上の数字」 |
| truncate | ファイルの中身を空にするコマンド。ベンチ前のログリセットに使う（`truncate -s 0 file`） |
| 塵積型 / 単発重量型 | 本ドキュメント内での分類。1回は速いが回数が膨大なもの / 1回が重いもの。改善アプローチが異なる |
| dstat | CPU・ディスク・ネットワークの使用状況を時系列で表示する監視ツール |
| サイクルタイム | 「計測→改善→再計測」の1周にかかる時間。短いほど試行回数を稼げる |

---

**自分用メモ →** [分からなかった単語を書き足す](01-measurement-terms.md)
**次に読む →** [2-2. EXPLAINとインデックス設計](02-sql-explain-index.md)
