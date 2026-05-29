# KEYVOX MCP Skills

KEYVOX アクセス管理サービスを **自然言語で操作する** ための [Claude Skills](https://docs.claude.com/en/docs/claude-code/skills) コレクション。

予約管理・チェックイン確認・滞在中サポート・清掃計画の4業務領域をカバーします。

## なぜ作ったか

KEYVOX API には 70近いエンドポイントがあり、業務と API の紐付けには熟練が必要です。
本スキル群を `~/.claude/skills/` に配置すると、Claude に以下のような自然言語で依頼するだけで業務が完結します。

- 「今日の BCLtest の予約見せて」 → `keyvox-reservation`
- 「101 号室のゲスト入った？」 → `keyvox-checkin-status`
- 「ゲストが鍵忘れた」 → `keyvox-onsite-support`
- 「今日の清掃対象は？」 → `keyvox-housekeeping`

API キー覚える必要なし、コマンド覚える必要なし、自然な日本語で。

## スキル一覧

| スキル名 | 役割 | カバー業務 | SKILL.md |
|---|---|---|---|
| `keyvox-reservation` | 予約管理 | 新規予約 / 確認 / 延長 / 変更 / キャンセル | [リンク](./skills/keyvox-reservation-SKILL.md) |
| `keyvox-checkin-status` | チェックイン確認 | 本日の状況 / 個別確認 / 未チェックイン抽出 | [リンク](./skills/keyvox-checkin-status-SKILL.md) |
| `keyvox-onsite-support` | 滞在中サポート | 鍵忘れ対応 / 緊急解錠 / 状況確認 | [リンク](./skills/keyvox-onsite-support-SKILL.md) |
| `keyvox-housekeeping` | 清掃計画 | 本日の清掃対象 / 今すぐ入れる部屋 / 利用統計 | [リンク](./skills/keyvox-housekeeping-SKILL.md) |

## 必要なもの

- [Claude.ai](https://claude.ai)（Pro / Max / Team / Enterprise プラン）または [Claude Code](https://docs.claude.com/en/docs/claude-code)
- KEYVOX アカウント（管理画面 BACS にログインできる）
- KEYVOX MCP コネクタの **OAuth2 クライアント ID**
  - [KEYVOX サポートポータル](https://support.keyvox.co) からチケットで発行依頼してください
  - 依頼時に必要な情報: `redirect_uri = https://claude.ai/api/mcp/auth_callback`

> **環境別の動作**
> - **Claude.ai (Web版) / Claude Desktop**: 公式サポート対象。Claude Desktop は Claude.ai の Web 設定を共有するので同じコネクタ設定が使えます。
> - **Claude Code (CLI/IDE)**: Claude.ai のコネクタは Claude Code には自動同期されません。Claude Code から使う場合は別途 `.mcp.json` 等で MCP server を設定する必要があります。

---

## 🚀 非エンジニア向けクイックスタート (Claude Desktop / Claude.ai)

> ターミナル / git / コマンドライン操作なしで使う方法です。Claude.ai の **Project 機能** にスキルを「知識」として読み込ませます。
> Pro / Max / Team / Enterprise プランが必要 (Project 機能はFreeプラン非対応)。

### ステップ 1: KEYVOX MCP コネクタを接続する (5〜10 分)

1. [KEYVOX サポートポータル](https://support.keyvox.co) で **「Claude 連携用の OAuth2 クライアント ID 発行希望」** とチケットを投げる
   - 伝える情報: `redirect_uri = https://claude.ai/api/mcp/auth_callback`
   - 返ってくるもの: クライアント ID (英数字の文字列)
2. Claude Desktop または Claude.ai を開く → **設定 → コネクタ → カスタムコネクタを追加**
   - 名前: `keyvoxMCP`
   - URL: `https://eco.blockchainlock.io/mcp/sse`
   - クライアント ID: ステップ 1 で取得した値
3. **「連携」** → BACS にログイン → 同意画面で **対象組織を必ず明示的に選択** (デフォルトのままだと業務データを操作できません)

### ステップ 2: このリポジトリを ZIP でダウンロード (1 分)

1. このページ右上の **緑色の「Code」ボタン** → **「Download ZIP」** をクリック
   ![Download ZIP](https://docs.github.com/assets/cb-13128/mw-1440/images/help/repository/code-button.webp)
2. ダウンロードした `keyvox-mcp-main.zip` を解凍
3. フォルダの中に `skills/` と `references/` があることを確認

### ステップ 3: Claude の Project にスキルを読み込ませる (5 分)

1. Claude Desktop / Claude.ai のサイドバー → **「Projects」** → **「Create project」**
2. プロジェクト名: `KEYVOX 業務` (任意)
3. **「Project knowledge」** エリアに、解凍したフォルダから以下 9 ファイルをドラッグ&ドロップ:
   ```
   skills/keyvox-reservation-SKILL.md
   skills/keyvox-checkin-status-SKILL.md
   skills/keyvox-onsite-support-SKILL.md
   skills/keyvox-housekeeping-SKILL.md
   references/keyvox-mcp-setup.md
   references/keyvox-entities.md
   references/keyvox-tool-map.md
   references/keyvox-id-resolution.md
   references/keyvox-enums.md
   ```
4. **「Project instructions」** 欄に以下をコピペ:
   ```
   あなたは KEYVOX アクセス管理サービスの業務アシスタントです。
   Project knowledge にアップロードされている 4 つの SKILL.md と
   5 つの references/*.md を必ず参照してください。

   - ユーザーの発話から該当スキル (reservation / checkin-status /
     onsite-support / housekeeping) を判定し、対応する SKILL.md の
     手順に従って MCP ツール (keyvoxMCP コネクタ) を呼び出す
   - 不可逆操作 (createReservation, cancelReservation, unlock,
     createLockPin 等) は必ずユーザーの最終承認を取ってから実行
   - 401 / E2003 エラー時は keyvox-mcp-setup.md の「定型応答」を
     そのままユーザーに出力 (要約・言い換え禁止)
   ```
5. **保存** して、このプロジェクト内で新しいチャットを開始

### ステップ 4: 動作確認 (1 分)

プロジェクト内のチャットで以下を試す:

```
今日の予約見せて
```

→ Claude が `keyvoxMCP` コネクタの `listReservations` を呼んで、本日の予約一覧を返せば成功 🎉

### ❓ うまく動かないとき

| 症状 | 対処 |
|---|---|
| Claude が「ツールが見つからない」と言う | コネクタが接続済みか確認 (設定 → コネクタ → `keyvoxMCP` が「接続済み」になっているか) |
| 「権限がない」「データがない」と言われる | 対象組織の選択ミス。コネクタを一度「切断」して再連携、同意画面で正しい組織を選ぶ |
| Project knowledge にアップロードできない | ファイル数 / サイズ上限の可能性。Free プランでは Project 自体が使えないので Pro 以上か確認 |
| その他 | [Discord](https://discord.gg/Etd3vZPKhb) または [KEYVOX サポート](https://support.keyvox.co) へ |

### 💡 アップデートの取り込み方

このリポジトリが更新されたら、再度 ZIP をダウンロードして Project knowledge のファイルを上書きしてください。手作業ですが「git の使い方を覚える」より圧倒的に早いです。

---

## セットアップ手順 (エンジニア向け)

### Step 1: KEYVOX MCP コネクタを Claude.ai に追加

1. [KEYVOX サポートポータル](https://support.keyvox.co) で OAuth2 クライアント ID を申請して取得
2. <https://claude.ai/customize/connectors> を開く
3. 右上 **「+」** → カスタムコネクタ追加:
   - **名前**: `keyvoxMCP` (任意。本READMEと各SKILL.md ではこの名前で参照しているので、別名にする場合は読み替えてください)
   - **URL**: `https://eco.blockchainlock.io/mcp/sse`
   - **クライアント ID**: Step 1 で取得した値
4. **「連携」** ボタンで OAuth 認可フロー → BACS ログイン → 同意
   - ⚠️ **対象組織の選択を間違えないこと**。デフォルトのままだと認証は通っても `E2003 権限不足` で API が失敗します
5. 接続確認: 「`keyvoxMCP` に接続しました」のトーストが出れば OK

### Step 2: スキルを配置

Claude Code は `~/.claude/skills/<skill-name>/SKILL.md` という構造を期待するので、本リポのフラットな `skills/keyvox-*-SKILL.md` から **個別フォルダ + SKILL.md** にマッピングする必要があります。

#### A. シンボリックリンク方式（推奨・編集が即反映）

```bash
git clone https://github.com/Unlock-the-World/keyvox-mcp.git ~/keyvox-mcp

for skill in reservation checkin-status onsite-support housekeeping; do
  mkdir -p ~/.claude/skills/keyvox-$skill
  ln -sf ~/keyvox-mcp/skills/keyvox-$skill-SKILL.md \
         ~/.claude/skills/keyvox-$skill/SKILL.md
done
```

#### B. 単純コピー方式

```bash
git clone https://github.com/Unlock-the-World/keyvox-mcp.git

for skill in reservation checkin-status onsite-support housekeeping; do
  mkdir -p ~/.claude/skills/keyvox-$skill
  cp ./keyvox-mcp/skills/keyvox-$skill-SKILL.md \
     ~/.claude/skills/keyvox-$skill/SKILL.md
done
```

> 💡 **なぜマッピングが必要か**: 本リポは Claude Project (knowledge アップロード時にファイル名衝突を避ける必要) と Claude Code (フォルダ + `SKILL.md` 規約) の両方をサポートするため、内部ではフラットな `keyvox-*-SKILL.md` 形式で保持しています。Claude Code への配置時のみ、上記スクリプトで規約形式に展開します。

### Step 3: 動作確認

Claude.ai または Claude Code を **新規セッションで起動** し、以下のような発話を試す:

```
KEYVOX のテストして
```

`keyvox-reservation` 等のスキルが発火し、`place_list` を叩いて物件情報を返せば成功。

## 利用例

> 以下の例に登場する物件名 (`BCLtest`)、ゲスト名 (山田 / 佐藤 / 鈴木)、予約 ID、PIN コード、電話番号などは **すべて架空のサンプル** です。実際の挙動はあなたの環境のデータに置き換わります。

### 例1: 朝のルーチン確認

```
You: 今日の BCLtest 予約見せて

Claude:
[keyvox-reservation 発火]
本日 (05/28) BCLtest 予約一覧 (3件)
| # | 時刻        | 部屋 | ゲスト | 状態 |
|---|-------------|------|--------|------|
| 1 | 14:00-17:00 | 101  | 山田   | 確定 |
| 2 | 15:00-18:00 | 102  | 佐藤   | 確定 |
| 3 | 10:00-13:00 | 103  | 鈴木   | 確定 |
```

### 例2: チェックイン状況確認

```
You: 101 号室の山田さん入った？

Claude:
[keyvox-checkin-status 発火]
山田太郎さん (予約 ABCDEFGHI / 101 号室 14:00-17:00)
状態: ✅ チェックイン済
- 初回解錠: 14:05 (QR 経由)
- 直近解錠: 16:42
- 解錠回数: 3 回
```

### 例3: 鍵忘れ対応

```
You: 102 号室の佐藤さんが鍵を紛失したらしい

Claude:
[keyvox-onsite-support 発火]
状況確認します:
- 佐藤さんの既存 PIN: 836241 (有効期間 15:00-18:00)
このPINを SMS / LINE で再送するのが最速です。
それでも開かない場合、新規一時 PIN 発行か、リモート解錠を提案できます。

新規 PIN を発行しますか？ (Y/N)
```

### 例4: 清掃計画

```
You: 今日の清掃対象は？

Claude:
[keyvox-housekeeping 発火]
本日の清掃対象 (BCLtest)
| # | 部屋 | チェックアウト | 次チェックイン | 清掃可能時間枠 |
|---|------|----------------|----------------|----------------|
| 1 | 101  | 11:00          | 14:00 (今日)   | 3 時間         |
| 2 | 102  | 12:00          | 翌日 10:00     | 22 時間        |

合計 2 件 (うちノーショウ除外 1 件)
```

## 共通リファレンス

スキル群は以下のドキュメントを参照する設計です。

| ファイル | 内容 |
|---|---|
| [`references/keyvox-mcp-setup.md`](./references/keyvox-mcp-setup.md) | コネクタ環境前提・再認証手順・典型エラー対処 (全スキル共通) |
| [`references/keyvox-entities.md`](./references/keyvox-entities.md) | 7 リソース（place / unit / lock / pin / card / user / reservation）の仕様 + ER 図 |
| [`references/keyvox-tool-map.md`](./references/keyvox-tool-map.md) | 業務シナリオ → MCP ツール対応表 |
| [`references/keyvox-id-resolution.md`](./references/keyvox-id-resolution.md) | 自然言語 → ID 解決パターン集 |
| [`references/keyvox-enums.md`](./references/keyvox-enums.md) | `orderStateCode` 等の enum 値辞書 |

## トラブルシューティング

| 症状 | 対処 |
|---|---|
| スキルが発火しない | セッション再起動。SKILL.md の `description` が業務文脈に合っているか確認 |
| `401 Unauthorized` | コネクタの OAuth トークン期限切れ。[`references/keyvox-mcp-setup.md`](./references/keyvox-mcp-setup.md) の再認証手順を参照 |
| `E2003 権限不足` | OAuth 同意時の「対象組織」が間違っている。再認証して正しい組織を選択 |
| `E0043 データが存在しません` | `placeId` / `orderId` の typo or 他組織の ID |
| `Failed to start OAuth flow: ... DCR` | Claude Code の自動 OAuth は KEYVOX 非対応。Claude.ai 経由で接続すること |

## アーキテクチャ

```
[あなた]
   ↓ 自然言語
[Claude.ai / Claude Code]
   ↓ (1) Skills 発火 (description で判別)
   ↓ (2) スキル内手順に従い MCP ツールを選択
[KEYVOX MCP コネクタ (eco.blockchainlock.io)]
   ↓ OAuth2 認可済みリクエスト
[KEYVOX API]
   ↓ 業務データ取得・更新
[ロック・PIN・予約システム]
```

スキル本体は **MCP ツールの応用レシピ**。MCP ツール直叩きでも同じことはできますが、業務手順を覚えなくていい・自然言語で済むのが利点です。

## ライセンス

[MIT](./LICENSE) — Copyright © 2026 Blockchain Lock Inc. / KEYVOX Contributors

## Contributing

Issue / PR 歓迎です。特に以下のシナリオでの実利用フィードバックを募集中:

- 本番物理ロックでの `unlock` 動作確認
- バックトゥバック予約での清掃計画運用
- マルチ place（複数物件）運営での発火精度

コントリビュート手順の詳細は [CONTRIBUTING.md](./CONTRIBUTING.md) を参照してください。すべての参加者は [行動規範 (Code of Conduct)](./CODE_OF_CONDUCT.md) に従ってください。

## セキュリティ

物理スマートロックを操作する性質上、脆弱性報告は最優先で扱います。**GitHub Issues に公開で投稿せず**、[SECURITY.md](./SECURITY.md) の手順で非公開報告してください。

## 関連リンク

- [KEYVOX サービスサイト](https://keyvox.co)
- [KEYVOX 開発者ポータル](https://developers.keyvox.co)
- [KEYVOX サポートポータル](https://support.keyvox.co)
- [KEYVOX Discord コミュニティ](https://discord.gg/Etd3vZPKhb)
- [Claude Skills 公式ドキュメント](https://docs.claude.com/en/docs/claude-code/skills)
- [Model Context Protocol (MCP)](https://modelcontextprotocol.io/)
