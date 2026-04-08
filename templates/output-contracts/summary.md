# summary — タスク完了サマリーレポートテンプレート

> **用途**: supervise ステップの Summary レポート（APPROVE の場合のみ出力）
> **report 設定**: `Summary: summary.md`

---

## テンプレート

```markdown
# タスク完了サマリー

## タスク
{元の要求を1-2文で}

## 結果
完了

## 変更内容
| 種別 | ファイル | 概要 |
|------|---------|------|
| 作成 | `src/file.ts` | 概要説明 |

## レビュー結果
| レビュー | 結果 |
|---------|------|
{カスタマイズ: ワークフローのレビュー構成に応じてリスト変更}
| AI Review | APPROVE |
| Architecture | APPROVE |
| QA | APPROVE |
| Supervisor | APPROVE |

## 確認コマンド
```bash
npm test
npm run build
```
```

---

## カスタマイズ箇所

**レビュー結果テーブルのみ**ワークフローごとに変更する。
他のセクションは全ワークフローで同一。

| ワークフロー | レビュー一覧 |
|--------|------------|
| minimal | AI Review, Supervisor |
| coding | AI Review, Architecture |
| default | Architecture Design, AI Review, Architect Review, QA, Supervisor |
| expert | AI Review, Architecture, Frontend, Security, QA, Supervisor |
