---
title: "静的解析と単体テストがパスすることを PR のマージに必須の条件とする"
free: true
---

PR が提出されたら GitHub Actions で静的解析や単体テストが実行されるようにしておきます。

GitHub のブランチ保護ルールにて、これらの静的解析や単体テストをマージの必須条件としておきます。

https://docs.github.com/ja/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/managing-a-branch-protection-rule

ライブラリのアップデートで、アプリ内で利用しているメソッドが利用不可になったり deprecated になった場合は、ここで自動サイクルが止まります。
