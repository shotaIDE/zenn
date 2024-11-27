---
title: "モバイルアプリのリリース頻度ってどれくらいを目指せばいいの？"
emoji: "🔄"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["ios", "android"]
published: false
---

こんにちは、モバイルエンジニアのころむにーです。

普段、モバイルアプリを通じたユーザーへの価値提供を目指すプロジェクトに、技術リードという立場で参加しています。

本記事では、モバイルアプリのリリース頻度について、私が最近個人的に考えていることをまとめました。

# 導入

開発プロジェクトにおいて、開発チームの生産性を高めることは重要です。
開発チームの生産性の高さは、継続的なユーザーへの価値提供の土台となるためです。

開発チームの生産性においてよく利用される指標の 1 つにリリース頻度があります。

これは、DORA（DevOps Research and Assessment）が提唱する指標の 1 つで、リリース頻度が高いチームほど、ビジネス成果が高いという研究結果があります。

これを見ると、リリース頻度の一番高いレベル（エリートパフォーマー）では、オンデマンドリリースを実践しているようです。

しかし、モバイルアプリの場合は、ストアの審査もあるし、オンデマンドのリリースは無理だと考えられます。

そこで、モバイルアプリのリリース頻度について考えてみました。

# モバイルアプリの事情

## ストアの審査

モバイルアプリをリリースする際には、App Store や Google Play などのストアに審査を通過する必要があります。

この際、審査に通過するまで、iOS は 1 日〜数日程度、Android は数時間程度かかることが一般的です。

そのため、そもそもアプリが公開されるまでに、このような自分達で制御できない時間がかかることを考慮する必要があります。

## ユーザーの手元でのロールアウト

また、モバイルアプリの場合、ユーザーの手元にアプリが届くまでにも時間がかかります。

ユーザーがアプリをアップデートするタイミングは様々で、アプリがリリースされたからといって、すぐに全ユーザーに届くわけではありません。

そのため、この点も考慮する必要があります。

これらを踏まえると、モバイルアプリのリリース頻度において、オンデマンドリリースを目指すのは難しいと考えられます。

## 品質

モバイルアプリは、一度ユーザーの手元にインストールされると、アプリをロールバックするなどの対応ができません。

そのため、致命的なバグを含むアプリをリリースしてしまうと、ユーザーに重大な不便を与えてしまいます。

復旧までには、ストアの審査を通過する時間がかかることを考えると、数時間程度は少なくともかかります。

ユーザー数が多すぎると、このダウンタイムはビジネスに大きな影響を及ぼす可能性があります。

## 監視・運用

モバイルアプリは、ユーザーの手元にインストールされて稼働するタイミングがユーザーごとに異なります。
そのため、複数の異なるバージョンが同時に稼働するということは珍しくありません。

このような状況下で、ユーザーの手元で発生した問題を迅速に対応するためには、どのバージョンのアプリがインストールされているかを素早く把握し、そのバージョンにおける調査する必要があります。

このように、複数バージョンのアプリが同時に稼働するという状況は、運用の複雑さを増す要因となります。

# 世の一般的なリリース頻度

# まとめ

1 日数回といったオンデマンドなデプロイは、ストア時間やアプリのロールアウトスピードから現実的ではありません。
週一、隔週のデプロイ頻度が最も良い塩梅と言えるでしょう。

また、他プラットフォームよりも品質の高さを求められるため、1 日でもデプロイ頻度を早めるより、一定間隔で安定してデプロイするのに注力することが重要と言えます。