---
title: "ライブラリ更新に時間かかりすぎて、更新作業だけで人生終わっちゃうという危機感を持つ人に送る！"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "ios", "android"]
published: false
---

# はじめに

アプリ開発で、運用やメンテナンスにかかるコストはできるだけ削減することが重要です。

特に、ライブラリの更新は、アプリの品質を保つために欠かせない作業の一方で、その更新作業自体がコストを生むことがあります。

本記事では、Flutter プロジェクトでライブラリの更新を自動化する方法について紹介します。

# 概要

Renovate を導入し、PR 作成を自動化します。
これにより、Flutter のライブラリの依存関係定義ファイルとロックファイルが更新された PR が作成されます。

CI により、Flutter のライブラリが間接的に利用する iOS ネイティブのライブラリのロックファイルを更新し、PR にプッシュバックします。

Renovate の自動マージにより、マージされます。

# 詳細

GitHub で開発し、GitHub Actions で CI を組んでいる前提で説明します。

## Renovate の導入

Renovate は、GitHub のリポジトリにプルリクエストを作成し、ライブラリの更新を自動化するツールです。

https://docs.renovatebot.com/

以下の "Hosted GitHub.com App" を参考にして、GitHub App として該当のリポジトリに対してインストールします。

https://docs.renovatebot.com/getting-started/installing-onboarding/#hosted-githubcom-app

## Renovate の設定
