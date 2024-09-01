---
title: "Pythonを使ってAPNs経由でプッシュ通知を送信する"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ios", "python", "apns"]
published: false
---

<!-- cSpell:ignore asyncio, httpx -->

# 前提

- iOS でプッシュ通知を受信する準備ができている
- デバイストークンが取得できている
- APNs の認証キー(`*.p8`ファイル)が取得できている

# 1. httpx をインストールする

APNs にリクエストを送信するためには、HTTP/2 に対応している HTTP クライアントが必要です。

そのため、今回は httpx を使用します。

https://www.python-httpx.org/

以下のコマンドでインストールします。

```bash
pip install httpx
```

# 2. jwt をインストールする

APNs にリクエストを送信するには、認証キーを使って JWT を生成する必要があります。

今回は PyJWT を使用します。

https://pyjwt.readthedocs.io/en/stable/

以下のコマンドでインストールします。

```bash
pip install PyJWT
```

# 3. JWT を生成する

APNs にリクエストを送信するためには、JWT トークンを生成する必要があります。

https://developer.apple.com/documentation/usernotifications/establishing-a-token-based-connection-to-apns

以下のスクリプトで JWT トークンを生成します。

```python
from time import time

import jwt


PRIVATE_KEY_PATH = 'apns-auth-key.p8' # APNs の認証キーのパス
APNS_KEY_ID = 'XXXXXXXXXX' # 作成した認証キーの Key ID
TEAM_ID = 'XXXXXXXXXX' # Apple Developer Program の Team ID

private_key = open(PRIVATE_KEY_PATH, 'r').read()
headers = {
    'alg': 'ES256',
    'kid': APNS_KEY_ID
}
payload = {
    'iss': TEAM_ID,
    'iat': time()
}

token = jwt.encode(
    payload,
    private_key,
    algorithm='ES256',
    headers=headers
)
```

`token` として得られた値を、リクエストヘッダーの `authorization` に `bearer {token}` の形式で設定します。

# 4. プッシュ通知のリクエストを送信する

APNs に対してプッシュ通知のリクエストを送信します。

https://developer.apple.com/documentation/usernotifications/sending-notification-requests-to-apns

以下のスクリプトでプッシュ通知のリクエストを送信します。

```python
import httpx


BUNDLE_ID = 'com.example.PushTest' # iOS アプリの Bundle ID
USE_SANDBOX = True # プッシュ通知でサンドボックス環境を使用するかどうか
if USE_SANDBOX:
    URL_PREFIX = 'https://api.sandbox.push.apple.com/3/device/'
else:
    URL_PREFIX = 'https://api.push.apple.com/3/device/'

device_tokens = [
    # iOS のデバイストークン、1台目
    '3dcb4cb72e9abbdaa73a89d8fff449e0de4093cc2c0cbfa5ee9bed21c0a88a72',
    # iOS のデバイストークン、2台目
    '6f80ab76a6237ba9498a469e6f23c580d9572c0303706aa2c01a90dca3e795f6'
]

headers = {
    'authorization': f'bearer {token}', # `token` は前述の JWT トークン
    'apns-push-type': 'alert',
    'apns-topic': f'{BUNDLE_ID}'
}

payload = {
    'aps': {
        'alert': {
            'title': 'テストタイトル',
            'body': 'テスト本文'
        }
    }
}

async with httpx.AsyncClient(http2=True) as client:
    for index, device_token in enumerate(device_tokens):
        url = f'{URL_PREFIX}{device_token}'

        response = await client.post(
            url,
            headers=headers,
            json=payload
        )

        print(f'#{index}: device token = {device_token}')
        print(f'Response status code: {response.status_code}')
```

httpx で HTTP/2 の通信する方法の詳細は以下のリンクを参照してください。

https://www.python-httpx.org/http2/

# 5. Python スクリプトを実行する

前述のスクリプトを組み合わせると、以下のようになります。

```python
# coding: utf-8

import asyncio
from time import time

import httpx
import jwt


def __generate_jwt_token():
    PRIVATE_KEY_PATH = 'apns-auth-key.p8' # APNs の認証キーのパス
    APNS_KEY_ID = 'XXXXXXXXXX' # 作成した認証キーの Key ID
    TEAM_ID = 'XXXXXXXXXX' # Apple Developer Program の Team ID

    private_key = open(PRIVATE_KEY_PATH, 'r').read()
    headers = {
        'alg': 'ES256',
        'kid': APNS_KEY_ID
    }
    payload = {
        'iss': TEAM_ID,
        'iat': time()
    }

    return jwt.encode(
        payload,
        private_key,
        algorithm='ES256',
        headers=headers
    )


async def send_notification():
    BUNDLE_ID = 'com.example.PushTest' # iOS アプリの Bundle ID
    USE_SANDBOX = True # プッシュ通知でサンドボックス環境を使用するかどうか
    if USE_SANDBOX:
        URL_PREFIX = 'https://api.sandbox.push.apple.com/3/device/'
    else:
        URL_PREFIX = 'https://api.push.apple.com/3/device/'

    device_tokens = [
        # iOS のデバイストークン、1台目
        '3dcb4cb72e9abbdaa73a89d8fff449e0de4093cc2c0cbfa5ee9bed21c0a88a72',
        # iOS のデバイストークン、2台目
        '6f80ab76a6237ba9498a469e6f23c580d9572c0303706aa2c01a90dca3e795f6'
    ]

    token = __generate_jwt_token()

    headers = {
        'authorization': f'bearer {token}',
        'apns-push-type': 'alert',
        'apns-topic': f'{BUNDLE_ID}'
    }

    payload = {
        'aps': {
            'alert': {
                'title': 'テストタイトル',
                'body': 'テスト本文'
            }
        }
    }

    async with httpx.AsyncClient(http2=True) as client:
        for index, device_token in enumerate(device_tokens):
            url = f'{URL_PREFIX}{device_token}'

            response = await client.post(
                url,
                headers=headers,
                json=payload
            )

            print(f'#{index}: device token = {device_token}')
            print(f'Response status code: {response.status_code}')


if __name__ == '__main__':
    loop = asyncio.get_event_loop()
    response = loop.run_until_complete(
        send_notification()
    )
```

上記を実行すると、以下のように各デバイストークンに対してプッシュ通知の送信が成功することを確認できます。

```log
#0: device token = 3dcb4cb72e9abbdaa73a89d8fff449e0de4093cc2c0cbfa5ee9bed21c0a88a72
Response status code: 200
#1: device token = 6f80ab76a6237ba9498a469e6f23c580d9572c0303706aa2c01a90dca3e795f6
Response status code: 200
```
