---
title: "Terraformで既存のFirebaseプロジェクトを管理する"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["firebase", "terraform", "ios", "android"]
published: false
---

<!-- cspell:ignore cloudfunctions, firebaserules, ruleset, tfstate -->

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

tfstate に関しては、ローカルに管理します。

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

# 事前準備

# Terraform をセットアップする

以下コマンドを実行します。

```shell
terraform init
```

生成されたファイルをコミットしておきます。

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

## プロジェクトとアプリ

| リソース名                                                                                                                         | 説明                                    |
| ---------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------- |
| [google_project](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/google_project)                    | GCP プロジェクト本体                    |
| [google_firebase_project](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firebase_project)         | Firebase プロジェクト本体               |
| [google_firebase_apple_app](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firebase_apple_app)     | Firebase に登録された Apple(iOS) アプリ |
| [google_firebase_android_app](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firebase_android_app) | Firebase に登録された Android アプリ    |

## Authentication

| リソース名                                                                                                                                 | 説明                  |
| ------------------------------------------------------------------------------------------------------------------------------------------ | --------------------- |
| [google_identity_platform_config](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/identity_platform_config) | Authentication の設定 |

## Firestore

| リソース名                                                                                                                           | 説明                                     |
| ------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------- |
| [google_firestore_database](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firestore_database)       | Firestore 本体                           |
| [google_firebaserules_ruleset](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firebaserules_ruleset) | Firestore のセキュリティルール           |
| [google_firebaserules_release](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firebaserules_release) | Firestore のセキュリティルールの適用状態 |

## Firebase Storage

| リソース名                                                                                                                               | 説明                                              |
| ---------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------- |
| [google_firebase_storage_bucket](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firebase_storage_bucket) | Firebase Storage 本体                             |
| [google_firebaserules_ruleset](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firebaserules_ruleset)     | Firestore のセキュリティルール                    |
| [google_firebaserules_release](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firebaserules_release)     | Firestore のセキュリティルールの適用状態          |
| [google_app_engine_application](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/app_engine_application)   | Firestore によりプロビジョニングされる App Engine |

`google_firebaserules_ruleset` と `google_firebaserules_release` は Firestore で取り込んだリソースの種類と同じです。

Firestore を有効にすると、裏側で AppEngine が有効にされます。
これを Terraform で管理する必要があります。

## Cloud Functions

Firebase でラップされている Cloud Functions ではなく、GCP の Cloud Functions を直接利用していたので、以下のリソースを定義しました。

| リソース名                                                                                                                                              | 説明                           |
| ------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------ |
| [google_cloudfunctions_function](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/cloudfunctions_function)                | Cloud Functions の各関数       |
| [google_cloudfunctions_function_iam_member](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/cloudfunctions_function_iam) | Cloud Functions の公開ポリシー |

関数は認証不要で全ユーザーがアクセスできるようにしていました。そのため、以下のような IAM ポリシーが設定されています。
Terraform ではこのような IAM ポリシーもリソースとして定義されています。

## Cloud Tasks

GCP の Cloud Tasks を利用していたので、以下のリソースを定義しました。

| リソース名                                                                                                                                              | 説明                           |
| ------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------ |
| [google_cloud_tasks_queue](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/cloud_tasks_queue)                            | Cloud Tasks のキュー           |
| [google_cloudfunctions_function_iam_member](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/cloudfunctions_function_iam) | Cloud Functions の公開ポリシー |

## サービスアカウント

| リソース名                                                                                                                      | 説明                     |
| ------------------------------------------------------------------------------------------------------------------------------- | ------------------------ |
| [google_service_account](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/google_service_account) | サービスアカウント       |
| [google_project_iam_member](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/google_project_iam)  | サービスアカウントの IAM |

# その他 Terraform で管理するようになってから生成したもの

- google_storage_bucket
- google_iam_workload_identity_pool
- google_iam_workload_identity_pool_provider
- google_service_account_iam_member

# Terraform の定義ファイルを自動生成する

以下のコマンドを実行します。

```shell
terraform plan -generate=import
```

# Terraform plan で差分なく定義されているか確認する

# まとめ
