# ワークフローYAML スキーマリファレンス

このドキュメントはワークフローYAMLの構造を定義する。具体的なワークフロー定義は含まない。

## トップレベルフィールド

```yaml
name: workflow-name           # ワークフロー名（必須）
description: 説明テキスト      # ワークフローの説明（任意）
max_steps: 10                # 最大イテレーション数（必須）
initial_step: plan            # 最初に実行する step 名（必須）

workflow_config:              # ワークフローレベルのプロバイダ設定（任意）
  provider_options:
    codex:
      network_access: true
    claude:
      effort: high

# セクションマップ（キー → ファイルパスの対応表）
policies:                     # ポリシー定義（任意）
  coding: ../policies/coding.md
  review: ../policies/review.md
personas:                     # ペルソナ定義（任意）
  coder: ../personas/coder.md
  reviewer: ../personas/architecture-reviewer.md
instructions:                 # 指示テンプレート定義（任意）
  plan: ../instructions/plan.md
  implement: ../instructions/implement.md
report_formats:               # レポートフォーマット定義（任意）
  plan: ../output-contracts/plan.md
  review: ../output-contracts/architecture-review.md
knowledge:                    # ナレッジ定義（任意）
  architecture: ../knowledge/architecture.md

steps: [...]                  # step 定義の配列（必須）
loop_monitors: [...]          # ループ監視設定（任意）
```

### セクションマップの解決

各セクションマップのパスは **ワークフローYAMLファイルのディレクトリからの相対パス** で解決する。
step 内では**キー名**で参照する（パスを直接書かない）。

例: ワークフローが `~/.takt/workflows/coding.yaml` にあり、`personas:` セクションに `coder: ../facets/personas/coder.md` がある場合
→ 絶対パスは `~/.takt/facets/personas/coder.md`
→ step では `persona: coder` で参照

## Step 定義

### 通常 Step

```yaml
- name: step-name              # step 名（必須、一意）
  description: 説明テキスト     # step の説明（任意）
  persona: coder               # ペルソナキー（personas マップを参照、任意）
  policy: coding               # ポリシーキー（policies マップを参照、任意）
  policy: [coding, testing]    # 複数指定も可（配列）
  instruction: implement       # インストラクションキー（instructions マップを参照、任意）
  knowledge: architecture      # ナレッジキー（knowledge マップを参照、任意）
  edit: true                   # ファイル編集可否（必須）
  required_permission_mode: edit # 必要最小権限: edit / readonly / full（任意）
  session: refresh             # セッション管理（任意）
  pass_previous_response: true # 前の出力を渡すか（デフォルト: true）
  provider: claude             # プロバイダ指定（任意）
  model: model-name            # モデル指定（任意）
  provider_options:            # ステップレベルのプロバイダ設定（任意）
    claude:
      allowed_tools: [Read, Glob, Grep, Edit, Write, Bash]
      effort: high
    codex:
      network_access: true
  output_contracts: [...]      # 出力契約設定（任意）
  quality_gates: [...]         # 品質ゲート（AIへの指示、任意）
  rules: [...]                 # 遷移ルール（必須）
```

### Parallel Step

```yaml
- name: reviewers              # 親 step 名（必須）
  parallel:                    # 並列サブステップ配列（これがあると parallel step）
    - name: arch-review
      persona: architecture-reviewer
      policy: review
      knowledge: architecture
      edit: false
      instruction: review-arch
      output_contracts:
        report:
          - name: 05-architect-review.md
            format: architecture-review
      rules:
        - condition: "approved"
        - condition: "needs_fix"

    - name: qa-review
      persona: qa-reviewer
      policy: review
      edit: false
      instruction: review-qa
      rules:
        - condition: "approved"
        - condition: "needs_fix"

  concurrency: 2               # 並列実行数（任意、デフォルト: サブステップ数）
  rules:                       # 親の rules（aggregate 条件で遷移先を決定）
    - condition: all("approved")
      next: supervise
    - condition: any("needs_fix")
      next: fix
```

**重要**: サブステップの `rules` は結果分類のための condition 定義のみ。`next` は無視される（親の rules が遷移先を決定）。

**`parallel`、`arpeggio`、`team_leader` は排他的** — 1つの step に同時指定できない。

### Arpeggio Step（データ駆動バッチ処理）

```yaml
- name: batch-review
  persona: reviewer
  instruction: review-batch
  edit: false
  arpeggio:
    source: json                     # データソース種別（必須）
    source_path: ./items.json        # データソースパス（必須）
    batch_size: 1                    # バッチサイズ（デフォルト: 1）
    concurrency: 2                   # 並列実行数（デフォルト: 1）
    template: ./review-template.md   # テンプレートパス（必須）
    merge:                           # マージ設定（任意）
      strategy: concat               # concat / custom
      separator: "\n---\n"           # セパレータ（concat 時）
    max_retries: 2                   # リトライ回数（デフォルト: 2）
    retry_delay_ms: 1000             # リトライ遅延ms（デフォルト: 1000）
    output_path: ./output.md         # 出力先（任意）
  rules:
    - condition: 完了
      next: next-step
```

### Team Leader Step（動的パート分解）

