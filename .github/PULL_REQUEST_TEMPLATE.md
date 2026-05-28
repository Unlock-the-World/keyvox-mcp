## 概要 / Summary

この PR が何を変えるかを 1〜3 行で。

## 変更内容 / Changes

- 追加: `...`
- 変更: `...`
- 削除: `...`

## 動機 / Motivation

なぜこの変更が必要か。関連 Issue があれば `Closes #...` で紐づけ。

## 変更タイプ / Type

- [ ] 新スキル追加
- [ ] 既存スキル改善 (機能追加 / 挙動修正)
- [ ] バグ修正
- [ ] references / ドキュメント追加・改善
- [ ] OSS 体裁 (LICENSE / CONTRIBUTING / SECURITY 等)

## 不可逆操作・物理デバイス影響の有無 / Physical device impact

- [ ] 物理デバイスに影響しない (情報取得のみ)
- [ ] 物理デバイスに影響しうる (`unlock` / `createLockPin` / `disableLockPin` 等)
  - 影響する場合: ユーザー最終承認ゲートの設計を本文で説明してください

## 動作検証 / Test plan

- [ ] テスト環境 (BCLtest 等) で動作確認済み
- [ ] 本番物理ロックでの `unlock` テストは **実施していない**
- [ ] 新規予約のテスト後は `cancelReservation` でクリーンアップ済み
- [ ] スキル発火確認: 新規セッションで自然言語発話 → 期待通りスキルが発火する

## レビュー観点 / Review notes

レビュアーに特に見てほしい論点・トレードオフがあれば記載。

## チェックリスト / Checklist

- [ ] [CONTRIBUTING.md](../CONTRIBUTING.md) を確認した
- [ ] 機密情報 (API キー、実顧客名、実電話番号、実 PIN 等) が混入していない
- [ ] 既存の DRY 構造を破壊していない (再認証手順を SKILL.md に再コピーしていない等)
- [ ] OSS 公開リポであることを意識した内容になっている (社内 Issue 番号、社内 URL 等を含まない)
