---
name: keyvox-checkin-status
description: KEYVOXのチェックイン状況を確認する。今日のゲストが何人到着済みか知りたい、特定部屋のゲストが入ったか確認したい、チェックイン予定時刻を過ぎても入っていない未到着ゲストを抽出したい、といった「予約に対して実際に解錠されたか」を判定する業務で使用。listReservations × getLockHistory の照合がコア。
---

# keyvox-checkin-status

ゲストがチェックインしたか（鍵を取得→実際に解錠したか）を確認するスキル。

## ⚠️ 環境前提・再認証・典型エラー

このスキルは **Claude.ai のカスタムコネクタ** 経由でのみ動作します。
401 / E2003 / DCR エラー時の対処、再認証の定型応答、Claude 向けガイドラインは以下に集約しています:

👉 **必読**: [`references/keyvox-mcp-setup.md`](../../references/keyvox-mcp-setup.md)

エラー発生時は同ファイル内の「**401 / E2003 エラー時にユーザーへ返す定型応答**」を **そのまま** ユーザーに出力すること（要約・言い換え禁止）。

## 共通リファレンス
- `references/keyvox-entities.md` — リソース定義
- `references/keyvox-tool-map.md` — 業務→ツール対応
- `references/keyvox-id-resolution.md` — ID解決パターン
- `references/keyvox-enums.md` — orderStateCode/etype 等のenum
- `skills/keyvox-reservation/SKILL.md` — 予約管理スキル（補完関係）

## このスキルの中核判定ロジック

**「予約に対して解錠イベントがあったか？」** を以下の手順で判定:

```
Step 1. 対象予約から checkin時刻 + unitId を取得 (getReservation または listReservations結果)
Step 2. unitId → lockId に解決 (getUnits を使う / unit_list には lockId が含まれないため getUnits 優先)
Step 3. getLockHistory(lockId) で履歴取得
Step 4. 履歴中に { etype: "9" (解錠) かつ checkin ≦ etime ≦ now } があるか確認
Step 5. あれば「チェックイン済」、なければ「未チェックイン」
```

**重要**: 1つのlockIdに対する getLockHistory は **その lock 全体の履歴** であり、特定予約に紐づかない。予約ごとの時刻ウィンドウでフィルタする必要がある。

## シナリオ判別

| 発話パターン | シナリオ |
|---|---|
| 「今日のチェックイン状況」「何人到着した？」 | A. 本日の状況一覧 |
| 「101号室のゲストは入った？」「岡本さんチェックインした？」 | B. 個別確認 |
| 「未チェックインの人」「まだ来てない予約」 | C. 未チェックイン抽出 |

---

## A. 本日のチェックイン状況一覧

### 手順
1. **placeIdを取得**: `place_list` (複数物件なら確認)
2. **今日の予約一覧取得**:
   ```
   listReservations(placeId, fromDate=今日0:00 JST, toDate=今日23:59 JST,
                    page=1, count=50, total=1)
   ```
   `total=1` でキャンセル・チェックアウト済を除外
3. **各予約のlockId引き当て**:
   - `getUnits()` で unitId → lockIds の辞書を作る
   - 予約の unitId からlockId を取得
4. **解錠履歴取得** (ユニーク lockId ごとに1回ずつ):
   - `getLockHistory(lockId, records=50)` を呼ぶ
   - レスポンスを lockId→history[] の辞書化
5. **各予約に対して照合**:
   - その予約の `unitId → lockId` のhistoryを参照
   - `checkin ≦ etime ≦ now` かつ `etype == "9"` のイベントが1件以上あれば「✅ チェックイン済」
   - なければ「⏳ 未到着」(checkin未来) / 「⚠️ 遅延」(checkin過ぎてるのに未解錠)
6. **表で報告**:
   ```
   ## 本日(MM/DD) BCLtest チェックイン状況
   - 予約総数: 3件  チェックイン済: 1件  未到着: 1件  遅延: 1件
   
   | # | 時刻 | 部屋 | ゲスト | 状態 | 解錠時刻 |
   |---|------|------|--------|------|----------|
   | 1 | 14:00-17:00 | 101 | 山田 | ✅済 | 14:05 |
   | 2 | 15:00-18:00 | 102 | 佐藤 | ⏳予定 | - |
   | 3 | 10:00-13:00 | 103 | 鈴木 | ⚠️遅延 | - |
   ```

