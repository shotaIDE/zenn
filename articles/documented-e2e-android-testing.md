---
title: "Android の E2E テストで QA メンバーとの協業を考える"
emoji: "🌟"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["android"]
published: false
---

ある程度の規模のチームを想定しています。

# 背景

エンジニアメンバーが E2E テストを書き、自分たちで自分たちの書いたコードが正しく動作することを確認しています。

一方で、QA メンバーは手動テストを行っています。

自動の E2E テストを運用しています。

一方で、技術的な難易度が理由ですべてのテストケースを自動に置き換えることは難しいです。

手動と自動との組み合わせを最適化する必要があります。

単体テストや自動 E2E テストの中身をすべて把握している開発者が対応できるとある程度簡単にできます。

ただ、プロジェクトの規模が大きくなると、その対応が難しくなります。

本記事ではその解決のためのアイデアを紹介します。

# 基本的なアイデア

- クラスやメソッド名である程度内容の想像がつくようにしておく
- 開発者が Doc コメント形式でテストメソッドに関して、非開発者が見ても分かる文章を作る
- Doc コメントからドキュメントを自動で生成し、QA メンバーはそれを見るようにする

# 具体的なやり方

## クラスやメソッド名の整理方法

クラスはカテゴリー分けのためのものとします。

メソッド名は、テストケースの名前がそのまま愚直に書かれたものとします。
これは通常のテスト実装における命名と何ら変わらないでしょう。

## ドキュメントの自動生成

dokka を利用します。

```diff gradle:app/build.gradle
+import org.jetbrains.dokka.gradle.DokkaTask

tasks.withType(DokkaTask.class) {
    dokkaSourceSets {
        named("main") {
            moduleName.set("Automated E2E Testing")
            noStdlibLink.set(true)
            noJdkLink.set(true)
            noAndroidSdkLink.set(true)
            includes.from("automated-e2e-testing.md")
            sourceRoots.from(file("src/androidTest/path/to/your/e2e/package"))

            perPackageOption {
                matchingRegex.set("^(?!.*your\\.e2e\\.package\\.name).*")
                suppress.set(true)
            }
        }
    }
}
```

## デプロイ

GitHub Pages を利用します。
