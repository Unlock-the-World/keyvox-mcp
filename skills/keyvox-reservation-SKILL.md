---
name: keyvox-reservation
description: KEYVOX予約の管理業務を行う。新規予約を作成したい、今日や明日の予約一覧を見たい、予約を延長したい、予約内容を変更したい、予約をキャンセルしたい、といった予約レコードを操作するすべての場面で使用する。空き状況の確認や、予約に紐づくPIN/QRコードの取得もこのスキルでカバー。
---

# keyvox-reservation

KEYVOX予約管理5業務を1スキルで対応する。

## ⚠️ 環境前提・再認証・典型エラー

このスキルは **Claude.ai のカスタムコネクタ** 経由でのみ動作します。
401 / E2003 / DCR エラー時の対処、再認証の定型応答、Claude 向けガイドラインは以下に集約しています:

👉 **必読**: [`references/keyvox-mcp-setup.md`](../references/keyvox-mcp-setup.md)

エラー発生時は同ファイル内の「**401 / E2003 エラー時にユーザーへ返す定型応答**」を **そのまま** ユーザーに出力すること（要約・言い換え禁止）。

## 共通リファレンス
スキル本文を読み始める前に、以下の参照ドキュメントを把握しておく:
- `references/keyvox-entities.md` — リソース定義（place / unit / reservation 等）
- `references/keyvox-tool-map.md` — 業務→ツール対応
- `references/keyvox-id-resolution.md` — 部屋名/日付/顧客名 → ID 解決
- `references/keyvox-enums.md` — orderStateCode 等のenum

## ⚠️ 不可逆操作の原則

以下のツールは **必ずユーザーの最終確認を取ってから呼ぶ**:
- `createReservation` — 予約作成
- `updateReservation` — 予約変更・延長
- `cancelReservation` — 予約キャンセル

確認の形式は「対象予約の概要 + 変更内容 + "実行してよいですか？"」を提示し、ユーザーが「はい」「OK」等を返した時のみ実行する。

## シナリオ判別

ユーザー発話から以下のシナリオを判別:

| 発話パターン例 | シナリオ |
|---|---|
| 「明日14時から3時間予約して」「新規予約」 | A. 新規予約作成 |
| 「今日の予約」「明日の予約一覧」 | B. 予約確認 |
| 「101号室の予約を3時間延長」「終了時刻を遅らせて」 | C. 延長 |
| 「予約時間を変更」「部屋を変更」 | D. 変更 |
| 「予約キャンセル」「予約を取り消し」 | E. キャンセル |

---

## A. 新規予約作成

### 手順
1. **必須情報を聞き取る** (不足があれば質問):
   - 日時範囲 (開始/終了)
   - 場所 または 部屋タイプ
   - ゲスト情報 (名前・連絡先)
2. **ID解決** (`references/keyvox-id-resolution.md`):
   - 場所名 → `place_list` → `placeId`
   - 部屋特定が必要なら → `unit_list` または `getUnits` → `unitId`
3. **プラン取得** (必須):
   - `plan_list(placeId)` で利用可能プランを取得
   - 用途に合う `commodityId` / `commodityName` / `priceTax` を選択
4. **空き状況確認**:
   - `place_availableList(placeId, fromTime, toTime)` (場所単位の空き枠取得)
5. **候補をユーザーに提示** (表形式):
   ```
   | # | 部屋 | 時間枠 | 料金/プラン |
   |---|------|--------|-----------|
   | 1 | 101号室 | 14:00-17:00 | スタンダード |
   ```
6. **最終確認** (例):
   ```
   以下で予約を作成します:
   - 部屋: 101号室 (BCLtest)
   - 時間: 5/28 14:00 - 17:00
   - ゲスト: 山田太郎 (070-xxxx-xxxx)
   - プラン: 時間貸しプラン
   実行してよいですか？
   ```
7. **承認後 `createReservation` 実行** — 必須パラメータ:
   - placeId, checkin, checkout (UNIX秒), orderContact, customerNum
   - payTypeCode (デフォルト `offlinePayment`)
   - earlyTime/extendTime (通常 0)
   - **commodityList** (Step 3 で取得した plan を [{commodityId, commodityName, commodityCategory:"Plan", commodityDate:checkin, commodityNum:1, price, priceTax, unitId}] で渡す)
   - unitNum: 1
   - unitList: [{unitId, checkin, checkout, contactName, customerNum}]
8. **結果報告** (レスポンスから自動取得できる):
   ```
   ✅ 予約を作成しました
   - 予約ID: ABCDEFGHI
   - PIN: 123456
   - QR画像URL: https://...
   - 鍵共有URL: shareUrl
   ```
   レスポンスの `unitList[0]` に `pinCode`, `qrCode`, `qrUrl`, `shareUrl` が含まれる

### エラー対応
- 空きゼロ → ±30分・前後日の代替案を提示
- ID解決失敗 → 候補リストを返してユーザーに選択を促す
- `createReservation` がエラー → 原因（重複予約等）を解釈してユーザーへ

---

## B. 予約確認 (今日/明日/任意日)

### 手順
1. **日時範囲を決定**:
   - 「今日」「明日」「来週」等は `keyvox-id-resolution.md` の日付変換ロジックに従い UNIX秒へ
   - 物件の `placeUtc` を考慮
