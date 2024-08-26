---
title: "AndroidアプリのWebソケットクライアントで送受信するメッセージを、プロキシで改ざん＆遅延させてみる"
emoji: "🪢"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["android", "mitmproxy", "kotlin"]
publication_name: "sun_asterisk"
published: false
---

<!-- cSpell:ignore asyncio, inet, mitmdump, mitmproxy -->

再現困難なバグの原因を特定するために習得した方法について、紹介してみます。

以下のように、Android アプリの Web ソケットクライアントで受信するメッセージを、プロキシで改ざんし、遅延させる方法を紹介します。

- サーバーからクライアントに送信されるメッセージを監視し、**特定の条件を満たす場合はメッセージをドロップ**する
- ドロップしたメッセージは、**一定時間経った後に再送信**する

![](/images/delay-websocket-message-to-android/summary.png)

紹介するスクリプトの内容を変更することで、クライアントからサーバーに送信するメッセージを改ざん対象に広げたり、その他様々な応用的な処理が可能です。

# 手順

手順は以下の通りです。

1. mitmproxy をインストールする
2. Android 端末に、mitmproxy のルート証明書をインストールする
3. アプリのネットワーク設定で、ユーザーが後からインストールしたルート証明書を信頼する
4. アプリの Web ソケットクライアントにプロキシーを設定する
5. mitmproxy でメッセージを遅延させるスクリプトを書く
6. 動作確認

# mitmproxy をインストールする

公式ページを参考に、mitmproxy をインストールします。

https://docs.mitmproxy.org/stable/overview-installation/

# Android 端末に、mitmproxy のルート証明書をインストールする

公式ページを参考に、mitmproxy のルート証明書を Android 端末にインストールします。

https://docs.mitmproxy.org/stable/concepts-certificates/

以下は上記のページの内容を抜粋したものです。

まず、以下のコマンドにより mitmproxy を起動します。

```bash
mitmproxy
```

次に、Android 端末のネットワーク設定におけるプロキシーを以下のように設定します。

| 項目       | 設定値                                              |
| ---------- | --------------------------------------------------- |
| ホスト名   | mitmproxy を起動している PC の LAN 内の IP アドレス |
| ポート番号 | `8080`（mitmproxy のデフォルトのポート番号）        |

![](/images/delay-websocket-message-to-android/android-os-proxy-settings.png =300x)

Android 端末のブラウザーで、 http://mitm.it にアクセスします。
Android の証明書入手ボタンをタップし、ルート証明書を Android 端末にダウンロードします。

![](/images/delay-websocket-message-to-android/mitmproxy-certificate-page.png =300x)

OS 設定で「CA 証明書」で検索し、CA 証明書の画面を開きます。
![](/images/delay-websocket-message-to-android/android-os-ca-certificates-settings.png =300x)

ダウンロードしたルート証明書を選択し、インストールします。

![](/images/delay-websocket-message-to-android/select-root-certificates.png =300x)

ブラウザーで https://example.com にアクセスします。
これにより、mitmproxy のコンソール画面に通信内容が表示されれば、設定は成功です。

![](/images/delay-websocket-message-to-android/example-request-summary.png)

各行をクリックすると詳細が表示されます。

![](/images/delay-websocket-message-to-android/example-request-details.png)

# アプリのネットワーク設定で、ユーザーが後からインストールしたルート証明書を信頼する

Android アプリのネットワーク設定で、ユーザーが後からインストールしたルート証明書を信頼するようにします。
関連する公式ドキュメントは以下です。

https://developer.android.com/privacy-and-security/security-config?hl=ja

まず、Android Manifest に `networkSecurityConfig` の設定を追加します。

```xml:app/src/main/AndroidManifest.xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">
    <application
        ...
        android:networkSecurityConfig="@xml/network_security_config"
        ...>
    </application>
</manifest>
```

ネットワークセキュリティの設定ファイルを作成します。

