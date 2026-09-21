# ISUCON開始から初回ベンチマークまで

private-isuをCloudFormationで構築し、競技開始時の流れを練習するための作業メモ。実際に進めた内容を順次追記する。

実際の大会では環境の配布方法や事前に許可される準備が異なるため、その回のマニュアル・レギュレーションを優先する。ここでは自分のAWSアカウントに環境を構築する想定。

## 事前準備（毎回の競技開始時には繰り返さない）

| 状態 | 作業 | 今回の記録 |
|---|---|---|
| 完了 | 自分用IAMユーザーを作成 | コンソールアクセスを有効にし、AdministratorAccessを直接アタッチ |
| 完了 | IAMユーザーのMFAを設定 | パスキーを登録 |
| 完了 | IAMユーザーでログイン確認 | 2026-09-17に確認 |
| 完了 | 東京リージョンにEC2キーペアを作成 | PEMを保存し、chmod 600を実施 |
| 完了 | GitHubのSSH公開鍵を確認 | 自分のユーザー名.keysに公開鍵が表示された |
| 未着手 | チームメンバーのSSH公開鍵を準備 | 各自が鍵を作り、公開鍵だけを共有する |

IAMユーザーとMFAは今回準備したものを使用する。当日はログインできることを確認すればよい。キーペアも同じリージョンで再利用できるものがあれば作り直さない。

### 今回のアクセス方針

- 代表者が既存のAWSアカウントでCloudFormation・EC2を操作する。
- 専用AWSアカウントやスイッチロールは今回作成しない。
- 管理者権限は既存の他の環境にも及ぶため、操作対象のスタック・リソースを確認する。
- メンバーは各自のSSH鍵でサーバーへ接続する。AWSコンソール用ユーザーは今のところ作成しない。
- AWSのログイン情報、秘密鍵、アクセスキーはリポジトリに保存しない。

### 実施した事前準備：EC2キーペア

1. AWSコンソールのリージョンを東京（ap-northeast-1）にする。
2. EC2の「キーペア」から作成画面を開く。
3. 名前は `private-isu-practice`、タイプはRSA、形式は `.pem` を指定する。
4. ダウンロードされた秘密鍵をPCで安全に保管する。メンバーには渡さない。

## 当日の流れと進捗

2026-09-20までに代表者のSSH接続・画面表示・初回ベンチマークまで成功。メンバーの接続設定は未実施。

