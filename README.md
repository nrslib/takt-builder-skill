# takt-builder-skill

[TAKT](https://github.com/nrslib/takt) のピースYAMLとファセットを作成・更新する Claude Code スキル。

## できること

- ピースYAML の新規作成（ワークフロー設計 → ファセット作成 → YAML生成）
- ファセットの新規作成・更新（ペルソナ / ポリシー / ナレッジ / インストラクション / 出力契約）
- スタイルガイドに準拠したセルフチェック

## インストール

```bash
git clone https://github.com/nrslib/takt-builder-skill.git ~/.claude/skills/takt-builder
```

## 使い方

Claude Code で `/takt-builder` コマンドを実行する。

```
/takt-builder 調査用の deep-research ピースを作りたい
/takt-builder default ピースにセキュリティレビューを追加
/takt-builder research 用の policy を作りたい
```

## 出力先

スキルは実行ディレクトリに応じて出力先を自動判定する。

| 条件 | 出力先 | 用途 |
|------|--------|------|
| TAKT プロジェクト内 | `builtins/{lang}/` | TAKT 本体の開発 |
| それ以外 | `.takt/` or `~/.takt/` | プロジェクトローカル / ユーザーグローバル |

## 同梱リファレンス

スタイルガイドとテンプレートが同梱されているため、TAKT プロジェクト外でも動作する。

```
references/
  STYLE_GUIDE.md                 # 全体アーキテクチャと分離判断フロー
  PERSONA_STYLE_GUIDE.md         # ペルソナ作成ガイド
  POLICY_STYLE_GUIDE.md          # ポリシー作成ガイド
  KNOWLEDGE_STYLE_GUIDE.md       # ナレッジ作成ガイド
  INSTRUCTION_STYLE_GUIDE.md     # インストラクション作成ガイド
  OUTPUT_CONTRACT_STYLE_GUIDE.md # 出力契約作成ガイド
  yaml-schema.md                 # ピースYAMLスキーマ
templates/
  personas/        # ペルソナテンプレート (simple, expert, character)
  policies/        # ポリシーテンプレート
  knowledge/       # ナレッジテンプレート
  instructions/    # インストラクションテンプレート (9種)
  output-contracts/ # 出力契約テンプレート (6種)
```

## ライセンス

MIT
