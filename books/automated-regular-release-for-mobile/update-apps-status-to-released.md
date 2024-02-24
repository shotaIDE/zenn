---
title: "審査が通り公開された際に、データストアにリリース済みとマーク"
---

リリース完了のメール通知が来た際に、Google スプレッドシートを更新します。

# iOS のリリース通知を受信した際に、スプレッドシートに記述

App Store のユーザー設定画面で、「アプリステータスレポート」の通知を受信するように設定しておきます。

https://appstoreconnect.apple.com/access/users

![](/images/automated-regular-release-for-mobile/app-store-notification-settings.png)

Zapier を利用しています。

![](/images/automated-regular-release-for-mobile/zapier-ios-01.png)
![](/images/automated-regular-release-for-mobile/zapier-ios-02.png)

Search String を以下のように設定します。「アプリ名」は App Store Connect でのアプリ名に置き換えてください。

```plaintext:Search String
from:(no_reply@email.apple.com) subject:(The status of your (iOS) app, アプリ名, is now "Ready for Distribution")
```

![](/images/automated-regular-release-for-mobile/zapier-ios-03.png)
![](/images/automated-regular-release-for-mobile/zapier-ios-04.png)

# Android のリリース通知を受信した際に、スプレッドシートに記述

![](/images/automated-regular-release-for-mobile/zapier-android-01.png)
![](/images/automated-regular-release-for-mobile/zapier-android-02.png)
![](/images/automated-regular-release-for-mobile/zapier-android-03.png)
![](/images/automated-regular-release-for-mobile/zapier-android-04.png)

# スプレッドシートの Apps Script

Google スプレッドシートの「変更時」に発動する、Apps Script を利用します。

![](/images/automated-regular-release-for-mobile/spreadsheet-trigger.png)

以下のように、iOS と Android 両方ともリリース通知が揃った場合に、スプレッドシートを更新します。

```javascript:script.gs
const summarySheetName = 'Summary';
const statusRange = 'A2';
const statusValueBothInReleaseProcess = 'iOS と Android の両方ともリリース進行中';
const statusValueOnlyOneSideInReleaseProcess = 'iOS と Android のいずれかがリリース進行中';
const statusValueBothReleased = 'iOS と Android の両方ともリリース済み';

function checkLatestReleaseHasCompleted() {
  const status = getValueInRangeForSheetByName(summarySheetName, statusRange);

  const iosLastRow = getLastRowForSheetByName('iOS');

  const androidLastRow = getLastRowForSheetByName('Android');

  if (iosLastRow !== androidLastRow) {
    console.log(`Last row number is different: iOS = ${iosLastRow}, Android = ${androidLastRow}.`);
    console.log(`Release status is "${status}".`);

    if (status == statusValueBothInReleaseProcess) {
      console.log('The above means only one of either iOS or Android is in release process.');

      setValueInRangeForSheetByName(summarySheetName, statusRange, statusValueOnlyOneSideInReleaseProcess);
    }

    return;
  }

  console.log(`Last row number is same: iOS = ${iosLastRow}, Android = ${androidLastRow}.`);
  console.log(`Release status is "${status}".`);

  if (status == statusValueBothInReleaseProcess) {
    console.log('The above means that both of iOS and Android are in release process.');
    return;
  }

  console.log('The above means that both iOS and Android releases have been completed.');

  const spreadsheet = SpreadsheetApp.getActiveSpreadsheet();
  const iosSheet = spreadsheet.getSheetByName('iOS');
  const iosLastRange = iosSheet.getRange(iosLastRow, 1);
  const iosLastReleaseMetaData = iosLastRange.getValue();
  const iosVersionNameRegex = /App Version Number: ([0-9]+\.[0-9]+\.[0-9]+)/;
  const matched = iosLastReleaseMetaData.match(iosVersionNameRegex);

  if (!matched && matched.length <= 1) {
    console.log(`Failed to extract version name from meta data: "${iosLastReleaseMetaData}".`);
  }

  const versionName = matched[1];
  console.log(`Found "${versionName}" release.`);

  setValueInRangeForSheetByName(summarySheetName, statusRange, statusValueBothReleased);
}

function getLastRowForSheetByName(sheetName) {
  const spreadsheet = SpreadsheetApp.getActiveSpreadsheet();
  const sheet = spreadsheet.getSheetByName(sheetName);
  return sheet.getLastRow();
}

function getValueInRangeForSheetByName(sheetName, range) {
  const spreadsheet = SpreadsheetApp.getActiveSpreadsheet();
  const sheet = spreadsheet.getSheetByName(sheetName);
  const targetRange = sheet.getRange(range);
  return targetRange.getValue();
}

function setValueInRangeForSheetByName(sheetName, range, value) {
  const spreadsheet = SpreadsheetApp.getActiveSpreadsheet();
  const sheet = spreadsheet.getSheetByName(sheetName);
  const targetRange = sheet.getRange(range);
  targetRange.setValue(value);
}
```
