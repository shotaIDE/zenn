---
title: "Terraformで既存のFirebaseプロジェクトを管理する"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["firebase", "terraform", "ios", "android"]
published: false
---

<!-- cspell:ignore appspot, cloudfunctions, firebaserules, ruleset, rulesets, tfstate, tfvars -->

# 前提の方針

以下は、現時点では Terraform による管理に対応していないため、手動でセットアップ＆管理する必要があります。

- Firebase Cloud Messaging
- Firebase Remote Config
- Firebase Crashlytics
- Firebase Analytics

上記以外に関しては、Terraform で管理します。

また、Terraform で管理されているリソースに関しては、Firebase Console での変更は避けるようにします。
手動で変更すると、Terraform 側にその変更を伝えるために、tfstate ファイルにも変更を反映する必要があります。

tfstate とは、Terraform が管理するリソースの状態を記録するファイルです。
これを利用することで、Terraform はリソースの状態を把握し、変更があった場合にはその変更を計算し、必要な更新手順を提案します。

また、tfstate ファイルは、GCP の Cloud Storage や AWS の S3 などのリモートストレージに保存し、チームで共有できます。
本記事では、簡単のために、tfstate に関しては、ローカルに管理することとします。

# 本記事で書くこと

本記事では、新たに Firebase プロジェクトを作成する際に Terraform によりセットアップする方法は書きません。

これは公式のサンプルの通りに作ればおおよそ問題ないためです。

https://firebase.google.com/docs/projects/terraform/get-started?hl=ja

また、実際に CI/CD を組んでデプロイを自動化するところまでは書きません。

ローカルのマシンで Terraform で管理、デプロイするまでを本記事では書きます。

# 手順の概要

手順としては以下のように進めます。

1. 必要なツールをインストールする
2. 作業ディレクトリを作成する
3. (オプション)Terraform 上で Firebase を管理する方法について知る
4. Terraform で管理できるリソースを洗い出す
5. Terraform で管理できるリソースを定義する。
6. 各リソースを import するための ID を取得する。
7. Terraform の定義ファイルを自動生成する。
8. Terraform plan で差分なく定義されているか確認する。

# 必要なツールをインストールする

以下のページを参考に Terraform をセットアップします。

https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli

また、以下のページを参考に Firebase CLI をセットアップします。
CLI をインストールしたら、**CLI 上での Firebase へのログイン**も実施しておきます。

https://firebase.google.com/docs/cli?hl=ja

さらに、Google Cloud CLI もセットアップします。
こちらも **CLI 上でのログイン**を実施しておきます。

https://cloud.google.com/sdk/docs/install?hl=ja

:::message
Firebase CLI と Google Cloud CLI でログインするアカウントは、プロジェクトオーナーの権限を持つアカウントで行うと便利です。
これにより、Terraform コマンド実行時の権限エラーに悩まされることなく、手順をスムーズに進められます。
一方で、本格的にチームで運用する際や自動デプロイを組む場合は、必要最小限の権限を持つアカウントを用意することをおすすめします。
:::

# 作業ディレクトリを作成する

作業用のディレクトリを作成し、その中に 3 つの Terraform の設定ファイルを配置します。

```
.
├── main.tf
├── import.tf
└── terraform.tfvars
```

```hcl:main.tf
terraform {
  required_providers {
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "5.34.0"
    }
  }
}
```

```hcl:import.tf
# 後から記載するため、一旦空ファイルとしておく
```

```hcl:terraform.tfvars
# 後から記載するため、一旦空ファイルとしておく
```

最新バージョン名は以下を確認してください。

https://registry.terraform.io/providers/hashicorp/google-beta/latest

以下コマンドを実行します。

```shell
terraform init
```

生成されたファイルをコミットしておきます。

# Terraform 上で Firebase を管理する方法について知る

:::message
本項目はオプションです。不要な方はスキップしてください。
:::

Firebase プロジェクトをセットアップし各種機能を有効にした際、リソースがどのようにプロビジョニングされているかの知識が必要と思われます。
そのため、Firebase のリソースを一旦 Terraform で定義してみることがおすすめです。