---

## B. 個別確認

### 手順
1. **対象予約特定** (`references/keyvox-id-resolution.md` 参照):
   - 「101号室のゲスト」→ unitName から unit解決 → そのunitの今日の予約を listReservations + searchWord 等で絞り込み
   - 「岡本さんの予約」→ listReservations + searchWord="岡本"
2. **予約詳細取得**: `getReservation(placeId, orderId)`
3. **lockId引き当て**: `getUnits()` から該当 unitId のlockId取得
4. **解錠履歴照合**:
   - `getLockHistory(lockId, records=50)`
   - 予約の checkin ≦ etime ≦ checkout (or now) の解錠イベントを抽出
5. **報告**:
   ```
   山田太郎さん (予約ABCDEFGHI / 101号室 14:00-17:00)
   
   状態: ✅ チェックイン済
   - 初回解錠: 14:05 (API経由)
   - 直近解錠: 16:42 (API経由)
   - 解錠回数: 3回
   ```

---

## C. 未チェックイン抽出 (遅延ゲスト)

### 手順
1. **placeId 取得** (Aと同様)
2. **本日の予約一覧** (Aの Step 2-4 と同じく一覧 + lockId引き当て + 履歴取得)
3. **フィルタ**: 各予約に対し以下の条件で抽出:
   - `checkin < 現在時刻` (チェックイン予定時刻を過ぎている)
   - **かつ** `checkin ≦ etime ≦ now` の解錠イベントが0件
   - **かつ** `orderStateCode` が キャンセル(`C`) / チェックアウト済 でない
4. **報告** (件数ゼロなら「現時点で遅延ゲストはいません」):
   ```
   ⚠️ 未チェックインゲスト 2件
   
   | # | 予定時刻 | 経過 | 部屋 | ゲスト | 連絡先 |
   |---|---------|------|------|--------|--------|
   | 1 | 10:00 | +1h30m | 103 | 鈴木 | 070-xxxx |
   | 2 | 11:00 | +30m | 105 | 高橋 | 070-yyyy |
   ```

### 判定の閾値
- デフォルトでは「checkin時刻を1分でも過ぎたら遅延候補」
- 業務要件で「30分以上経過したら警告」等の閾値を設けたい場合、ユーザーに確認してから設定

---

## ⚠️ 注意事項

### lock と unit の多重度
1つの unit に複数 lock が紐づくケース（玄関+部屋扉等）あり。その場合は **全 lock の history を結合** してから判定する。

### タイムゾーン
- 予約の `checkin`/`checkout` は UNIX秒（タイムゾーンレス）
- lockHistory の `etime` も UNIX秒
- 比較はそのまま UNIX秒同士でOK。表示時のみ JST に変換 (物件の `placeUtc` を参照)

### `getLockHistory` のページング
- 50件超を取得したい場合、`position` をレスポンスから取り出して次回引数に渡す
- 通常の業務（今日のチェックイン照合）では50件で十分

### `etype` の意味
- `etype: "9"` = 解錠系イベント（実機検証で確認）
- 他の etype 値は `references/keyvox-enums.md` 参照（未確定値あり）
- 施錠/異常/タイムアウト等の区別が必要な場面では etype の正確な定義をメンターに確認

### `userName` フィールド
- `userName: "API"` → APIまたはQRコードによる自動解錠
- `userName: "Ken Okamoto"` 等の人名 → モバイルアプリ等の認証ユーザー
- 「ゲスト本人がQR/PINで開けた」か「管理者がリモート解錠した」かを区別する場合、`userName` を確認

---

## トラブルシューティング

| 症状 | 対処 |
|---|---|
| `getLockHistory` で履歴ゼロ | lockId が間違っている / そのlockに通信ログが無い (バーチャル等) / 期間内に解錠なし |
| 同じ予約に複数解錠イベント | 正常。出入りで複数回解錠される。「初回解錠」を「チェックイン時刻」と扱うのが慣例 |
| 401 / E2003 | コネクタ認証期限切れ / 対象組織違い (reservationスキルと同様) |
