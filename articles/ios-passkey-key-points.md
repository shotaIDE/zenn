---
title: "iOSでパスキー認証を導入する際の仕様検討や実装のポイント"
emoji: "🔐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ios", "swift"]
published: true
---

iOS ネイティブでパスキー認証(FIDO2)を実装するため方法を調査しましたが、引っかかる部分が多かったので記事を書いてみることにしました。

Android の記事も以下に書きました。

https://zenn.dev/colomney/articles/ios-passkey-key-points

# 本記事で書くこと

本記事では以下の内容が含まれています。

- iOS アプリにおけるパスキー認証の仕様検討時の注意点
- iOS アプリにおけるパスキー認証の実装や動作確認方法

以下のような内容は含まれていません。

- パスキー認証とは
- サーバー側におけるパスキー認証の実装

# ポイントの概要

パスキーを導入する際の動作環境や、動作確認用の端末については、**iOS 16 以上**が必要です。
また、画面ロックを設定している必要があります。

https://support.apple.com/ja-jp/guide/iphone/iphf538ea8d0/ios

動作させるには、**特定のドメインに対してアプリを紐づけるためのサーバー構築が必要**です。
[Charles](https://www.charlesproxy.com/) などのプロキシーツールがあれば、特定の API リクエストを改竄することでサーバーなしでも動作が可能です。

アプリには、**パスキーの登録と、登録済みのパスキーで認証という二段階の実装**が必要になります。
iOS では、ユーザーがパスキー認証をキャンセルしたのかそもそも認証情報が存在しないのかの区別がつかないという制約があるため、それを考慮した仕様を策定する必要があります。

# 詳細

## ドメインに対してアプリを紐づける

まず、パスキーを利用するドメインを決定します。
例えば、以下では `your.domain.com` というドメインに決定したと仮定します。

次に `/.well-known/apple-app-site-association` に JSON ファイルを配置します。
フルの URL は `https://your.domain.com/.well-known/apple-app-site-association` のようになります。

JSON ファイルの中身は以下のように記載します。
`XXXXXXXX` と記載している箇所は、Apple Developer 契約のチーム ID です。

```json:apple-app-site-association
{
  "webcredentials": {
    "apps": ["XXXXXXXX.ide.shota.colomney.PasskeyTest"]
  }
}
```

:::message

JSON の URL にアクセスした際のレスポンスは、Content-Type が `application/json` で返却される必要があります。

:::

さらに、Xcode 上で "Associated Domains" に `webcredentials:your.domain.com` を追加します。

![](/images/ios-passkey-key-points/02_associated-domain.png)

より詳細な情報は Apple のドキュメントを参考にしてください。

https://developer.apple.com/documentation/xcode/supporting-associated-domains

:::message

本来なら信頼される HTTPS 証明書を持つサーバー環境を用意する必要があります。
ただし、以下の手順を実施しておくことで、自己証明書や Charles などのプロキシーツールによる JSON ファイルのシミュレートが利用可能になります。

iOS 端末において、「設定」 > 「デベロッパ」 > 「ユニバーサルリンク」 > 「関連ドメインの開発」を ON にします。

![](/images/ios-passkey-key-points/01_associated-domains-of-developer-settings.jpg =250x)

かつ、Xcode 内で関連ドメインの末尾に `?mode=developer` を指定します。

![](/images/ios-passkey-key-points/03_developer-mode-for-associated-domain.png)

:::

## iOS アプリを実装する

詳細な実装に関しては、公式が公開しているサンプルプロジェクトを参考にしてみてください。

https://developer.apple.com/documentation/authenticationservices/connecting_to_a_service_with_passkeys

以下では要点をかいつまんで解説します。

### パスキーを登録する

以下のコードによりパスキー登録を開始します。
`UIWindow` のインスタンスを `anchor` 引数に指定し、`signUpWith` を実行します。

```swift:PasskeyResource.swift
import AuthenticationServices

class PasskeyResource: NSObject, ASAuthorizationControllerPresentationContextProviding, ASAuthorizationControllerDelegate {
    let domain = "your.domain.com"

    var authenticationAnchor: ASPresentationAnchor?

    func signUpWith(userName: String, anchor: ASPresentationAnchor) {
        self.authenticationAnchor = anchor
        let publicKeyCredentialProvider = ASAuthorizationPlatformPublicKeyCredentialProvider(relyingPartyIdentifier: domain)

        let challenge = Data() // 本来はサーバーから取得したチャレンジのバイナリデータを利用します
        let userID = Data(UUID().uuidString.utf8)

        let registrationRequest = publicKeyCredentialProvider.createCredentialRegistrationRequest(
            challenge: challenge,
            name: userName,
            userID: userID
        )

        let authController = ASAuthorizationController(authorizationRequests: [ registrationRequest ] )
        authController.delegate = self
        authController.presentationContextProvider = self
        authController.performRequests()
    }
}
```

デリゲートメソッドにより登録結果を受け取ります。

```swift:PasskeyResource.swift
extension PasskeyResource {
    func authorizationController(controller: ASAuthorizationController, didCompleteWithAuthorization authorization: ASAuthorization) {
        switch authorization.credential {
        case let credentialRegistration as ASAuthorizationPlatformPublicKeyCredentialRegistration:
            print("新しいパスキーが登録されました")
            let clientDataJSON = credentialRegistration.rawClientDataJSON
            print("認証データ: \(clientDataJSON)") // このJSONデータをサーバーと共有します
        default:
        }
    }
}
```

上記のコードにより、以下のような画面が表示されます。

![](/images/ios-passkey-key-points/04_register-passkey.png =250x)

作成したパスキーの確認や削除は、iOS 端末の「設定」 > 「パスワード」で確認できます。

![](/images/ios-passkey-key-points/05_passkey-in-settings.jpg =250x)

https://support.apple.com/ja-jp/HT211146

:::message

同じ Apple ID でサインインした複数の iOS 端末間では基本的にパスキーが共有されるので、その前提で仕様やサーバーの API を設計する必要があります。

:::

## 登録済みのパスキーで認証する

以下のコードにより登録済みのパスキーで認証します。

```swift:PasskeyResource.swift
import AuthenticationServices

class PasskeyResource: NSObject, ASAuthorizationControllerPresentationContextProviding, ASAuthorizationControllerDelegate {
    let domain = "your.domain.com"

    var authenticationAnchor: ASPresentationAnchor?

    func signInWith(anchor: ASPresentationAnchor, preferImmediatelyAvailableCredentials: Bool) {
        self.authenticationAnchor = anchor
        let publicKeyCredentialProvider = ASAuthorizationPlatformPublicKeyCredentialProvider(relyingPartyIdentifier: domain)

        let challenge = Data() // 本来はサーバーから取得したチャレンジを利用します

        let assertionRequest = publicKeyCredentialProvider.createCredentialAssertionRequest(challenge: challenge)

        let authController = ASAuthorizationController(authorizationRequests: [ assertionRequest ] )
        authController.delegate = self
        authController.presentationContextProvider = self

        authController.performRequests(options: .preferImmediatelyAvailableCredentials)
    }
}
```

上記のコードにより、以下のような画面が表示されます。

![](/images/ios-passkey-key-points/06_sign-in-with-passkey.png =250x)

ユーザーがパスキー認証をキャンセルしたり、パスキーの認証情報が存在しなかったなどのエラーは以下のコードでハンドリングできます。

```swift
extension PasskeyResource {
    func authorizationController(controller: ASAuthorizationController, didCompleteWithError error: Error) {
        guard let authorizationError = error as? ASAuthorizationError else {
            return
        }

        if authorizationError.code == .canceled {
            print("ユーザーがパスキー認証をキャンセルしたか、または、認証情報が存在しませんでした")
        }
    }
}
```

ただし、iOS でパスキー認証する関数で、ユーザーがパスキー認証をキャンセルしたのかそもそも認証情報が存在しないのかの区別がつきません。
これは、iOS のセキュリティ上の意図的な仕様ということです。
そのため、この仕様を考慮した画面設計が必要です。

https://developer.apple.com/forums/thread/735867

# 参考にしたサイト

https://qiita.com/mogmet/items/1c9720a311686ff02de3
