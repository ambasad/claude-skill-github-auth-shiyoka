# CHANGELOG

## v0.4.0 - 2026-05-20

### 改善
- SKILL.md: 全体手順の案内を HTTPS 方式 / SSH 方式の分岐フローに刷新
- SKILL.md: Step 4b（WSL2）と Step 4c（複数アカウント）の位置づけを明確化
- SKILL.md: Step 2a の `op item list` を API Credential カテゴリで絞り込む形に改善

### 追加
- SKILL.md: 解除手順に GitHub 側の PAT 削除 URL を追記
- SKILL.md: 解除手順に GitHub 側の Deploy Key 削除 URL を追記

---

## v0.3.0 - 2026-05-20

### 修正
- テスト: `grep -c` の出力比較を `-eq 1` から `= "1"` に修正（文字列比較に統一）
- テンプレート: `templates/ssh_config_github` の先頭の余分な空行を削除
- SKILL.md: 冒頭の「WSL2 環境を想定」という記述を「WSL2 / macOS / Linux 対応」に修正

### 追加
- テスト: `op.env` の `.gitignore` への追記・重複防止テストを追加
- テスト: Deploy Key の Host エイリアス追記・重複防止テストを追加
- README.md: 「設定後にできること」を方式別の機能比較テーブルに刷新

---

## v0.2.0 - 2026-05-20

### 追加
- SKILL.md: YAML frontmatter に `version` フィールドを追加
- SKILL.md: 事前チェックに `op --version` によるバージョン確認を追加
- SKILL.md: トラブルシューティングセクションを追加（Permission denied / op not found / WSL2 config 反映されない / セッションタイムアウト / ssh.exe not found）
- README.md: スキル起動時の自動チェック内容を明記
- README.md: 開発中バナー（NOTE）を追加

### 改善
- SKILL.md: Step 2a のサブ手順を `###` 見出し形式に統一（他 Step と一致）

---

## v0.1.0 - 2026-05-20

### 追加
- Fine-grained PAT + 1Password CLI（Step 1 / 2a）
- GCM（Git Credential Manager）（Step 2b）
- Deploy Key（Step 3）
- SSH + 1Password SSH Agent（Step 4 / 4b）
- 複数 GitHub アカウント対応（Step 4c）
- WSL2 専用対応（`ssh.exe` 経由・Windows / Linux 両側 SSH config 設定）
- 動作確認・解除手順（Step 5）
- bats によるテスト（`tests/github-auth-shiyoka.bats`）
- `op.env.example` テンプレート
- `templates/ssh_config_github` テンプレート