```xml:app/src/main/res/xml/network_security_config.xml
<network-security-config xmlns:tools="http://schemas.android.com/tools">
    <debug-overrides>
        <trust-anchors>
            <certificates
                src="user"
                tools:ignore="AcceptsUserCertificates" />
            <certificates src="system" />
        </trust-anchors>
    </debug-overrides>
</network-security-config>
```

`<certificates src="user" />` の設定値により、ユーザーが後からインストールしたルート証明書を用いた暗号化通信が Android アプリで有効になります。

# アプリの Web ソケットクライアントにプロキシーを設定する

アプリの Web ソケットクライアントにプロキシーを設定します。

Web ソケットクライアントのライブラリは複数ありますが、本記事では OkHttp を使用した例を示します。

https://github.com/square/okhttp

プロキシーのセットアップにおいて、以下のように設定します。

```diff kotlin
class WebSocketClient : WebSocketListener() {
    private val webSocket: WebSocket

    init {
        val client = OkHttpClient.Builder()
+            .proxy(
+                Proxy(
+                    Proxy.Type.HTTP,
+                    InetSocketAddress("192.168.11.13", 8080)
+                )
            )
            .build()
        val request = Request.Builder()
            .url("wss://your.domain/path")
            .build()
        webSocket = client.newWebSocket(request, this)
    }

    // ...
}
```

`192.168.11.13` の部分は、mitmproxy を起動している PC の LAN 内の IP アドレスに置き換えてください。

:::message
[Java-WebSocket](https://github.com/TooTallNate/Java-WebSocket) を使用している場合、以下のようなエラーが出て、プロキシーが設定できませんでした。

```log
Invalid proxy
```

Issue など漁ってみましたが、原因や解決策が不明でした。
そのため、Java-WebSocket での動作確認はできていません。
:::

具体的にテストする例として、本記事では、以下に用意された Web ソケットのテストサーバーを利用する方法を紹介します。
Web ソケットでクライアントからテキストを送信すると、サーバーから同じテキストが返ってきます。

https://websocket.org/tools/websocket-echo-server/

上記のテストサーバーを利用した簡単な Android アプリを以下のように実装します。
Web ソケットに接続し、0 から 1 ずつカウントアップするメッセージを送信できるアプリです。

https://github.com/shotaIDE/web-socket-test/blob/ea1118947cc0b88ebae71c1cad8905190c208944/app/src/main/java/ide/shota/colomney/websockettest/MainActivity.kt

![](/images/delay-websocket-message-to-android/web-socket-test-app.gif =300x)

# mitmproxy でメッセージを遅延させるスクリプトを書く

mitmproxy は、Python でスクリプトを書いてプロキシーの動作を細かくカスタマイズできます。
公式ページにサンプルスクリプトが豊富に掲載されています。

https://docs.mitmproxy.org/stable/addons-examples/

今回は、`1` という内容のメッセージを遅延させるスクリプトを書きます。
以下のような Python スクリプトを任意の場所に保存してください。

https://github.com/shotaIDE/web-socket-test/blob/1ad3ed0af5bd5230a0f0160db5f1fbc51ef911fb/delay-websocket-message.py

以下のコマンドで mitmproxy のログ出力モードを起動します。

```shell
mitmdump -s delay-websocket-message.py
```

`delay-websocket-message.py` の部分は、スクリプトのパスに適宜置き換えてください。

:::message
スクリプト内で出力しているログを確認するために `mitmdump` コマンドを利用しています。
これをそのまま `mitmproxy` に置き換えても動作確認できます。
:::

以下のように遅延処理が行われることを確認できます。

![](/images/delay-websocket-message-to-android/delay-mitmdump-log.png)

Android アプリでも、メッセージの受信が遅延して順番が入れ替わっていることが確認できます。

![](/images/delay-websocket-message-to-android/delay-android-logcat.png)
