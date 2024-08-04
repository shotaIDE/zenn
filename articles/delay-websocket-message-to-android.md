---
title: "AndroidアプリのWebソケットクライアントで受信する一部のメッセージを遅延させる"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["android", "mitmproxy", "kotlin"]
published: false
---

<!-- cSpell:ignore asyncio, mitmproxy -->

# 手順

1. mitmproxy をインストールする
2. アプリの Web ソケットクライアントにプロキシーを設定する
3. mitmproxy でメッセージを遅延させるスクリプトを書く

# mitmproxy でメッセージを遅延させるスクリプトを書く

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
