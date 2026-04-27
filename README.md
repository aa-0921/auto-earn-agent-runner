# auto-earn-agent-runner

GitHub Actions の public 無料枠で動かすための runner リポジトリ。
ロジック本体は別 repo に置いている (workflow から PAT で clone して実行)。

## 構成

- `.github/workflows/scout.yml` — 1時間ごと cron + workflow_dispatch
- secrets: `PRIVATE_REPO_PAT` (classic, repo スコープ) / `GEMINI_API_KEY`

## 手動実行

```bash
gh workflow run Scout --repo aa-0921/auto-earn-agent-runner
```
