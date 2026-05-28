# KEYVOX enum値辞書

API/MCPツールのレスポンスに出てくる **コード値・enum 値** の意味を集約した辞書。

## 出典

このドキュメントは公式 **[KEYVOX 開発者ポータル](https://developers.keyvox.co)** が公開している OpenAPI 仕様 (`openapi.yaml`) を一次ソースとしており、実機検証 (BCLtest 環境) で観測した値も補足として記載しています。差分がある場合は公式仕様を優先してください。

## 1. `placeType` (場所カテゴリ)

API ごとに2系統の値が返るので注意。

### 場所登録系 (`addPlace`/`updatePlace` 等)

| 値 | 意味 |
|---|---|
| `hotel` | ビルディング（常に `hotel` 固定、指定しても無視される旨が公式仕様に記載） |

### ユニット詳細系 (`unit_detail` / `getUnits` 等)

| 値 | 意味 |
|---|---|
| `facility` | ドア（部屋） ※スマートロック用 |
| `locker_single` | スマートロッカー（片面開き） |
| `locker_double` | スマートロッカー（両面開き） |

> 補足: `mcp-api.yaml` の `unit_list` パラメータ記述では `hotel`/`locker`/`doubleLocker`/`vendingMachine`/`facility` も列挙されており、MCP ラッパー側で別系統の値を受け付ける可能性あり。

## 2. `orderStateCode` (予約ステータス)

| 値 | 意味 |
|---|---|
| `Q` | 予約申込 |
| `B` | 予約拒否 |
| `A` | 未チェックイン |
| `C` | キャンセル済み |
| `I` | チェックイン済み |
| `M` | メンテ中 |
| `O` | チェックアウト済み |

## 3. `payStateCode` (支払ステータス)

| 値 | 意味 |
|---|---|
| `0` | 未支払 |
| `2` | 支払中 |
| `30` | 一部支払済 |
| `20` | 支払完了 |
| `40` | 支払失敗 |

## 4. `payTypeCode` (支払方法)

| 値 | 意味 |
|---|---|
| `stripeCreditCard` | クレジットカード (Stripe) |
| `offlinePayment` | オフライン決済 |
| `cash` | 現金 |

## 5. `etype` (ロックイベント種別、`getLockHistory`)

| 値 | 意味 |
|---|---|
| `1` | PIN (暗証番号) |
| `2` | Card (NFC カード) |
| `5` | OPEN/CLOSE Button (スマートロックのボタン) |
| `6` | Knob Button (サムターン) |
| `9` | Application (API 含むアプリ等のサービス) |

> `etype=1` のときに限り `pinCode` フィールドに使われた暗証番号が返却される。

## 6. `channelCode` (予約チャネル)

| 値 | 意味 |
|---|---|
| `BACS` | KEYVOX 管理画面 / API 経由作成 |
| `GO` | (公式仕様列挙のみ) |
| `googleCalendar` | Google カレンダー連携 |
| `KeyvoxWeb` | KEYVOX Web |
| `Mykeyvox` | Mykeyvox |
| `TABLET` | タブレット |
| `NEPPAN` | ねっぱん |
| `BATCH` | バッチ処理 |
| `BCLCHECKIN` | BCL チェックイン |
| `CHECKIN` | チェックイン |

> 補足: 実機観測で `line` (LINE 経由) が返ることがあるが公式 enum には未掲載。今後の追加候補。

## 7. `orderSource` (予約サイト名)

公式仕様上は自由記述 (`type: string`)。例: `楽天トラベル`, `line`, `BACS` 等。

## 8. `lockType` (ロック機種)

公式 OpenAPI に enum 列挙はなし（自由記述）。実機観測:

| 値 | 意味 |
|---|---|
| `BCL-QR1` | KEYVOX QR1 (QR コード対応スマートロック) |

> KEYVOX 他機種 (BR1 等)、`L!NKEY` / `OPELO` / `igloohome` 等他社ロックの正規値は実環境で観測して追加すること。

## 9. `relateType` (BCL-QR1 と連動するロックのタイプ)

公式仕様での説明: 「BCL-QR1 と連動して利用しているロックのタイプ」。BCL-QR1 に連動するロックがある場合のみ返却。

| 値 | 意味 |
|---|---|
| `PiACK II` | (公式仕様の example) |
| `BCL-BR1` | KEYVOX BR1 (実機観測) |

> ⚠️ 旧版ドキュメントでは「関連ゲートウェイ機種」と説明していたが、公式仕様では「連動するロックの機種」が正しい。

## 10. `pinType` (暗証番号方式)

| 値 | 意味 |
|---|---|
| `1` | オンライン方式 |
| `2` | オフライン方式 |
| `3` | 暗証番号非対応 |

> 項目が存在しない場合は `3:暗証番号非対応` として扱うこと。

## 11. `pinStatus` (暗証番号ステータス)

| 値 | 意味 |
|---|---|
| `0` | 未発行 |
| `1` | 発行中 |
| `2` | 発行済 |
| `4` | 削除済 |

## 12. `wifi` (Wi-Fi 接続状態、`getLockStatus`)

| 値 | 意味 |
|---|---|
| `"1"` | 接続中 |
| `"0"` | 切断（推測） |

## 13. `stateType` (注文の通常/重要区分)

| 値 | 意味 |
|---|---|
| `0` | 通常 |
| `1` | 重要 |

## 14. `isAssigned` (割当フラグ、`listReservations` 引数)

| 値 | 意味 |
|---|---|
| `null` (未設定) | フィルタなし |
| `0` | 未割当 |
| `1` | 割当済み |

## 15. `total` (total フィルタ、`listReservations` 引数)

| 値 | 意味 |
|---|---|
| `0` | すべて |
| `1` | (キャンセル・チェックアウト) 以外 |

## 16. `businessType` (ビジネスタイプ、reservation / plan 返却)

| 値 | 意味 |
|---|---|
| `housing` | 宿泊 |
| `rentalSpace` | レンタルスペース |
| `conferenceRoom` | コーワーキング |
| `locker` | ロッカー |
| `airdrop` | ドロップイン |
| `vendingMachine` | 自動販売機 |

## 17. `releaseType` (掲載タイプ、plan 返却)

| 値 | 意味 |
|---|---|
| `0` | 公開しない |
| `1` | 一般公開 |
| `2` | 限定公開 |

## 18. `stockType` (利用単位、plan 返却)

| 値 | 意味 |
|---|---|
| `0` | 独占 |
| `1` | 共有 |

## 19. `cancelOrderPayFlag` (キャンセル料金区分)

| 値 | 意味 |
|---|---|
| `1` | キャンセル可能 |
| `0` | キャンセル不可 |

## 値の用法ガイドライン

- **コードを直接画面に出さない**: ユーザー向け回答では `O` → 「チェックアウト済み」のように日本語に変換する
- **未知の値が返ったら警告**: 上記表にない値が返ってきたら「未知のコード `<値>` が返却された」とログに残し、安全側に倒した挙動をする
- **公式仕様優先**: 実機観測値と公式仕様が食い違う場合は、公式仕様を一次情報として扱い、観測値は補足として記録する

## 関連ドキュメント
- リソース定義: `keyvox-entities.md`
- 業務シナリオ→ツール: `keyvox-tool-map.md`
- ID 解決パターン: `keyvox-id-resolution.md`
- セットアップ・再認証: `keyvox-mcp-setup.md`
- 公式 API リファレンス: <https://developers.keyvox.co>
