---
title: "個人アプリ「うちのコメロディー」の技術構成"
emoji: "⚒️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["firebase", "gcp", "flutter", "dart", "個人開発"]
published: false
---

:::message
この記事は[個人開発 Advent Calendar 2023](https://qiita.com/advent-calendar/2023/personal-developement) の 22 日目の記事です。
:::

以下のようなモバイルアプリをリリースしたので、技術的な話をまとめておきます。

https://apps.apple.com/us/app/%E3%81%86%E3%81%A1%E3%81%AE%E3%82%B3%E3%83%A1%E3%83%AD%E3%83%87%E3%82%A3%E3%83%BC/id6450181110

https://play.google.com/store/apps/details?id=ide.shota.colomney.MyPetMelody

# どういったアプリか？

猫の動画から、以下のようなオリジナルの音楽を作ることができるアプリです。

https://twitter.com/colomney/status/1730919744558293034

技術的には、以下のような仕組みが含まれています。

- 選択された猫の動画を解析し、鳴き声部分を自動で検知する
- BGM に鳴き声の音声を合成し、音声ファイルを生成する
- 音声ファイルと一枚の画像から、静止画の動画を生成する

# 選定の方針

音声から猫の鳴き声を検知したり、自動で動画を生成する必要があり、かつ、それらの知見がほとんどない状態でした。
そのため、それら以外はできるだけ個人的に知見のある技術を使うようにしました。

# 構成

## モバイルアプリ

Flutter を使い、iOS/Android 両方とも実装しています。

## API

アプリから呼び出す API は Cloud Functions を利用しています。

Cloud Functions for Firebase ではなく Cloud Functions を直接使っているのは、Cloud Tasks から呼び出しがしやすかったためです。

## 非同期処理

動画を生成する処理は、Cloud Tasks のタスクキューによって非同期で実行されるようにしています。

## ストレージ

Cloud Storage for Firebase を利用しています。

モバイルアプリから動画などをアップロード、ダウンロードする際は、API を介さず直接アクセスしています。
そのために、Storage のセキュリティルールにより、各ユーザーはそのユーザー専用のディレクトリ階層のみ読み書きできるように制限しています。

```firestore-security-rules:firestore.rules
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /userMedia/{userId}/pieces/{allPaths=**} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

Cloud Functions からも動画生成の際にユーザーがアップロードした動画を参照するために、アクセスしています。

## データベース

Firebase の Cloud Firestore を利用しています。

モバイルアプリで UI 表示のために参照する際やプッシュ通知のトークンを登録する際に、API を介さず直接アクセスしています。
そのために、Firestore のセキュリティルールにより、各ユーザーはそのユーザー専用のコレクション配下のみ読み書きできるように制限しています。

```firestore-security-rules:firestore.rules
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

モバイルアプリからアクセスする際は、リアルタイム更新のコネクションを張り、Firestore でデータ更新された際にモバイルアプリの UI へ即時反映しています。

Cloud Functions からも、生成された動画のメタデータを書き込んだり、プッシュ通知の送信先トークンを取得するために、アクセスしています。

## 鳴き声検知

Pydub ライブラリを使っています。

https://pydub.com/

いくつか機械学習を使ったライブラリも試してみましたが、上記の音量レベルの閾値により検知する単純なアルゴリズムが最も高い精度を発揮しました。

## サブスクリプション課金

RevenueCat を利用し、モバイルアプリ側の課金処理と、Cloud Functions 側の課金状態チェックを実装しています。

https://www.revenuecat.com/

## 認証

Firebase Authentication を使っています。

匿名認証を利用し、アカウント作成せずにアプリの機能が使えるようにしています。
また、Google アカウントと X(Twitter)アカウント、Apple でサインインが利用できるようにしています。

## プッシュ通知

Firebase Cloud Messaging を利用しています。

アプリでプッシュ通知送信用のトークンが発行されたら、Firestore にトークンを登録します。
また、Cloud Functions 上で Firestore からトークンを取得し、プッシュ通知を送信します。

## 分析

Firebase Analytics を利用しています。

デフォルト設定で自動的に収集してくれるものの他に、画面遷移のイベントを収集するようにしています。

また、アプリ中最も重要なユーザー体験の一連の流れにおいて、どの段階でユーザーが離脱したかを分析できるように、カスタムイベントを一部だけ埋め込んでいます。

## 監視

Firebase Crashlytics を利用しています。

デフォルト設定で自動的に収集してくれるものの他に、Flutter における例外発生を収集するようにしています。

## ユーザーテスト用のアプリ配布

Firebase App Distribution を利用しています。

少数の端末しかインストールする必要がないので、iOS は事前にインストールする端末の UDID を収集して Ad Hoc ビルドを利用しています。

## 単体テスト

API における、鳴き声検知の認識スコアが落ちていないかを確認する単体テストを書いています。

猫の鳴き声が含まれるサンプルの動画を 6 つほど用意し、それらの鳴き声が過不足なく検知できたかを数値化して認識スコアとしています。
この数値が初期実装時の値から変化していないかをテストとして確認します。
API の修正やライブラリのアップデートなどによりこの値が変化した場合、アプリ内の鳴き声検知機能の使い勝手に影響を与える可能性があるため、その影響を早期検知するための仕組みです。

また、モバイルアプリにおける複雑なロジックが発生する部分だけ単体テストを書いています。

モバイルアプリのロジック周りの単体テストは Flutter の標準のテストフレームワークを利用しています。

## E2E テスト

E2E テストは Maestro を利用して記載しています。

https://maestro.mobile.dev/

単純なテストケースなら YAML に操作内容を記載していくだけで済むので、非常に手軽で使い勝手が良いです。
アプリ中最も重要なユーザー体験がデグレしていないかを確認する 1 つのテストケースのみ書いています。

## ローカル開発

Firebase のローカルエミューレーターを使い、普段の開発時はローカルのみで完結するようにしています。

https://firebase.google.com/docs/emulator-suite?hl=ja

開発中における Google Cloud や Firebase の利用頻度を減らし、コストを削減することが目的です。

また、モバイルアプリをローカルエミュレーターに向けて場合、PC の LAN におけるプライベート IP アドレスをアプリに渡す必要があります。
プライベート IP アドレスは PC の環境により変化するので、リポジトリ上でのコミットを避けた方が良いです。
そのため、Flutter でビルド時に変数の記載したファイルを渡せる仕組みを使い、変数を記載したファイルを Git の管理対象外としています。

```json:dart-defines.json
{
  "SERVER_HOST": "192.168.10.13"
}
```

```bash
flutter run --dart-define-from-file "dart-defines.json"
```

## IaC

Terraform を使い、Firebase プロジェクトの作成をはじめとする以下の構築を自動化しています。

- Google Cloud プロジェクトと、それに紐づく Firebase プロジェクトの作成
- Firebase Authentication の匿名認証の有効化
- Firebase Storage の有効化とセキュリティルールの設定
- Firebase Firestore の有効化とセキュリティルールの設定
- Cloud Tasks におけるタスクキューの作成と設定

Terraform の Firebase のプロバイダー自体がまだベータ機能なので、一部手の届かない部分があります。

https://firebase.google.com/docs/projects/terraform/get-started?hl=ja

## CI/CD

GitHub Actions を利用し、以下の CI/CD を組んでいます。

- PR に対する静的解析
- PR に対する単体テスト
- PR に対してビルドができるかのチェック
- タグプッシュ時のユーザーテスト用アプリのデプロイ
- タグプッシュ時の Cloud Functions の自動デプロイ
- タグプッシュ時の一般公開用アプリのデプロイと審査提出＆公開

また、Fastlane を利用して上記の内容を 1 つのコマンドで行えるようにし、GitHub Actions 上ではそのコマンドを叩くだけにしています。
これにより、ローカル実行用と CI/CD 用で同じスクリプトを二重に書く必要がなくなります。

https://fastlane.tools/
