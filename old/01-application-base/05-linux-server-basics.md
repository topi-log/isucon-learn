# 1-5. Linuxサーバー操作の基礎

> **このドキュメントで学ぶこと**
> - ISUCON の競技サーバーで必要になる Linux 操作（systemd 中心）
> - サービスの再起動・ログ確認・設定ファイルの場所
> - リソース使用状況の見方

## 競技サーバーに入ったらまず把握すること

```bash
ssh isucon@<サーバーIP>

# 何が動いているか
sudo systemctl list-units --type=service --state=running

# CPU コア数とメモリ（チューニング値を決める材料になる）
nproc
free -h
df -h            # ディスク空き（ログの吐きすぎで埋まることがある）

# アプリのコードはどこか（典型: /home/isucon/webapp/ 以下に言語別ディレクトリ）
ls ~/webapp/
```

## systemd — サービス管理

ISUCON のアプリ・nginx・MySQL はすべて systemd のサービスとして動いています。

```bash
# 状態確認
sudo systemctl status isuride-node.service

# 再起動（コード変更を反映するときに叩く）
sudo systemctl restart isuride-node.service

# 参考実装の言語切り替え（例: Go を止めて Node.js を有効化）
sudo systemctl disable --now isuride-go.service
sudo systemctl enable --now isuride-node.service

# unit ファイルの中身を見る（起動コマンド・環境変数がわかる）
systemctl cat isuride-node.service
```

unit ファイルの例（`ExecStart` と `EnvironmentFile` が重要）:

```ini
[Service]
WorkingDirectory=/home/isucon/webapp/nodejs
EnvironmentFile=/home/isucon/env.sh     # DBホスト等の環境変数はここ
ExecStart=/home/isucon/.local/node/bin/node dist/main.js
Restart=always
```

**`EnvironmentFile` の場所は序盤に必ず確認**してください。複数台構成（2-7）で DB の接続先を変えるときにここを書き換えます。

## ログの見方 — journalctl

```bash
# サービスのログを追いかける（アプリがクラッシュしてないか確認）
sudo journalctl -u isuride-node.service -f

# 直近100行
sudo journalctl -u isuride-node.service -n 100 --no-pager
```

ベンチ実行後にアプリのエラーが出ていないかを見る癖をつけましょう。**スコアが急に落ちたときの原因は大抵ログに出ています。**

## 主要な設定ファイルの場所

| 対象 | 場所 |
|------|------|
| nginx 本体設定 | `/etc/nginx/nginx.conf` |
| nginx サイト設定 | `/etc/nginx/sites-enabled/*.conf` または `/etc/nginx/conf.d/*.conf` |
| MySQL 設定 | `/etc/mysql/mysql.conf.d/mysqld.cnf`（Ubuntu系） |
| アプリの環境変数 | `/home/isucon/env.sh` など（unit ファイルで確認） |

設定変更後の反映:

```bash
# nginx: 文法チェックしてからリロード（reload なら接続を切らない）
sudo nginx -t && sudo systemctl reload nginx

# MySQL: 再起動が必要（数秒〜数十秒止まるのでベンチ中は避ける）
sudo systemctl restart mysql
```

## リソース使用状況の見方

ベンチマーク実行中に別ターミナルで眺めるのが基本動作です。

```bash
top   # または htop / dstat
```

`top` で見るポイント:

```
%Cpu(s): 85.0 us,  5.0 sy,  0.0 ni,  5.0 id, 5.0 wa
         ↑ユーザー処理      ↑アイドル(暇) ↑I/O待ち

  PID USER   %CPU %MEM COMMAND
 1234 mysql   160  40.0 mysqld     ← MySQL が2コア近く食っている = DBがボトルネック
 5678 isucon   30   5.0 node
```

| 観察結果 | 意味 | 打ち手 |
|---------|------|--------|
| `mysqld` の CPU が支配的 | DB がボトルネック | インデックス・N+1・クエリ改善（2-2, 2-3） |
| `node` の CPU が支配的 | アプリがボトルネック | キャッシュ・処理削減・プロセス複数化（2-5, 2-7） |
| `wa`（I/O待ち）が高い | ディスクI/O がボトルネック | ログ削減、バッファプール拡大（2-4） |
| CPU も I/O も余っているのに遅い | ロック待ち・コネクション枯渇・設定上限 | PROCESSLIST、プールサイズ確認（1-4） |

## 競技でよく使うその他のコマンド

```bash
# ポートを誰が listen しているか（アプリが起動できてないときの調査）
sudo ss -ltnp

# git 管理（競技開始直後に必ずやる。壊したとき戻れるように）
cd ~/webapp && git init && git add -A && git commit -m "initial"

# ファイル転送（ローカルで編集する派の場合）
scp -r isucon@server:~/webapp/nodejs ./
rsync -av ./nodejs/ isucon@server:~/webapp/nodejs/
```

---

これで Part 1 は完了です。

## 用語集

| 用語 | 意味 |
|------|------|
| systemd | Linuxのサービス（常駐プログラム）管理の仕組み。起動・停止・自動起動・ログをまとめて面倒を見る |
| サービス / unit | systemdが管理する対象の単位。`isuride-node.service`のような単位で起動・停止する |
| unit ファイル | サービスの定義ファイル。起動コマンド（ExecStart）や環境変数ファイル（EnvironmentFile）が書いてある |
| enable / disable | サーバー起動時にサービスを自動起動する/しない設定。`enable`忘れは再起動試験で失格の原因 |
| journalctl | systemd管理下のサービスのログを見るコマンド。`-f`で流し見、`-u`でサービス指定 |
| EnvironmentFile | サービスに渡す環境変数（DBホスト名など）を書いたファイル。複数台構成でここを書き換える |
| ssh / scp / rsync | リモートサーバーへのログイン / ファイルコピー / 差分同期。rsyncはデプロイの定番 |
| top / htop | プロセスごとのCPU・メモリ使用率をリアルタイム表示するコマンド |
| us / sy / id / wa | topのCPU内訳。ユーザー処理 / カーネル処理 / アイドル（暇）/ I/O待ち。waが高い=ディスクがボトルネックの兆候 |
| nproc | CPUコア数を表示するコマンド |
| df / free | ディスク使用量 / メモリ使用量の確認コマンド |
| listen（リッスン） | プログラムが特定ポートで接続を待ち受けている状態。`ss -ltnp`で確認 |
| ポート | 1台のサーバー上で通信の宛先を区別する番号（HTTP=80, HTTPS=443, MySQL=3306など） |
| reload と restart | reload=設定だけ読み直す（接続を切らない）、restart=プロセスを再起動する（一瞬止まる）。nginxはreloadで十分なことが多い |

---

**自分用メモ →** [分からなかった単語を書き足す](05-linux-server-basics-terms.md)
**次に読む →** [2-1. 計測ファースト](../02-performance/01-measurement.md)
