# .github

april-knights-solution の repo で共通に使う GitHub の設定。

| パス | 中身 |
|---|---|
| `.github/workflows/claude-review.yml` | PR を Claude がレビューしてコメントする共通の workflow。各 repo から呼ぶ |
| `review/` | レビューの本体（Claude Agent SDK を使う Node.js のスクリプト） |
| `workflow-templates/claude-review.yml` | 各 repo に置く呼び出し側の雛形（`.properties.json` は Actions の画面に出す説明） |
| `pull_request_template.md` | 自前の PR テンプレートを持たない repo で使われる、既定の PR テンプレート |

## Claude の PR レビューを入れる

PR が開かれたとき、push で更新されたとき、下書きから外れたときに、Claude が差分を読んで日本語でコメントを 1 件付ける。
見るのはバグ・セキュリティ・本番に出る影響の 3 つ。同じ PR で動き直すと、前のコメントを書き換える。
下書きの PR、dependabot の PR、変更が 1,500 行を超える PR はレビューしない。続けて push したときは、古い実行を止めて新しいほうだけを回す。

### 1. repo に secret を置く

repo の Settings → Secrets and variables → Actions → New repository secret で、
`ANTHROPIC_API_KEY` を登録する。値は Claude Console の API キー。

GitHub の無料プランでは、組織の secret を非公開の repo から読めないので、repo ごとに登録する。

### 2. 呼び出し側の workflow を置く

repo の Actions → New workflow で「Claude PR review」を選ぶか、
`workflow-templates/claude-review.yml` を `.github/workflows/claude-review.yml` としてコピーする。

マージで本番に出る repo では `release_impact: true` にする。コメントの冒頭に「本番に出るもの」が 1〜2 行で入る。

`pull_request` で動くので、呼び出し側の workflow を足した PR からレビューが付く。

### 変えられる値

| 名前 | 既定 | 意味 |
|---|---|---|
| `release_impact` | `false` | コメントの冒頭に本番への影響を書くか |
| `runs_on` | `"ubuntu-latest"` | ランナーのラベル（JSON）。自前のランナーなら `'["self-hosted","ラベル"]'` |
| `model` | `sonnet` | 使うモデル（`sonnet`・`opus`・`haiku` などの別名か、モデル ID） |
| `max_turns` | `30` | 1 回のレビューの上限ターン数 |
| `max_budget_usd` | `2` | 1 回のレビューの費用の上限（米ドル）。超えるとそこで止まる |
| `max_changed_lines` | `1500` | これを超える PR はレビューしない |

コメントの末尾に、使った token 数と費用の見込みが小さく出る。費用は SDK が token 数から計算した見込みで、Console の請求額と一致するとは限らない。

### 呼び出し側の権限

呼び出し側の job には `contents: read` と `pull-requests: write` が要る（雛形に入っている）。
フォークからの PR では secret が渡らないので、レビューは失敗する。

### 手元で試す

```sh
cd review && npm ci
node review.mjs review --dry-run --meta pr.json --diff pr.diff --repo ../対象の repo
```

`pr.json` は `{"repo","number","title","body","author","base"}` を持つ JSON。`--dry-run` ではコメントの本文を標準出力に出し、GitHub には投稿しない。
`ANTHROPIC_API_KEY` が無いときは Claude を呼ばず、プロンプトを標準エラーに、空のレビューの本文を標準出力に出す。
