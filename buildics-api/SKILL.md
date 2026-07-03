---
name: buildics-api
description: >-
  BUILDICS Restful API (Ver 1.0.1, 2025-11-17) を使ってビル資産・空間・デバイス・
  アラーム・不具合履歴などを取得するアプリを実装するためのルール。
  ユーザーが BUILDICS、Buildics、ビルディックス、資産データ、querySpaceInfo、
  queryAssetInfo、queryAlarmDevice、APゲートウェイ、udfBuildingId、`/common/` で
  始まる API などに言及した場合や、`Apikey` ヘッダーで認証する Restful API を
  扱うコードを書く際に適用する。
---

# BUILDICS Restful API ルール

BUILDICS 向けデータプラットフォームの Restful API を呼び出すアプリ
(Web/Node/フロント/モバイル) を実装するときの共通ルール。
出典: `BUILDICS® Restful API仕様書 Ver 1.0.1 (2025-11-17)`。

## 1. サーバーと認証

- **ベース URL**: `{Host}` はプラットフォームから提供される値をそのまま使う。
  ハードコードせずに **環境変数 (`BUILDICS_API_HOST` など) から読む**こと。
- **認証**: すべての API は `Apikey` ヘッダーで認証する。値は
  プラットフォームから発行される API Key をそのまま設定する。
- **メソッド**: 仕様書記載のすべての API は **POST** + `raw/json`。
  GET は使わない。
- **必須ヘッダー**:

```
Content-Type: application/json;charset=UTF-8
Apikey: <API KEY>
```

- **セキュリティ**:
  - `Apikey` を **ソースコードにハードコードしない**。`process.env` や
    Cloud Functions のシークレットから読み取る (ワークスペースの
    `Security-1` ルールに従う)。
  - ブラウザから直接呼ぶ構成では CORS 問題が出るので、
    必要に応じて Cloud Functions / Nginx のプロキシ経由にすること
    (AiMeet と同じ運用)。

## 2. 共通レスポンス構造

```json
{
  "code": 200,
  "msg": "success",
  "data": [ ... ]
}
```

| code  | 意味                       |
|-------|----------------------------|
| 200   | 成功                       |
| 500   | サーバ内部エラー           |
| 20001 | 照会エラー (パラメータ不正等) |

- **必ず `code` を確認してからデータを使う**。`200` 以外は `msg` を
  ログに出して呼び出し側にエラーを伝える。
- `data` は **配列または単一オブジェクト**。仕様書に「Array」と書かれて
  いるものは空配列が返る可能性があるので null/長さを確認。

## 3. 主要パラメータ・識別子

頻出する ID はフィールド名を変更しない。**仕様書の綴りをそのまま使う**。

| フィールド       | 型     | 意味                                  |
|------------------|--------|---------------------------------------|
| `udfBuildingId`  | String | ビル ID (キー)。ほぼ全 API で必須     |
| `udfSpaceId`     | String | 空間記号 (`querySpaceInfo`)           |
| `udfAssetId`     | String | カスタム資産 ID                       |
| `symbol`         | String | 資産記号 (`queryAssetInfo` の必須)    |
| `assetSymbol`    | String | 資産記号 (`problem-reports` 等)       |
| `deviceId`       | String | デバイス ID                           |
| `deviceSn`       | String | デバイスシリアル                      |
| `Apikey`         | Header | 認証キー                              |

### 時刻フィールドは「ミリ秒の Unix タイムスタンプ」

`latestDataTime`, `receive_ts`, `latestHeartbeatTs`, `startTime`, `endTime`
は **秒ではなくミリ秒**。`Date.now()` をそのまま使えるが、Python 等で
実装する場合は `int(time.time() * 1000)` を使うこと。

- `queryDeviceData` で最新データだけを取る場合は、実測上 `startTime` / `endTime`
  を指定せず、リクエスト body を **配列**にして `{ deviceId }` を送る。
  期間指定が必要なケースだけ `startTime` / `endTime`（7 日以内）を使う。

