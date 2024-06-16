---
title: "Terraformで既存のFirebaseプロジェクトを管理する"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["firebase", "terraform", "ios", "android"]
published: false
---

# 前提の方針

以下は、現時点では Terraform による管理に対応していないため、諦める必要があります。

- Firebase Cloud Messaging
- Firebase Remote Config
- Firebase Crashlytics
- Firebase Analytics

これらは手動でセットアップ＆管理する必要があります。

これ以外に関しては、Terraform で管理します。
また、Terraform で管理されているリソースに関しては、Firebase Console での変更は避けるようにします。
tfstate による管理が面倒になるためです。

# 本記事で書くこと

本記事では、新たに Firebase プロジェクトを作成する際に Terraform によりセットアップする方法は書きません。

これは公式のサンプルの通りに作ればおおよそ問題ないためです。

https://firebase.google.com/docs/projects/terraform/get-started?hl=ja

# 手順の概要

1. 既存の Firebase プロジェクトで作成されているリソースのうち、Terraform で管理できるリソースを洗い出す。
2. Terraform で管理できるリソースを定義する。
3. 各リソースを import するための ID を取得する。
4. Terraform の定義ファイルを自動生成する。
5. Terraform plan で差分なく定義されているか確認する。

# Terraform で管理できるリソースを洗い出す

Firebase プロジェクトをセットアップし各種機能を有効にした際、リソースがどのようにプロビジョニングされているかの知識が必要と思われます。

そのため、Firebase のリソースを一旦 Terraform で定義してみることがおすすめです。

import する必要があります。
リソースを見つけ、その import に必要な ID フォーマットを確認し、Firebase や GCP のコンソール、CLI ツールから ID を取得します。

そして、以下のような形式で Terraform で定義します。

```hcl:import.tf
import {
  id = "{{project_id}}"
  to = google_project.default
}

# 他のリソースの定義が続く
```

私のプロジェクトの場合、以下のようになりました。

## プロジェクト

| リソース名                                                                                                                         | 説明                                    |
| ---------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------- |
| [google_project](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/google_project)                    | GCP プロジェクト                        |
| [google_firebase_project](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firebase_project)         | Firebase プロジェクト                   |
| [google_firebase_apple_app](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firebase_apple_app)     | Firebase に登録された Apple(iOS) アプリ |
| [google_firebase_android_app](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firebase_android_app) | Firebase に登録された Android アプリ    |
