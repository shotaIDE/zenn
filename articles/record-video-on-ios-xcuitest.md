---
title: "iOSでE2Eテストの自動実行環境を構築する際、失敗したテストケースのキャプチャ動画を残す"
emoji: "📸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dependabot", "gradle", "android"]
published: true
---

CI 環境で E2E テストを自動実行する際に、テストがコケた原因を調査するための情報をできるだけ多く用意しておくことは重要です。
特に、テストがコケた際のキャプチャ動画は、強力な助けとなります。

Firebase Test Lab や Bitrise などを利用すると自動でキャプチャ動画を収集してくれる機能があります。
ただ、GitHub Actions や Circle CI などの利用している場合は、自前でその環境を構築する必要があります。

本記事ではその方法を解説します。

# 前提

XCUITest を用いて E2E テストケースを実装している状況を想定しています。

また、CI 環境としては CircleCI を想定しています。

iOS の Simulator には、Simulator を動作させている Mac から画面のキャプチャ動画を撮影できる機能があります。

```shell
xcrun simctl io booted recordVideo "test-video.mov"
```

今回はこの機能を利用して、テストケース実行時のキャプチャを残します。

# 概要

Simulator を実行するマシンで、キャプチャ動画を録画開始・終了するサーバーを立てます。

テストケースの開始時に、サーバーに録画開始を指示し、テストケースの終了時に、サーバーに録画終了を指示します。
この際、テストケースが成功した場合録画を破棄し、失敗した場合録画を残します。

残った録画を CI 実行結果の成果物として残します。

# 詳細

## キャプチャ動画を録画開始・終了するサーバーを立てる

サーバーのスクリプトはどのような技術でも問題ないですが、今回は Ruby を用いたスクリプトを紹介します。

```ruby:recording_server.rb
require 'fileutils'
require 'sinatra'

# Server to start and stop recording screen of `XCUITest` running on Simulator on Mac.
#
# Called before the start and after the end of each test case in XCUITest,
# leaving the screen recording only if it fails.
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
    puts `xcrun simctl io #{params['udid']} recordVideo --codec h264 --force #{video_file} &`
  end
end
```

スクリプトのポイントは以下です。

- Simulator から HTTP 通信を受信するサーバーを立てる
- サーバーのプロセスで Simulator に対するコマンド `xcrun simctl` を実行する

## テストケースの開始・終了時に、ローカルサーバーに録画の指示を出す

# 参考

https://testableapple.com/video-recording-of-failed-tests-on-ios/

https://riccardocipolleschi.medium.com/video-record-your-uitest-4cc5b75079af

https://qiita.com/Cychow/items/02c99b9fe8f8b822cca2
