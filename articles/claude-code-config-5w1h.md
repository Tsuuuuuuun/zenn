---
title: "Claude Code 設定ファイルの「なぜ・どこに・何を」を 5W1H で整理した"
emoji: "🗂️"
type: "idea"
topics: ["AI", "claudecode"]
published: true
---

Claude Code を使い込んでいくと新しい機能にたくさん出くわす。`CLAUDE.md`、`skills/`、`agents/`、`commands/`、`hooks/`、`rules/`、`settings.json`、そして Claude Code の機能ではないが頻繁に用いられる `docs/`。「この情報はどこに書けばいいのか」と手が止まる瞬間が出てくる。

これらの機能に対して 5W1H に当てはめて整理したところ、「この情報はどこに書けばよいのか」という判断がしやすくなった。本記事ではその整理の方法を共有する。

## 全体の見取り図

まず、各設定ファイルが 5W1H のどこに対応するかを以下の一覧に示す。

| 要素 | 問い | 場所 | 一言で言うと |
|---|---|---|---|
| `CLAUDE.md` | Where | プロジェクトルート | プロジェクトの地図 |
| `docs/` | Why | `docs/decisions/` など | 意思決定の背景 |
| `skills/` | How | `.claude/skills/` | 作業手順・規約の教科書 |
| `agents/` | Who | `.claude/agents/` | サブエージェントの定義 |
| `commands/` | What | `.claude/commands/` | 定型フローの定義 |
| `hooks/` | When | `.claude/hooks/` | イベントベースの自動処理 |
| `rules/` | When | `.claude/rules/` | パスベースの自動読み込み |
| `settings.json` | 前提条件 | `.claude/settings.json` | 環境・権限の宣言 |

以降ではそれぞれの役割を、土台から積み上げる順番で説明していく。

## `CLAUDE.md` — Where：プロジェクトの地図

`CLAUDE.md` はセッション開始時に自動で読み込まれるファイルで、プロジェクトの基本情報を Claude に伝える役割を持つ。

`CLAUDE.md` はプロジェクトの地図である。プロジェクトの概要、技術スタック、ディレクトリ構造を与えることで Claude はこのプロジェクトの全体像を把握できる。ビルドやテストなどの実行コマンドを与えることで、地図を頼りにプロジェクトをどう探検すればよいかも伝えられる。また、禁止事項を明文化しておけば、地図上の立ち入り禁止地区として Claude に認識させることができる。

```md:CLAUDE.md
# プロジェクト概要
ユーザー認証とプロフィール管理を提供する REST API。
FastAPI + SQLAlchemy + Pydantic で構成。

## ディレクトリ構造
.
├─ app/
│   ├─ models/  — データベースモデル
│   ├─ api/     — ルートハンドラ
│   └─ core/    — 設定とユーティリティ
└─ tests/       — テストコード

## コマンド
- `make dev`  — 開発サーバー起動
- `make test` — テスト実行
- `make lint` — リンター実行

## 禁止事項
- `app/core/config.py` を直接編集しない（環境変数で上書きすること）
- マイグレーションファイル（`migrations/`）を手動で変更しない
- `main` ブランチへの直接プッシュ禁止
```

`CLAUDE.md` は毎セッション読み込まれるため、あらゆる規約や手順を詰め込むとコンテキストウィンドウを圧迫し、Claude の注意力が分散する。詳細な手順は後述する `skills/` に、設計の背景は `docs/` に分離して、`CLAUDE.md` からはパスで参照する形にする。

## `docs/` — Why：意思決定の背景

### コードが what の正、`docs/` が why の正

API のエンドポイント仕様やディレクトリ構造はコードを読めばわかる。それを `docs/` に書き写しても二重管理になるだけで、遅かれ早かれどちらかが古くなる。

`docs/` に残すのはコードからは消える「なぜ」だけにする。なぜ PostgreSQL を選んだのか、なぜイベントソーシングを採用したのか、なぜ別のアプローチを取らなかったのか。こうした判断の根拠は、コードをどれだけ読んでも復元できない。

### PR description の延長として考える

