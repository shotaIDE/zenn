---
title: "GitHub Actionsで差分に応じたCIの選択実行と必須ジョブ設定を両立させる"
emoji: "🎯"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ios", "android"]
published: false
---

# 背景

GitHub Actions では、PR の差分に応じてワークフローを実行させるか否かを選択できる `paths` や `paths-ignore` フィルターが用意されています。

https://docs.github.com/ja/actions/using-workflows/workflow-syntax-for-github-actions#onpushpull_requestpull_request_targetpathspaths-ignore

これにより、差分の内容から影響を受けず確認の必要がないワークフローをスキップし、CI の実行時間や経済的コストを削減できます。

一方で、PR で実行されるべきワークフローやジョブを指定し、そのチェックがパスしていないと PR をマージできないようにするブランチに対する「ステータスチェック必須」設定が用意されています。

https://docs.github.com/ja/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/managing-a-branch-protection-rule

これにより、PR マージにより、マージ先のブランチが壊れてしまうことを防ぐことができ、安全な開発プロセスにつながります。

上記の 2 つの設定は併用して開発することが望ましいです。

ところが、`paths` フィルターを利用すると、ステータスチェックの結果がそもそも残らず、チェックが「実行不要で満たしている」という状態であることを GitHub Actions が認識してくれません。

（ここに図を挿入します）

本記事では、この解決方法を記載します。

# やり方の概要

`paths` や `paths-ignore` フィルターは利用せず、ジョブにおける `if` 構文を利用します。

`if` 構文により、チェックを行うジョブと、チェックが不要なので何もせずに終了するジョブのいずれかが実行されるようにします。

ステータスチェック必須設定では、2 つのジョブを両方とも必須として設定します。

# 詳細な方法

```yaml:.github/workflows/ci_api.yaml
name: CI / API

on:
  pull_request:
    branches:
      - "main"
    paths:
      - "functions/**"

jobs:
  test:
    name: Test API
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install Python dependencies
        run: pip install -r requirements.txt
      - name: Test
        run: pytest
```
