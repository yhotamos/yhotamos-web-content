---
id: dtn7h7d6ujt
title: Next.jsでGoogleSheetsAPIを使ってデータを取得するための手順
date: "2025-08-25"
updated: "2025-08-25"
category: blog
thumbnail: ""
tags: [Next.js, Google Sheets API, GCP]
slug: blog/2025-08-25-Next.jsでGoogleSheetsAPIを使ってデータを取得するための手順
lang: ja
---

Next.js で Google Sheets API を使ってデータを取得するための手順を，**GCP のサービスアカウントを利用する方法**でまとめます．
以下の手順は，サーバーサイド（API Route や`getServerSideProps`）でのフェッチを前提としています．

---

## 手順概要

1. **GCP プロジェクトを作成**
2. **Google Sheets API を有効化**
3. **サービスアカウントを作成・秘密鍵（JSON）を取得**
4. **スプレッドシートの共有設定（サービスアカウントメールを編集者または閲覧者に追加）**
5. **Next.js プロジェクトにライブラリ導入（googleapis）**
6. **API Route またはサーバーサイドで認証してデータ取得**
7. **環境変数で秘密情報を管理**
8. **API Route（サーバーサイドで Sheets API を呼び出す）**
9. **フロントエンドから fetch()する**
10. **おまけ なぜ /api/sheet 経由なのか？**

---

## 詳細手順

### 1. GCP プロジェクトを作成

- [Google Cloud Console](https://console.cloud.google.com/)にアクセスし，新規プロジェクトを作成します．
- プロジェクト名を設定し，作成します．

---

### 2. Google Sheets API を有効化

- GCP コンソールの「API とサービス」>「ライブラリ」から **Google Sheets API** を検索し，有効化します．

---

### 3. サービスアカウントを作成・秘密鍵（JSON）を取得

1. 「IAM と管理」>「サービスアカウント」>「サービスアカウントを作成」．
2. 名前を設定し，「完了」まで進める．
3. 作成したサービスアカウントの「キー」タブから **新しい鍵 > JSON 形式** を選択してダウンロード．
4. この JSON ファイルには以下のような情報が含まれます．

   ```json
   {
     "type": "service_account",
     "project_id": "your-project-id",
     "private_key_id": "xxxxxxxx",
     "private_key": "-----BEGIN PRIVATE KEY-----\n....\n-----END PRIVATE KEY-----\n",
     "client_email": "your-service-account@your-project-id.iam.gserviceaccount.com",
     "client_id": "xxxxxx",
     ...
   }
   ```

---

### 4. スプレッドシートの共有設定

- 対象の Google スプレッドシートを開く．
- 「共有」設定から，サービスアカウントの **`client_email`（例: `xxxx@project-id.iam.gserviceaccount.com`）** を「閲覧者（または編集者）」として追加．

---

### 5. Next.js プロジェクトにライブラリ導入

```bash
npm install googleapis
```

または

```bash
yarn add googleapis
```

---

### 6. API Route またはサーバーサイドで認証してデータ取得

**例：`pages/api/sheet.js`**

```ts
import { google } from "googleapis";

export default async function handler(req, res) {
  try {
    // サービスアカウント認証
    const auth = new google.auth.JWT(process.env.GOOGLE_CLIENT_EMAIL, undefined, process.env.GOOGLE_PRIVATE_KEY.replace(/\\n/g, "\n"), [
      "https://www.googleapis.com/auth/spreadsheets.readonly",
    ]);

    const sheets = google.sheets({ version: "v4", auth });

    // スプレッドシートからデータ取得
    const response = await sheets.spreadsheets.values.get({
      spreadsheetId: process.env.GOOGLE_SHEET_ID,
      range: "Sheet1!A1:C10", // 範囲指定
    });

    res.status(200).json({ data: response.data.values });
  } catch (error) {
    console.error(error);
    res.status(500).json({ error: "Failed to fetch data" });
  }
}
```

---

### 7. 環境変数で秘密情報を管理（秘密鍵 JSON から設定）

ダウンロードしたサービスアカウントの秘密鍵 JSON ファイルから，以下のキーの値を抜き出して `.env.local` に設定します．

- **`client_email` → `GOOGLE_CLIENT_EMAIL`**
- **`private_key` → `GOOGLE_PRIVATE_KEY`**

**例：`.env.local`**

```env
GOOGLE_CLIENT_EMAIL="your-service-account@project-id.iam.gserviceaccount.com"
GOOGLE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\nYOUR_KEY\n-----END PRIVATE KEY-----\n"
GOOGLE_SHEET_ID="スプレッドシートのID"
```

> **注意:** > `private_key` の改行はそのままでは `.env` に書けないため，`\n` に置き換える必要があります（ダブルクォーテーション内で `\n` を改行コードとしてエスケープ）．
> 実行時には `replace(/\\n/g, '\n')` を使用して復元します．

**例：復元コード**

```ts
const auth = new google.auth.JWT(process.env.GOOGLE_CLIENT_EMAIL, undefined, process.env.GOOGLE_PRIVATE_KEY.replace(/\\n/g, "\n"), [
  "https://www.googleapis.com/auth/spreadsheets.readonly",
]);
```

---

### 8. API Route（サーバーサイドで Sheets API を呼び出す）

`pages/api/sheet.ts`

```ts
import { google } from "googleapis";
import type { NextApiRequest, NextApiResponse } from "next";

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  try {
    const auth = new google.auth.JWT(process.env.GOOGLE_CLIENT_EMAIL, undefined, process.env.GOOGLE_PRIVATE_KEY?.replace(/\\n/g, "\n"), [
      "https://www.googleapis.com/auth/spreadsheets.readonly",
    ]);

    const sheets = google.sheets({ version: "v4", auth });

    const response = await sheets.spreadsheets.values.get({
      spreadsheetId: process.env.GOOGLE_SHEET_ID,
      range: "Sheet1!A1:C10",
    });

    res.status(200).json({ data: response.data.values });
  } catch (error) {
    console.error("Sheets API Error:", error);
    res.status(500).json({ error: "Failed to fetch data" });
  }
}
```

---

### 9. フロントエンドから fetch()する

例えば，`pages/index.tsx` 内で以下のように書きます：

```tsx
import { useEffect, useState } from "react";

export default function Home() {
  const [sheetData, setSheetData] = useState<string[][]>([]);

  useEffect(() => {
    async function fetchData() {
      try {
        const res = await fetch("/api/sheet");
        const json = await res.json();
        setSheetData(json.data || []);
      } catch (error) {
        console.error("Fetch Error:", error);
      }
    }
    fetchData();
  }, []);

  return (
    <div>
      <h1>Google Sheets Data</h1>
      <ul>
        {sheetData.map((row, i) => (
          <li key={i}>{row.join(" , ")}</li>
        ))}
      </ul>
    </div>
  );
}
```

---

### 10. おまけ なぜ /api/sheet 経由なのか？

- **秘密鍵 (`GOOGLE_PRIVATE_KEY`) をフロントに直接埋め込まないため**．
- Next.js の API Routes はサーバーサイドで動作し，`.env.local` に定義した秘密情報を安全に扱える．
- フロントエンドは `/api/sheet` を `fetch()` するだけで Google Sheets のデータを取得できる．
