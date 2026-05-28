# KEYVOX ID解決パターン集

ユーザー発話に含まれる自然言語の表現（部屋名・日付・顧客名等）を MCPツールが要求する ID（unitId, orderId, lockId 等）に変換するパターンを集約。

すべての業務スキルはこのドキュメントの解決ロジックを参照すること。

## 1. 部屋名・ユニット名 → `unitId`

### パターン
| ユーザー発話 | 解決ステップ |
|---|---|
| 「101号室」「ドロップインドア」 | `unit_list` (placeId + searchWord) → list内の unitId |
| 「会議室A」（曖昧） | `unit_list` で全件取得 → unitName を曖昧マッチ → 候補1件なら採用、複数なら確認 |

### 実装例
```
1. ユーザー: 「101号室の予約見せて」
2. → place_list で placeId 取得（複数物件なら確認）
3. → unit_list(placeId, searchWord="101") で候補絞り込み
4. → 候補1件なら unitId 確定、複数なら「どれですか？」
```

### 注意
- `unit_list` の `searchWord` は部分一致っぽい挙動。完全一致を期待しない
- `unitName` と `unitNo` 両方マッチ対象

## 2. 場所名 → `placeId`

### パターン
| ユーザー発話 | 解決ステップ |
|---|---|
| 「BCLtest」「銀座スペースA」 | `place_list` で list を取得 → placeName マッチ |
| 「東京の物件」 | placeAddress を住所マッチ |

### 実装例
```
1. ユーザー: 「BCLtestの今日の予約」
2. → place_list(page=1, count=100) で全件取得（少数前提）
3. → list[].placeName で "BCLtest" マッチ → placeId 取得
4. → listReservations(placeId, fromDate=今日0:00, toDate=今日23:59)
```

### 注意
- 同名物件が複数ある場合は placeAddress で区別
- 物件が多い場合（>100件）はページング考慮

## 3. 日付表現 → UNIX秒 (`fromDate` / `toDate`)

### パターン
| ユーザー発話 | 解決ロジック |
|---|---|
| 「今日」 | 当日 00:00:00 JST 〜 23:59:59 JST |
| 「明日」 | 翌日 00:00:00 JST 〜 23:59:59 JST |
| 「今週」 | 月曜00:00 〜 日曜23:59 (週開始は要合意) |
| 「3/15」「3月15日」 | 当該日付 00:00 〜 23:59 |
| 「来週月曜から3日間」 | 月曜00:00 〜 水曜23:59 |

### 実装注意
- KEYVOXは `placeUtc` フィールドで物件のタイムゾーンを持つ（多くは JST=9）
- date計算は **対象物件の UTC オフセット** に従うこと
- UNIX秒に変換: `Math.floor(date.getTime() / 1000)`

### 実装例
```python
# 「今日」のJST範囲をUNIX秒に
今日_start = today.replace(hour=0, minute=0, second=0).timestamp()  # JST 0:00
今日_end   = today.replace(hour=23, minute=59, second=59).timestamp()
listReservations(placeId=..., fromDate=今日_start, toDate=今日_end, page=1, count=50)
```

## 4. 顧客名 → `orderId`

### パターン
| ユーザー発話 | 解決ステップ |
|---|---|
| 「岡本さんの予約」 | `listReservations` (placeId + searchWord="岡本") → list[].orderId |
| 「岡本さんの今日の予約」 | listReservations(placeId, fromDate, toDate, searchWord="岡本") |

### 注意
- `searchWord` は orderContact (予約者名) と contactTel (電話) 両方を対象とする模様（要検証）
- 同名複数件ある場合は連絡先や予約日時で区別を促す

## 5. 予約ID → `orderId` (直接指定)

ユーザーが直接 `QUPHLLMCI` のような9文字英大文字+数字を発話したら、それを orderId として直接使う。

```
正規表現: ^[A-Z0-9]{9}$
```

## 6. ロックID → `lockId` (発話よりはunit経由が多い)

ユーザーが直接 `QRZEURCMC2FWQ9OI` 等の lockId を発話するケースは稀。

通常は：
```
ユーザー: 「101号室の解錠履歴」
→ unit解決 (上記#1)
→ getUnits で該当 unitId の lockIds 取得
→ lockIds[0] を lockId として getLockHistory に渡す
```

### 注意
- **`unit_list` の戻り値には lockIds が含まれない** → `getUnits` を使うこと
- 1つの unit に複数 lock が紐づくケース（玄関+部屋扉等）あり、その場合はユーザーに「どのロック？」と確認

## 7. 「現在の状態」「今」 → 時刻計算

| 発話 | 解決 |
|---|---|
| 「今チェックインしてるゲスト」 | listReservations(today range) → 各予約の `getLockHistory` で最終解錠時刻が checkin 後にあるかチェック |
| 「滞在中の予約」 | `orderStateCode` が滞在中状態（enums.md 参照、要検証） |

## 8. 曖昧マッチ・複数候補時のUXパターン

候補が複数あるときの応答テンプレ：
```
「101号室」で2件見つかりました：
1. 101号室 (BCLtest)
2. 101号室 (銀座スペースA)
どちらですか？
```

完全一致なし時：
```
「102号室」は見つかりませんでした。
利用可能なユニット: 101号室、ドロップインドア、…
```

## 関連ドキュメント
- リソース定義: `keyvox-entities.md`
- 業務シナリオ→ツール: `keyvox-tool-map.md`
- enum値: `keyvox-enums.md`
