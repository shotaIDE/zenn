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

以下のような Python のテスト実行の CI が組まれていたとします。
これに対して、修正を適用していきます。

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

`paths` フィルターを削除し、修正が入ったパスを判定するジョブ `check-impact` を追加します。

https://github.com/marketplace/actions/changed-files

```diff yaml:.github/workflows/ci_api.yaml
on:
  pull_request:
    branches:
      - "main"
-    paths:
-      - "functions/**"

jobs:
+  check-impact:
+    name: Check impact
+    runs-on: ubuntu-latest
+    outputs:
+      has-changed-related-files: ${{ steps.check-related-files.outputs.any_changed == 'true' }}
+    steps:
+      - uses: actions/checkout@v4
+        with:
+          fetch-depth: 0
+      - name: Check related files
+        id: check-related-files
+        uses: tj-actions/changed-files@v40
+        with:
+          files: "functions/**"
```

`if` 構文により、チェックを行うジョブと、チェックが不要なので何もせずに終了するジョブのいずれかが実行されるようにします。

```diff yaml:.github/workflows/ci_api.yaml
  test:
    name: Test API
+    needs: check-impact
+    if: needs.check-impact.outputs.has-changed-related-files == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install Python dependencies
        run: pip install -r requirements.txt
      - name: Test
        run: pytest
+  test-no-need:
+    name: Test API (no need)
+    needs: check-impact
+    if: needs.check-impact.outputs.has-changed-related-files == 'false'
+    runs-on: ubuntu-latest
+    steps:
+      - name: Skip
+        run: echo "No changes in files related to API, skipping."
```

リポジトリの設定から "Branch protection rule" を開き、以下のように 2 つのジョブを両方とも設定します。
ジョブは GitHub Actions 上で一度以上実行されないと候補として表示されないので、試しに動作させた後でこの設定は行います。

![](/images/required-job-depends-on-diffs-for-github-actions/required-check-settings.png)

以上のように設定することで、 `functions/**` の差分に関わらずチェックステータスが記録されるので、必須設定が適切に動作します。

![](/images/required-job-depends-on-diffs-for-github-actions/check-result-on-needed.png)

![](/images/required-job-depends-on-diffs-for-github-actions/check-result-on-no-need.png)
