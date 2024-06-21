---
title: "Fastlaneで自作のアクションに対してテストコードを書く"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["fastlane", "ruby", "ios", "android"]
published: false
---

Fastlane とは、iOS や Android のビルド、テスト、デプロイなどのタスクを自動化するためのツールです。

Fastlane には、標準で提供されているアクションがありますが、自作のアクションを作成できます。

自作のアクションに対してテストコードを書くことで、アクションの品質を保つことができます。

この記事では、Fastlane で自作のアクションに対してテストコードを書く方法について解説します。

## 前提条件

- Fastlane の基本的な使い方を理解していること
- Ruby の基本的な文法を理解していること

## 自作のアクションを作成する

Fastlane では、以下のような記述方式でレーンを定義します。

```ruby
lane :my_lane do
  # ここにタスクを記述
end
```

レーンの定義には、Ruby における DSL（Domain Specific Language）を使用しています。

これは、ビルドやデプロイなど、単純な手続的なタスクを記述することに適していると言えます。

そのため、ある程度のロジックを組む際には、自作のアクションを作成し、Ruby の言語で記述することが適しています。

自作のアクションを作成するには、`fastlane new_action` コマンドを使用します。

```sh
fastlane new_action
```

そうすると、以下のようなファイルが作成されます。

```sh
./fastlane/actions/my_custom_action.rb
```

このファイルに、自作のアクションを記述します。
例えば、以下のようなアクションを作成できます。

```ruby

```

## テストコードを書く

自作のアクションに対してテストコードを書くには、Rspec を使用します。

まず、`Gemfile` に `rspec` を追加します。

```ruby:Gemfile
group :test do
  gem 'rspec'
end
```

以下のコマンドで、`rspec` をインストールします。

```sh
bundle install
```

`spec/` ディレクトリを作成し、その中にテストコードを記述します。

```ruby:spec/test.rb

```

以下のコマンドでテストが実行できます。

```sh
bundle exec rspec -format d
```

以下のようにテスト結果が得られます。

```sh

```
