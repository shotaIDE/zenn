---
title: "fastlaneで組んでいるロジックが複雑になってきたのでテストコードを書きたい"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["fastlane", "rspec", "ruby", "ios", "android"]
publication_name: "sun_asterisk"
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

# 実践方法

## 複雑なロジックをどこに書くべきか

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

`Fastfile` 中では、Ruby 上で動作する専用のドメイン固有言語（Domain Specific Language、以下 DSL と表記）を使用して記述していきます。
これにより、ビルドやデプロイタスクを簡単に記述できるようになっています。

:::message
ドメイン固有言語とは、特定の領域の内容を記述するのに特化した言語のことです。
:::

ただし、あくまでビルドやデプロイなどのタスクを記述するための DSL です。
そのため、複雑なロジックを記述するには、**DSL ではなく Ruby で直接記述し、Ruby そのものに付帯するテストやデバッグのフレームワークを活用する**方が適していると考えられます。

fastlane のビルドタスクで利用するロジックを Ruby で記述する場合、以下の 2 つの方法が取れます。

1. 自作の「アクション」を作り、その中にロジックを記述する
2. 自作の「プラグイン」を作り、その中にロジックを記述する

本記事では、比較的簡単にロジックを組むことができる 1 の方法を取り上げます。

:::message
2 の方法は、より大規模なロジックを組むには適していると考えられます。
やり方については、以下ドキュメントを参照してください。

https://docs.fastlane.tools/plugins/create-plugin/
:::

## 複雑なロジックを fastlane のアクションとして定義する

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
例えば、以下のようなアクションを作成してみます。

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

ほとんどはドキュメントコメントの記述であり、**実際のロジックは `self.run` の部分**です。

ここでは例として、Slack に Incoming Webhook でメッセージを送信する際に、`&`, `<`, `>` といった文字をエスケープするアクションを作成しています。

## アクションに対して RSpec でテストコードを書く

作成したアクションに対してテストコードを書くには、RSpec を使用します。

RSpec とは Ruby におけるテストフレームワークの 1 つです。

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
