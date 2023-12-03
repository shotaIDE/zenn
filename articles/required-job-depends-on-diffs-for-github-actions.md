---
title: "GitHub Actionsで差分に応じたジョブの選択実行とPRマージにおける必須ジョブ設定を両立させる"
emoji: "🎯"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["github", "githubactions"]
published: false
---

本記事では、GitHub Actions 利用時における、PR の差分に基づくジョブの選択実行と、PR マージ時の必須ジョブ設定を両立させる方法について紹介します。

通常、PR の差分に基づいてジョブを選択実行する際は、`paths` や `paths-ignore` フィルターが利用されます。
一方でこの方法では、PR マージ時の必須ジョブ設定と競合を起こしてしまうことがあります。

以下では、この問題の回避策を紹介します。

# やり方の概要

`paths` や `paths-ignore` フィルターの代わりに、Changed Files アクションを利用します。

https://github.com/marketplace/actions/changed-files

Changed Files アクションにてファイル差分を確認し、チェックジョブの影響範囲内のファイルが含まれていた場合は、`if` 構文によりチェックジョブを実行します。

一方で、チェックジョブの影響範囲内のファイルが含まれていなかった場合は、同じく `if` 構文により何もしないというジョブを実行します。

さらに、PR マージ時の必須ジョブ設定では、上記のチェックジョブと何もしないジョブを両方とも必須として設定します。

これにより、不要なチェック実行を避けつつ、必要なステータスチェックを確実に行うことが可能になります。

# 背景となる課題

:::message
以下では背景となる課題をより詳細に記載します。回避策を確認したい場合は、次の章まで読み飛ばしてください。
:::

GitHub Actions では、`paths` や `paths-ignore` フィルターを使って、PR の差分に応じてワークフローを実行するかどうかを選択できます。

https://docs.github.com/ja/actions/using-workflows/workflow-syntax-for-github-actions#onpushpull_requestpull_request_targetpathspaths-ignore

この機能により、影響を受けないワークフローやジョブをスキップし、GitHub Actions の実行時間とコストを削減できます。

また、「ステータスチェック必須」設定を用いて、特定のワークフローやジョブの完了を PR のマージ条件に設定できます。

https://docs.github.com/ja/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/managing-a-branch-protection-rule

この機能により、安全な開発プロセスを確保できます。

しかし、`paths` フィルターや `paths-ignore` フィルターを利用すると、**スキップされたチェックジョブが「待機状態」となり、GitHub が「マージ不可」と判断する**問題が生じます。

例えば、以下のように `Test API` というジョブを PR のマージ条件に設定します。

![](/images/required-job-depends-on-diffs-for-github-actions/required-check-settings-before.png)

この状況下で、`paths` フィルターや `paths-ignore` フィルターにより `Test API` ジョブがスキップされると、必要なチェックステータスが待機状態のままとなりマージがブロックされます。

![](/images/required-job-depends-on-diffs-for-github-actions/check-result-with-paths.png)

# やり方の詳細

以下では具体的な回避策を紹介します。

## 前提となるチェックジョブ

たとえば、以下のような Python のテスト実行 CI が設定されているとします。

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

## paths フィルターを changed-files アクションに差し替える

ここで、`paths` フィルターを削除し、変更があるかどうかを判定する `check-impact` ジョブを追加します。

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
# ...
```

## チェックジョブの発動条件を if に差し替え、かつ、何もしないジョブを追加する

`if` 構文を用いて、関連ファイルに変更がある場合のみテストジョブを実行し、そうでない場合は何もしない `Test API (no need)` ジョブを実行します。

```diff yaml:.github/workflows/ci_api.yaml
  test:
    # ...
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

## PR マージにおける必須ジョブ設定を変更する

リポジトリの設定で、2 つのジョブを両方必須として設定します。

![](/images/required-job-depends-on-diffs-for-github-actions/required-check-settings-after.png)

## GitHub Actions の実行結果を確認する

以上の設定により、`functions/**` の差分に関わらず、必要なチェックが適切に記録され、PR のマージが安全に行えるようになります。

![](/images/required-job-depends-on-diffs-for-github-actions/check-result-on-needed.png)

![](/images/required-job-depends-on-diffs-for-github-actions/check-result-on-no-need.png)
