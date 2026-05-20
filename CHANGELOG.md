# CHANGELOG

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
