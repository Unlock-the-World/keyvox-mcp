# KEYVOX enum値辞書

API/MCPツールのレスポンスに出てくる **コード値・enum値** の意味を集約した辞書。

## ⚠️ ステータス

このドキュメントは #13 学習②の実機検証で観測した値と、`mcp-api.yaml` / `openapi.yaml` に書かれている記述から構成されている。**`?` 印は値の意味が未確定で、メンター確認後に正式値で更新する必要がある**。

## 1. `placeType` (場所カテゴリ)

`mcp-api.yaml` の `unit_list` parameters 記述より：

| 値 | 意味 |
|---|---|
| `hotel` | ビルディング |
| `locker` | ロッカー |
| `doubleLocker` | 両面開きロッカー |
| `vendingMachine` | 自動販売機 |
| `facility` | 施設（`getUnits` 経由で観測） |

**注意**: 同じ場所でも `place_list` では `"hotel"`、`getUnits` では `"facility"` が返ることを観測（#13）。スキーマが API ごとに違うため、用途に応じて使い分け。

## 2. `orderStateCode` (予約ステータス)

実機観測値（要メンター確認）：

| 値 | 観測コンテキスト | 推測される意味 |
|---|---|---|
| `O` | `checkin/checkout` が過去、`livedDays=1` | チェックアウト済み / 完了 ? |
| `Q` | `checkin/checkout` が未来、`livedDays` が極端に大きい (575等) | 確定済み・未利用 ? |

**確認必要**: 全ステータス一覧（O/Q 以外の値の有無）、各値の正式な意味、状態遷移図

## 3. `payStateCode` (支払ステータス)

実機観測値：

| 値 | 観測 |
|---|---|
| `20` | 全予約で観測 |

**確認必要**: 他の値の存在、値の意味（10=未払い, 20=支払済み 等？）

## 4. `etype` (ロックイベント種別、`getLockHistory`)

実機観測値：

| 値 | 観測 |
|---|---|
| `9` | userName=API のとき | 解錠系? |

**確認必要**: 値の全リスト。施錠/解錠/タイムアウト/異常等の区別

## 5. `channelCode` (予約チャネル)

実機観測値：

| 値 | 意味 |
|---|---|
| `line` | LINE経由予約 |

**確認必要**: 他のチャネル（直販、agoda、Beds24、Booking.com 等）の値

## 6. `lockType` (ロック機種)

実機観測値：

| 値 | 意味 |
|---|---|
| `BCL-QR1` | KEYVOX QR1（QRコード対応スマートロック） |

**確認必要**: KEYVOX 他機種（BR1、新機種等）、L!NKEY、OPELO、igloohome 等の他社ロック値

## 7. `relateType` (関連ゲートウェイ機種)

実機観測値：

| 値 | 意味 |
|---|---|
| `BCL-BR1` | KEYVOX BR1（ブリッジ機器） |

## 8. `pinType` (PIN種別、`getLockStatus`)

実機観測値：

| 値 | 観測 |
|---|---|
| `1` | デフォルト値？ |

**確認必要**: 種別の意味（オンラインQR/オフラインQR/手動入力等？）

## 9. `wifi` (Wi-Fi接続状態、`getLockStatus`)

| 値 | 意味 |
|---|---|
| `"1"` | 接続中 |
| `"0"` | 切断 (推測) |

## 10. `stateType` (注文ステータス・listReservations引数)

`mcp-api.yaml` 記述より：

| 値 | 意味 |
|---|---|
| `0` | 通常 |
| `1` | 重要 |

## 11. `isAssigned` (割当フラグ・listReservations引数)

`mcp-api.yaml` 記述より：

| 値 | 意味 |
|---|---|
| `null` (未設定) | フィルタなし |
| `0` | 未割当 |
| `1` | 割当済み |

## 12. `total` (totalフィルタ・listReservations引数)

`mcp-api.yaml` 記述より：

| 値 | 意味 |
|---|---|
| `0` | すべて |
| `1` | (キャンセル・チェックアウト)以外 |

## 13. `businessType` (業務カテゴリ、reservation返却)

実機観測値：

| 値 | 意味 |
|---|---|
| `rentalSpace` | レンタルスペース業 |

**確認必要**: 他のbusinessType（hotel, coworking, locker 等）

## 14. `orderSource` (予約ソース、reservation返却)

実機観測値：

| 値 | 意味 |
|---|---|
| `line` | LINE経由 |

`channelCode` との関係要確認。

## 値の用法ガイドライン

- **コードを直接画面に出さない**: ユーザー向け回答では `O` → 「チェックアウト済み」のように日本語に変換する
- **不明値が返ったら警告**: 上記表にない値が返ってきたら「未知のコード `<値>` が返却された」とログに残し、安全側に倒した挙動をする

## 関連ドキュメント
- リソース定義: `keyvox-entities.md`
- 業務シナリオ→ツール: `keyvox-tool-map.md`
- ID解決パターン: `keyvox-id-resolution.md`
