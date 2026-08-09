# ISUCON2026 学習ドキュメント

ISUCON2026 参加に向けた学習ドキュメント集です。
**番号順に読む**ことを前提に構成しています（前の章の知識を後の章が使います）。

- 言語: コード例は **TypeScript (Node.js)**、DB は **MySQL** を前提
- 閲覧方法: VS Code なら Markdown ファイルを開いて `Ctrl+Shift+V` でプレビュー表示できます
- **用語集**: 各ドキュメントの末尾に、そのドキュメントに出てくる専門用語の解説（用語集）があります
- **自分用メモ**: 各ドキュメントと対になる `〜-terms.md` は自分用のメモ帳です。読んでいて分からなかった単語や調べたことを書き足して、読みながら育てていってください（本文末尾からリンク）

## 読む順番

### Part 1: アプリケーションのベース — `01-application-base/`

ISUCON という競技の理解と、チューニング対象となる Web アプリケーションの基礎構造。

| # | ファイル | 内容 |
|---|---------|------|
| 1-1 | [ISUCONとは何か](01-application-base/01-isucon-overview.md) | 競技ルール・スコアの仕組み・当日の流れ |
| 1-2 | [Webアプリケーションの構成](01-application-base/02-web-architecture.md) | nginx → app → MySQL、リクエストの一生、ボトルネックの考え方 |
| 1-3 | [Node.jsバックエンドの基礎](01-application-base/03-nodejs-basics.md) | イベントループ、async/await、参考実装の読み方 |
| 1-4 | [MySQLの基礎](01-application-base/04-mysql-basics.md) | コネクションプール、トランザクション、ロック |
| 1-5 | [Linuxサーバー操作の基礎](01-application-base/05-linux-server-basics.md) | systemd、journalctl、リソース確認 |

### Part 2: パフォーマンス向上の知識 — `02-performance/`

ISUCON の本体。**「計測 → ボトルネック特定 → 改善」のループ**を部分ごとに学ぶ。

| # | ファイル | 内容 |
|---|---------|------|
| 2-1 | [計測ファースト](02-performance/01-measurement.md) | alp、pt-query-digest、top。推測せず計測する |
| 2-2 | [EXPLAINとインデックス設計](02-performance/02-sql-explain-index.md) | ★最重要。B+Tree、複合インデックス、EXPLAINの読み方 |
| 2-3 | [N+1問題](02-performance/03-n-plus-one.md) | 見つけ方と3つの解消パターン（Before/Afterコード付き） |
| 2-4 | [DBチューニング](02-performance/04-db-tuning.md) | バルクINSERT、スキーマ改善、MySQL設定 |
| 2-5 | [アプリケーション最適化](02-performance/05-app-optimization.md) | オンメモリキャッシュ、非同期化、外部API |
| 2-6 | [ミドルウェア (nginx)](02-performance/06-middleware-nginx.md) | 静的ファイル配信、keepalive、unix socket |
| 2-7 | [複数台構成](02-performance/07-multi-server.md) | 3台の使い方、DB分離、役割分担 |

### Part 3: ISUCON14 (ISURIDE) 分析 — `03-isucon14/`

直近の過去問を題材に、Part 1・2 の知識を実戦に接続する。

| # | ファイル | 内容 |
|---|---------|------|
| 3-1 | [問題概要](03-isucon14/01-problem-overview.md) | ISURIDE の仕様・スコア計算・満足度の仕組み |
| 3-2 | [ボトルネック分析](03-isucon14/02-bottlenecks.md) | 初期実装のどこが遅いのか |
| 3-3 | [改善方法案](03-isucon14/03-improvements.md) | 公式講評ベースの具体的な改善リスト |
| 3-4 | [練習ガイド](03-isucon14/04-practice-guide.md) | 環境構築と素振りの進め方 |

### チームドキュメント — `by-sora/`

チームメンバー sora さん作成のドキュメント（GitHub Issue から転記）。番号順に読む。

| # | ファイル | 内容 |
|---|---------|------|
| s-1 | [事前学習Overview](by-sora/01-pre-study-overview.md)（[要約版](by-sora/01-pre-study-overview-summary.md)） | ISUCONで学ぶべき知識の全体像・7領域・学習フェーズ（[元Issue #7](https://github.com/topi-log/isucon2026-team-sakaguchi/issues/7)） |

## 学習ロードマップの目安

| 期間 | やること |
|------|---------|
| 1週目 | Part 1 を通読。手元に MySQL を立てて 1-4 のSQLを実際に叩く |
| 2〜3週目 | Part 2 を1章ずつ。特に 2-2 (EXPLAIN) と 2-3 (N+1) は手を動かして身につける |
| 4週目 | Part 3 を読み、ISUCON14 の初期実装コードを読んでボトルネックを自分で探してみる |
| 以降 | private-isu や ISUCON14 環境で素振り（3-4 参照）。「計測→改善→再計測」を体で覚える |

## ISUCON で一番大事な原則（先に言っておく）

> **推測するな、計測せよ。**

ISUCON で伸び悩む最大の原因は「たぶんここが遅い」で手を動かすこと。
本ドキュメントも一貫して「まず計測、次に改善」の順で書いています。
