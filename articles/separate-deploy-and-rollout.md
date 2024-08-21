---
title: "デプロイとロールアウトを独立させた方がいいですよ"
emoji: "🛫"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["ios", "android"]
published: false
---

# 前提

トランクベース開発では、短いサイクルでトランクブランチにマージしていきます。
そのため、開発途中のコードもトランクブランチにマージされていくことになります。

こうした状況で、トランクブランチをリリース可能な状態にするには、フィーチャートグルの導入がほぼ必須です。

マージとデプロイの分離が必須です。

その 1 つの手段としてフィーチャートグルがありますが、その他にも手段があります。
本記事では、マージとデプロイの手段としてのフィーチャートグルを導入している前提で解説します。

# TL;DR

デプロイとロールアウトを独立させた方がいいですよ。

デプロイとロールアウトを独立させると、リリース時のリスクへの対策になります。
また、開発者や QA の負担を減らすことができます。

デプロイとロールアウトを独立させるということは、開発プロセスの早い段階でロールアウトのことを考える必要があるということです。

デプロイを開発プロセスの早い段階で考慮することで、デプロイの際に問題が発生することを防ぐことができます。

運用と開発が分離することを防ぐことができます。

# マージとデプロイを独立させるとは？

マージされたコードがユーザーの元で動作しても、機能が発動しないということです。

その後、デプロイの作業をすることで、機能が発動するようになります。

# 開発プロセスの早い段階でデプロイのことを考える必要がある

マージされても即時ユーザーへリリースされるという状態にはしません。
そのため、独自でデプロイの方法を考える必要があります。

例 1

新機能 A については、サーバー側で特定のファイルがデプロイされることで、クライアント側でも新機を利用可能になる状態としましょう。
そうすると、クライアント側はサーバー側のファイルがデプロイされない限り一般ユーザーに対して機能は利用可能にはならないですね。

例 2

# デプロイの際に問題が発生することを防ぐことができる

（この例ってあるかな？）

# 運用と開発が分離することを防ぐことができる

DevOps の考え方に近い。開発と運用のチームやメンバーが分離することによるサイロ化を防ぐことができる。

# 各種機能の取り組む際への難しさ

特定のバージョンからの新機能を案内するための機能を作成する場合、その機能は特定のバージョンからのみ利用可能である必要があります。

その機能をフィーチャートグルで実現する場合、より複雑なロジックが必要です。

最終的には以下のようなロジックが必要になります。

- 「機能を含まない旧アプリ」から「機能を含む新アプリ」への上書きアップデート
- 「機能を含む新アプリ」の新規インストール
- 「機能を含む新アプリ」から「機能を含む新アプリの未来バージョン」への上書きアップデート

一方でフィーチャートグルの分岐がある最中は、以下のパターンで試す必要があります。

- フィーチャートグル OFF から ON に変更した
- フィーチャートグル OFF の場合

# 参考

https://logmi.jp/tech/articles/329504