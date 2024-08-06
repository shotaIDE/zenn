---
title: "AndroidアプリのWebソケットクライアントで受信する一部のメッセージをプロキシから遅延させる"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["android", "mitmproxy", "kotlin"]
published: false
---

<!-- cSpell:ignore asyncio, mitmdump, mitmproxy -->

非常にニッチな内容です。

以下のようなことをやります。

![](/images/delay-websocket-message-to-android/summary.png)

# 手順

前提として、Android アプリとプロキシーの通信内容を復号するため、Android アプリ側でユーザーがインストールしたルート証明書を信頼する必要があります。
これには、Android アプリのネットワーク設定の修正が必要です。
詳細は、以下の記事をご確認ください。

手順としては以下です。

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

まず、mitmproxy を起動します。

```bash
mitmproxy
```

次に、Android 端末のプロキシーを以下のように設定します。

- ホスト名: mitmproxy を起動している PC の IP アドレス
- ポート: 8080
  - mitmproxy のデフォルトのポート番号

Android 端末のブラウザーで、 http://mitm.it にアクセスします。

Android のボタンをタップし、ルート証明書をダウンロードします。

OS 設定で「CA 証明書」で検索し、CA 証明書の画面を開きます。ダウンロードしたルート証明書をインストールします。

ブラウザーで https://example.com にアクセスします。
これにより、mitmproxy のコンソール画面に通信内容が表示されれば、設定は成功です。

# アプリの Web ソケットクライアントにプロキシーを設定する

アプリの Web ソケットクライアントにプロキシーを設定します。

例えば、OkHttp を使用している場合、以下のように設定します。

:::message
Java-WebSocket を使用している場合、以下のようなエラーが出て、プロキシーが設定できませんでした。

```log
Invalid proxy
```

Issue など漁ってみましたが、原因や解決策が不明でした。そのため、Java-WebSocket での動作確認はできていません。
:::

# mitmproxy でメッセージを遅延させるスクリプトを書く

mitmproxy は、Python でスクリプトを書いて動作をカスタマイズできます。
公式ページにサンプルスクリプトが載っているので、参考にしてください。

https://docs.mitmproxy.org/stable/addons-examples/

今回は、特定のメッセージを遅延させるスクリプトを書きます。
以下のようにしました。

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

    # 最新のメッセージを取得
    latest_message = flow.websocket.messages[-1]

    if latest_message.from_client:
        logging.info(f"[{LOG_TAG}] Received a message from the client: {latest_message.text}")
    else:
        # スクリプト内で生成したメッセージに対して同じ処理を繰り返さないよう、`injected` フラグで識別
        if not latest_message.injected and 'keyword' in latest_message.text
            logging.info(f"[{LOG_TAG}] Will delay a message from the server: {latest_message.text}")

            # サーバーから送信されたメッセージをキャンセルする
            latest_message.drop()

            # サーバーから送信されたメッセージと同一のメッセージを、遅延させた後に再送する
            # websocket_message の処理をブロックすると、他のメッセージも送信されないため、非同期で処理する
            asyncio.create_task(post_websocket_message_async(flow, latest_message.text))
        else:
            logging.info(f"[{LOG_TAG}] Received a message from the server: {latest_message.text}")


async def post_websocket_message_async(flow: http.HTTPFlow, message: str):
    await asyncio.sleep(0.5)

    to_client = True
    ctx.master.commands.call("inject.websocket", flow, to_client, message.encode())

    logging.info(f"[{LOG_TAG}] Send the delayed message from the server: {message}")
```

以下のコマンドで mitmproxy を起動します。

```shell
mitmproxy -s delay-websocket-message.py
```

スクリプト内で出力しているログを確認するには、以下のコマンドにてログ出力モードにするとわかりやすいです。

```shell
mitmdump -s delay-websocket-message.py
```