「なぜこの実装にしたか」を PR の description に書く習慣は多くのエンジニアに馴染みがあるはずだ。`docs/` に ADR（Architecture Decision Records）を残すのは、その延長にある。PR がマージされたら、そこに書いた why を `docs/decisions/` に昇格させるイメージだ。

逆に言えば、検討中の議論は Issue や PR に任せればいい。`docs/` の中身を全部確定情報に保てることで、Claude に「ここに書いてあることは信頼してよい」と宣言できる。

### 「なぜ採用しなかったか」も ADR に統合する

否定の理由には特に価値がある。「DynamoDB を検討したが、トランザクション要件との相性から PostgreSQL を選んだ」のように、却下した選択肢との根拠を残すことで、後から同じ議論を繰り返すのを防げる。Claude に対しても「この方向はすでに検討済み。こういう理由で却下された」と伝えられる。

### ディレクトリ構成例

```
docs/
├── decisions/      # 確定済みの設計判断（ADR）
│   ├── 0001-use-postgres-over-dynamodb.md
│   └── template.md
├── context/        # ビジネスドメイン・制約条件
├── architecture/   # 構造の「なぜ」
├── history/        # 大規模な変更の経緯
└── archive/        # 役目を終えたドキュメントの退避先
```

`decisions/` が中心で、ADR を連番で管理する。`context/` にはコードに現れない前提（「利用者は社内の非エンジニアが中心」「レイテンシよりデータ整合性を優先する」など）を置く。`architecture/` にはモジュール分割やレイヤー構成の意図を、`history/` には大規模なマイグレーションの経緯を残す。

### 陳腐化への備え

`docs/` に why を蓄積していくと、いずれ内容が古くなる問題に直面する。古すぎる why は「ない」より厄介で、Claude が自信を持って間違った方向に進む原因になる。

ただし、完全な鮮度維持を目標にする必要はない。以下の2つを押さえておけば、安全に壊れる設計になる。

まず、ADR の frontmatter に `status` を持たせる。

```md
---
status: accepted     # accepted / superseded / deprecated
date: 2025-06-15
superseded_by: "0012"  # 上書きされた場合のみ
---
```

役目を終えたドキュメントは `archive/` へ移動する。`CLAUDE.md` に「`archive/` は明示的に指示されない限り読まない」と書いておけば、古い why がコンテキストを汚さない。

次に、`CLAUDE.md` にセーフティネットを一行加える。

```md:CLAUDE.md
## `docs/` の読み方
- `docs/` の記述とコードの実態が矛盾している場合、コードを正とする
```

これだけで、ドキュメントの更新が多少遅れても Claude が致命的な誤判断をするリスクを下げられる。古い情報があっても Claude が正しく重み付けできる状態を作ることがゴールであって、ドキュメントを常に最新に保つこと自体は目標にしない。

## `skills/` — How：作業手順・規約の教科書

Claude に毎回同じ説明をしているならば、それは Skill にする合図だ。`skills/` にはプロジェクト固有の手順や規約をマークダウンで置く。

```
.claude/skills/
├── api-conventions/
│   └── SKILL.md
├── testing-patterns/
│   └── SKILL.md
└── db-migrations/
    └── SKILL.md
```

`SKILL.md` にはフロントマターでメタ情報を記述し、本文に具体的な手順やパターンを書く。

```md:.claude/skills/api-conventions/SKILL.md
---
name: api-conventions
description: REST API の設計規約。エンドポイント追加や修正時に参照する。
allowed-tools: Read, Grep, Glob
---

# API 設計規約

## エンドポイントの命名
- リソース名は複数形（/users, /orders）
- ネストは2階層まで（/users/{id}/orders は可、それ以上は避ける）
```

```md
## レスポンス形式
- 成功時は {"data": ...} で包む
- エラー時は {"error": {"code": "...", "message": "..."}}
- ページネーションは cursor ベース
```

`description` の内容を Claude が意味的にマッチングして、関連する作業のときに自動で参照するかどうかを判断する。また `/api-conventions` のようにスラッシュコマンドで明示的に呼び出すこともできる。

`docs/` との棲み分けとしては、`docs/` が「なぜそうするか」、`skills/` が「具体的にどうやるか」という対応になる。たとえば「cursor ベースのページネーションを採用した理由」は `docs/decisions/` に、「その実装パターン」は `skills/` に置く。

