---
title: "AndroidアプリのWebソケットクライアントで受信する一部のメッセージをプロキシから遅延させる"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["android", "mitmproxy", "kotlin"]
published: false
---

<!-- cSpell:ignore asyncio, inet, mitmdump, mitmproxy -->

非常にニッチな内容です。

以下のようなことをやります。

![](/images/delay-websocket-message-to-android/summary.png)

# 手順

:::message
前提として、Android アプリからプロキシーに対して暗号化通信をし、その内容をプロキシー側で復号します。
これには、Android アプリのネットワーク設定において、ユーザーが後からインストールしたルート証明書を信頼し通信可能にする設定が必要です。
詳細は、以下の公式ドキュメントをご確認ください。

https://developer.android.com/privacy-and-security/security-config?hl=ja
:::

手順は以下の通りです。

1. mitmproxy をインストールする
2. Android 端末に、mitmproxy のルート証明書をインストールする
3. アプリの Web ソケットクライアントにプロキシーを設定する
4. mitmproxy でメッセージを遅延させるスクリプトを書く
5. 動作確認

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

![プロキシー設定のスクショ]()

Android 端末のブラウザーで、 http://mitm.it にアクセスします。
Android の証明書入手ボタンをタップし、ルート証明書を Android 端末にダウンロードします。

![証明書ダウンロードページのスクショ]()

OS 設定で「CA 証明書」で検索し、CA 証明書の画面を開きます。
![CA証明書のスクショ]()

ダウンロードしたルート証明書を選択し、インストールします。

![選択画面のスクショ]()

ブラウザーで https://example.com にアクセスします。
これにより、mitmproxy のコンソール画面に通信内容が表示されれば、設定は成功です。

![](/images/delay-websocket-message-to-android/example-request-summary.png)

各行をクリックすると詳細が表示されます。

![](/images/delay-websocket-message-to-android/example-request-details.png)

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

以下で Web ソケットのテストサーバーを利用できます。

https://websocket.org/tools/websocket-echo-server/

`192.168.11.13` の部分は、mitmproxy を起動している PC の LAN 内の IP アドレスに置き換えてください。

:::message
[Java-WebSocket](https://github.com/TooTallNate/Java-WebSocket) を使用している場合、以下のようなエラーが出て、プロキシーが設定できませんでした。

```log
Invalid proxy
```

Issue など漁ってみましたが、原因や解決策が不明でした。
そのため、Java-WebSocket での動作確認はできていません。
:::

# mitmproxy でメッセージを遅延させるスクリプトを書く

mitmproxy は、Python でスクリプトを書いてプロキシーの動作を細かくカスタマイズできます。
公式ページにサンプルスクリプトが豊富に掲載されています。

https://docs.mitmproxy.org/stable/addons-examples/

今回は、特定のメッセージを遅延させるスクリプトを書きます。
以下のような Python スクリプトを任意の場所に保存してください。

```python:delay-websocket-message.py
# coding: utf-8

import asyncio
import logging
import json

from mitmproxy import http
from mitmproxy import ctx

LOG_TAG = 'DelayWebSocketMessageAddon'

async def websocket_message(flow: http.HTTPFlow):
    global pending_message_from_server

    assert flow.websocket is not None

    # Webソケットにおける最新のメッセージを取得
    latest_message = flow.websocket.messages[-1]

    if latest_message.from_client:
        # クライアントから送信されたメッセージは、特に何もしない
        logging.info(f"[{LOG_TAG}] Received a message from client: {latest_message.text}")
    else:
        # サーバーから送信されたメッセージに対して、特定のキーワードが含まれていたら、遅延処理を行う
        # また、スクリプト内で生成したメッセージに対して再起的に同じ処理を繰り返さないよう、`injected` フラグで識別
        if not latest_message.injected and 'keyword' in latest_message.text
            logging.info(f"[{LOG_TAG}] Will delay a message from server: {latest_message.text}")

            # サーバーから送信された元々のメッセージをキャンセルする
            latest_message.drop()

            # サーバーから送信されたメッセージと同一のメッセージを、遅延させた後に再送する
            # websocket_message の処理をブロックすると、後続の他メッセージも送信されずに止まってしまうため、非同期で処理する
            asyncio.create_task(post_websocket_message_async(flow, latest_message.text))
        else:
            logging.info(f"[{LOG_TAG}] Received a message from server: {latest_message.text}")


async def post_websocket_message_async(flow: http.HTTPFlow, message: str):
    await asyncio.sleep(0.5)

    to_client = True
    # サーバーから送信されたメッセージと同じものをクライアントに再送信する
    ctx.master.commands.call("inject.websocket", flow, to_client, message.encode())

    logging.info(f"[{LOG_TAG}] Send the delayed message from server: {message}")
```

以下のコマンドで mitmproxy を起動します。

```shell
mitmproxy -s delay-websocket-message.py
```

`delay-websocket-message.py` の部分は、スクリプトのパスに適宜置き換えてください。

スクリプト内で出力しているログを確認するには、以下のコマンドにてログ出力モードにするとわかりやすいです。

```shell
mitmdump -s delay-websocket-message.py
```

以下のように遅延処理が行われることを確認できます。

![処理の様子]()