https://firebase.google.com/docs/projects/terraform/get-started?hl=ja

# 1. Terraform で管理できるリソースを洗い出す

Terraform で既存のインフラリソースを管理するためには、各リソースを Terraform にインポートする操作が必要です。
インポートにより、Terraform はリソースの状態を把握し、tfstate ファイルに記録します。
Terraform は tfstate ファイルを参照して現状のリソースを把握し、さらにファイルに定義されたリソースの状態を比較し、必要な更新手順を計算します。

インポートには、コマンドにより 1 つずつリソースをインポートする方法と、tf ファイルに定義されたリソースを一気にインポートする方法があります。

今回は、tf ファイルに定義されたリソースを一気にインポートする方法を採用します。

一気にインポートする方法では、tf ファイル自体を自動生成するという機能もあるため、そちらを利用して tf ファイル生成も省力化して実施します。

インポート定義を作成するには、以下の手順が必要です。

- リソースを見つけ、その import に必要な ID フォーマットを確認し、Firebase や GCP のコンソール、CLI ツールから ID を取得します。
- リソースに対し、Terraform 上で管理するための名前をつけます

そして、以下のような形式で Terraform ファイルに定義します。

```hcl:import.tf
import {
  id = "{{リソースの ID}}"
  to = {{リソースの種別}}.{{リソースの管理名}}
}
```

私のプロジェクトの場合、以下のようになりました。

## Firebase のセキュリティールールの名前を調べる

Firebase CLI や GCP CLI から、Firebase のセキュリティールールの名前を調べる方法が分かっていません。
そのため、一旦 Terraform で関連するリソースをインポートすることで、間接的にセキュリティールールの名前を調べることにします。

まず、以下の一時ファイルを作成します。
これは、Firestore と Storage のセキュリティールールのリソースをインポートするためのファイルです。

```hcl:temporary.tf
resource "google_firebaserules_release" "firestore" {
  provider     = google-beta
  name         = "cloud.firestore"
  ruleset_name = ""
}

resource "google_firebaserules_release" "storage" {
  provider     = google-beta
  name         = "cloud.storage"
  ruleset_name = ""
}
```

GCP のプロジェクト ID を調べておきます。

以下のコマンドを実行します。

```shell
PROJECT_ID="{{GCPのプロジェクトIDを記載}}"
terraform import google_firebaserules_release.firestore "projects/$PROJECT_ID/releases/cloud.firestore"
terraform import google_firebaserules_release.storage "projects/$PROJECT_ID/releases/firebase.storage/$PROJECT_ID.appspot.com"
```

すると、`terraform.tfstate` という名前の JSON ファイルが生成されます。
その中から、`ruleset_name` というキー名に対するバリューを探してメモしておきます。

![](/images/manage-existing-firebase-by-terraform/firebase-ruleset-name-in-tfstate.png)

以下のようなフォーマットです。

```text
projects/{{GCPのプロジェクトID}}/rulesets/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` の部分を後から利用します。

:::message
`x` は数字またはアルファベットを示しています。
:::

メモが完了したら、`temporary.tf` と `terraform.tfstate` を削除しておきます。

## プロジェクトとアプリ

まず、プロジェクト本体と、Firebase に登録されているアプリのインポート定義を追加します。

| リソース名                                                                                                                         | 説明                                    |
| ---------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------- |
| [google_project](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/google_project)                    | GCP プロジェクト本体                    |
| [google_firebase_project](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firebase_project)         | Firebase プロジェクト本体               |
| [google_firebase_apple_app](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firebase_apple_app)     | Firebase に登録された Apple(iOS) アプリ |
| [google_firebase_android_app](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firebase_android_app) | Firebase に登録された Android アプリ    |

