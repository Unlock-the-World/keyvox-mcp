# Contributing to KEYVOX MCP Skills

KEYVOX MCP Skills へのコントリビュートに興味を持っていただきありがとうございます。

## どんな貢献を歓迎しているか

- 🐛 **バグ報告**: スキルが期待通りに発火しない、想定外のツール呼び出しが発生する、定型応答が壊れている等
- 💡 **新スキル提案**: 現在の 4 スキル (`reservation` / `checkin-status` / `onsite-support` / `housekeeping`) でカバーできない業務シナリオ
- 📝 **リファレンス改善**: `references/keyvox-enums.md` の `?` 印未確定値の確定、`keyvox-tool-map.md` への新シナリオ追加
- 🧪 **実機検証フィードバック**: 本番物理ロックでの `unlock` 動作、マルチ place 運営での発火精度、清掃所要時間デフォルトの妥当性 等

## はじめての方へ

1. このリポジトリを Fork
2. ローカルにクローン
3. ブランチ作成 (`feat/...`, `fix/...`, `docs/...`)
4. 変更を加える
5. Pull Request を作成

## SKILL.md を書く際のガイドライン

- **description は意図ベースで書く**: キーワード列挙ではなく業務文脈を表現する（発火精度に直結）
- **不可逆操作には必ずユーザー最終確認ゲートを設ける**: `createReservation` / `cancelReservation` / `unlock` / `disableLockPin` 等
- **環境前提・再認証手順は重複させない**: [`references/keyvox-mcp-setup.md`](./references/keyvox-mcp-setup.md) を参照するスタイルに統一
- **実機検証ベースで書く**: 推測で書いたパラメータ・enum 値は `⚠️ 要確認` として明示

## 動作検証のお願い

PR を提出する前に、可能な範囲で以下を実施してください:

- [ ] テスト用環境（BCLtest 等）で動作確認
- [ ] 本番物理ロックに影響する操作 (`unlock` 等) のテストは**実施しない**こと
- [ ] 新規予約のテスト後は `cancelReservation` でクリーンアップ
- [ ] スキル発火確認: 新規セッションで自然言語発話 → 期待通りスキルが発火するか

## Issue を作る前に

- 既存の Issue を検索して重複していないか確認
- バグ報告には以下を含めてください:
  - 利用環境 (Claude.ai Web / Claude Code CLI)
  - 発話した内容
  - 期待した挙動
  - 実際の挙動（エラーメッセージ含む）
  - コネクタ設定（コネクタ名、組織選択）

## 質問・相談

- 軽い質問は [KEYVOX Discord コミュニティ](https://discord.gg/Etd3vZPKhb) で
- 業務インパクトを伴う質問は [KEYVOX サポートポータル](https://support.keyvox.co) で

## ライセンス

コントリビュートしたコード・ドキュメントは [MIT License](./LICENSE) の下で公開されることに同意したものとみなします。
