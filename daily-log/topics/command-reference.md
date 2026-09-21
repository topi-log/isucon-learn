# ISUCON練習で使うコマンド集

今回のprivate-isu環境向け。実際の大会では当日マニュアルのサービス名・パス・ルールを優先する。

## 実行する場所

| 場所 | 主な作業 |
|---|---|
| Macのターミナル | Git管理、コード編集、SSH接続、ファイル転送、deploy.sh |
| 競技用サーバー（192.168.1.10） | アプリ・MySQL・Nginxの操作、ログ確認 |
| ベンチマーカー用サーバー（192.168.1.20） | ベンチマーク実行 |

```bash
whoami   # 現在のユーザー
hostname # 接続しているサーバー
pwd      # 現在のディレクトリ
```

SSHでubuntuに入った後、作業ユーザーへ切り替える。

```bash
sudo su - isucon
exit # 切り替え前のシェルに戻る。SSHログイン直後のシェルなら接続終了
```

## systemctl：サービスの確認・操作

実行場所：競技用サーバー。以下のisu-nodeをmysql・nginx・isu-rubyに置き換えて使う。

| コマンド | 意味 |
|---|---|
| `sudo systemctl status isu-node --no-pager -l` | 状態と直近のログを省略せず表示 |
| `sudo systemctl is-active isu-node` | 現在動いているか確認 |
| `sudo systemctl start isu-node` | 今すぐ起動 |
| `sudo systemctl stop isu-node` | 今すぐ停止 |
| `sudo systemctl restart isu-node` | 停止して起動し直す |
| `sudo systemctl enable isu-node` | OS起動時の自動起動を有効化（今すぐ起動はしない） |
| `sudo systemctl disable isu-node` | 自動起動を無効化（今すぐ停止はしない） |
| `sudo systemctl enable --now isu-node` | 自動起動を有効化し、今すぐ起動 |
| `sudo systemctl disable --now isu-node` | 自動起動を無効化し、今すぐ停止 |
| `sudo systemctl is-enabled isu-node` | 自動起動の設定を確認 |
| `sudo systemctl cat isu-node` | 起動コマンド・環境設定などのサービス定義を表示 |

- `active` は起動状態の確認。アプリが正しい応答を返すかは画面やベンチマークでも確認する。
- `restart` は処理を中断するので、ベンチマーク中には行わない。
- `reload` はサービスが対応している場合の設定再読み込み。すべてのサービスで使えるわけではない。
- `sudo systemctl daemon-reload` はsystemdのサービス定義（.service）変更時に使う。MySQLの.cnf変更を反映するコマンドではない。

## journalctl：サービスのログ

実行場所：競技用サーバー。

```bash
sudo journalctl -u isu-node -n 100 --no-pager # 最新100行
sudo journalctl -u isu-node -f               # 新しいログを追い続ける
sudo journalctl -u isu-node --since '10 minutes ago' --no-pager
sudo journalctl -u mysql -n 100 --no-pager
```

`-f` を終えるにはCtrl+C。ログ表示を終えるだけで、サービスは止まらない。

ファイルのログを見る場合：

```bash
sudo tail -n 100 /var/log/nginx/error.log
sudo tail -f /var/log/nginx/error.log
```

## MySQL設定：配置・検証・反映

Gitで管理するのはPC側の `infra/mysql/mysqld.cnf`。実際の設定場所はサーバーの `/etc/mysql/mysql.conf.d/mysqld.cnf`。

### 1. Macから一時配置する

練習リポジトリ直下で実行。`.env` は自分で作った設定ファイルを使う。

```bash
source .env
scp -i "$DEPLOY_KEY" infra/mysql/mysqld.cnf \
  "ubuntu@$DEPLOY_HOST:~/mysqld.cnf.new"
```

転送先は `/home/ubuntu/mysqld.cnf.new`。scpで指定した接続ユーザーにより `~/` の場所が決まる。

### 2. 競技用サーバーで配置・検証する

変更前の実設定がGitに保存されていることを確認してから上書きする。

```bash
sudo install -o root -g root -m 644 \
  /home/ubuntu/mysqld.cnf.new \
  /etc/mysql/mysql.conf.d/mysqld.cnf
sudo /usr/sbin/mysqld --validate-config
echo $?
```

`echo $?` は直前のコマンドの終了コード。0なら成功。検証エラー時は再起動せず、設定を修正するか変更前の内容に戻す。

### 3. 再起動・確認する

```bash
sudo systemctl restart mysql
sudo systemctl is-active mysql
sudo mysql -e "
SHOW GLOBAL VARIABLES WHERE Variable_name IN (
  'slow_query_log', 'slow_query_log_file', 'long_query_time', 'log_output'
);"
```

