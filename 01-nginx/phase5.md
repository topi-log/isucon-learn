# phase5: ISUCON 実戦チューニング

このフェーズでわかるようになること

- nginx でやる定番チューニングの中身と優先順位
- 複数台構成での振り分け方
- 競技当日、nginx まわりで最初にやることのチェックリスト

ここまでの知識（location、proxy_pass、keepalive、alp）を全部使います。

## チューニングの優先順位

先に結論です。効果と手間のバランスで、この順番で入れるのがおすすめです。

1. アクセスログの JSON 化（phase4。改善ではなく計測の準備）
2. 静的ファイルの nginx 直接配信 + expires
3. upstream keepalive（phase3 の 3 点セット）
4. gzip などその他の設定
5. 複数台構成での振り分け（サーバーが複数あるなら）

nginx のチューニングは「アプリに仕事をさせない」ことが目的です。
ただし、最大のボトルネックはたいてい DB やアプリ側にあります。
nginx の定番設定をひととおり入れたら、深追いせず alp の結果に従って
アプリ・DB の改善に移るのが得点効率の良い動き方です。

## 改善1: 静的ファイルは nginx が直接返す

phase2 で見たとおり、初期状態は静的ファイルまでアプリが返しています。
これを nginx に肩代わりさせます。定番かつ効果の出やすい改善です。

```nginx
server {
    listen 80;
    root /home/isucon/webapp/public;    # 静的ファイルの実体の場所（要確認）

    location /assets/ {
        expires 24h;
        add_header Cache-Control public;
    }

    location = /favicon.ico {
        expires 24h;
    }

    location / {
        proxy_set_header Host $host;
        proxy_pass http://app;
    }
}
```

- location /assets/ には proxy_pass がないので、nginx は root からファイルを探して
  直接返します。アプリには一切リクエストが行きません
- expires はレスポンスに「24 時間キャッシュしてよい」というヘッダーを付けます。
  ベンチマーカーがこれを見てキャッシュしてくれる問題では、2 回目以降の
  リクエスト自体がなくなります

注意点が 2 つあります。

- 静的ファイルの実体がどこにあるかは問題ごとに違います。必ず
  `ls /home/isucon/webapp/` あたりで確認してから root を書いてください
- キャッシュしてよいかは競技マニュアルに書いてあることがあります。
  当日は必ずマニュアルを確認してください

反映後、alp で /assets/ の SUM が消えていれば成功です。

## 改善2: gzip 圧縮

レスポンスを圧縮して転送量を減らします。http ブロックに書きます。

```nginx
http {
    gzip on;
    gzip_types text/css application/javascript application/json;
    gzip_min_length 1k;
}
```

- gzip_types: 圧縮する種類。HTML はデフォルトで対象なので、CSS・JS・JSON を足します
- gzip_min_length: 小さすぎるレスポンスは圧縮しない（圧縮のほうが高くつくため）

さらに、静的ファイルについては事前に圧縮済みファイルを置いておく
gzip_static という仕組みもあります（`app.js.gz` を置いておくと、
リクエストのたびに圧縮する代わりにそれをそのまま返す）。
余裕があれば、で構いません。

## 改善3: nginx 本体の設定

nginx.conf の上のほう（グローバルと events）にある設定です。
初期状態で問題ないことも多いですが、一応確認しておきます。

```nginx
worker_processes auto;      # CPU コア数に合わせてワーカーを起動する

events {
    worker_connections 4096;   # ワーカー1つが同時に扱える接続数
}

http {
    sendfile on;               # ファイル送信を効率化する
    tcp_nopush on;             # パケットをまとめて送る
    keepalive_requests 10000;  # クライアントとの接続1本で受けるリクエスト数の上限
}
```

worker_connections が初期値の 1024 のままだと、ベンチの同時接続をさばききれず
接続エラーが出ることがあります。大きめにしておくのが無難です。

## 改善4: 複数台構成での振り分け

ISUCON ではサーバーが 3 台渡されることが多いです。
たとえば「1 台目 = nginx + DB、2〜3 台目 = アプリ」のような分担にするとき、
振り分けを担当するのが nginx の upstream です。

```nginx
upstream app {
    server 192.168.0.12:8080;    # 2台目のアプリ
    server 192.168.0.13:8080;    # 3台目のアプリ
    keepalive 128;
}

server {
    location / {
        proxy_pass http://app;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
    }
}
```

server を複数並べるだけで、nginx が交互にリクエストを振り分けてくれます
（ラウンドロビンと呼びます）。台数に差をつけたいときは
`server 192.168.0.12:8080 weight=2;` のように重みを付けられます。

注意: アプリをオンメモリキャッシュ化している場合、2 台に分けるとキャッシュも
2 つに割れて不整合が起きることがあります。複数台化はアプリの実装と相談しながら
進めてください。詳しくは [02-performance/07-multi-server.md](../old/02-performance/07-multi-server.md) を見てください。

## 当日チェックリスト: nginx まわりで最初にやること

競技開始直後、nginx について確認・準備することのリストです。30 分以内が目安です。

```
[ ] sudo nginx -T で現在の設定の全体像を見る
[ ] 設定ファイルを git 管理下にコピーしてバックアップする
[ ] 静的ファイルの実体の場所を確認する（ls で webapp/public など）
[ ] log_format を JSON 化して alp が使える状態にする（phase4）
[ ] マニュアルを読み、キャッシュ可否・レスポンスの制約を確認する
[ ] 初回ベンチを回し、alp の結果を保存する（改善前の基準値になる）
```

改善はこのあと、alp の結果を見ながらです。
「計測の準備が先、改善は計測してから」を当日も守ってください。

## 壊したときの戻し方

最後に保険の話です。設定変更でベンチが失敗するようになったら、
悩む前に直前の状態へ戻します。

- 設定ファイルを git 管理していれば `git diff` で変更点を確認し、checkout で戻す
- 戻したら nginx -t → reload → ベンチで復旧確認

「1 変更 1 ベンチ」を守ると、壊れたとき原因が一瞬で特定できます。
複数の変更をまとめて入れたくなっても、反映と計測は 1 つずつが安全です。

## このフェーズのまとめ

- 優先順位は「計測準備 → 静的ファイル直接配信 → keepalive → gzip → 複数台」
- nginx の定番を入れ終えたら、深追いせずアプリ・DB の改善へ移る
- 当日はまず現状確認・バックアップ・ログ JSON 化。改善は初回ベンチの計測後
- 1 変更 1 ベンチ。壊れたらすぐ戻す

これで nginx 入門は完了です。続きとして、
[02-performance/06-middleware-nginx.md](../old/02-performance/06-middleware-nginx.md)（実戦チューニングの詳細）と
[02-performance/01-measurement.md](../old/02-performance/01-measurement.md)（計測全般）に進んでください。
