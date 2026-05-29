---
name: keyvox-checkin-status
description: KEYVOXのチェックイン状況を確認する。今日のゲストが何人チェックイン済みか知りたい、特定予約・特定ゲストがチェックインしたか確認したい、チェックイン予定時刻を過ぎてもステータスが未チェックインのままの予約を抽出したい、といった「予約のチェックインステータス(orderStateCode)を確認する」業務で使用する。チェックイン判定は予約ステータスで行い、ロックの解錠履歴では判定しない（解錠＝実利用の判定は keyvox-housekeeping の領域）。
---

# keyvox-checkin-status

ゲストがチェックインしたか（＝予約の **`orderStateCode`** が `I` に遷移したか）を確認するスキル。

## ⚠️ 最重要：チェックイン判定は「予約ステータス」で行う

**チェックイン = 予約レコードの `orderStateCode`。ロックの解錠履歴とは無関係。**

| 判定したいこと | 正しい指標 | 使うツール |
|---|---|---|
| **チェックインしたか**（このスキル） | `orderStateCode`（`A`未／`I`済／`O`チェックアウト済） | `listReservations` / `getReservation` |
| 実際に解錠したか／ノーショウか（別概念） | `getLockHistory` の解錠イベント（etype=9） | `keyvox-housekeeping` を参照 |

> ❌ **やってはいけない**: チェックインしたかを `getLockHistory`（解錠履歴）で判定すること。
> 「鍵は開けたが `orderStateCode` は `A` のまま」「`I` に遷移済みだが解錠はまだ」というケースが普通に発生するため、解錠の有無でチェックインを判定すると誤る。
> このスキルでは **`getLockHistory` を呼ばない。**

### `orderStateCode` の意味（`references/keyvox-enums.md` 準拠）

| 値 | 意味 | このスキルでの扱い |
|---|---|---|
| `Q` | 予約申込 | 未確定 |
| `A` | 未チェックイン | ⏳ 未チェックイン（予定前 or 遅延） |
| `I` | チェックイン済み | ✅ チェックイン済み |
| `O` | チェックアウト済み | 退出済み |
| `C` | キャンセル済み | 集計対象外 |
| `B` | 予約拒否 | 集計対象外 |
| `M` | メンテ中 | 集計対象外 |

### 補足フィールド（`getReservation` 取得時）
- `realCheckin`（UNIX秒）: 実際にチェックイン処理が行われた時刻。`0` は未チェックイン。`orderStateCode=I` と整合する想定（要検証）。
- `realCheckout`（UNIX秒）: 実際のチェックアウト時刻。`0` は未チェックアウト。
- これらは「いつチェックインしたか」を時刻で知りたいときの補助。状態の正否はあくまで `orderStateCode` が一次情報。

### ⚠️ 運用上の前提（重要）
`orderStateCode` が `A→I` に遷移するのは、以下いずれかでチェックイン処理が実行されたとき:
- `checkin(placeId, orderId, unitId)` ツール呼び出し
- タブレット／セルフチェックイン／チャネル連携によるチェックイン

**運用としてチェックイン処理を一切行っていない現場では、ステータスは恒常的に `A` のまま**になる。
その場合「チェックイン状況」を `orderStateCode` で見ても全件 `A` になるため、
- ユーザーに「この物件はチェックイン処理を運用していますか？」を確認するか、
- 「到着確認」を別途やりたいなら実利用判定（`keyvox-housekeeping` の解錠照合）を使う、
のどちらかに切り替える。判定基準を取り違えないこと。

## ⚠️ 環境前提・再認証・典型エラー

このスキルは **Claude.ai のカスタムコネクタ** 経由でのみ動作します。
401 / E2003 / DCR エラー時の対処、再認証の定型応答、Claude 向けガイドラインは以下に集約しています:

👉 **必読**: [`references/keyvox-mcp-setup.md`](../references/keyvox-mcp-setup.md)

エラー発生時は同ファイル内の「**401 / E2003 エラー時にユーザーへ返す定型応答**」を **そのまま** ユーザーに出力すること（要約・言い換え禁止）。

## 共通リファレンス
- `references/keyvox-entities.md` — リソース定義
- `references/keyvox-tool-map.md` — 業務→ツール対応
- `references/keyvox-id-resolution.md` — ID解決パターン
- `references/keyvox-enums.md` — orderStateCode 等のenum
- `skills/keyvox-reservation-SKILL.md` — 予約管理スキル（補完関係）
- `skills/keyvox-housekeeping-SKILL.md` — 実利用/ノーショウ判定（解錠照合はこちら）

## このスキルの中核判定ロジック

```
Step 1. listReservations で対象範囲（今日 等）の予約を取得
Step 2. 各予約の orderStateCode を読む
Step 3. A=未チェックイン / I=チェックイン済み / O=チェックアウト済み に分類
Step 4. 未チェックイン(A)のうち checkin < 現在時刻 のものを「遅延」候補とする
```

**解錠履歴は参照しない。** チェックイン判定は orderStateCode のみで完結する。

## シナリオ判別

| 発話パターン | シナリオ |
|---|---|
| 「今日のチェックイン状況」「何人チェックインした？」 | A. 本日の状況一覧 |
| 「101号室の予約チェックインした？」「岡本さんチェックインした？」 | B. 個別確認 |
| 「未チェックインの人」「予定過ぎてるのに未チェックインの予約」 | C. 未チェックイン抽出（遅延） |

---

## A. 本日のチェックイン状況一覧

