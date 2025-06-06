# 執筆ガイドライン

## スタイル

スタイルは、[style_tech.md](/docs/style_tech.md)を参照してください。

## 執筆手順

記事を執筆した後は、静的解析の結果を確認し、指摘があれば修正してください。

### スペルチェック

```shell
npx cspell path/to/article.md
```

### 構文

```shell
npx textlint -f checkstyle path/to/article.md
```
