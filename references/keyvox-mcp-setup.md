---
name: keyvox-mcp-setup
description: KEYVOX MCP コネクタの環境前提・再認証手順・典型エラー対処を集約したリファレンス。全SKILL.mdから参照される共通ドキュメント。
---

# KEYVOX MCP Setup & Troubleshooting

全KEYVOXスキルが共通で参照する、コネクタ環境前提と再認証手順、典型エラーの対処方法を集約。

## 環境前提

KEYVOXスキル群は **Claude.ai のカスタムコネクタ** 経由でのみ動作します。

| 環境 | 動作 |
|---|---|
| **Claude.ai** (Web版) | ✅ 動く — `claude.ai/customize/connectors` で `keyvoxMCP` 接続済みが前提 |
| **Claude Code** (CLI/IDE) | ⚠️ Claude.ai のコネクタは Claude Code には自動同期されません。Claude Code から KEYVOX を使う場合は、別途 `.mcp.json` 等で MCP server を設定する必要があります |

> **注記**: 以降の手順における「`keyvoxMCP`」は claude.ai のカスタムコネクタ追加時に付けた名前です。あなたが別名で登録している場合はその名前に読み替えてください。

---

## 401 / E2003 エラー時にユーザーへ返す **定型応答**

> ⚠️ **Claude へ**: 401 や E2003 を受け取ったら、以下のコードブロック内の文章を **そのままユーザーに出力** してください（要約・言い換え禁止）。

```
KEYVOX MCPコネクタの認証が切れています。以下の手順で再認証してください:

👉 直接リンク: https://claude.ai/customize/connectors

1. 上記URLを開く
2. 「Web」セクションの「keyvoxMCP」(あなたが付けた名前) を選択
3. 右上の「切断する」をクリック → 確認ダイアログでも「切断する」
4. 「未接続」セクションに移動した「keyvoxMCP」の「連携 / 連携させる」をクリック
5. KEYVOXログイン画面で ユーザーID + パスワード でログイン
6. 同意画面で「対象組織」ドロップダウンを開き → 業務データのある組織 (BCL 社等) を選択 → 「はい」
7. 画面右上に「keyvoxMCP に接続しました」のトーストが出れば完了

⚠️ Step 6 で対象組織を間違える (デフォルトの組織のままにする等) と、認証は通っても E2003 で失敗します。
業務データを操作したい組織を必ず明示的に選び直してください。

なお、Claude in Chrome や Playwright 等のブラウザMCPがこのセッションで使えるなら、Step 1〜4 はClaudeが代行可能です。
「ブラウザ操作で再認証して」と指示してもらえれば自動化します (Step 5, 6 のログイン・組織選択はユーザー操作必須)。
```

---

## Claude 向けの追加ガイドライン

1. **`/mcp` 手動認証を絶対に案内しない**
   - Claude Code TUI の `/mcp` は KEYVOX OAuth サーバーが DCR (Dynamic Client Registration) 非対応のため完走できない
   - 案内すると無限ループに入るため誤誘導

2. **ブラウザMCP使用可否を確認した上で代行可否を判断**
   - 使えない: 上記定型応答のみ提示
   - 使える: 「Step 1〜4を自動で代行しますか？」とユーザーに確認後、ブラウザ操作開始

3. **`Failed to start OAuth flow: ... does not support dynamic client registration` エラー** が出たら
   - そのまま定型応答を出力する
   - 「dynamic client registration」「DCR」等の技術用語をユーザーに見せない

---

## 典型エラーと対処早見表

| エラー | 原因 | 対処 |
|---|---|---|
| `401 Unauthorized` | トークン期限切れ | 上記定型応答を出力 |
| `E2003 リクエストに失敗しました：権限不足です` | OAuth同意時の対象組織が違う | 定型応答 Step 6 の組織選択を強調 |
| `E0043 データが存在しません` | placeId/orderId/unitId のtypo or 他組織のID | ID を再確認 |
| `E0005 パラメータエラー` (updateReservation) | パラメータ構造が厳格 | 延長は `update_reservation_checkout`、その他変更は cancel + recreate パターン |
| `Failed to start OAuth flow: ... DCR` | Claude Code の auto OAuth は KEYVOX 非対応 | 定型応答を出力。`/mcp` 案内禁止 |

## 関連ドキュメント
- リソース定義: `keyvox-entities.md`
- 業務シナリオ→ツール: `keyvox-tool-map.md`
- ID解決パターン: `keyvox-id-resolution.md`
- enum値: `keyvox-enums.md`
