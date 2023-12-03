---
title: "CI環境でiOSのUIテストを自動実行する際、失敗したテストケースの画面録画を残す"
emoji: "📸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ios", "xcode", "test", "ci"]
published: true
---

CI 環境で UI テストを自動実行する際に、テストが失敗した原因を調査するための情報をできるだけ多く用意しておくことは重要です。
特に、テストが失敗した際の画面録画は、強力な助けとなります。

Firebase Test Lab や Bitrise などを利用すると自動で画面録画を収集してくれる機能があります。
ただ、GitHub Actions や Circle CI などの利用している場合は、自前でその機能を構築する必要があります。

本記事では GitHub Actions や Circle CI などで利用できる、画面録画を残す方法を解説します。

# 前提

XCUITest を用いて UI テストケースを実装している状況を想定します。

また、iOS の Simulator には、Simulator が動作している Mac からコマンドにより画面録画を収集する機能が提供されています。

```shell
xcrun simctl io booted recordVideo "test-video.mov"
```

本記事では、上記の機能を利用して、テストケース実行時の画面録画を収集します。

# やり方の概要

Simulator を実行する Mac で、リクエストに応じて画面録画の開始と終了、破棄の機能を持ったサーバーを立てます。

テストケースの開始時に、サーバーに画面録画の開始をリクエストします。

また、テストケースの終了時に、サーバーに画面録画の終了をリクエストします。
この際、テストケースが成功した場合、画面録画の破棄をリクエストします。

破棄されず残った画面録画を CI 実行結果の成果物として保持します。

# やり方の詳細

## 画面録画の開始と終了、破棄の機能を持ったサーバーを立てる

サーバーはどのような技術を利用しても問題ないですが、今回は Ruby を利用したスクリプト例を紹介します。

```ruby:recording_server.rb
require 'fileutils'
require 'sinatra'

post '/record_video/:udid/:test_name' do
  recordings_dir = 'recording'
  video_base_name = "#{recordings_dir}/#{params['test_name']}"
  recordings = (0..Dir["#{recordings_dir}/*"].length + 1).to_a
  body = JSON.parse(request.body.read)
  FileUtils.mkdir_p(recordings_dir)

  video_file = ''
  if body['delete']
    recordings.reverse_each do |i|
      video_file = "#{video_base_name}_#{i}.mp4"
      break if File.exist?(video_file)
    end
  else
    recordings.each do |i|
      video_file = "#{video_base_name}_#{i}.mp4"
      break unless File.exist?(video_file)
    end
  end

  if body['stop']
    simctl_processes = `pgrep simctl`.strip.split("\n")
    simctl_processes.each { |pid| `kill -s SIGINT #{pid}` }
    File.delete(video_file) if body['delete'] && File.exist?(video_file)
  else
    # 録画の開始
    puts `xcrun simctl io #{params['udid']} recordVideo --codec h264 --force #{video_file} &`
  end
end
```

上記のスクリプトでは、Simulator から HTTP 通信を受信するサーバーを立てています。

また、サーバーのプロセスで、Simulator に対するコマンド `xcrun simctl` の実行や中断をしています。

## テストケースの開始・終了時に、ローカルサーバーに録画の指示を出す

以下のように、`XCUITest` のテストケースの親クラスを作成します。

```swift:XCTestCaseWithRecording.swift
import ImageIO
import MobileCoreServices
import XCTest

class XCTestCaseWithRecording: XCTestCase {
    static private let recordingServerOrigin = "http://localhost:4567"

    override func setUpWithError() throws {
        try super.setUpWithError()
        recordVideo()
    }

    override func tearDownWithError() throws {
        recordVideo(stop: true)
        try super.tearDownWithError()
    }

    private func recordVideo(stop: Bool = false) {
        let udid = ProcessInfo.processInfo.environment["SIMULATOR_UDID"] ?? ""

        // `name` は "-[(テストクラス名) (テストケース名)]" のような形式
        let parts = name.components(separatedBy: " ")
        let testClassName = String(parts[0].dropFirst(2))
        let testCaseName = String(parts[1].dropLast(1))

        let urlString = "\(XCTestCaseWithRecording.recordingServerOrigin)/record_video/\(udid)/\(testClassName).\(testCaseName)"
        guard let url = URL(string: urlString) else { return }

        // テストケースが成功していたら、画面録画は削除する
        let json: [String: Any] = ["delete": !isTestFailed(), "stop": stop]

        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.httpBody = try? JSONSerialization.data(withJSONObject: json, options: [])
        URLSession.shared.dataTask(with: request).resume()
    }

    private func isTestFailed() -> Bool {
        if let testRun = testRun {
            let failureCount = testRun.failureCount + testRun.unexpectedExceptionCount
            return failureCount > 0
        }
        return false
    }
}
```

上記のクラスを各テストケースで利用します。

```swift
import XCTest

final class AccountTests: XCTestCaseWithRecording {
    func testLogin() throws {
        let app = XCUIApplication()
        app.launch()

        // ここにテスト処理を実装する
    }
}
```

## 残った録画を CI 実行結果の成果物として残す

サーバーを以下のコマンドで実行します。

```shell
ruby recording_server.rb
```

この状態で XCUITest を実行します。
これにより、 `recording` フォルダーに失敗時の録画が残されます。

`recording` フォルダーをビルドの成果物として残しておけば、原因調査に利用できます。

# 参考

https://testableapple.com/video-recording-of-failed-tests-on-ios/

https://riccardocipolleschi.medium.com/video-record-your-uitest-4cc5b75079af

https://qiita.com/Cychow/items/02c99b9fe8f8b822cca2