```diff hcl:import.tf
+variable "import_google_project_id" {
+  type        = string
+  description = "ID for GCP project."
+}
+
+variable "import_firebase_apple_app_id" {
+  type        = string
+  description = "App ID for Firebase Apple app, such as 1:000000000000:ios:xxxxxxxxxxxxxxxxxxxxxx."
+}
+
+variable "import_firebase_android_app_id" {
+  type        = string
+  description = "App ID for Firebase Android app, such as 1:000000000000:android:xxxxxxxxxxxxxxxxxxxxxx."
+}
+
+import {
+  id = var.import_google_project_id
+  to = google_project.default
+}
+
+import {
+  id = "projects/${var.import_google_project_id}"
+  to = google_firebase_project.default
+}
+
+import {
+  id = "projects/${var.import_google_project_id}/iosApps/${var.import_firebase_apple_app_id}"
+  to = google_firebase_apple_app.default
+}
+
+import {
+  id = "projects/${var.import_google_project_id}/androidApps/${var.import_firebase_android_app_id}"
+  to = google_firebase_android_app.default
+}
```

```diff hcl:terraform.tfvars
+import_google_project_id       = "{{GCPのプロジェクトIDを記載}}"
+import_firebase_apple_app_id   = "{{Firebaseに登録されているAppleアプリのアプリIDを記載}}"
+import_firebase_android_app_id = "{{Firebaseに登録されているAndroidアプリのアプリIDを記載}}"
```

Firebase に登録されているアプリ ID は、Firebase Console から確認できます。

![](/images/manage-existing-firebase-by-terraform/firebase-apple-app-id.png)

## Authentication

次に、Firebase Authentication に関するリソースのインポート定義を追加します。

| リソース名                                                                                                                                 | 説明                  |
| ------------------------------------------------------------------------------------------------------------------------------------------ | --------------------- |
| [google_identity_platform_config](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/identity_platform_config) | Authentication の設定 |

```diff hcl:import.tf
# ...

import {
  id = "projects/${var.import_google_project_id}/androidApps/${var.import_firebase_android_app_id}"
  to = google_firebase_android_app.default
}
+
+import {
+  id = vars.import_google_project_id
+  to = google_identity_platform_config.auth
+}
```

## Firestore

Firestore に関するリソースのインポート定義を追加します。

| リソース名                                                                                                                           | 説明                                     |
| ------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------- |
| [google_firestore_database](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firestore_database)       | Firestore 本体                           |
| [google_firebaserules_ruleset](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firebaserules_ruleset) | Firestore のセキュリティルール           |
| [google_firebaserules_release](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firebaserules_release) | Firestore のセキュリティルールの適用状態 |

```diff hcl:import.tf
# ...

variable "import_firebase_android_app_id" {
  type        = string
  description = "App ID for Firebase Android app, such as 1:000000000000:android:xxxxxxxxxxxxxxxxxxxxxx."
}

+variable "import_firestore_ruleset_name" {
+  type        = string
+  description = "Firestore rule set name."
+}
+
import {
  id = var.import_google_project_id
  to = google_project.default
}

# ...

import {
  id = vars.import_google_project_id
  to = google_identity_platform_config.auth
}
+
+import {
+  id = "projects/${vars.import_google_project_id}/databases/(default)"
+  to = google_firestore_database.default
+}
+
+import {
+  id = "projects/${vars.import_google_project_id}/rulesets/${var.import_firestore_ruleset_name}"
+  to = google_firebaserules_ruleset.firestore
+}
+
+import {
+  id = "projects/${vars.import_google_project_id}/releases/cloud.firestore"
+  to = google_firebaserules_release.firestore
+}
```

```diff hcl:terraform.tfvars
# ...
import_firebase_android_app_id = "{{Firebaseに登録されているAndroidアプリのアプリIDを記載}}"
+import_firestore_ruleset_name  = "{{Firestoreのルールセット名を記載}}"
```

Firebase で Firestore を有効にすると、データベース名が `(default)` になります。
そのため、`google_firestore_database.default` の ID の末尾は `(default)` 固定にしています。

もし、お使いの Firestore のデータベース名が `(default)` 以外の場合は、その名前を指定してください。

Firestore のデータベース名は、Firebase Console から確認できます。