### 手順
1. **placeIdを取得**: `place_list`（複数物件なら確認）
2. **今日の予約一覧取得**:
   ```
   listReservations(placeId, fromDate=今日0:00 JST, toDate=今日23:59 JST,
                    page=1, count=50, total=1)
   ```
   `total=1` でキャンセル・チェックアウト済を除外（必要に応じ `total=0` で全件）
3. **各予約の `orderStateCode` を分類**:
   - `I` → ✅ チェックイン済み
   - `A` かつ `checkin > 現在時刻` → ⏳ 予約開始前
   - `A` かつ `checkin ≦ 現在時刻` → ⚠️ 遅延（予定を過ぎても未チェックイン）
   - `O` → 退出済み
4. **表で報告**（コードは日本語化）:
   ```
   ## 本日(MM/DD) BCLtest チェックイン状況
   - 予約総数: 3件  チェックイン済(I): 1件  予約開始前: 1件  遅延: 1件

   | # | 時刻 | 部屋 | ゲスト | ステータス |
   |---|------|------|--------|-----------|
   | 1 | 14:00-17:00 | 101 | 山田 | ✅ チェックイン済(I) |
   | 2 | 15:00-18:00 | 102 | 佐藤 | ⏳ 予約開始前(A) |
   | 3 | 10:00-13:00 | 103 | 鈴木 | ⚠️ 遅延(A) |
   ```
   ※ `orderStateCode` は `references/keyvox-enums.md` を参照して日本語化する

---

## B. 個別確認

### 手順
1. **対象予約特定**（`references/keyvox-id-resolution.md` 参照）:
   - 「101号室の予約」→ unitName から unit解決 → そのunitの今日の予約を listReservations + searchWord で絞り込み
   - 「岡本さんの予約」→ listReservations + searchWord="岡本"
2. **予約詳細取得**: `getReservation(placeId, orderId)`
3. **`orderStateCode` を読む**（必要なら `realCheckin` で時刻も確認）
4. **報告**:
   ```
   山田太郎さん (予約ABCDEFGHI / 101号室 14:00-17:00)

   ステータス: ✅ チェックイン済み (orderStateCode=I)
   チェックイン時刻: 14:05 (realCheckin)
   ```
   未チェックインなら:
   ```
   ステータス: ⏳ 未チェックイン (orderStateCode=A)
   ※ チェックイン予定 14:00（現在 13:30 / 予約開始前）
   ```

---

## C. 未チェックイン抽出（遅延）

### 手順
1. **placeId 取得**（Aと同様）
2. **本日の予約一覧**（Aの Step 2）
3. **フィルタ**: 以下を満たす予約を抽出:
   - `orderStateCode == A`（未チェックイン）
   - **かつ** `checkin < 現在時刻`（チェックイン予定時刻を過ぎている）
4. **報告**（件数ゼロなら「現時点で遅延（未チェックイン）の予約はありません」）:
   ```
   ⚠️ 予定時刻を過ぎても未チェックイン 2件

   | # | 予定時刻 | 経過 | 部屋 | ゲスト | 連絡先 |
   |---|---------|------|------|--------|--------|
   | 1 | 10:00 | +1h30m | 103 | 鈴木 | 070-xxxx |
   | 2 | 11:00 | +30m | 105 | 高橋 | 070-yyyy |
   ```

### 判定の閾値
- デフォルト: 「checkin予定時刻を過ぎてもステータス `A`」なら遅延候補
- 「30分以上経過で警告」等の閾値はユーザーに確認してから設定

---

## ⚠️ 注意事項

### チェックインと実利用は別概念
- **チェックイン** = `orderStateCode`（このスキル）
- **実利用 / ノーショウ** = 解錠イベントの有無（`keyvox-housekeeping` の `getLockHistory` 照合）
- 「予約はあったが解錠ゼロ＝ノーショウ」を見たい場合は housekeeping を使う。チェックインステータスと混同しない。

### タイムゾーン
- 予約の `checkin`/`checkout`、`realCheckin`/`realCheckout` は UNIX秒
- 比較はそのまま UNIX秒同士でOK。表示時のみ JST に変換（物件の `placeUtc` を参照）

### コードは日本語化して出す
- `orderStateCode` の生値（`A`/`I`/`O`）をそのまま画面に出さず、`references/keyvox-enums.md` を参照して日本語に変換する

---

## トラブルシューティング

| 症状 | 対処 |
|---|---|
| 全件 `A`（チェックイン済みが常にゼロ） | その物件でチェックイン処理を運用しているか確認。運用していなければ orderStateCode は遷移しない。到着確認したいなら housekeeping の実利用判定に切替 |
| `listReservations` が0件 | 日付範囲の timezone 確認、`total` フィルタ調整（`total=0` で全件） |
| ステータスの意味が不明な値 | `references/keyvox-enums.md` 未掲載の値。「未知のコード」として警告し、安全側に倒す |
| 401 / E2003 | コネクタ認証期限切れ / 対象組織違い（reservationスキルと同様） |

## 旧版からの主な変更点（レビュー用メモ）
- 中核判定を「解錠履歴(getLockHistory)」から「予約ステータス(orderStateCode)」へ全面変更
- このスキルでは `getLockHistory` / `getUnits`(lockId引き当て) を**使わない**
- 「実利用/ノーショウ」判定は `keyvox-housekeeping` の役割として明示的に分離
- `realCheckin`/`realCheckout` を補助フィールドとして追加
- チェックイン処理を運用していない現場では全件 `A` になる運用前提を明記
