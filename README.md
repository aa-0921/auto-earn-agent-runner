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

---

## 運用フロー: public runner で private repo を回す（無料枠無制限）

### 背景・狙い

private repo の GitHub Actions は月 2000 分制限あり。1 時間ごとの cron だと月 720 分使う想定で長期運用するとじわじわ消費する。一方 **public repo の Actions は同一機能で完全無料・無制限**。

そこでこの runner repo (public) は workflow 定義だけを持ち、実装本体は private repo `aa-0921/auto-earn-agent` に残す構成にしている。public 側はコードを公開しないため、AI スロップ評判リスクや実装パクリも防げる。

検証実績: 2026-04-28 切替直後の run 25008462050 で 58 秒・Issues #32-#41 起票・bot commit b1459d0 push 成功。意図通り動作確認済み。

### 必要 Secrets（runner 側に登録）

- `PRIVATE_REPO_PAT` — classic Personal Access Token / scope `repo`（private 読み書き）
  - private 側 `aa-0921/auto-earn-agent` の clone・data/scout.db への commit/push に使用
  - Issue 起票 (`gh issue create` 系) も同 PAT で実行
- `GEMINI_API_KEY` — 候補スコアリング用 Gemini API キー（`gemini-2.0-flash` 推奨。`gemini-2.5-flash` は無料枠 RPD 20 で枯渇しやすい — `~/.claude/lessons.md` 参照）

### Workflow ステップ（`.github/workflows/scout.yml` の構成）

1. **Checkout (private)** — `actions/checkout@v4` で `repository: aa-0921/auto-earn-agent` + `token: ${{ secrets.PRIVATE_REPO_PAT }}` を指定し、`path: auto-earn-agent` のサブディレクトリに clone
2. **Setup Python** — `actions/setup-python@v5` (Python 3.11) + pip キャッシュ (`cache-dependency-path: auto-earn-agent/requirements.txt`)
3. **Install deps** — `pip install -r auto-earn-agent/requirements.txt`
4. **Run scout** — `working-directory: auto-earn-agent` で `python -m scout`
   - env: `GEMINI_API_KEY` / `GH_TOKEN: PRIVATE_REPO_PAT` / `GH_REPO: aa-0921/auto-earn-agent` / `DEBUG` / `THRESHOLD` / `SOURCES`
5. **Commit DB to private repo** — `if: always()` で `data/scout.db` を private 側へ commit & push（`[skip ci]` を付けて再実行ループ防止）

### permissions・concurrency

- `permissions: contents: read` で十分（private 側への書き込みは PAT 経由のため public 側 GITHUB_TOKEN は read のみで OK）
- `concurrency: group: scout, cancel-in-progress: false` を必ず指定 — 並行 cron 実行時に SQLite (`data/scout.db`) の競合 commit を直列化するため

### 応用範囲

- 「private 側にコード集約・public 側に runner だけ置く」パターンは他の cron 自動化でも転用可能
- 副次効果: コード非公開 + Actions 分浪費なし + 実行回数を実質無制限化
- 注意: PAT は classic を使う（fine-grained だと concurrency 周りや commit 認証で詰まる事例あり）

### 教訓ソース

`~/.claude/lessons.md` の **「public Actions runner から PAT で private repo を clone/commit すれば無料枠が無制限（2026-04-28）」** エントリ参照。
