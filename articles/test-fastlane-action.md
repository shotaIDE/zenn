---
title: "fastlaneに記述しているロジックに対してテストコードを書く"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["fastlane", "rspec", "ruby", "ios", "android"]
published: false
---

<!-- cspell:ignore gsub, testflight -->

fastlane とは、iOS や Android アプリのビルド、テスト、デプロイなどのタスクを自動化するためのツールです。

https://fastlane.tools/

fastlane には、iOS や Android アプリ をテストしてリリースするまでに必要な自動化タスクが標準で豊富に用意されています。

しかし、各プロジェクトの業務フローに合わせた**複雑なロジックを含む自動化タスクをカスタムで組みたい**シーンもあります。

そのような場合には、**複雑なロジックはテストコードを書いて武装しないと不安になる**こともあるでしょう。

本記事では、fastlane で自作の複雑なロジックに関してテストコードを書く方法について解説します。

# 想定する読者

fastlane の基本的な使い方を理解していることを前提として進めます。

# 前提など

fastlane をセットアップ済みのプロジェクトがある前提で進めます。

また、本記事で出てくる fastlane 関連のコマンドは、fastlane を Bundler により組み込んでいることを前提としています。

詳しい情報は、以下ドキュメントにおける "Managed Ruby environment + Bundler (macOS/Linux/Windows)" の部分を参照してください。

https://docs.fastlane.tools/

# 実施方法

## 複雑なロジックは fastlane の「アクション」として定義する

fastlane では、1 つのタスクを「レーン」と呼ばれる単位で定義します。

```ruby:fastlane/Fastfile
lane :send_results_to_slack do
  slack(
    message: "Build Succeeded!",
    success: true
  )
end
```

レーンの定義は、`Fastfile`というファイル名に記載します。

`Fastfile` 中では、Ruby におけるドメイン固有言語（Domain Specific Language ＝ DSL）を使用して記述していきます。

DSL とは、特定の領域の内容を記述するのに特化した言語のことです。

fastlane では、Ruby の言語上で動作する DSL を使用して、iOS や Android アプリのビルドやデプロイなどのタスクを記述します。

これは、ビルドやデプロイなど、単純な手続的なタスクを記述することに適していると言えます。

そのため、ある程度のロジックを組む際には、自作のアクションを作成し、Ruby の言語で記述することが適しています。

自作のアクションを作成するには、`fastlane new_action` コマンドを使用します。

https://docs.fastlane.tools/create-action/

```shell
bundle exec fastlane new_action
```

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

`Gemfile` に RSpec を追加します。

```diff ruby:Gemfile
source 'https://rubygems.org'

gem 'fastlane'
+gem 'rspec'
# Other dependencies...
```

以下コマンドでインストールします。

```shell
bundle install
```

`spec/escape_for_slack_message_spec.rb` という名前でファイルを作成し、その中にテストコードを記述します。

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
bundle exec rspec
```

以下のようにテスト結果が得られます。

```log
Fastlane::Actions::EscapeForSlackMessageAction
  #run
    "&"はエスケープされる
    通常の文字はエスケープされない

Finished in 0.00243 seconds (files took 0.22909 seconds to load)
2 examples, 0 failures
```