失敗したら `sudo journalctl -u mysql -n 100 --no-pager` を確認。設定の検証に通っても、ファイル権限などの実行時の問題がないとは限らない。

調査用の設定例（MySQLが読む.cnfの `[mysqld]` 内）：

```ini
[mysqld]
slow_query_log = ON
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 0
log_output = FILE
```

`long_query_time = 0` は原則すべてのクエリを記録する設定。ログ量と負荷が増えるため、計測条件をそろえてスコアを比較する。出力先はMySQLが書き込める場所にする。

## Nginx設定：検証・反映

実行場所：競技用サーバー。

```bash
sudo nginx -t
# 検証に成功した場合だけ実行
sudo systemctl reload nginx
sudo systemctl status nginx --no-pager
```

## Node.js（TypeScript）：ビルド・デプロイ

競技用サーバーで手動ビルドする場合：

```bash
cd /home/isucon/private_isu/webapp/node
export PATH="/home/isucon/.local/node/bin:$PATH"
npm run build
# ビルドに成功した場合だけ実行
sudo systemctl restart isu-node
sudo systemctl status isu-node --no-pager -l
```

Macから反映する場合：

```bash
cd ~/workspace/projects/private-isu-practice
./deploy.sh
```

`.env` に `DEPLOY_HOST`（競技用サーバーの公開IP）と `DEPLOY_KEY`（秘密鍵のパス）を設定しておく。

現在のdeploy.shはwebapp/nodeの転送・依存インストール・ビルド・再起動を行う。MySQL・Nginx設定やwebapp/publicは対象外。PCで削除したファイルはサーバーから自動削除しない。

## ベンチマーク

実行場所：ベンチマーカー用サーバー、isuconユーザー。

```bash
cd /home/isucon/private_isu/benchmarker
./bin/benchmarker -u ./userdata -t http://192.168.1.10/
```

宛先は競技用サーバーのプライベートIP。結果のpass・score・fail・messagesを記録する。比較時はアプリのバージョン、宛先、計測設定をそろえる。

## 負荷・ファイル・権限を見る

```bash
top                 # CPU・メモリ・プロセスを見る（qで終了）
free -h             # メモリ使用状況
df -h               # ディスク空き容量
ls -la              # 隠しファイルも含む一覧
ls -lh              # ファイルサイズを読みやすく表示
du -h --max-depth=1 . # ディレクトリごとの容量（Linux）
command -v mysqldumpslow pt-query-digest alp # ツールの場所。未導入なら表示されない
```

`top` の主な列：%CPUはCPU使用率、RESはRAM上のメモリ量、Sは状態（R=実行中または実行待ち、S=待機、D=割り込み不可の待機）。

```bash
chmod 600 ~/.ssh/isucon2026.pem # 秘密鍵を所有者だけが読み書きできる状態にする（Mac）
```

権限不足のときは必要な操作だけsudoを使う。設定ファイルを誰でも書ける権限に変えない。

## Git：変更の保存と確認

実行場所：Macの練習リポジトリ。

```bash
git status                 # 作業状態
git diff                   # 未ステージの変更
git add infra/mysql/mysqld.cnf # 保存したいファイルを選ぶ
git diff --cached          # コミット対象の変更内容
git diff --cached --stat   # コミット対象の概要
git commit -m "Enable MySQL slow query log"
git push
git log -5 --oneline
```

GitHubへのpushとサーバーへの反映は別。pushしても自動で設定は反映されない。秘密鍵・パスワード・ローカルの.envはコミットしない。

変更前のファイルを別ファイルに取り出す場合（コミットIDは置き換える）：

```bash
git show COMMIT_ID:infra/mysql/mysqld.cnf > /tmp/mysqld.cnf.restore
```

内容を確認してから転送・配置・検証・再起動する。Gitで戻すだけではサーバーの設定は戻らない。

## SSHとファイル転送

Macから競技用サーバーへ接続する：

```bash
source .env
ssh -i "$DEPLOY_KEY" "ubuntu@$DEPLOY_HOST"
```

- 公開IPに `http://` は付けない。
- ubuntuへSSH接続してから `sudo su - isucon` する方法と、isuconへ直接SSH接続する方法では、認証に使える鍵が異なることがある。
- `Permission denied (publickey)` はSSH認証の問題。ファイルを読む権限のエラーとは区別する。
- 場所を変えた後に接続できない場合は、接続元公開IPv4とセキュリティグループの22番許可を確認する。

## 終了・再開

作業終了時はAWSコンソールで今回のEC2を2台とも「停止」する。「終了」は削除につながるため選ばない。停止中もEBSとElastic IPの料金は残る。

再開時は2台を起動し、状態を確認してSSH接続する。
