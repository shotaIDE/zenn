---
title: "Flutterライブラリの更新地獄から脱出！自動化でスマートなFlutter開発を！"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "renovate", "ios", "android"]
publication_name: "sun_asterisk"
published: true
---

<!-- cspell:ignore automerge, noreply, podfile, precache, subosito, temurin -->

# はじめに

アプリ開発で、運用やメンテナンスにかかるコストをできるだけ削減することは重要です。

特にライブラリの更新は、アプリの品質を保つために欠かせない作業の一方で、その更新作業自体に一定のコストがかかります。
さらに更新作業はこまめに実施しないと、一気に多くのライブラリの更新が必要になり、更新作業自体が困難になることもあります。

本記事では、Flutter アプリでライブラリの更新を自動化し、これらの負担やリスクを軽減する方法について紹介します。

# 前提

GitHub で開発し、GitHub Actions で CI を組んでいる前提で説明します。

ライブラリの更新による影響は CI で検証され、**CI が通れば安全にライブラリ更新の PR がマージできるという前提**に立ちます。
ライブラリ更新の影響を確認する CI 自体の組み方は、本記事では記載しません。

:::message
プロジェクトの品質基準によっては、ライブラリの更新による影響を手動で検証する必要があるという状況も考えられます。
その場合は、本記事の手法は使えません。
:::

ライブラリの更新 PR は、**1 ライブラリにつき 1PR** としています。
ただし、Renovate の設定により、複数ライブラリを 1PR にまとめることも可能です。

また、本記事で前提としているのはアプリ開発です。
そのため、**利用しているライブラリの新しいバージョンがリリースされたら、その新しいバージョンのみを利用するようバージョン指定の制約を更新**します。

:::message
一方ライブラリ自体の開発では、バージョン制約は広く取り、制約の更新自体も緩やかに行うことが一般的です。
これは、他の人や他のチームが定義する様々なバージョン制約の中で利用可能にするためです。
:::

# 概要

**Renovate** を導入し、PR 作成を自動化します。
これにより、**Flutter ライブラリのバージョン制約定義ファイル `pubspec.yaml` とロックファイル `pubspec.lock` が更新された PR が作成**されます。

https://docs.renovatebot.com/

PR 作成後に発火するワークフローを定義します。
そのワークフローで Flutter のライブラリが**間接的に利用する iOS ネイティブライブラリに関するロックファイル `Podfile.lock` を更新**し、プッシュバックします。

Renovate の自動マージ設定を有効化し、**各種 CI のチェックが通った後自動マージ**されるようにしておきます。

:::message
Flutter のライブラリは iOS ネイティブライブラリを利用するものがあります。
そのため、Flutter ライブラリ更新時に、それらの間接的な依存関係も同時に更新する必要があります。
このような間接的な依存関係の更新は、Renovate では現状対応していないため、自前でワークフローを定義する必要があります。
:::

# 詳細

## 1. Flutter のライブラリ依存関係の定義方法を変更

`pubspec.yaml` におけるライブラリのバージョン指定は、キャレット演算子 `^` を使うことが一般的です。
これはライブラリの公式サイトやドキュメントで標準の書き方として記載されているためです。

しかしこの状態だと、ライブラリの新しいバージョンがリリースされた際に、Renovate がバージョン制約を変更しなくて良いと判断することが多く、更新 PR があまり提出されなくなります。

本記事では、新しいバージョンのライブラリがリリースされたら、すぐに最新版のみを利用するようバージョン指定の制約に更新したいです。

そこで、**`pubspec.yaml` に記載されているライブラリのバージョン制約を、絶対指定に変更します**。

```diff yaml:pubspec.yaml
name: app_name
description: App description.
publish_to: "none"
version: 1.0.0+1

environment:
  sdk: ">=2.18.5 <3.0.0"
  flutter: ">=3.19.5"

dependencies:
-  cloud_firestore: ^4.15.10
+  cloud_firestore: 4.15.10
-  collection: ^1.18.0
+  collection: 1.18.0
# ...
```

:::message
バージョン制約の絶対指定への変更は、Renovate の設定における `rangeStrategy` を `pin` に指定することによっても対応可能です。
指定すると、絶対指定へ修正する PR を自動で立ててくれます。
https://docs.renovatebot.com/configuration-options/#rangestrategy
:::

