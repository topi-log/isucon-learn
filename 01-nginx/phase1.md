# phase1: nginx に触れる — 起動・停止・設定の場所

このフェーズでわかるようになること

- nginx の設定ファイルがどこにあり、どうつながっているか
- nginx を安全に再起動・反映する手順（変更 → nginx -t → reload の型）
- 困ったときにどのログを見ればいいか

## nginx は何ができるのか

phase0 で見たとおり、nginx はいちばん手前でリクエストを受ける受付です。
できることは大きく 3 つです。

- 静的ファイルの配信: ディスク上のファイルをそのまま返す
- リバースプロキシ: 受けたリクエストを後ろのアプリに横流しする
- ロードバランス: 複数のアプリに仕事を振り分ける

ISUCON ではこの 3 つを全部使います。

## 設定ファイルの場所

nginx の動きはすべて設定ファイルで決まります。大元は 1 つです。

```
/etc/nginx/nginx.conf
```

ただし、この 1 ファイルに全部書いてあるわけではありません。
nginx.conf の中に include という行があり、別のディレクトリのファイルを読み込んでいます。

```nginx
http {
    # ... 全体の設定 ...
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

- `/etc/nginx/conf.d/` や `/etc/nginx/sites-enabled/` に、サイトごとの設定が置かれます
- ISUCON で主に編集するのはこちら側です。競技が始まったらまず
  `sudo cat /etc/nginx/nginx.conf` で include 先を確認し、そこにあるファイルを開いてください

```bash
# 実際に読み込まれている設定を全部まとめて見るコマンド
sudo nginx -T | less
```

`-T`（大文字）は include をすべて展開して表示してくれるので、全体像の把握に便利です。

## 起動・停止・reload

nginx は systemd（Linux のサービス管理の仕組み）経由で動いています。
操作は systemctl コマンドです。

```bash
sudo systemctl status nginx    # 今動いているか確認
sudo systemctl start nginx     # 起動
sudo systemctl stop nginx      # 停止
sudo systemctl restart nginx   # 完全に再起動
sudo systemctl reload nginx    # 設定だけ読み直す
```

restart と reload の違いは重要です。

- restart: プロセスを止めてから起動し直す。一瞬リクエストを受けられない時間ができる
- reload: 動いたまま設定だけ読み直す。リクエストを取りこぼさない

設定変更の反映は reload で足ります。restart が必要になるのは nginx 自体の調子が
おかしいときくらいです。

## 設定変更の型: 変更 → nginx -t → reload

設定ファイルに文法ミスがある状態で reload すると、反映に失敗します。
さらに悪いことに、その状態で restart すると nginx が起動できず、サイト全体が落ちます。

そこで、反映の前に必ず文法チェックを挟みます。

```bash
sudo nginx -t
```

成功ならこう出ます。

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

失敗なら、どのファイルの何行目が悪いかを教えてくれます。

```
nginx: [emerg] unexpected "}" in /etc/nginx/sites-enabled/isucon.conf:15
```

まとめると、設定をいじるときは常にこの 3 ステップです。

```bash
sudo vim /etc/nginx/sites-enabled/isucon.conf   # 1. 編集
sudo nginx -t                                    # 2. 文法チェック
sudo systemctl reload nginx                      # 3. 反映
```

この型を体に入れておけば、nginx を壊して時間を溶かす事故はほぼ防げます。

## ログの場所

nginx は 2 種類のログを書きます。

```
/var/log/nginx/access.log   # 受けた全リクエストの記録
/var/log/nginx/error.log    # エラーや警告の記録
```

- access.log は phase4 の主役です。ここを集計して遅いエンドポイントを探します
- error.log は「なんか動かない」ときに最初に見る場所です

```bash
sudo tail -f /var/log/nginx/error.log   # エラーログを流しながら見る
```

たとえば 502 エラーが出ているとき、error.log には
「connect() failed ... while connecting to upstream」のような行が残っていて、
アプリにつなげていないことがわかります（アプリが落ちている、ポート番号が違う、など）。

## このフェーズのまとめ

- 大元は /etc/nginx/nginx.conf。実際の編集対象は include されている conf.d や sites-enabled 側
- 全体を見るなら sudo nginx -T
- 反映の型は「編集 → nginx -t → systemctl reload nginx」。restart は極力使わない
- 困ったら /var/log/nginx/error.log

次の phase2 では、開いた設定ファイルの中身を読めるようになります。
