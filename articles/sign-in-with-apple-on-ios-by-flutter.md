---
title: "FlutterによるiOSアプリでTestFlightにアップロードしたアプリのみ「Appleでサインイン」が失敗する問題の解消法メモ"
emoji: "🍎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "ios", "xcode", "apple"]
published: false
---

Flutter プロジェクトで「Apple でサインイン」を実装している際に、以下の事象に遭遇しました。

- TestFlight にアップロードしたアプリで「Apple でサインイン」を実行すると、内部で何かしらのエラーが発生し、サインインモーダルが表示されない
- ローカルでビルドしたアプリでは問題なく「Apple でサインイン」が動作する

解決策をメモしておきます。

## 前提

- Flutter プロジェクトにおける iOS アプリ
- Apple でサインイン機能を実装済み
- Xcode プロジェクトファイルの設定は以下の通り
  - Automatic Signing が有効
- Flutter のビルドコマンドと xcodebuild コマンドを使用して、**Manual Signing によりビルド**したものを TestFlight にアップロード

## 結論

Xcode プロジェクトファイルの構成そのものを Manual Signing に変更しました。
これにより、TestFlight にアップロードしたアプリでも「Apple でサインイン」機能が正常に動作するようになりました。

## 詳細

### 問題の発生

Automatic Signing を有効にして xcodebuild でビルドしたアプリでは、Sign in with Apple 機能が無効になっていました。

### 試した解決策 1: Code Signing の有効化

最初に試した解決策は、アーカイブ時に Code Signing を有効にする方法でした：

```bash
xcodebuild -archivePath path/to/archive \
           -authenticationKeyPath path/to/AuthKey.p8 \
           -authenticationKeyID KEY_ID \
           -authenticationKeyIssuerID ISSUER_ID
```

この方法で Sign in with Apple 機能は動作するようになりましたが、CI/CD ビルドで以下のエラーが発生しました：

- 開発者アカウントに登録されていないデバイスエラー
- マッチする Provisioning Profile が見つからないエラー

### 最終的な解決策: Manual Signing の採用

上記の問題を受けて、Release 構成で Manual Signing を使用する方針へ変更しました。

#### 設定手順

1. Xcode でプロジェクトの Release 構成を開く
2. Signing の設定を Manual Signing に変更
3. 適切な Signing Identity と Provisioning Profile を指定
4. Flutter の `build ipa` コマンド実行時は、構成で指定した設定をそのまま使用

この方法により、以下の利点が得られました：

- Sign in with Apple 機能が正常に動作
- CI/CD ビルドでのエラーが解消
- 予測可能なビルド結果

## まとめ

Flutter プロジェクトで Sign in with Apple 機能を実装する際は、以下の点が重要と考えています：

- Automatic Signing は便利ですが、Sign in with Apple 機能との組み合わせで問題の発生する可能性がある
- Manual Signing を使用することで、より予測可能で安定したビルド結果を得られる
- CI/CD パイプラインでの運用を考慮した署名設定の選択が重要

同様の問題に遭遇した方の参考になれば幸いです。