![](/images/manage-existing-firebase-by-terraform/firestore-database-id.png)

Firestore のルールセット名は、最初の方の手順でメモした以下のフォーマットのものを記載します。

```text
xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

## Firebase Storage

Firebase Storage に関するリソースのインポート定義を追加します。

| リソース名                                                                                                                               | 説明                                              |
| ---------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------- |
| [google_firebase_storage_bucket](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firebase_storage_bucket) | Firebase Storage 本体                             |
| [google_firebaserules_ruleset](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firebaserules_ruleset)     | Firestore のセキュリティルール                    |
| [google_firebaserules_release](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firebaserules_release)     | Firestore のセキュリティルールの適用状態          |
| [google_app_engine_application](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/app_engine_application)   | Firestore によりプロビジョニングされる App Engine |

```diff hcl:import.tf
# ...

import {
  id = "projects/${vars.import_google_project_id}/releases/cloud.firestore"
  to = google_firebaserules_release.firestore
}
+
+import {
+  id = "projects/${vars.import_google_project_id}/buckets/${vars.import_google_project_id}.appspot.com"
+  to = google_firebase_storage_bucket.default
+}
+
+import {
+  id = "projects/${vars.import_google_project_id}/rulesets/${var.import_firebase_storage_ruleset_name}"
+  to = google_firebaserules_ruleset.storage
+}
+
+import {
+  id = "projects/${vars.import_google_project_id}/releases/firebase.storage/${vars.import_google_project_id}.appspot.com"
+  to = google_firebaserules_release.storage
+}
+
+import {
+  id = vars.import_google_project_id
+  to = google_app_engine_application.default
+}
```

```diff hcl:terraform.tfvars
# ...
import_firestore_ruleset_name        = "{{Firestoreのルールセット名を記載}}"
+import_firebase_storage_ruleset_name = "{{Firebase Storageのルールセット名を記載}}"
```

Firebase Storage のルールセット名は、最初の方の手順でメモした以下のフォーマットのものを記載します。

```text
xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

Firestore を有効にすると、裏側で AppEngine が有効にされます。
これを Terraform で管理する必要があります。

`google_firebaserules_ruleset` と `google_firebaserules_release` は Firestore で取り込んだリソースの種類と同じです。

## Cloud Functions

Firebase でラップされている Cloud Functions ではなく、GCP の Cloud Functions を直接利用していたので、以下のインポート定義を追加します。

| リソース名                                                                                                                                              | 説明                           |
| ------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------ |
| [google_cloudfunctions_function](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/cloudfunctions_function)                | Cloud Functions の各関数       |
| [google_cloudfunctions_function_iam_member](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/cloudfunctions_function_iam) | Cloud Functions の公開ポリシー |

```diff hcl:import.tf
variable "import_google_project_id" {
  type        = string
  description = "ID for GCP project."
}

+variable "import_google_project_location" {
+  type        = string
+  description = "Location for GCP project."
+}
+
variable "import_firebase_apple_app_id" {
  type        = string
  description = "App ID for Firebase Apple app, such as 1:000000000000:ios:xxxxxxxxxxxxxxxxxxxxxx."
}

# ...

import {
  id = vars.import_google_project_id
  to = google_app_engine_application.default
}
+
+import {
+  id = "${vars.import_google_project_id}/${var.import_google_project_location}/function1"
+  to = google_cloudfunctions_function.function1
+}
+
+import {
+  id = "${vars.import_google_project_id}/${var.import_google_project_location}/detect roles/cloudfunctions.invoker allUsers"
+  to = google_cloudfunctions_function_iam_member.function1_invoker
+}
```

```diff hcl:terraform.tfvars
import_google_project_id             = "{{GCPのプロジェクトIDを記載}}"
+import_google_project_location       = "{{GCPプロジェクトのロケーションを記載}}"
import_firebase_apple_app_id         = "{{Firebaseに登録されているAppleアプリのアプリIDを記載}}"
# ...
```

Function を複数定義している場合は、それぞれに対してインポート定義が必要です。

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
