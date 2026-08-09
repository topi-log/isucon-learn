# 01-nginx: ISUCON のための nginx 入門

nginx をまったく知らない状態から、ISUCON で nginx を触れるようになるまでを
phase0 〜 phase5 の 6 段階で学ぶドキュメントです。
前のフェーズの知識を次のフェーズが使うので、順番に読んでください。

## フェーズ一覧

| フェーズ | ファイル | 内容 |
|---------|---------|------|
| phase0 | [前提知識](phase0.md) | Web サーバーとは何か、HTTP・ポートの基本、ISUCON の構成の中での nginx の位置 |
| phase1 | [nginx に触れる](phase1.md) | 設定ファイルの場所、起動・停止・reload、nginx -t、ログの場所 |
| phase2 | [設定ファイルを読む](phase2.md) | server / location ブロック、マッチの優先順位、よく見るディレクティブ |
| phase3 | [リバースプロキシ](phase3.md) | proxy_pass、proxy_set_header、upstream、keepalive、unix socket |
| phase4 | [アクセスログで計測](phase4.md) | ログを JSON 化して alp で集計し、遅いエンドポイントを見つける |
| phase5 | [実戦チューニング](phase5.md) | 静的ファイル配信、gzip、本体設定、複数台構成、当日チェックリスト |

## 読み方の目安

- phase0 〜 2 は読むだけでも身につきます。手元に環境があれば実際にファイルを開いてみてください
- phase3 〜 5 は ISUCON の点数に直結する部分です。過去問環境（ISUCON14 など）で
  実際に設定を変えて、ベンチマークを回して確かめるのがいちばん効きます

## このドキュメントのゴール

- 初期状態の nginx 設定を読んで「何をしているか」説明できる
- 設定を変更して、壊さずに反映できる（nginx -t → reload の型）
- アクセスログから遅いエンドポイントを特定できる
- 定番のチューニング（静的ファイル配信・keepalive など）を自分で入れられる

より実戦寄りの内容は [02-performance/06-middleware-nginx.md](../old/02-performance/06-middleware-nginx.md) と
[02-performance/01-measurement.md](../old/02-performance/01-measurement.md) に続きます。
