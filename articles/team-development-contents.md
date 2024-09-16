---
title: "チーム開発をうまくやるための内容"
emoji: "🕌"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["ios", "android"]
published: false
---

アジリティを高めるために以下のような施策を実施しました。

- 内部品質向上
  - PR 提出時に、静的解析の指摘が増えないことを自動チェック
  - PR 提出時に、デッドコードが存在しないことを自動チェック
  - PR 提出時に、Typo が増えないことを自動チェック
- E2E テストの導入
- ワークフロー効率化
  - PR 提出時に、正しい Jira への紐付けを自動チェック
  - ライブラリ更新の PR を自動で作成
  - スプリント開発用のビルドのデプロイ自動化
  - リリース用のビルドのデプロイ半自動化（トランクベース開発を考慮）
- リスク低減
  - 各種ツールで利用する Ruby のバージョンを厳密に固定
- デバッグ効率
  - E2E 自動テスト実行時の画面収録を蓄積
- ドキュメンテーション
  - E2E 自動テストにおけるテストケースと確認観点のドキュメントデプロイを自動化
- アプリのメトリクス可視化
  - 内部品質の定量的な指標の可視化
  - 開発プロセスのメトリクスの可視化
  - 主要な機能のパフォーマンス可視化

## 安全にプロジェクトを進めるための新しい取り組み

- アプリのメトリクスに基づく意思決定
  - Android プッシュ通知の EOL 対応
- 定量的に測れる内部品質が悪化しないようにするための CI
  - 静的解析による指摘事項が増えないようにする
  - デッドコードが増えないようにする
- アプリが依存するライブラリのライセンス情報が変化したことを検知するためのツールの作成
  - ライセンス違反に気づきやすくなった

## 効率よくプロジェクトを進めるための新しい取り組み

- Android におけるライブラリ更新の手動作業を削減
  - 週次で 0.5pt 程度の作業がなくなった
- E2E テストの導入開始
  - 開発メンバーによる自律的な E2E テストの追加とメンテナンス、精度向上の取り組み。開発メンバーが外部品質にオーナーシップをより持っていくことに繋げられそう
- 開発プロセスやコード品質の定量的な測定と観察
  - コード品質が順調に維持・向上しているかを確認できるようになった
  - 開発メンバーがコード品質のメトリクス(単体テストのコードカバレッジ)を意識する場面が見られた
- 開発やテストを効率化するための仕組み
  - プロキシーツールに関するドキュメンテーションと動作確認・テストへの導入
  - Android の Web ソケットにおけるプロキシーでの可視化機能の導入

## その他

- トランクベース開発の効果、つらさ
  - 効果
    - 普段の開発作業における複雑なブランチ管理が不要、デプロイ手順の容易さ
    - リグレッションテストの手間を最小にできる
  - つらさ
    - トランクブランチの状態を常に一定以上の品質を保たないと、つらい

## E2E テストについて

### E2E テストを運用することで、デグレを検知できる

プロダクトコード側で、選択肢ダイアログ表示中に別のダイアログが上に表示されてようになってしまい、選択肢ダイアログの操作がしづらくなりました。
つまり、操作性が悪くなるデグレでした。
E2E テストが失敗するようになったので、この事象を検知し修正できた。

### E2E テストを実装することで間接的にプロダクトコードの品質向上に寄与している

テストの実行結果が不安定となる場合があり、それらはプロダクトコード側の品質が悪いことに原因があります。

- ネットワーク速度が遅いあるいは端末のスペックが低い場合に、正しい順番で処理が行われない
- プログラム中の準正常系、異常系のパターンが漏れている

### 開発が立ち上がるまでのコストがそれなりに大きい

- CI 環境で、ネットワーク速度が遅かったり端末の動作が遅かったりする場合にもロバストに動くテストコードの書き方
- CI 環境で E2E テストを動かすための環境構築

### メンテナンスのための開発者の作業コストが増える

- テストが失敗した際の調査
- 日々の機能追加や修正による、E2E テストコードの修正

### 自動実行のための CI 環境の経済的コストがかかる

- マシンにある程度のスペックが求められるため、静的解析や単体テストを回すスペックよりも良いマシンスペックを選択したり、専用の SaaS を利用した方がいい