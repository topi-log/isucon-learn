# 2-6. ミドルウェア (nginx) チューニング

> **このドキュメントで学ぶこと**
> - 静的ファイルを nginx から直接配信する設定
> - アプリへのプロキシを速くする設定（keepalive / unix socket）
> - nginx 本体の基本設定

nginx の改善は「アプリに仕事をさせない」ことが目的です。DB・アプリの改善（2-2〜2-5）と比べると効果は問題依存ですが、静的ファイル配信は定番で、設定もほぼ毎回同じです。

## 現状確認

```bash
cat /etc/nginx/sites-enabled/*.conf   # or /etc/nginx/conf.d/
```

初期状態は大抵「全リクエストをアプリに丸投げ」:

```nginx
server {
    listen 80;
    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

## 改善1: 静的ファイルは nginx が直接返す

JS/CSS/画像をアプリ（Node.js）が返すのは無駄。nginx はファイル配信が本業で、圧倒的に速い。

```nginx
server {
    listen 80;
    root /home/isucon/webapp/public;   # 静的ファイルの実体の場所を確認して指定

    # 静的アセット: nginxが直接返す + ブラウザキャッシュ許可
    location /assets/ {
        expires 24h;
        add_header Cache-Control public;
    }
    location = /favicon.ico { expires 24h; }

    # それ以外はアプリへ
    location / {
        proxy_pass http://app;
    }
}
```

- `expires` を付けるとベンチマーカーがクライアントキャッシュしてくれる問題もある（マニュアル要確認）
- 2-4 パターンBで DB から追い出した画像も、ここで `location /images/` として配信する

## 改善2: アプリへのプロキシを速くする

### upstream keepalive

デフォルトでは nginx → アプリの接続が毎回張り直されます。keep-alive で使い回す:

```nginx
upstream app {
    server 127.0.0.1:8080;
    keepalive 128;               # 保持するアイドル接続数
}

server {
    location / {
        proxy_pass http://app;
        proxy_http_version 1.1;          # ← keepaliveに必須
        proxy_set_header Connection "";  # ← keepaliveに必須
        proxy_set_header Host $host;
    }
}
```

### unix domain socket

nginx とアプリが同居しているなら、TCP より unix socket の方がオーバーヘッドが小さい:

```nginx
upstream app {
    server unix:/tmp/app.sock;
    keepalive 128;
}
```

アプリ側 (Fastify の例):

```typescript
await app.listen({ path: "/tmp/app.sock" });
// 注意: 起動時に既存のsockファイルがあると失敗する。起動前に削除する処理を入れる
```

（複数台構成でアプリを別サーバーに出すときは TCP に戻す）

## 改善3: nginx 本体設定

`/etc/nginx/nginx.conf`:

```nginx
worker_processes auto;           # CPUコア数に合わせる

events {
    worker_connections 4096;     # デフォルト768は少ない
}

http {
    keepalive_requests 10000;    # クライアント側keep-aliveの再利用回数
    sendfile on;
    tcp_nopush on;

    # ベンチマーカーがgzip対応ならCPUと相談して検討（静的ファイルはgzip_static）
    # gzip on;
}
```

反映は毎回:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

## 改善4: 複数アプリサーバーへの振り分け（2-7 の準備）

```nginx
upstream app {
    server 127.0.0.1:8080;        # 自分
    server 192.168.0.12:8080;     # サーバー2のアプリ
    keepalive 128;
}
```

- デフォルトはラウンドロビン。重み付けは `server ... weight=2;`
- **特定エンドポイントだけ特定サーバーへ**も可能。オンメモリキャッシュ（2-5）を使う場合、同じデータを扱うエンドポイントを同じサーバーに寄せる、という設計とセットで使う（→ 2-7）:

```nginx
location /api/chair/ {
    proxy_pass http://chair_app;   # 椅子系APIはサーバー2に固定
}
```

## 最終盤のチェック

- アクセスログ (`access_log`) は計測用。**最終ベンチ前に `access_log off;` にする**（書き込み負荷が消える）
- `sudo nginx -t` を通してから reload。文法エラーで nginx が起動しない状態のまま競技終了、が最悪のパターン

---

## 用語集

| 用語 | 意味 |
|------|------|
| location | nginx設定の「このURLパスへのリクエストはこう処理する」というブロック。`location = /exact`は完全一致 |
| proxy_pass | リクエストを後ろのアプリサーバーへ転送する指示 |
| upstream | 転送先サーバー（群）の定義ブロック。複数書けばロードバランスされる |
| ラウンドロビン | 複数の転送先に順番に振り分ける方式。upstreamのデフォルト動作 |
| upstream keepalive | nginx→アプリ間の接続を使い回す設定。`proxy_http_version 1.1`と`Connection ""`ヘッダ指定が必須 |
| unix domain socket | 同一マシン内のプロセス間通信。nginxとアプリが同居しているならTCPより速い |
| root | 静的ファイルの実体を探すベースディレクトリの指定 |
| expires / Cache-Control | 「このファイルはしばらく再取得不要」とクライアントに伝えるヘッダ。ブラウザ（ベンチマーカー）キャッシュを促す |
| worker_processes | nginxのワーカープロセス数。`auto`でCPUコア数に合わせる |
| worker_connections | ワーカー1つが同時に扱える接続数。デフォルトは小さめなので増やす |
| sendfile / tcp_nopush | ファイル送信をカーネル内で効率化する設定。静的配信を速くする |
| gzip / gzip_static | レスポンス圧縮。転送量が減るがCPUを使う。事前圧縮版（gzip_static）ならCPU消費なし |
| `nginx -t` | 設定ファイルの文法チェックコマンド。reload前に必ず実行する習慣をつける |
| access_log off | アクセスログ出力の停止。計測が終わった最終盤に設定してログ書き込み負荷を消す |

---

**自分用メモ →** [分からなかった単語を書き足す](06-middleware-nginx-terms.md)
**次に読む →** [2-7. 複数台構成](07-multi-server.md)
