# KEYVOX 業務シナリオ → MCPツール対応表

各業務シナリオで使う MCPツールの組み合わせ。スキル設計時にこの表から逆引きする。

## 表の見方

- **主ツール**: そのシナリオで必ず叩くツール
- **補助ツール**: 必要に応じて追加で叩くツール（ID解決・詳細取得など）
- **前提**: そのシナリオを実行するために事前に必要な情報・条件
- **注意点**: エラーになりやすい点・特殊な仕様

## 9 業務シナリオ

### 1. 今日の予約確認（朝のルーチン）
- **主ツール**: `listReservations` (date filter, `fromDate`/`toDate` を今日の0:00〜23:59 UNIX秒で指定)
- **補助**: `getReservation` で個別予約の詳細
- **前提**: placeId（複数物件運営ならplace_listから選択）
- **注意**: `total=1` を指定するとキャンセル・チェックアウト済を除外できる

### 2. チェックイン状況の確認（鍵取得済みか）
- **主ツール**: `getReservation` で `unitPinList.qrCode` の存在を確認
- **補助**: `getLockHistory` でユーザーが実際に解錠したか確認
- **前提**: orderId（listReservationsから取得）
- **注意**: PINが発行されていても解錠していない＝鍵は受け取ったが利用未開始

### 3. 実利用の確認（解錠イベント）
- **主ツール**: `getLockHistory` (etypeで解錠系イベントをフィルタ、enums.md参照)
- **補助**: `getUnits` で lockId 取得（`unit_list` では lockId が取れないため）
- **前提**: lockId
- **注意**: `userName="API"` は自動解錠（QRコード/リモート）の場合

### 4. 滞在中の鍵忘れ・閉じ込め対応
- **主ツール**:
  - 鍵忘れ → `getReservation` で `unitPinList.qrUrl` を取得 → ゲストに再送
  - または `createLockPin` で追加PIN発行
  - 閉じ込め緊急時 → `unlock` で直接遠隔解錠
- **前提**: orderId または lockId
- **注意**: `createLockPin` は利用開始日時の72時間前から発行可能。`getLockPinStatus` で配信完了を確認してから使うこと

### 5. 予約延長
- **主ツール**: `updateReservation` (checkout時刻を後ろにずらす) または `update_reservation_checkout`
- **前提**: orderId, 新checkout時刻 (UNIX秒)
- **注意**: 既存PINの有効期限も同時に延長される仕様か要確認

### 6. 事前変更
- **主ツール**: `updateReservation`
- **前提**: orderId と変更内容
- **注意**: チェックイン前のみ変更可能（業務ルール上）

### 7. キャンセル
- **主ツール**: `cancelReservation`
- **前提**: orderId
- **注意**: キャンセル後の status 確認は `getReservation` で `orderStateCode` を見る

### 8. 清掃タイミング判定（今日の利用件数・清掃可能時間）
- **主ツール**: `listReservations` で今日チェックアウト予定を取得
- **補助**: `getLockHistory` で最終解錠時刻を確認（ゲストがまだ滞在中なら清掃NG）
- **前提**: placeId, 今日の日付範囲
- **判定ロジック**:
  1. 今日のchekout予定一覧 → `livedDays > 0` でフィルタ（実利用済み）
  2. 各ユニットの `getLockHistory` から最終解錠時刻取得
  3. 最終解錠時刻 + 30分 以降が清掃開始可能時刻

### 9. 新規予約作成（空き時間確認 + Reservation API Insert）
- **主ツール**: `place_availableList` で空き確認 → `createReservation`
- **補助**: `getUnits` で unitId 取得
- **前提**: placeId, 希望日時範囲, 顧客情報
- **注意**:
  - `place_availableList` の返却単位（分・時間）は要検証
  - 作成後、自動でPINが発行される（`getReservation` の `unitPinList` で確認可能）

## ツール別呼び出し頻度（参考）

| ツール | 使用シナリオ数 | 用途 |
|---|---|---|
| `listReservations` | 4 | 一覧系の基本 |
| `getReservation` | 4 | 詳細・PIN情報取得 |
| `getLockHistory` | 3 | 利用実績確認 |
| `getUnits` | 3 | lockId引き当て |
| `place_list` | (前段) | placeId引き当て |
| `createLockPin` | 1 | 鍵忘れ対応のみ |
| `unlock` | 1 | 閉じ込め緊急時のみ |
| `cancelReservation` | 1 | キャンセル |
| `place_availableList` | 1 | 空き時間確認 |
| `createReservation` | 1 | 新規予約 |

## スキル割当て

| スキル | カバー業務 |
|---|---|
| `keyvox-reservation` | 1, 5, 6, 7, 9 |
| `keyvox-checkin-status` | 2, 3 |
| `keyvox-onsite-support` | 4 |
| `keyvox-housekeeping` | 8 |

## 関連ドキュメント
- リソース定義: `keyvox-entities.md`
- ID解決パターン: `keyvox-id-resolution.md`
- enum値（orderStateCode等）: `keyvox-enums.md`