`occuredAt` 等の日付文字列は `^\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}$` 形式。

## 4. API 一覧 (13 エンドポイント)

| #  | パス                                            | 用途                                         |
|----|-------------------------------------------------|----------------------------------------------|
| 1  | `/common/queryAssetInfo`                        | 資産データ 1 件照会 (`symbol` 必須)          |
| 2  | `/common/querySpaceInfo`                        | 空間 + 紐づく資産一覧                        |
| 3  | `/common/queryAssetInfoByClass`                 | 大/中/小分類で資産を検索                     |
| 4  | `/common/queryAlarmDevice`                      | 異常デバイス (警報中) の一覧                 |
| 5  | `/common/queryClass`                            | ビル内の資産分類ツリー                       |
| 6  | `/common/problem-reports/summaries`             | 不具合 + 修繕履歴                            |
| 7  | `/common/getS3FileUrl`                          | ファイルキー → S3 署名付き URL 変換          |
| 8  | `/common/addBuilding`                           | ビル情報追加                                 |
| 9  | `/common/queryBuilding`                         | ビル情報照会                                 |
| 10 | `/common/queryKanriRoidMaintenanceRecord`       | 管理ロイド修繕履歴 (別名 `/datacenter/v1/sgs/query`) |
| 11 | `/common/queryCancelAlarmDevice`                | 警報解除履歴                                 |
| 12 | `/common/apgateway/status`                      | AP ゲートウェイのオンライン状態              |
| 13 | `/common/device/queryDeviceData`                | デバイスの最新/期間データ                    |

**注意**: 仕様書本文では「3.5 資産分類の照会」のパスが表中で
`queryAlarmDevice` と誤記されているが、正しくは **`/common/queryClass`** で
ある (本文に記載のパスを優先)。

## 5. ファイル (画像・PDF) の扱い

- `imageInfo[]` / `pdfList[]` は **直接 URL ではなく `imageKey` / `fileKey` を
  返す**。
- 実際にダウンロードするには `/common/getS3FileUrl` に
  `{ "keys": ["<key1>", "<key2>", ...] }` を投げて、返された
  `data[].url` (S3 署名付き URL) を使う。
- 取得した URL は **有効期限がある** ので、毎回都度取得し、DB 等に
  永続化しない。
- `imageInfo` の最大件数は **3**、`querySpaceInfo` の `pdfList` は
  建築設備・電気設備・給排気設備・衛生設備・消防設備・その他の **6 種類**
  という点を UI 設計時に考慮する。

## 6. エラーハンドリングと運用

- `code !== 200` のときは `msg` をアプリログに出し、ユーザー向けには
  汎用メッセージ (「データ取得に失敗しました」) を表示する。
  (`Security-3` ルール: 内部エラーを直接ユーザーに見せない。)
- `Apikey` 関連のエラー (401/403 相当) は リトライしても成功しない
  ので、即座に失敗扱いにする。
- `queryDeviceData` の「7 日以内」の制約を超えた場合は 20001 が返る。
  アプリ側でバリデーションしてから呼ぶ。
- 一覧系 API (`queryAssetInfoByClass`, `queryAlarmDevice`,
  `apgateway/status` 等) は仕様書にページング指定がないので、大きな
  ビルでは **データ量が増える**。受信後にフロントでフィルタする設計に
  する。

## 7. 呼び出しサンプル

### curl (資産データ照会)

```bash
curl -X POST "${BUILDICS_API_HOST}/common/queryAssetInfo" \
  -H "Content-Type: application/json;charset=UTF-8" \
  -H "Apikey: ${BUILDICS_API_KEY}" \
  -d '{
    "udfBuildingId": "BLD001",
    "symbol": "AC-101"
  }'
```

### JavaScript / TypeScript (fetch)