これにより、Renovate が新しいバージョンのライブラリがリリースされた際に、バージョン制約を以下のように変更した PR を作成してくれるようになります。

```diff yaml:pubspec.yaml
name: app_name
description: App description.
publish_to: "none"
version: 1.0.0+1

environment:
  sdk: ">=2.18.5 <3.0.0"
  flutter: ">=3.19.5"

dependencies:
-  cloud_firestore: 4.15.10
+  cloud_firestore: 4.15.11
# ...
```

## 2. Renovate の導入

Renovate は、GitHub のリポジトリにプルリクエストを作成し、ライブラリの更新を自動化するツールです。

以下の "Hosted GitHub.com App" を参考にして、GitHub App として該当のリポジトリに対してインストールします。

https://docs.renovatebot.com/getting-started/installing-onboarding/#hosted-githubcom-app

## 3. Renovate の設定

Renovate をリポジトリにインストールしたら、ライブラリ自動更新の推奨設定が適用された PR が自動で作成されます。

![](/images/automatically-upgrade-flutter-dependencies/renovate-configure-pr.png)

この PR は**そのままマージせず、目的に適した設定へと修正**します。

以下が修正後の設定ファイルの内容です。

```json:renovate.json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "enabledManagers": ["pub"],
  "packageRules": [
    {
      "matchManagers": ["pub"],
      "automerge": true
    }
  ],
  "prConcurrentLimit": 4
}
```

Flutter のライブラリは pub というパッケージマネージャーを通じて管理されているため、`enabledManagers` に `pub` を指定します。

`prConcurrentLimit` は、同時に提出される PR の数を制限するための設定です。
PR が同時に大量に発生してノイズにならないように、適切な数に設定すると良いです。

このように修正後、PR をマージします。

Renovate の設定ファイルに関する詳細は以下のドキュメントを参照してください。

https://docs.renovatebot.com/configuration-options/

## 4. GitHub Actions でプッシュバックする際のアクセストークンを用意する

後述のステップで、iOS ネイティブのライブラリのロックファイルを更新し、プッシュバックし、プッシュバックをトリガーとして CI が実行されるようにします。

この際、GitHub Actions で**デフォルトで利用できるアクセストークンの `GITHUB_TOKEN` によりプッシュバックすると、CI がトリガーされません**。
これは、意図せず CI が大量に動作して経済的な損失が発生しないようにするための、GitHub の仕様です。

https://docs.github.com/ja/actions/using-workflows/triggering-a-workflow#triggering-a-workflow-from-a-workflow

これを解消するためには、以下の 3 種類の方法があります。

1. GitHub Apps によるトークンを使う
2. 個人トークン(Fine-grained)を使う
3. 個人トークン(クラシック)を使う

チーム開発の運用やセキュリティ的な観点では 1 の GitHub Apps によるトークンを使う方法が良いですが、今回は一番簡単な 3 の個人トークン(クラシック)を使う方法をご紹介します。

個人トークンのページを開きます。

https://github.com/settings/tokens

"Generate new token (classic)" をクリックします。

![](/images/automatically-upgrade-flutter-dependencies/create-personal-access-token.png)

"Note" に任意の名前を入力し、"Expiration" を好きな値に設定します。
また、"Select scopes" における "repo" のスコープにチェックを入れます。

![](/images/automatically-upgrade-flutter-dependencies/personal-access-token-settings.png)

作成したトークンをコピーします。

![](/images/automatically-upgrade-flutter-dependencies/created-personal-access-token.png)

リポジトリの "Settings" に移動し、"Secrets and variables" > "Actions" を開き、"New repository secret" をクリックします。

![](/images/automatically-upgrade-flutter-dependencies/repository-secrets.png)

`GH_PERSONAL_ACCESS_TOKEN` としてコピーしたアクセストークンを登録しておきます。

![](/images/automatically-upgrade-flutter-dependencies/add-repository-secrets.png)

:::message
`GITHUB_` という名前は GitHub により禁止されているため、`GH_` としています。
:::

## 5. iOS ネイティブのライブラリのロックファイルを更新し、プッシュバックする

次に、ワークフローを作成します。