## `agents/` — Who：サブエージェントの定義

`agents/` の本質はコンテキストの分離だ。チーム開発で調査担当、実装担当、レビュー担当を分けるのと同じように、Claude にもサブエージェントとして役割ごとに異なるコンテキストを与えられる。

```md:.claude/agents/api-reviewer.md
---
name: api-reviewer
description: API の変更に対してレビューを行う。
tools:
  - Read
  - Grep
  - Glob
skills:
  - api-conventions
  - testing-patterns
---

# API Reviewer

## レビュー観点
- エンドポイントの命名が規約に沿っているか
- エラーハンドリングが統一されているか
- 既存のテストが壊れていないか
- 破壊的変更がないか（ある場合はマイグレーション計画を確認）

## 制約
- ファイルの変更は行わない
- 指摘は具体的なコード箇所とともに報告する
```

ここで重要なのは、知識は `skills/` に一元管理し、`agents/` のサブエージェントはそれを参照するだけという構造にすることだ。上の例では `skills: [api-conventions, testing-patterns]` と `preload` しているが、規約の中身自体は `skills/` 側に書いている。こうしておけば規約を更新したときに `agents/` を個別に修正する必要がない。

ツールの制限もサブエージェントの重要な機能で、調査用のサブエージェントにはファイル変更を許可しない、といった制御ができる。

## `commands/` — What：定型フローの定義

`commands/` は複数のサブエージェントや作業ステップを束ねるパイプラインの入口になる。`/command-name` で呼び出す。

`skills/` が「どうやるか」を定義するのに対し、`commands/` は「何をどの順番で実行するか」を定義する。たとえば「PR レビューコマンド」であれば、変更差分の取得、`api-reviewer` サブエージェントによるレビュー、結果のサマリー出力、といった一連のフローを1つのコマンドにまとめられる。定型的な作業手順を自動化し、繰り返し実行するのに向いている。

ただし、現在の Claude Code では `commands/` と `skills/` は統合されており、`.claude/commands/deploy.md` と `.claude/skills/deploy/SKILL.md` はどちらも `/deploy` として同じように動作する。`skills/` でもフローの定義は可能ではある。実装上は `skills/` に一本化してもよいし、概念的な区分けとして `commands/` を残してもよいと思う。

## `hooks/` ・ `rules/` — When：条件付きの自動処理

`hooks/` と `rules/` はどちらも「あるタイミングで自動的に何かを差し込む」仕組みだが、トリガーの性質が異なる。

### `hooks/`：イベントベースの割り込み

`hooks/` は「何かが起きたとき」に処理を自動実行する。設定は `settings.json` に記述する。

たとえば、Claude がファイルを編集するたびにフォーマッターを走らせたい場合はこう書く。

```jsonc:.claude/settings.json
// hooks 部分の抜粋
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write \"$CLAUDE_FILE_PATH\""
          }
        ]
      }
    ]
  }
}
```

`PostToolUse` はツールの実行直後に発火するイベントで、`matcher` で対象のツールを絞り込む。マッチャーの中に `hooks` 配列をネストし、各フックに `type` を指定する構造になっている。この例ではファイルの編集・作成が行われるたびに `prettier` が走り、スタイルの揺れをセッション中に潰せる。

他にも、`PreToolUse` フックで `rm -rf` や `git push --force` の実行前にブロックする、といった防御的な使い方もある。

### `rules/`：パスベースの自動読み込み

`rules/` は「特定のファイルを触るとき」にだけ読み込まれるルールを定義する。ルールファイルの先頭でパスの glob パターンを指定し、本文にそのスコープで適用したい指示を書く。

たとえば、テストファイルを扱うときだけテスト規約を読み込ませたい場合はこう書く。

```markdown:.claude/rules/testing-conventions.md
---
paths:
  - "tests/**"
  - "**/*_test.py"
---

# テスト規約

- テスト関数名は test_<対象>_<条件>_<期待結果> の形式にする
- フィクスチャは conftest.py に集約し、テストファイル内で定義しない
- モックは unittest.mock を使い、サードパーティのモックライブラリは導入しない
- 1テスト関数につき assert は原則1つ。複数の検証が必要な場合はパラメタライズを使う
```

