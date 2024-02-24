---
title: "前回のリリースから差分があるかを判定する"
free: true
---

リリース時に付与される Git のタグを元に、最新コミットまでの差分を確認し、差分がある場合はリリース作業を進めます。

これには、GitHub Actions の Changed Files というアクションを利用しています。

https://github.com/marketplace/actions/changed-files

```yaml
# ...

check-unreleased-diff:
  name: Check some diffs exist related to app
  runs-on: ubuntu-latest
  outputs:
    has-diff-related-to-app: ${{ steps.check-related-files.outputs.any_changed == 'true' }}
  steps:
    - uses: actions/checkout@v4
      with:
        ref: ${{ github.head_ref }}
        fetch-depth: 0
    - name: Get latest RC tag name
      id: get-latest-rc-tag-name
      run: |
        latest_rc_tag_name="$(git describe --tags --abbrev=0 --match 'rc/*')"
        echo "Latest RC tag name: $latest_rc_tag_name"
        echo "tag-name=$latest_rc_tag_name" >> $GITHUB_OUTPUT
    - name: Check latest release is not based on latest commit
      id: check-latest-release-is-not-based-on-latest-commit
      run: |
        latest_release_commit_sha="$(git rev-list -n 1 HEAD)"
        latest_rc_tag_commit_sha="$(git rev-list -n 1 ${{ steps.get-latest-rc-tag-name.outputs.tag-name }})"
        if [ "$latest_release_commit_sha" = "$latest_rc_tag_commit_sha" ]; then
          echo "Latest release is based on the latest commit."
          echo "result=false" >> $GITHUB_OUTPUT
        else
          echo "Latest release is not based on latest commit."
          echo "result=true" >> $GITHUB_OUTPUT
        fi
    - name: Check related files
      id: check-related-files
      uses: tj-actions/changed-files@v42
      if: ${{ steps.check-latest-release-is-not-based-on-latest-commit.outputs.result == 'true' }}
      with:
        base_sha: ${{ steps.get-latest-rc-tag-name.outputs.tag-name }}
        files: |
          .maestro/**
          android/**
          ios/**
          lib/**
          test/**
          .tool-versions
          .xcode-version
          pubspec.lock
          pubspec.yaml
        sha: "HEAD"
```