- [ ] マニュアル・レギュレーションを読み、構築方法と競技対象を確認する。
- [x] 配布されたCloudFormationテンプレートの変更内容を確認する。
  - リージョン、インスタンスの台数・種類、AMI、ネットワーク、接続元の許可を確認する。
  - 元の競技用AMIは検索結果に表示されなかった。READMEのAMI概要をユーザーが確認し、両サーバーを `ami-09201e964bee13733` に変更した。
  - 競技用は `c7a.large`、ベンチマーカー用は `c7a.xlarge` に変更した。
  - SSHの接続元を公開IPv4アドレスの `/32` に限定した。会場に着いたら公開IPv4を調べ直し、構築前にテンプレートを更新する。構築済みならセキュリティグループの許可も更新する。
  - 公開IPの確認には [みんなのネット回線速度：IPアドレス確認](https://minsoku.net/ip_confirmations) を使用した。会場でもこのページで公開IPv4アドレスを確認し、SSHの許可には末尾に `/32` を付ける。
  - 保存先は `practice/private-isu.yaml`。スタックのCREATE_COMPLETEを確認済み。両サーバーを起動し、SSH接続できた。
- [x] テンプレートと必要なパラメーターを指定してスタックを作成する。
- [x] スタックの作成完了と、サーバーの起動・初期設定完了を確認する。
- [x] 代表者がSSH接続し、アプリの画面表示を確認する。
- [ ] 両サーバーにメンバーのSSH公開鍵と接続元の許可を設定し、全員の接続を確認する。
- [x] アプリを変更する前に、ベンチマーカー用サーバーから競技用サーバーのプライベートIPへ初回ベンチマークを実行する。
- [x] 成否・スコア・実行日時・使用した構成を記録し、改善前の基準にする。

## 実行した接続・ベンチマーク手順

1. Macから各サーバーへ `ssh -i ~/.ssh/private-isu-practice.pem ubuntu@公開IPv4` で接続（鍵の名前は実際のファイル名に合わせる）。
2. サーバー上で `sudo su - isucon` を実行する。
3. ブラウザで `http://競技用サーバーの公開IPv4/` を開き、画面表示を確認する。
4. ベンチマーカー用サーバーのisuconユーザーで以下を実行する。

```bash
/home/isucon/private_isu/benchmarker/bin/benchmarker \
  -u /home/isucon/private_isu/benchmarker/userdata \
  -t http://192.168.1.10
```

### 初回結果（Ruby・公開IP経由）

後からユーザーが、以前の測定では公開IPを宛先にしていたと確認。上のプライベートIP宛てコマンドは今後の標準手順。

記録日：2026-09-20。正確な実行時刻は未記録。構成は上記AMI、競技用c7a.largeとベンチマーカー用c7a.xlarge。

```json
{"pass":true,"score":1054,"success":917,"fail":0,"messages":[]}
```

初回ベンチマークは成功。今回は1人で進めるため、メンバーのSSH公開鍵登録と接続確認は保留。

### チームのSSH準備

- 各メンバーのGitHub IDを事前に把握し、`https://github.com/ユーザー名.keys` で公開鍵が取得できることを確認する。
- メンバー本人が対応する秘密鍵を当日使うPCに持っていることを確認する。秘密鍵は共有しない。
- 今回のテンプレートはGitHubUsernameに指定した1人分だけを登録するため、メンバー分は両サーバーのisuconユーザーのauthorized_keysへ追記する。
- 同じ会場でも公開IPv4が同じか確認する。同じならSSHの接続元許可は1つでよい。
- 今回はEC2キーペアでubuntuに接続したため、GitHub由来の鍵でisuconに直接接続できるかは未確認。


## 起動前の料金メモ

2026-09-17に確認した東京・Linuxオンデマンドの参考単価は、c7a.largeが$0.1292/時、c7a.xlargeが$0.2584/時。公開IPv4を2個（合計$0.010/時）含めて$0.3976/時。1ドル150円と仮定すると約60円/時、8時間で約477円。税・EBS・通信料は別。EBS容量・種類はテンプレートに明示がなく、AMIの設定を引き継ぐため未確認。

参考：[large料金](https://aws-pricing.com/c7a.large.html)、[xlarge料金](https://cloudcostdb.com/aws/instances/c7a.xlarge/)、[AWS公開IPv4料金](https://aws.amazon.com/vpc/pricing/)。EC2単価は第三者の料金表による参考値であり、実際の適用単価はAWS側で確認する。

GitHubの公開鍵取得は起動時にインターネット接続を必要とする。GitHub由来の鍵の登録成否は、直接SSH接続する際に確認する。

## 練習終了時

- [ ] 変更したコードや必要な記録を保存する。
- [ ] 環境が不要なら対象のCloudFormationスタックを削除する。
- [ ] EBSやElastic IPなど、課金対象のリソースが残っていないか確認する。

EC2の停止だけでは、残っているディスクやElastic IPの料金は止まらない。

## 参照先

- [private-isu README](https://github.com/catatsuy/private-isu#ami)
- [CloudFormationテンプレート](https://gist.github.com/tohutohu/024551682a9004da286b0abd6366fa55)
- [private-isu 当日マニュアル](https://github.com/catatsuy/private-isu/blob/master/manual.md)

最終更新：2026-09-20

## Git管理・Node.jsへの切り替えとデプロイ

- サーバーの初期コードをGit管理し、初期コミットを作成。依存ライブラリとビルド成果物は除外した。
- `.git` ごとMacの `~/workspace/projects/private-isu-practice` にコピーし、`git@github.com:topi-log/private-isu-practice.git` へのpushに成功。
- 運用はPCで編集・commit・pushし、PCからrsyncでサーバーへ転送する方式。サーバーからGitHubに接続しないため、サーバー用Deploy keyは不要。
- 競技用サーバーでNode.jsのPATHを設定し、`npm run build` に成功。`sudo systemctl disable --now isu-ruby` と `sudo systemctl enable --now isu-node` で言語を切り替えた。
- Node.js初回結果（公開IP経由）：pass=true、score=1982、success=2009、fail=12。
- 再実行（公開IP経由）：pass=true、score=1869、success=1896、fail=12。
- 両実行とも `1ページに表示される画像の数が足りません (GET /posts)` が再現。原因は未調査。
- PCの練習リポジトリに `deploy.sh` を作成。Git管理対象外の `.env` に `DEPLOY_HOST` と `DEPLOY_KEY` を設定し、`./deploy.sh` でNode.jsディレクトリを転送し、サーバーで `npm ci --include=dev`、ビルド、isu-node再起動、起動状態確認を実施する。
- スクリプトのシェル構文は確認済み。ユーザーが実サーバーへの実行完了を報告済み。転送対象はwebapp/nodeのみ。PCで消したファイルはサーバーから自動削除しない。publicやNginx設定の変更はこのスクリプトの対象外。
- ビルドに失敗した場合は再起動に進まないが、転送済みファイルのロールバックは行わない。ベンチマーク中は実行しない。

## Node.jsの比較基準（プライベートIP経由・デプロイ後）

ベンチマーカー用サーバーで以下を実行する。PCからのSSH接続は公開IP、サーバー間のベンチマークはプライベートIPを使う。

```bash
cd /home/isucon/private_isu/benchmarker
./bin/benchmarker -u ./userdata -t http://192.168.1.10/
```

| 実行 | pass | score | success | fail |
|---|---|---|---|---|
| 1 | true | 1894 | 1921 | 12 |
| 2 | true | 2110 | 2147 | 13 |

両方で `1ページに表示される画像の数が足りません (GET /posts)` が再現。公開IPを使ったことだけでは、このエラーを説明できない。原因は未調査。今後はこのプライベートIP経由の結果を比較基準とする。過去の公開IP経由の測定と条件が異なるため、スコア差をそのまま改善効果としない。
