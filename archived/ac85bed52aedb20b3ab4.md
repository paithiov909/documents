---
ID: ac85bed52aedb20b3ab4
Title: スプレッドシートをJSONにして配信するGASのスニペット
Tags: gas
Author: Kato Akiru
Private: false
---

## 手順

1. 配信したいスプレッドシートを開く
2. ツールからスクリプトエディタを開く
3. 以下のスニペットを貼る
4. 公開からWebアプリケーションとして導入を選ぶ
    1. 「Execute the app as Me」
    2. 「Who has access to the app: Anyone, even anonymous」
    3. Googleにログインしてアプリを認証する

## スニペット

``` js
function getData() {
  const spreadsheet = SpreadsheetApp.getActiveSpreadsheet();
  const sheet1 = spreadsheet.getSheetByName('シート1');
  const range = sheet1.getRange('A1:B979'); // 979行目まで
  const values = range.getValues();
  const whole = values.map(row => {
    let col = 0;
    return {
      id: row[col++],
      body: row[col++]
    }
  });
  const trim = whole.filter(row => row.id !== "");
  const data = trim.reverse();
  return data.slice(0, 30); // 30行だけ
}

function doGet() {
  const data = getData();
  const response = ContentService.createTextOutput();
  response.setMimeType(MimeType.JSON);
  response.setContent(JSON.stringify(data));
  return response;  
}

```
