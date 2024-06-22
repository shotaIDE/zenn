---
title: "Fastlaneに記述しているロジックに対してテストコードを書く"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["fastlane", "rspec", "ruby", "ios", "android"]
published: false
---

<!-- cspell:ignore gsub, testflight -->

Fastlane とは、iOS や Android アプリのビルド、テスト、デプロイなどのタスクを自動化するためのツールです。

https://fastlane.tools/

Fastlane には、iOS や Android アプリ をテストしてリリースするまでに必要な様々な自動化の要望に応えてくれる標準機能が豊富に用意されています。

しかし、自分のプロジェクトに合わせた複雑なロジックを含む処理を行いたい場合もあります。

そのような場合には、処理のテストコードを書きたくなるのがエンジニアの性でしょう。

この記事では、Fastlane で自作の複雑なロジックに関してテストコードを書く方法について解説します。

## 前提条件

- Fastlane の基本的な使い方を理解していること
- Fastlane をセットアップ済みのプロジェクトがあること
- Ruby の基本的な文法を理解していること

## 複雑なロジックは Fastlane の「アクション」として定義する

Fastlane では、1 つのタスクを「レーン」と呼ばれる単位で定義します。

```ruby:fastlane/Fastfile
lane :send_results_to_slack do
  slack(
    message: "Build Succeeded!",
    success: true
  )
end
```

レーンの定義は、`Fastfile`というファイル名に記載し、この中では Ruby における DSL（Domain Specific Language）を使用して記述していきます。

DSL とは、特定のドメイン（分野）に特化した言語のことです。

Fastlane では、Ruby の言語上で動作する DSL を使用して、iOS や Android アプリのビルドやデプロイなどのタスクを記述します。

これは、ビルドやデプロイなど、単純な手続的なタスクを記述することに適していると言えます。

そのため、ある程度のロジックを組む際には、自作のアクションを作成し、Ruby の言語で記述することが適しています。

自作のアクションを作成するには、`fastlane new_action` コマンドを使用します。

https://docs.fastlane.tools/create-action/

```shell
fastlane new_action
```

:::message
Fastlane をディレクトリの Gemfile で管理している場合、`bundle exec fastlane new_action` として実行する必要があります。
:::

以下のようにアクションの名前を入力するように求められます。
例として、`escape_for_slack_message` という名前を入力します。

```log
Must be lowercase, and use a '_' between words. Do not use '.'
examples: 'testflight', 'upload_to_s3'
[11:15:55]: Name of your action:
```

そうすると、以下のような出力とともに `./fastlane/actions/escape_for_slack_message.rb` というファイルが作成されます。

```log
[11:17:04]: Created new action file './fastlane/actions/escape_for_slack_message.rb'. Edit it to implement your custom action.
```

このファイルに、自作のアクションを記述します。
例えば、以下のようなアクションを作成できます。

```ruby:fastlane/actions/escape_for_slack_message.rb
module Fastlane
  module Actions
    module SharedValues
      ESCAPE_FOR_SLACK_MESSAGE_ESCAPED_VALUE = :ESCAPE_FOR_SLACK_MESSAGE_ESCAPED_VALUE
    end

    class EscapeForSlackMessageAction < Action
      def self.run(params)
        text = params[:text]

        # See https://api.slack.com/reference/surfaces/formatting#escaping
        text
          .gsub(/&/, '&amp;')
          .gsub(/</, '&lt;')
          .gsub(/>/, '&gt;')
      end

      #####################################################
      # @!group Documentation
      #####################################################

      def self.description
        'Escape string for sending Slack message by incoming webhooks'
      end

      def self.details
        'You can use this action to escape "&", "<" and ">" into HTML entities.'\
        'They are used control characters in Slack message.'
      end

      def self.available_options
        [
          FastlaneCore::ConfigItem.new(
            key: :text,
            env_name: 'ESCAPE_FOR_SLACK_MESSAGE_TEXT',
            description: 'Text to be escaped',
            type: String,
            optional: false
          )
        ]
      end

      def self.output
        [
          ['ESCAPE_FOR_SLACK_MESSAGE_ESCAPED_VALUE', 'Escaped text']
        ]
      end

      def self.return_value; end

      def self.authors
        ['colomney']
      end

      def self.is_supported?(platform)
        %i[ios mac].include?(platform)
      end
    end
  end
end
```

ほとんどはドキュメントのための記述です。
実際のロジックとしては、`self.run` のみが重要です。

ここでは例として、Slack に Incoming Webhook でメッセージを送信する際に、`&`, `<`, `>` といった文字をエスケープするアクションを作成しています。

## 「アクション」に対して RSpec でテストコードを書く

作成したアクションに対してテストコードを書くには、RSpec を使用します。

https://rspec.info/

RSpec をインストールします。

```shell
gem install rspec
```

:::message
Fastlane をディレクトリの Gemfile で管理している場合、`Gemfile` に `gem 'rspec'` を追記し、`bundle install` でインストールします。
:::

`spec/` ディレクトリを作成し、その中にテストコードを記述します。

```ruby:spec/escape_for_slack_message_spec.rb
require 'fastlane/action'
require './fastlane/actions/escape_for_slack_message'

describe Fastlane::Actions::EscapeForSlackMessageAction do
  let(:action) { Fastlane::Actions::EscapeForSlackMessageAction }

  describe '#run' do
    it '"&"はエスケープされる' do
      expect(action.run(text: 'A & B')).to eq('A &amp; B')
    end

    it '通常の文字はエスケープされない' do
      expect(action.run(text: 'A B')).to eq('A B')
    end
  end
end
```

## テストを実行する

以下のコマンドでテストが実行できます。

```shell
rspec
```

:::message
Fastlane をディレクトリの Gemfile で管理している場合、`bundle exec rspec` として実行する必要があります。
:::

以下のようにテスト結果が得られます。

```log
Fastlane::Actions::EscapeForSlackMessageAction
  #run
    "&"はエスケープされる
    通常の文字はエスケープされない

Finished in 0.00243 seconds (files took 0.22909 seconds to load)
2 examples, 0 failures
```
