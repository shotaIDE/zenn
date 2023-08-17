---
title: "Androidでパスキー認証を導入する際の仕様検討や実装のポイント"
emoji: "🔐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["android", "kotlin"]
published: true
---

Android ネイティブでパスキー認証(FIDO2)を実装するため方法を調査しましたが、引っかかる部分が多かったので記事を書いてみることにしました。

iOS の記事も以下に書きました。

https://zenn.dev/colomney/articles/android-passkey-key-points

# 本記事で書くこと

本記事では以下の内容が含まれています。

- Android アプリにおけるパスキー認証の仕様検討時の注意点
- Android アプリにおけるパスキー認証の実装や動作確認方法

以下のような内容は含まれていません。

- パスキー認証とは
- サーバー側におけるパスキー認証の実装

# ポイントの概要

パスキーを導入する際の動作環境や、動作確認用の端末については、**Android 9 以上**が必要です。
また、画面ロックを設定している必要があります。

動作させるには、**特定のドメインに対してアプリを紐づけるためのサーバー構築が必要**です。
[Charles](https://www.charlesproxy.com/) などのプロキシーツールでドメイン紐付けを捏造するのは不可能と考えられるので、サーバーの準備が必須です。

アプリには、**パスキーの登録と、登録済みのパスキーで認証という二段階の実装**が必要になります。

# 詳細

## ドメインに対してアプリを紐づける

まず、パスキーを利用するドメインを決定します。
例えば、以下では `your.domain.com` というドメインに決定したと仮定します。

次に `/.well-known/assetlinks.json` に JSON ファイルを配置します。
フルの URL は `https://your.domain.com/.well-known/assetlinks.json` のようになります。

JSON ファイルの中身は以下のように記載します。

```json:assetlinks.json
[
  {
    "relation": [
      "delegate_permission/common.handle_all_urls",
      "delegate_permission/common.get_login_creds"
    ],
    "target": {
      "namespace": "android_app",
      "package_name": "ide.shota.colomney.PasskeyTest",
      "sha256_cert_fingerprints": [
        "0A:0A:0A:0A:0A:0A:0A:0A:0A:0A:0A:0A:0A:0A:0A:0A:0A:0A:0A:0A:0A:0A:0A:0A:0A:0A:0A:0A:0A:0A:0A:0A"
      ]
    }
  }
]
```

:::message

JSON の URL にアクセスした際のレスポンスは、Content-Type が `application/json` で返却される必要があります。

:::

JSON ファイルが正しく記載できているかは以下の URL で確認できます。

```
https://digitalassetlinks.googleapis.com/v1/statements:list?source.web.site=https://${ココにサイトのドメイン}&relation=delegate_permission/common.handle_all_urls
```

:::message

Android OS は、以下 URL の Google がクローリングしている結果にアクセスすることで間接的に JSON の中身を取得します。
`https://digitalassetlinks.googleapis.com`

このレスポンスは Charles 等のプロキシーツールで書き換えできません。
そのため、パスキーの動作確認には信頼される HTTPS 証明書を持つサーバー環境を用意する必要があります。

:::

より詳細な情報は Google のドキュメントを参考にしてください。

https://developer.android.com/training/app-links/verify-android-applinks?hl=ja

## Android アプリを実装する

詳細な実装に関しては、公式が公開しているサンプルプロジェクトを参考にしてみてください。

https://developer.android.com/training/sign-in/passkeys?hl=ja#create-passkey

以下では要点をかいつまんで解説します。

### パスキーを登録する

以下のコードによりパスキー登録を開始します。

```kotlin:MainViewModel.kt
class MainViewModel : ViewModel() {
    private val domain = "your.domain.com"
    private val challenge = "neKjg-lPlgvdOuFxDb9HCLeFD5726DmLkrZofdWsoWk" // 本来はサーバーから取得したチャレンジのデータを利用します
    private val userName = "Test User"

    suspend fun register(context: Context) {
        val credentialManager = CredentialManager.create(context)
        val requestJson = """
{
  "rp": { "id": "$domain", "name": "Passkey Test" },
  "user": {
    "id": "VGVzdCBVc2Vy",
    "name": "test.user",
    "displayName": "$userName"
  },
  "challenge": "$challenge",
  "pubKeyCredParams": [
    {
      "type": "public-key",
      "alg": -7
    },
    {
      "type": "public-key",
      "alg": -257
    }
  ],
  "timeout": 1800000,
  "attestation": "none",
  "excludeCredentials": [],
  "authenticatorSelection": {
    "authenticatorAttachment": "platform",
    "requireResidentKey": true,
    "residentKey": "required",
    "userVerification": "required"
  }
}
        """.trimIndent()

        val request = CreatePublicKeyCredentialRequest(
            requestJson = requestJson,
            preferImmediatelyAvailableCredentials = false
        )

        val result = credentialManager.createCredential(context, request)

        println(result.data) // このデータをサーバーと共有します
    }
}
```

上記のコードにより、以下のような画面が表示されます。
![](/images/android-passkey-key-points/01_register-passkey.png =250x)

作成したパスキーの確認・削除は、Android 端末の「設定」 > 「Google パスワードマネージャー」で確認できます。
![](/images/android-passkey-key-points/02_passkey-on-google-password-manager.png =250x)

## 登録済みのパスキーで認証する

以下のコードにより登録済みのパスキーで認証します。

```kotlin:MainViewModel.kt
class MainViewModel : ViewModel() {
    private val domain = "your.domain.com"
    private val challenge = "neKjg-lPlgvdOuFxDb9HCLeFD5726DmLkrZofdWsoWk" // 本来はサーバーから取得したチャレンジのデータを利用します
    private val base64EncodedCredentialId = "go6b2cAEfjnWC4hZpRecmw" // 本来はサーバーから取得した認証情報のIDを利用します

    suspend fun signIn(context: Context) {
        val credentialManager = CredentialManager.create(context)
        val requestJson = """
{
  "challenge": "$challenge",
  "allowCredentials": [
    {
      "type": "public-key",
      "id": "$base64EncodedCredentialId"
    }
  ],
  "timeout": 1800000,
  "userVerification": "required",
  "rpId": "$domain"
}
        """.trimIndent()
        val getPublicKeyCredentialOption = GetPublicKeyCredentialOption(
            requestJson = requestJson
        )
        val getCredRequest = GetCredentialRequest(
            listOf(getPublicKeyCredentialOption)
        )

        val result = credentialManager.getCredential(context, getCredRequest)

        println(result) // このデータをサーバーと共有します
    }
}
```

画面の設計としては、以下のようになるでしょう。

- サーバーに登録された対象ユーザーの公開鍵が存在しない場合、パスキーの新規登録に進む
- サーバーに登録された対象ユーザーの公開鍵が存在する場合
  - クライアント側でサーバーに登録された公開鍵と対応する認証情報でパスキー認証を試みる。クライアント側にパスキーが存在しない場合は、パスキーの新規登録に進む

既存のパスキーがないかどうかは、`NoCredentialException` という例外をキャッチすることで確認できます。

```kotlin
try {
    credentialManager.getCredential(context, getCredRequest)
} catch (error: NoCredentialException) {
    println("認証情報が見つかりませんでした")
    // この後、パスキー登録のフローに進む
}
```

# 参考にしたサイト

https://codelabs.developers.google.com/codelabs/fido2-for-android#0

https://logmi.jp/tech/articles/322823
