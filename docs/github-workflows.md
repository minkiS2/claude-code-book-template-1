# GitHub Actions ワークフロー設定

このドキュメントでは、`.github/workflows` ディレクトリに配置されている GitHub Actions ワークフローの設定内容について説明します。

このリポジトリには以下の2つのワークフローが定義されています。

| ファイル名 | 概要 |
| --- | --- |
| [`claude.yml`](../.github/workflows/claude.yml) | Issue や PR のコメントで `@claude` にメンションすると Claude Code が応答・作業を行う |
| [`claude-code-review.yml`](../.github/workflows/claude-code-review.yml) | Pull Request が作成・更新された際に Claude Code が自動でコードレビューを行う |

いずれも [`anthropics/claude-code-action@v1`](https://github.com/anthropics/claude-code-action) を利用しています。

---

## 1. `claude.yml`（Claude Code）

`@claude` のメンションをトリガーに Claude Code を起動し、Issue への回答や PR の作成・修正などの作業を行うワークフローです。

### トリガー（`on`）

以下のイベントで実行されます。

- `issue_comment`（`created`）: Issue や PR のコメントが作成されたとき
- `pull_request_review_comment`（`created`）: PR のレビューコメント（コード行への差分コメント）が作成されたとき
- `issues`（`opened`, `assigned`）: Issue が作成された、またはアサインされたとき
- `pull_request_review`（`submitted`）: PR レビューが送信されたとき

### 実行条件（`if`）

ワークフロー自体は上記イベントで毎回起動しますが、`claude` ジョブは次のいずれかの条件を満たす場合のみ実行されます。

- `issue_comment` イベントで、コメント本文に `@claude` が含まれる
- `pull_request_review_comment` イベントで、コメント本文に `@claude` が含まれる
- `pull_request_review` イベントで、レビュー本文に `@claude` が含まれる
- `issues` イベントで、Issue の本文またはタイトルに `@claude` が含まれる

つまり、`@claude` というキーワードを含む場合にのみ Claude Code が起動します。

### 権限（`permissions`）

| 権限 | 値 | 用途 |
| --- | --- | --- |
| `contents` | `read` | リポジトリの内容を読み取るため |
| `pull-requests` | `read` | PR の情報を読み取るため |
| `issues` | `read` | Issue の情報を読み取るため |
| `id-token` | `write` | OIDC トークンの発行のため |
| `actions` | `read` | PR に紐づく CI（Actions）の実行結果を Claude が参照できるようにするため |

### ジョブ内容（`jobs.claude`）

1. **`actions/checkout@v4`**: リポジトリをチェックアウトします（`fetch-depth: 1` で最新コミットのみ取得）。
2. **`anthropics/claude-code-action@v1`**: Claude Code 本体を実行します。
   - `claude_code_oauth_token`: `secrets.CLAUDE_CODE_OAUTH_TOKEN` を使用して認証します。
   - `additional_permissions`: `actions: read` を追加設定し、PR に対する CI 結果を Claude が読み取れるようにしています。
   - `prompt`（任意・コメントアウト中）: カスタムプロンプトを指定したい場合に使用します。指定しない場合、`@claude` を含むコメント本文の指示に従って動作します。
   - `claude_args`（任意・コメントアウト中）: `--allowed-tools` など、Claude Code の動作をカスタマイズする追加オプションを指定できます。詳細は [claude-code-action の usage ドキュメント](https://github.com/anthropics/claude-code-action/blob/main/docs/usage.md) および [CLI リファレンス](https://code.claude.com/docs/en/cli-reference) を参照してください。

---

## 2. `claude-code-review.yml`（Claude Code Review）

Pull Request の作成・更新時に、Claude Code によるコードレビューを自動実行するワークフローです。

### トリガー（`on`）

- `pull_request`（`opened`, `synchronize`, `ready_for_review`, `reopened`）
  - PR の作成時、コミット追加時（synchronize）、Draft から Ready for Review への変更時、再オープン時に実行されます。
  - コメントアウトされた `paths` 設定を有効化することで、特定のファイルパス（例: `src/**/*.ts` など）が変更された場合のみ実行するよう制限することも可能です。

### 実行条件

デフォルトでは `if` 条件は設定されておらず、上記トリガー発生時に常に実行されます。コメントアウトされた設定を有効化すると、特定の PR 作成者（例: 外部コントリビューターや初回コントリビューター）に限定してレビューを実行することもできます。

### 権限（`permissions`）

| 権限 | 値 | 用途 |
| --- | --- | --- |
| `contents` | `read` | リポジトリの内容を読み取るため |
| `pull-requests` | `read` | PR の情報を読み取るため |
| `issues` | `read` | Issue の情報を読み取るため |
| `id-token` | `write` | OIDC トークンの発行のため |

### ジョブ内容（`jobs.claude-review`）

1. **`actions/checkout@v4`**: リポジトリをチェックアウトします（`fetch-depth: 1`）。
2. **`anthropics/claude-code-action@v1`**: Claude Code によるレビューを実行します。
   - `claude_code_oauth_token`: `secrets.CLAUDE_CODE_OAUTH_TOKEN` を使用して認証します。
   - `plugin_marketplaces`: `https://github.com/anthropics/claude-code.git` をプラグインマーケットプレイスとして指定します。
   - `plugins`: `code-review@claude-code-plugins` プラグインを使用します。
   - `prompt`: `/code-review:code-review --comment <owner>/<repo>/pull/<PR番号>` を実行し、対象 PR に対してコードレビューを行い、その結果をインラインコメントとして投稿します。
   - `claude_args`: `--allowedTools "mcp__github_inline_comment__create_inline_comment"` を指定し、レビュー結果を PR にインラインコメントとして投稿できるように許可しています。

---

## 共通の前提事項

- 両ワークフローとも、実行には `secrets.CLAUDE_CODE_OAUTH_TOKEN` というリポジトリシークレットが必要です。事前に GitHub リポジトリの Settings > Secrets and variables > Actions に設定しておく必要があります。
- `anthropics/claude-code-action@v1` の詳細な設定オプションについては、[claude-code-action の usage ドキュメント](https://github.com/anthropics/claude-code-action/blob/main/docs/usage.md) を参照してください。
- Claude Code は `.github/workflows` 配下のワークフローファイル自体を変更する権限は持っていません（GitHub App の権限による制限）。