```yaml:.github/workflows/ios.yml
name: iOS

on:
  pull_request:
    branches:
      - "main"

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref }}
  cancel-in-progress: true

permissions:
  contents: read
  pull-requests: write

jobs:
  push-back-diffs-if-needed:
    name: Push back diffs after resolving dependencies if needed
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.head_ref }}
          token: ${{ secrets.GH_PERSONAL_ACCESS_TOKEN }}
      - name: Setup Git
        # Git のユーザーとして "github-actions[bot]" を設定する
        # https://github.com/actions/checkout/issues/13#issuecomment-724415212
        run: |
          git config --global user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git config --global user.name "github-actions[bot]"
      - uses: actions/setup-java@v4
        with:
          distribution: "temurin"
          java-version: "17"
      - uses: subosito/flutter-action@v2
      - name: Install iOS dependencies
        run: |
          flutter pub get --no-example
          flutter precache --ios
          cd ios
          pod install
      - name: Commit
        run: |
          git add ios/Podfile.lock
          if git diff --cached --quiet; then
            echo "No changes to commit"
          else
            git commit -m 'build: fix Podfile.lock'
          fi
      - name: Push back if needed
        run: |
          BRANCH_NAME="${{ github.event.pull_request.head.ref }}"
          git push origin "$BRANCH_NAME"
```

ワークフローの説明は以下のとおりです。

ワークフローでコードをチェックアウトする際、Git のチェックアウト先の Ref として `github.head_ref` を指定しています。

```yaml
- uses: actions/checkout@v4
  with:
    ref: ${{ github.head_ref }}
```

これは、**プルリクエストのトリガーによるワークフローでは、デフォルトで、プルリクエストがベースブランチにマージされた状態のブランチがチェックアウトされるため**です。
**PR のブランチ上でプッシュバックを実現したいため、明示的に PR のブランチそのものを指定**します。

また、チェックアウトの際には、GitHub のアクセストークンとして `GH_PERSONAL_ACCESS_TOKEN` を使っています。

```yaml
- uses: actions/checkout@v4
  with:
    # ...
    token: ${{ secrets.GH_PERSONAL_ACCESS_TOKEN }}
```

`Install iOS dependencies` は、iOS ネイティブのライブラリのロックファイルを更新するためのステップです。

```shell:Install iOS dependencies
flutter pub get --no-example
flutter precache --ios
cd ios
pod install
```

Flutter のライブラリが更新された後に上記のように実行することで、iOS ネイティブのライブラリのロックファイルを更新される場合があります。

その後、`Commit` ステップにより、更新されたロックファイルをコミットします。

```shell:Commit
git add ios/Podfile.lock
if git diff --cached --quiet; then
  echo "No changes to commit"
else
  git commit -m 'build: fix Podfile.lock'
fi
```

`ios/Podfile.lock` に更新がある場合、`build: fix Podfile.lock` というコミットメッセージでコミットします。

`Push back if needed` は、更新されたロックファイルをプッシュバックするためのステップです。

```shell:Push back if needed
BRANCH_NAME="${{ github.event.pull_request.head.ref }}"
git push origin "$BRANCH_NAME"
```

チェックアウトの際に `GH_PERSONAL_ACCESS_TOKEN` を使っているため、プッシュの際もそのトークンが使われます。
これにより、GitHub Actions の CI が再度トリガーされます。

# 実際に動作している様子

Renovate が定期的に PR を作成します。

![](/images/automatically-upgrade-flutter-dependencies/pr-by-renovate.png)

最初は以下のようなコミットが作成されます。

![](/images/automatically-upgrade-flutter-dependencies/first-commit-by-renoate.png)

PR 作成後しばらくすると以下のようなコミットがプッシュバックされ、CI が再度トリガーされます。

![](/images/automatically-upgrade-flutter-dependencies/push-back-commit.png)

CI がパスすると、以下のように PR がマージされます。

![](/images/automatically-upgrade-flutter-dependencies/auto-merge-by-renovate.png)

# 最後に

本記事で紹介した方法により、ライブラリ更新が大幅に自動化できるため、開発者の負担を軽減できます。

一方で、まだライブラリ更新時に依存関係の解決が失敗して手動で修正する必要がある場合もあります。
そのため、まだ完全な自動化とはなっていません。

今後も、完全な自動化を目指していきたいです。
