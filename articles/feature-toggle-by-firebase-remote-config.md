---
title: "Firebase Remote Configによるフィーチャートグルを実現する際の運用に関する概要"
emoji: "📡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["firebase", "ios", "android"]
published: false
---

# はじめに

モバイルアプリのプロジェクトで Firebase Remote Config によりフィーチャートグルを導入する機会がありました。
その際に、プロジェクトメンバーに対して運用の概要を説明するために作成したドキュメントの内容を転記したものです。

プロジェクトマネージャーや QA、エンジニアなどを想定の読者としています。

# 前提知識

## Firebase Remote Config とは

アプリのアップデートをストアから公開しなくても、アプリの動作を変更できるようにする仕組みです。

https://firebase.google.com/docs/remote-config?hl=ja

## フィーチャートグルとは

アプリをアンインストール・インストールなどにより入れ替えることなく、特定の機能の動作を変えるようにできる仕組みです。

https://ja.wikipedia.org/wiki/%E3%83%95%E3%82%A3%E3%83%BC%E3%83%81%E3%83%A3%E3%83%BC%E3%83%88%E3%82%B0%E3%83%AB

実現方法はさまざまなものがあります。
アプリ内のスイッチ UI で機能の ON/OFF を切り替えるやり方もありますし、サーバー側の設定に反応してアプリ側で機能の ON/OFF を切り替えるやり方もあります。

Firebase Remote Config によるフィーチャートグルは、後者の方法で実現するものです。

# 実現できるようになること

Firebase Remote Config によるフィーチャートグルを導入すると、以下が実現できるようになります。

- 特定の機能の ON/OFF を、アプリ内の操作やアプリのアップデート公開なしに切り替える
- 特定の機能の ON/OFF を、パーセンテージロールアウトで段階的に公開していく
- 特定の機能の ON/OFF を、特定の端末のみ ON/OFF を変更する（主に開発や検証用途）

# 導入の際の事前準備

## サーバー

Web ページから Firebase のプロジェクトを作成し、Remote Config の画面でリモート設定のキー・バリューを記述します。

![](/images/feature-toggle-by-firebase-remote-config/create-first-key-value.png)

## モバイルアプリ

Firebase Remote Config のライブラリを依存関係に追加します。

https://firebase.google.com/docs/remote-config/get-started?hl=ja

# 実装・運用の概要

1 つの機能に Firebase Remote Config によるフィーチャートグルを適用する流れは以下のようになります。

1. リモート設定のキー・バリューを設計する
2. モバイルアプリ側で、リモート設定の同期と、定義されたキー・バリューを元にした条件分岐を実装する
3. テスト環境の Firebase プロジェクト内で、Remote Config のキー・バリューを変更しながらテストする
4. モバイルアプリをストアからリリースする
5. 商用環境の Firebase プロジェクト内で、Remote Config のキー・バリューを変更して、機能を一般公開する

以下は機能公開が安定した後に、タイミングを見て行う手順です。

6. 機能の一般公開に問題ないことが確認できたら、モバイルアプリ側で条件分岐を削除して機能 ON 固定状態とした上で、モバイルアプリをストアからリリースする
7. モバイルアプリの強制アップデートなどにより、キー・バリューを利用するアプリが存在しなくなったあとに、Firebase プロジェクト内で Remote Config のキー・バリューを削除する

# 実装と運用の詳細

## サーバー

リモート設定のキー・バリューを設計し、それを Firebase Remote Config の Web 画面で設定します。

1 つのキーには条件に応じたバリューを複数設定できるため、設計時には条件とバリューのセットを考慮しておく必要があります。
モバイルアプリ側では、条件に合致したバリューが 1 つだけ取得されます。

![](/images/feature-toggle-by-firebase-remote-config/value-conditions.png)

## モバイルアプリ

モバイルアプリ側では、リモート設定の値に応じてプログラム中に分岐を書き、フィーチャートグル ON の時の挙動と OFF の時の挙動を両方定義します。

```swift
let isFeatureAEnabled = RemoteConfig.remoteConfig.configValue(forKey: "featureA").boolValue
if isFeatureAEnabled {
    // 機能A のフィーチャートグル ON の時の挙動
} else {
    // 機能A のフィーチャートグル OFF の時の挙動
}
```
