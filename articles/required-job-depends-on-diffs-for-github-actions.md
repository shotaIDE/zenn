---
title: "GitHub Actionsで差分に応じたCIの選択実行と必須ジョブ設定を両立させる"
emoji: "🎯"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["github", "githubactions"]
published: false
---

この記事では、GitHub Actions を使用して、ファイルの差分に基づく継続的インテグレーション（CI）の効率化と、必須ジョブの安全な設定を両立させる方法について紹介します。

CI の効率化は、リソースの節約と迅速なフィードバックの提供に不可欠ですが、安全なプロセスの保持も同様に重要です。

この記事では、この 2 つの要素のバランスをとる方法を解説します。

# 背景

GitHub Actions では、`paths`や`paths-ignore`フィルターを使って、プルリクエスト（PR）の差分に応じてワークフローを実行するかどうかを選択できます。

https://docs.github.com/ja/actions/using-workflows/workflow-syntax-for-github-actions#onpushpull_requestpull_request_targetpathspaths-ignore

この機能により、影響を受けないワークフローをスキップし、CI の実行時間とコストを削減できます。

また、「ステータスチェック必須」設定を用いて、特定のワークフローやジョブの完了を PR のマージ条件に設定できます。

https://docs.github.com/ja/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/managing-a-branch-protection-rule

この機能により、安全な開発プロセスを確保できます。

しかし、`paths`フィルターや`paths-ignore`フィルターを利用すると、必要なステータスチェックが実行されず、GitHub Actions が「実行不要」と判断する問題が生じます。

例えば、以下のように `Test API` というジョブを PR のマージ条件に設定します。

![](/images/required-job-depends-on-diffs-for-github-actions/required-check-settings-before.png)

この状況下で、``paths`フィルターや`paths-ignore`フィルターにより`Test API`ジョブがスキップされると、必要なチェックステータスが満たされない状態のままとなってしまいます。

![](/images/required-job-depends-on-diffs-for-github-actions/check-result-with-paths.png)

本記事では、CI の効率化と安全性を両立させるための解決策を提案します。

# やり方の概要

`paths`や`paths-ignore`フィルターの代わりに、`if`構文を使用してジョブの実行必要性を決定します。

この方法では、ファイルの変更がある場合にのみ重要なチェックを実行し、変更がない場合はチェックをスキップします。

これにより、不要なジョブの実行を避けつつ、必要なステータスチェックを確実に行うことが可能になります。

# 詳細な方法

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

ここで、`paths`フィルターを削除し、変更があるかどうかを判定する`check-impact`ジョブを追加します。

https://github.com/marketplace/actions/changed-files

```diff yaml

:.github/workflows/ci_api.yaml
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

`if`構文を用いて、関連ファイルに変更がある場合のみテストジョブを実行し、そうでない場合は何もしない（`Test API (no need)`）というジョブを実行します。

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

リポジトリの設定で、2 つのジョブを両方必須として設定します。

![](/images/required-job-depends-on-diffs-for-github-actions/required-check-settings-after.png)

以上の設定により、`functions/**`の差分に関わらず、必要なチェックが適切に記録され、PR のマージが安全に行えるようになります。

![](/images/required-job-depends-on-diffs-for-github-actions/check-result-on-needed.png)

![](/images/required-job-depends-on-diffs-for-github-actions/check-result-on-no-need.png)
