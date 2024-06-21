---
title: "Fastlaneで自作のアクションに対してテストコードを書く"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["fastlane", "ruby", "ios", "android"]
published: false
---

Fastlane とは、iOS や Android のビルド、テスト、デプロイなどのタスクを自動化するためのツールです。

https://fastlane.tools/

Fastlane には、iOS や Android をテストしてリリースするまでの様々な自動化の要望に応えてくれる標準機能が豊富に用意されています。

しかし、自分のプロジェクトに合わせた複雑なロジックを含む処理を行いたい場合もあります。

そのような場合には、処理のテストコードを書きたくなるのがエンジニアの性でしょう。

この記事では、Fastlane で自作の複雑なロジックに関してテストコードを書く方法について解説します。

## 前提条件

- Fastlane の基本的な使い方を理解していること
- Fastlane をセットアップ済みのプロジェクトがあること
- Ruby の基本的な文法を理解していること

## 複雑なロジックは Fastlane の「アクション」として定義する

Fastlane では、以下のような記述方式でレーンを定義します。

```ruby:fastlane/Fastfile
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

## 「アクション」に対してテストコードを書く

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

## テストを実行する

以下のコマンドでテストが実行できます。

```sh
bundle exec rspec -format d
```

以下のようにテスト結果が得られます。

```sh

```
