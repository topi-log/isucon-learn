# phase4: アクセスログで計測する

このフェーズでわかるようになること

- access.log を集計しやすい形式（JSON）に変える設定
- alp というツールで「どのエンドポイントが遅いか」を出す方法
- 計測 → 改善 → 再計測のループの回し方

## なぜ nginx のログで計測するのか

ISUCON の鉄則は「推測するな、計測せよ」です。
そして nginx は全リクエストが通る受付なので、ここのログを集計すれば
「どの URL に何回アクセスがあり、それぞれ何秒かかったか」が全部わかります。

つまり access.log は、最初に見るべき計測データです。
「たぶんこの機能が遅い」ではなく「合計時間ワースト 1 位はこの URL」から着手できます。

## ログを JSON 形式にする

デフォルトの access.log は人間向けの 1 行テキストで、応答時間も入っていません。
集計ツールが読みやすいよう、JSON 形式に変えます。

nginx.conf の http ブロック（server の外側）に log_format を定義し、
access_log でそれを使うよう指定します。

```nginx
http {
    log_format json escape=json '{'
        '"time":"$time_iso8601",'
        '"remote_addr":"$remote_addr",'
        '"method":"$request_method",'
        '"uri":"$request_uri",'
        '"status":"$status",'
        '"body_bytes":"$body_bytes_sent",'
        '"request_time":"$request_time",'
        '"upstream_time":"$upstream_response_time"'
    '}';

    access_log /var/log/nginx/access.log json;
}
```

大事な変数は 2 つです。

- request_time: nginx がリクエストを受けてから返し終わるまでの秒数
- upstream_response_time: そのうちアプリの処理にかかった秒数

この 2 つがほぼ同じなら遅いのはアプリ（か DB）、大きく差があるなら
nginx とクライアントの間（転送量が大きすぎるなど）に原因がある、と切り分けられます。

書いたら phase1 の型どおり反映します。

```bash
sudo nginx -t && sudo systemctl reload nginx
```

## alp で集計する

alp は nginx のアクセスログをエンドポイント別に集計してくれるツールです。
（インストールは GitHub の tkuchiki/alp からバイナリを取得して置くだけです。
競技用サーバーには事前にセットアップ手順をチームで用意しておきましょう。）

```bash
sudo cat /var/log/nginx/access.log | alp json --sort sum -r
```

- `json`: 上で設定した JSON 形式のログを読む
- `--sort sum -r`: 合計時間（SUM）の大きい順に並べる

出力はこんな表になります。

```
+-------+--------+--------------+-------+-------+-------+--------+
| COUNT | METHOD | URI          |  MIN  |  AVG  |  MAX  |  SUM   |
+-------+--------+--------------+-------+-------+-------+--------+
|  1276 | GET    | /api/posts   | 0.004 | 0.083 | 0.522 | 105.9  |
|   312 | POST   | /api/login   | 0.010 | 0.150 | 0.480 |  46.8  |
+-------+--------+--------------+-------+-------+-------+--------+
```

見るべきは SUM（合計時間）です。
1 回あたりは速くても回数が多い URL は、合計では最大のボトルネックになりえます。
平均（AVG）だけ見ていると見落とすので、まず SUM 順で見る癖をつけてください。

### パスのグルーピング

`/posts/123` `/posts/456` のように ID がパスに入る URL は、そのままだと
全部バラバラの行として集計されてしまいます。`-m` で正規表現をまとめられます。

```bash
sudo cat /var/log/nginx/access.log | \
  alp json --sort sum -r -m '/posts/[0-9]+,/users/[0-9]+'
```

これで `/posts/123` も `/posts/456` も 1 行にまとまり、正しく比較できます。

## 計測 → 改善 → 再計測のループ

ISUCON 中の基本サイクルはこれです。

1. ログを空にする（前回の結果が混ざらないように）
2. ベンチマークを実行する
3. alp で集計し、SUM ワーストの URL を特定する
4. その URL の処理を改善する（インデックス、N+1 解消、nginx で配信、など）
5. 1 に戻って効果を確認する

ログを空にするのは次のコマンドが手軽です。

```bash
sudo truncate -s 0 /var/log/nginx/access.log
```

truncate はファイルを消さずに中身だけ空にします。
rm で消してしまうと nginx がログを書けなくなる（正確には、開いたままの古いファイルに
書き続けて新しいファイルが作られない）ので、truncate を使ってください。

このループを何周も回すのが ISUCON です。1 周を速く回せるように、
ベンチ実行から alp 表示までをシェルスクリプトにしておくチームが多いです。

```bash
#!/bin/bash
# bench 前に実行する例
sudo truncate -s 0 /var/log/nginx/access.log
echo "ログを空にしました。ベンチを実行してください"
```

## このフェーズのまとめ

- nginx は全リクエストの通り道なので、access.log が最初の計測データになる
- log_format で JSON 化し、request_time と upstream_response_time を記録する
- alp で SUM 順に集計し、合計時間ワーストから手を付ける
- ベンチ前に truncate でログを空にする。計測 → 改善 → 再計測を高速に回す

より詳しい計測手法（pt-query-digest や top との組み合わせ）は
[02-performance/01-measurement.md](../old/02-performance/01-measurement.md) にあります。

次の phase5 では、計測で見つけた問題に対する nginx 側の定番チューニングをまとめます。