```typescript
type BuildicsResponse<T> = { code: number; msg: string; data: T };

async function callBuildics<TReq, TRes>(
  path: string,
  body: TReq,
): Promise<TRes> {
  const host = process.env.BUILDICS_API_HOST;
  const apiKey = process.env.BUILDICS_API_KEY;
  if (!host || !apiKey) throw new Error("BUILDICS host / apikey not configured");

  const res = await fetch(`${host}${path}`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json;charset=UTF-8",
      Apikey: apiKey,
    },
    body: JSON.stringify(body),
  });
  const json = (await res.json()) as BuildicsResponse<TRes>;
  if (json.code !== 200) {
    console.error("Buildics API error:", json.code, json.msg);
    throw new Error(`Buildics API ${path} failed: ${json.msg}`);
  }
  return json.data;
}

const assets = await callBuildics("/common/queryAssetInfo", {
  udfBuildingId: "BLD001",
  symbol: "AC-101",
});
```

### ファイルキー → URL 解決の流れ

```typescript
const space = await callBuildics<_, SpaceInfo[]>("/common/querySpaceInfo", {
  udfBuildingId: "BLD001",
  udfSpaceId: "ROOM-101",
});
const keys = space.flatMap(s => [
  ...s.imageInfo.map(i => i.imageKey),
  ...s.pdfList.map(p => p.fileKey),
]);
if (keys.length > 0) {
  const urls = await callBuildics<_, { key: string; url: string }[]>(
    "/common/getS3FileUrl",
    { keys },
  );
}
```

### デバイス最新データ

最新データを取得する場合、body はオブジェクトではなく **配列**で送る。
`startTime` / `endTime` は不要。

```typescript
const rows = await callBuildics("/common/device/queryDeviceData", [
  { deviceId: "350976658106031" },
]);
```

実レスポンス例:

```json
{
  "code": 200,
  "msg": "Success",
  "data": [
    {
      "deviceId": "350976658106031",
      "deviceSn": "350976658106031",
      "latestRawData": "{\"350976658106031_temp\":25.54,\"350976658106031_hum\":40.10}",
      "latestDataTime": 1778569833000,
      "typeUnit": null,
      "dataValue": "25.54,40.10"
    }
  ]
}
```

温湿度センサーの場合、`latestRawData` は JSON 文字列なので `JSON.parse`
して、`${deviceId}_temp` / `${deviceId}_hum` を読む。

```typescript
const row = rows[0];
const raw = JSON.parse(row.latestRawData) as Record<string, number>;
const temperature = raw[`${row.deviceId}_temp`];
const humidity = raw[`${row.deviceId}_hum`];
```

## 8. 実装時チェックリスト

- [ ] `Apikey` を環境変数/シークレット経由で注入した
- [ ] `Content-Type: application/json;charset=UTF-8` を指定した
- [ ] `code !== 200` の分岐 (500 / 20001 / その他) を書いた
- [ ] ミリ秒タイムスタンプで送受信している (秒ではない)
- [ ] `queryDeviceData` の最新取得は body を配列 (`[{ deviceId }]`) にしている
- [ ] 温湿度は `latestRawData` の JSON 文字列から `${deviceId}_temp` / `${deviceId}_hum` を読む
- [ ] 期間指定で `queryDeviceData` を使う場合だけ、7 日以内にクリップしている
- [ ] 画像/PDF 表示時は `getS3FileUrl` で都度 URL 化している
- [ ] ブラウザ直叩きで CORS が出る場合は、プロキシ経由に切り替えた
- [ ] 仕様書のフィールド名 (`udfBuildingId`, `assetSymbol`, `imageKey` 等)
      を変更せずそのまま使っている

## 9. 追加資料

- 一次情報: `C:\Users\兪燮風\Documents\Bulidicsー API仕様書20251117.docx`
  (Ver 1.0.1, 2025-11-17)。レスポンスの全フィールドや細かい挙動が不明な
  ときは、この docx を再抽出して 3.1〜3.13 の該当セクションを確認する。
