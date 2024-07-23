---
title: "トランクベース開発を実際のプロジェクトに導入する"
emoji: "⛴️"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["ios", "android"]
published: false
---

# 運用上必要になってくること

## フィーチャーフラグやビルドツールの活用によりマージとリリースを分離する

開発中の中途半端な機能もフィーチャーフラグを用いて一般には公開されない状態にしつつ、リリースできる状態とする。

フィーチャートグルは 1 つの手段であり、比較的リッチな方法なので、その他の手段があればそちらを採用するようにします。

- サーバー側のデプロイにより発動する。本番環境ではデプロイをしないようにする
- 本番環境アプリでは設定項目を非表示にする

やっぱりこの機能まだリリースしたくないとなった場合は、トランクブランチに対して、フィーチャーフラグを用いてリリース対象外とする修正を適用します。
その後、リリースブランチにチェリーピックする。

## 作ったものはすぐにリリースしましょうという方向で握る

マージとリリースの分離は必ずできるものでないため、別の方法で成り立つようにする必要があります。

「次のリリースにはどのバグ修正を含めますか？」というコミュニケーションではなく、「次のリリースにはこれらのバグ修正が含まれる予定です」というコミュニケーションになってきます。

## 開発単位をできるだけ小さく保つ

チェリーピックする際にコンフリクトが発生する可能性を下げる。
複数のコミットが同時多発的に一本のブランチにマージされていくので、何か問題が発生した時に、切り分けができるように。

チケットを作成する際に、チケットを細分化し、結果的に PR が小さくなるようにしています。
例えば、新しい画面を作成する際、以下のように分割します。

## トランクブランチのいつでもある程度致命的なバグが存在しない状態の水準にする

### デグレを検知した場合は、早急に対応する

1 つのチケットを 1 つの PR で対応できればいいが、そうできない場合も発生する。
その際は、一定期間(2 週間程度)より長く滞留してしまったら、以下のいずれかの方針をとる。

1. 対応した PR により機能のデグレーションが発生している
   - 一旦トランクブランチ上でリバートし、再着手時に再度 PR を立てる
2. 対応した PR により機能のデグレーションが発生していないものの、チケットに記載しているバグが完全には解消していない
   - PR をリリースする。チケットに`記載時点までのPRをリリース可能`という日付フィールドに、判断した日付を入力する。
3. 対応した PR により機能のデグレーションは発生しておらず、チケットに記載しているバグが解消している一方で、別のバグが見つかっている
   - チケットを完了としてリリースまで持っていき、別のバグは別チケットで管理する。

品質を担保した後にマージというフェーズが取らないため。

## リリース前のイベントをできるだけ短くする

テスト期間が長すぎると適していないと考えられる。

## リリース担当者を立てる

開発内容もわかりつつ、リリース作業を行う人が必要。

チェリーピックなどの作業が発生するため、Git の操作や概念に慣れている人が望ましい。

## Squash マージを強制し、PR1 つに対し 1 つのコミットが生成されるようにする

GitHub の設定を変更し、Squash マージとする。
これは、最終的にメインブランチに PR のタイトルがコミットメッセージとされたコミット履歴を残し、コミットごとにリリース計画を立てやすくするため。
また、PR ごとの細かいコミットに気を使うのはコストもかかるし、自然とできるメンバーも少ないので、細かいコミットを捨てるため。