2. **placeId が未指定なら**: `place_list` で取得 (複数物件なら確認)
3. `listReservations(placeId, fromDate, toDate, page=1, count=50, total=1)` を呼ぶ
   - `total=1` でキャンセル・チェックアウト済みを除外（必要に応じ調整）
4. **結果を表で返す**:
   ```
   | # | 時刻 | 部屋 | ゲスト | 状態 |
   |---|------|------|--------|------|
   | 1 | 14:00-17:00 | 101号室 | 山田太郎 | O(チェックイン済) |
   ```
   `orderStateCode` は `keyvox-enums.md` を参照して日本語化
5. 件数が多い場合は要約 + 「詳細は予約番号を教えてください」

---

## C. 予約延長 (checkout時刻の後ろ倒し)

### 手順
1. **対象予約特定**:
   - ユーザー発話から (例: 「101号室の現在の予約」「岡本さんの予約」)
   - `keyvox-id-resolution.md` のパターンで `listReservations` + searchWord で絞り込み
   - 候補複数なら確認
2. `getReservation(placeId, orderId)` で現在の checkout 時刻取得
3. **延長後の時間枠の空き確認** (`place_availableList`)
4. **ユーザー最終確認**:
   ```
   以下に延長します:
   - 予約ID: XXX (101号室 / 山田太郎)
   - 現在: 17:00 → 延長後: 18:00 (+1h)
   実行してよいですか？
   ```
5. **承認後 `update_reservation_checkout(orderId, checkout)` を呼ぶ**
   - ⚠️ `updateReservation` (フル版) は必須パラメータが多くE0005エラーが頻発。**checkout 延長は `update_reservation_checkout` を使うこと**
6. 結果報告

---

## D. 予約変更 (時間シフト/部屋変更)

### ⚠️ 重要な実装方針: cancel + recreate パターン

`updateReservation` は必須パラメータが厳格でE0005エラーが頻発するため、**checkin時刻のシフト・部屋変更・人数変更は「キャンセル + 再作成」で対応** すること。

### 手順
1. **対象予約特定** (Cと同様)
2. **変更内容を聞き取り**: 新時間 / 新部屋 / 新人数
3. **変更先の空き確認** (`place_availableList`)
4. **ユーザー最終確認** (Before/After + cancel+recreateであることを明示):
   ```
   以下に変更します（一度キャンセルして再作成します）:
   - Before: 101号室 14:00-17:00
   - After:  102号室 15:00-18:00
   ※ 元の予約はキャンセルされ、新しい予約IDが発行されます。
     PINも新規発行されるため、ゲストへの再送が必要です。
   実行してよいですか？
   ```
5. **承認後**:
   1. `cancelReservation(placeId, orderId)` で既存予約キャンセル
   2. `createReservation(...)` で新条件の予約作成
6. 結果報告（新orderId, 新PIN, 新QRURL）

---

## E. 予約キャンセル

### 手順
1. **対象予約特定** (Cと同様)
2. **必ずユーザー最終確認** (キャンセルは特に重要):
   ```
   以下をキャンセルします:
   - 予約ID: XXX (101号室 / 山田太郎 / 5/28 14:00-17:00)
   実行してよいですか？
   ⚠️ キャンセル後は元に戻せません
   ```
3. **承認後 `cancelReservation(placeId, orderId)`**
4. 結果報告:
   ```
   ✅ 予約 XXX をキャンセルしました
   - PINも自動で無効化されました（pinStatus=4, delFlag=1）
   ```

**自動効果**:
- 予約に紐づいていた `pin` は自動で無効化される（delFlag=1, pinStatus=4）
- 別途 `disableLockPin` を呼ぶ必要は **ない**

---

## テスト時の安全策

このスキルを実機テストする際の注意:
- **テスト用ユニット (BCLtest 等) のみで実施する**
- 本番予約に影響を与える可能性のあるツール (`createReservation`/`updateReservation`/`cancelReservation`) は、テストアカウントで完結させる
- 新規予約のテスト後は **必ず `cancelReservation` でクリーンアップ** する
- バーチャルロックの場合、実際の物理ドアには影響しない

## トラブルシューティング

| 症状 | 原因 / 対処 |
|---|---|
| `401 Unauthorized` | MCPコネクタの認証期限切れ → claude.aiのconnectorsから再認証 |
| `E2003 権限不足` | OAuth同意時の対象組織が違う → 再認証して正しい組織を選択 |
| `E0043 データが存在しません` | placeId/orderId/unitId のtypo or 他組織のID |
| `E0005 パラメータエラー` (updateReservation) | パラメータ構造が厳格すぎて呼びにくい。延長は `update_reservation_checkout` で、その他変更は **cancel + recreate** パターンに切り替える |
| ID解決で候補ゼロ | `keyvox-id-resolution.md` の曖昧マッチパターンを参照 |

## 実機検証で判明した enum値（references/keyvox-enums.md 更新候補）

| フィールド | 値 | 意味 |
|---|---|---|
| `orderStateCode` | `A` | 新規作成・有効 |
| `orderStateCode` | `C` | キャンセル済み |
| `pinStatus` | `1` | 有効 |
| `pinStatus` | `4` | 無効化（キャンセル等） |
| `channelCode` | `BACS` | API経由作成 |
| `orderSource` | `BACS` | API経由作成 |

これらは references/keyvox-enums.md の正式更新（別PR or 設計フェーズPRへの追記コミット）で反映する。