```yaml
- name: develop
  persona: leader
  instruction: develop
  edit: true
  team_leader:
    max_parts: 3                     # 最大パート数（デフォルト: 3、最大3）
    refill_threshold: 0              # 補充閾値（デフォルト: 0）
    timeout_ms: 900000               # タイムアウトms（デフォルト: 900000）
    part_persona: worker             # パートのペルソナ（任意）
    part_edit: true                  # パートの編集可否（任意）
    part_permission_mode: edit       # パートの権限（任意）
  rules:
    - condition: 完了
      next: review
```

## Rules 定義

```yaml
rules:
  - condition: 条件テキスト      # マッチ条件（必須）
    next: next-step              # 遷移先 step 名（必須、サブステップでは任意）
    requires_user_input: true   # ユーザー入力が必要（任意）
    interactive_only: true      # インタラクティブモードのみ（任意）
    appendix: |                 # 追加情報（任意）
      補足テキスト...
```

### Condition 記法

| 記法 | 説明 | 例 |
|-----|------|-----|
| 文字列 | AI判定またはタグで照合 | `"タスク完了"` |
| `ai("...")` | AI が出力に対して条件を評価 | `ai("コードに問題がある")` |
| `all("...")` | 全サブステップがマッチ（parallel 親のみ） | `all("approved")` |
| `any("...")` | いずれかがマッチ（parallel 親のみ） | `any("needs_fix")` |
| `all("X", "Y")` | 位置対応で全マッチ（parallel 親のみ） | `all("問題なし", "テスト成功")` |

### 特殊な next 値

| 値 | 意味 |
|---|------|
| `COMPLETE` | ワークフロー成功終了 |
| `ABORT` | ワークフロー失敗終了 |
| step 名 | 指定された step に遷移 |

## Output Contracts 定義

Step の出力契約（レポート定義）。`output_contracts.report` 配列形式で指定する。

### 形式1: name + format（フォーマット参照）

```yaml
output_contracts:
  report:
    - name: 01-plan.md
      format: plan               # report_formats マップのキーを参照
```

`format` がキー文字列の場合、トップレベル `report_formats:` セクションから対応する .md ファイルを読み込み、出力契約指示として使用する。

### 形式1b: name + format（インライン）

```yaml
output_contracts:
  report:
    - name: 01-plan.md
      format: |                  # インラインでフォーマットを記述
        # レポートタイトル
        ## セクション
        {内容}
```

### 形式2: label + path（ラベル付きパス）

```yaml
output_contracts:
  report:
    - Summary: summary.md
    - Scope: 01-scope.md
    - Decisions: 02-decisions.md
```

各要素のキーがレポート種別名（ラベル）、値がファイル名。

## Quality Gates 定義

Step 完了時の品質要件を AI への指示として定義する。自動検証は行わない。

```yaml
quality_gates:
  - 全てのテストがパスすること
  - TypeScript の型エラーがないこと
  - ESLint 違反がないこと
```

配列で複数の品質基準を指定できる。エージェントはこれらの基準を満たしてから Step を完了する必要がある。

## テンプレート変数

インストラクションファイル内で使用可能な変数:

| 変数 | 説明 |
|-----|------|
| `{task}` | ユーザーのタスク入力（template に含まれない場合は自動追加） |
| `{previous_response}` | 前の step の出力（pass_previous_response: true 時、自動追加） |
| `{iteration}` | ワークフロー全体のイテレーション数 |
| `{max_steps}` | 最大イテレーション数 |
| `{step_iteration}` | この step の実行回数 |
| `{report_dir}` | レポートディレクトリ名 |
| `{report:ファイル名}` | 指定レポートファイルの内容を展開 |
| `{user_inputs}` | 蓄積されたユーザー入力 |
| `{cycle_count}` | loop_monitors 内で使用するサイクル回数 |

## Loop Monitors（任意）

```yaml
loop_monitors:
  - cycle: [step_a, step_b]           # 監視対象の step サイクル
    threshold: 3                       # 発動閾値（サイクル回数）
    judge:
      persona: supervisor              # ペルソナキー参照
      instruction: loop-monitor-judge  # インストラクションキー（参照解決）
      rules:
        - condition: 健全（進捗あり）
          next: step_a
        - condition: 非生産的（改善なし）
          next: alternative_step
```

特定の step 間のサイクルが閾値に達した場合、judge が介入して遷移先を判断する。

## provider_options について

`provider_options` はワークフローレベル（`workflow_config.provider_options`）とステップレベルの両方で指定可能。ステップレベルの設定がワークフローレベルより優先される。

```yaml
provider_options:
  claude:
    allowed_tools: [Read, Glob, Grep, Edit, Write, Bash]  # 許可ツール一覧
    effort: high                                           # 推論努力: low / medium / high / max
    sandbox:
      allow_unsandboxed_commands: true                     # サンドボックス外コマンド許可
      excluded_commands: [rm, git push]                    # 除外コマンド
  codex:
    network_access: true                                   # ネットワークアクセス許可
    reasoning_effort: medium                               # 推論努力: minimal / low / medium / high / xhigh
  opencode:
    network_access: true                                   # ネットワークアクセス許可
```

Claude Code の Skill として実行する場合、`allowed_tools` は参考情報として扱い、`edit` フィールドの方を権限制御に使用する。