`paths` に glob パターンを指定することで、該当ファイルに触れた時点で自動的にルールが読み込まれる。開発者が明示的に呼び出す必要がない。

同じ要領で、`**/*.py` に Python のスタイル規約を、`src/db/**` に DB マイグレーションのルールを、といった形でスコープを限定した指示を増やしていける。`skills/` との違いは、`rules/` は対象ファイルの作業中に常時適用される点だ。

## 全体のディレクトリ構成

ここまでの要素をすべて合わせると、プロジェクトのディレクトリ構成は以下のようになる。

```
your-project/
├── CLAUDE.md                  # Where：プロジェクトの地図
├── docs/                      # Why：意思決定の背景
│   ├── decisions/
│   │   ├── 0001-use-postgres-over-dynamodb.md
│   │   └── template.md
│   ├── context/
│   ├── architecture/
│   ├── history/
│   └── archive/
├── .claude/
│   ├── settings.json          # hooks の登録、権限設定
│   ├── skills/                # How：作業手順・規約の教科書
│   │   ├── api-conventions/
│   │   │   └── SKILL.md
│   │   └── testing-patterns/
│   │       └── SKILL.md
│   ├── agents/                # Who：サブエージェントの定義
│   │   └── api-reviewer.md
│   ├── commands/              # What：作業フローの定義
│   │   └── pr-review.md
│   └── rules/                 # When：パスベースの自動読み込み
│       ├── testing-conventions.md
│       └── python-style.md
```

## 全体の依存関係

ここまで紹介した要素は独立しているわけではなく、依存関係を持って連携する。

```
docs/（Why）+ 設計の意図
  ↓ 参照
skills/（How）+ 知識の一元管理
  ↓ preload
agents/（Who）+ 独立したサブエージェント
  ↓ Task ツールで起動
commands/（What）+ パイプラインの入口
  ↓
hooks/ / rules/（When）+ 条件付き割り込み
```

`docs/` の設計意図を踏まえて `skills/` に具体的な手順を書き、`agents/` のサブエージェントがそれを `preload` し、`commands/` がサブエージェントを組み合わせてフローを構成する。`hooks/` と `rules/` はその過程で条件に応じて割り込む。この流れを意識しておくと、新しいルールや手順を追加するときに「どこに書くか」で迷わなくなる。

## チームで運用するための issue テンプレート

5W1H の整理は、チームメンバーが新しい設定の追加を起票するときの判断軸にもなる。以下のような issue テンプレートを用意しておくと、起票者が各要素の違いを正確に把握していなくても、レビュワーが分類できる材料が揃う。

```md
## 解決したいこと
<!-- Claude に毎回説明していること、自動化したい作業、守らせたいルールなど -->

## いつ・どこで発生するか
<!-- 特定のファイルを触るとき？毎回？特定の作業フローの中で？ -->

## 対応する要素（該当するものを選択）

- [] `skills/` - 毎回同じ説明をしている手順や規約
- [] `rules/` - 特定のパスを触るときだけ適用したいルール
- [] `agents/` - 独立したコンテキストで実行させたいタスク
- [] `commands/` - 複数ステップに定型フロー
- [] `hooks/` - ツール実行前後に自動で走らせたい処理
- [] `docs/` - 設計判断の背景情報
- [] わからない

## 具体的な内容
<!-- わかる範囲で。規約の箇条書き、対象パス、実行タイミングなど -->

```

## まとめ

改めて全体の対応表を再掲する。

| 要素 | 問い | 一言で言うと |
|---|---|---|
| `CLAUDE.md` | Where | プロジェクトの地図 |
| `docs/` | Why | 意思決定の背景 |
| `skills/` | How | 作業手順・規約の教科書 |
| `agents/` | Who | サブエージェントの定義 |
| `commands/` | What | 定型フローの定義 |
| `hooks/`・`rules/` | When | 条件付きの自動処理 |

## 参考

https://docs.claude.com/ja/docs/claude-code/settings

https://gihyo.jp/book/2026/978-4-297-15354-0
