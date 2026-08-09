# phase3: リバースプロキシを理解する

このフェーズでわかるようになること

- リバースプロキシとは何か、なぜアプリの前に挟むのか
- proxy_pass と proxy_set_header が何をしているか
- upstream と keepalive、unix domain socket で速くする仕組み

## リバースプロキシとは

プロキシは「代理人」という意味です。
リバースプロキシは、サーバー側の代理人としてクライアントの相手をするプログラムを指します。

```
ブラウザ  --->  nginx（代理人）  --->  アプリ（本人）
          <---                  <---
```

ブラウザから見える相手は nginx だけで、アプリの存在は見えません。
nginx はリクエストを受け取り、それをアプリに転送し、アプリの返事をブラウザに返します。

わざわざ間に挟む理由は phase0 でも触れたとおりです。

- 得意な仕事（静的ファイル配信、圧縮）を代理人が肩代わりできる
- 全リクエストが通る場所なので、記録（ログ）を一箇所で取れる
- 後ろにアプリを複数並べて、振り分けができる

## proxy_pass — 転送先の指定

リバースプロキシの中心となるディレクティブです。

```nginx
location / {
    proxy_pass http://127.0.0.1:8080;
}
```

「この location に来たリクエストを 127.0.0.1:8080 に転送しろ」という意味です。

### 末尾スラッシュの罠

proxy_pass には有名な落とし穴があります。転送先 URL の末尾に `/` を付けるかどうかで
挙動が変わります。

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080;     # スラッシュなし
}
# /api/users へのリクエスト → アプリには /api/users が届く（パスそのまま）

location /api/ {
    proxy_pass http://127.0.0.1:8080/;    # スラッシュあり
}
# /api/users へのリクエスト → アプリには /users が届く（/api/ が削られる）
```

スラッシュありの場合、location のパス部分が転送先のパスに置き換えられます。
ISUCON では基本的にパスをそのまま渡したいので、スラッシュなしにしておくのが安全です。
「設定を変えたら急に 404 が増えた」ときはまずここを疑ってください。

## proxy_set_header — アプリに正しい情報を渡す

nginx が代理でリクエストを転送すると、アプリから見た通信相手は nginx になります。
すると、そのままではアプリに正しく伝わらない情報が出てきます。
それを補うのが proxy_set_header です。

```nginx
location / {
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_pass http://127.0.0.1:8080;
}
```

- Host: もともとブラウザが送ってきたホスト名。アプリが URL を組み立てるときに使うので、
  渡さないとリダイレクト先がおかしくなることがあります
- X-Forwarded-For: 本来のクライアントの IP アドレス。
  渡さないと、アプリにはすべてのアクセスが nginx（127.0.0.1）から来たように見えます

初期設定に書いてあったら消さずに残す、と覚えておけば十分です。

## upstream と keepalive — 接続を使い回す

転送先は proxy_pass に直接書く代わりに、upstream ブロックとして名前を付けられます。

```nginx
upstream app {
    server 127.0.0.1:8080;
}

server {
    location / {
        proxy_pass http://app;
    }
}
```

これだけだと書き方が変わっただけですが、upstream にすると追加の設定ができるように
なります。代表が keepalive です。

デフォルトでは、nginx はアプリへ転送するたびに接続を作って、返事をもらったら
切断します。この接続の作成・切断は毎回コストがかかります。
keepalive を設定すると、使い終わった接続を捨てずに次のリクエストで使い回します。

```nginx
upstream app {
    server 127.0.0.1:8080;
    keepalive 128;                       # 使い回す接続を最大128本キープ
}

server {
    location / {
        proxy_pass http://app;
        proxy_http_version 1.1;          # keepalive に必須
        proxy_set_header Connection "";  # keepalive に必須
        proxy_set_header Host $host;
    }
}
```

下 2 行の proxy_http_version と proxy_set_header Connection はセットで必要です。
理由まで覚える必要はありません。「keepalive を使うならこの 3 点セット」で覚えてください。
ISUCON では大量のリクエストが飛んでくるので、これだけでスコアが上がることがあります。

## unix domain socket — 同居ならさらに速く

nginx とアプリが同じサーバーにいる場合、127.0.0.1 への TCP 通信は少し無駄があります。
同じマシン内なのに、ネットワーク通信の手続きを踏んでいるためです。

同一マシン内専用の通信路として unix domain socket があります。
見た目はファイルで、TCP よりオーバーヘッドが小さくなります。

```nginx
upstream app {
    server unix:/tmp/app.sock;
    keepalive 128;
}
```

ただし nginx 側だけでなく、アプリ側も「ポート 8080 で待つ」のをやめて
「/tmp/app.sock で待つ」ように変更する必要があります（Node.js なら
`server.listen(8080)` を `server.listen('/tmp/app.sock')` に変えるイメージです）。
アプリ側の変更とソケットファイルの権限設定が必要なので、
効果は keepalive より小さめなことも多く、余裕があればやる改善という位置づけです。

## 502 と 504 の意味がわかるようになる

phase0 で予告したエラーの正体です。どちらも「代理人が本人と話せない」系のエラーです。

- 502 Bad Gateway: nginx がアプリに接続できない、または変な応答が返ってきた。
  アプリが落ちている・ポートやソケットのパスが間違っている、が定番の原因
- 504 Gateway Timeout: 接続はできたが、アプリの返事が遅すぎて待ちきれなかった。
  アプリや DB が重すぎるのが原因で、nginx 側は悪くないことが多い

ベンチマーク中に 502/504 が出たら、nginx ではなく後ろ（アプリ・DB）を見に行く、
という判断ができるようになります。

## このフェーズのまとめ

- リバースプロキシはサーバー側の代理人。proxy_pass で転送先を指定する
- proxy_pass の末尾スラッシュでパスの渡り方が変わる。基本はスラッシュなし
- proxy_set_header は「代理を挟んだせいで欠ける情報」を補うためのもの
- upstream + keepalive の 3 点セットは定番の高速化。同居構成なら unix socket も選択肢
- 502 はつながらない、504 は待ちきれない

次の phase4 では、nginx のアクセスログを使って「どこが遅いのか」を計測します。
