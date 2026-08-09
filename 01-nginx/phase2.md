# phase2: 設定ファイルを読めるようになる

このフェーズでわかるようになること

- 設定ファイルのブロック構造（http / server / location）
- location がどういうルールでリクエストとマッチするか
- 初期状態の設定を読んで「何をしているか」説明できる

## 設定ファイルの構造

nginx の設定は「ディレクティブ」の集まりです。書き方は 2 種類だけです。

```nginx
worker_processes auto;      # 単純な設定: 名前 値 ;

http {                      # ブロック: 名前 { 中に別の設定 }
    ...
}
```

ブロックは入れ子になっていて、外側の設定は内側に引き継がれます。
全体はこういう三層構造です。

```nginx
# いちばん外（グローバル）: nginx 全体の設定
worker_processes auto;

events {
    worker_connections 1024;
}

http {                          # HTTP に関する設定すべての入れ物
    access_log /var/log/nginx/access.log;

    server {                    # 1 つのサイト（待ち受け）の設定
        listen 80;

        location / {            # パスごとの設定
            proxy_pass http://127.0.0.1:8080;
        }
    }
}
```

読むときの頭の使い方はシンプルで、
「server = どのポートで受けるか」「location = どのパスをどう処理するか」の 2 つを追えば
だいたい理解できます。

## server ブロック

```nginx
server {
    listen 80;
    server_name isucon.example.com;
    ...
}
```

- listen: 待ち受けるポート番号。80 なら普通の HTTP を受けます
- server_name: どのホスト名（リクエストの Host ヘッダー）宛てを受けるか。
  1 台で複数サイトを住み分けるための仕組みですが、ISUCON ではサイトが 1 つのことが
  多いので、あまり気にしなくて大丈夫です

## location ブロック

server の中で、パスごとに処理を分けるのが location です。ここが読解の中心になります。

```nginx
location /assets/ {     # /assets/ で始まるパスはここ
    root /home/isucon/webapp/public;
}

location / {            # それ以外は全部ここ
    proxy_pass http://127.0.0.1:8080;
}
```

書き方は主に 3 種類あります。

```nginx
location = /favicon.ico { ... }   # 完全一致: このパスそのものだけ
location /assets/ { ... }         # 前方一致: /assets/ で始まるもの全部
location ~ \.(js|css)$ { ... }    # 正規表現: パターンにマッチするもの
```

複数の location に当てはまりそうなときの優先順位は次のとおりです。

1. `=` の完全一致が最優先
2. 次に正規表現（`~`）。書いた順に上から試される
3. 最後に前方一致。マッチする長さがいちばん長いものが勝つ

「上から順に見る」のではなくルール優先である点だけ注意してください。
たとえば `location /` はどんなパスにもマッチしますが優先度は最弱なので、
「どこにも当てはまらなかったときの受け皿」として機能します。

## よく見るディレクティブ

### root — ファイルを探す起点

```nginx
location /assets/ {
    root /home/isucon/webapp/public;
}
```

root は「パスの前にくっつける起点ディレクトリ」です。
`/assets/app.js` へのリクエストは `/home/isucon/webapp/public/assets/app.js` を探します。
パスがそのまま後ろに連結される、というのがポイントです。

### alias — パスを置き換える

```nginx
location /images/ {
    alias /home/isucon/private/img/;
}
```

alias は location のパス部分を置き換えます。
`/images/cat.png` は `/home/isucon/private/img/cat.png` を探します。
root との違いは「/images/ の部分が残るか消えるか」です。
URL とディスク上のディレクトリ名が一致しないときに alias を使います。

### index — ディレクトリのときに返すファイル

```nginx
index index.html;
```

`/` のようにディレクトリを指すリクエストが来たら index.html を返す、という指定です。

### try_files — 順番に探して、なければ次へ

```nginx
location / {
    try_files $uri /index.html;
}
```

「まず `$uri`（リクエストされたパス）のファイルを探し、なければ /index.html を返す」
という意味です。`$uri` のようにドル記号で始まるものは変数で、
リクエストの内容に応じて中身が変わります。
SPA（1 枚の HTML で動くフロントエンド）の配信でよく見る形です。

## 練習: ISUCON の初期設定を読んでみる

典型的な初期状態の設定です。上から読んでみてください。

```nginx
server {
    listen 80;

    client_max_body_size 10m;

    root /home/isucon/webapp/public/;

    location / {
        proxy_set_header Host $host;
        proxy_pass http://127.0.0.1:8080;
    }
}
```

読解するとこうなります。

- 80 番ポートで受ける
- client_max_body_size 10m: アップロードなどの本文サイズ上限を 10MB にする
  （超えると 413 エラー。画像投稿がある問題で引っかかることがあります）
- root は指定されているが、location がひとつしかなく、
  すべてのリクエストが `proxy_pass` で 127.0.0.1:8080 のアプリに横流しされる。
  つまり root は実質使われていない
- 結論: 「nginx は何もせず全部アプリに投げている」状態。
  静的ファイルすらアプリが返しており、ここに改善の余地がある（phase5 でやります）

初期設定がこの形だと分かっていれば、当日は差分だけ見ればよくなります。

## このフェーズのまとめ

- 設定は http > server > location の入れ子。server はポート、location はパスごとの処理
- location は「完全一致 > 正規表現 > 前方一致の最長」の順で選ばれる
- root はパスを連結、alias は置き換え
- ISUCON の初期設定は「全部アプリに横流し」がお決まりの形

次の phase3 では、その「横流し」の中身であるリバースプロキシをちゃんと理解します。
