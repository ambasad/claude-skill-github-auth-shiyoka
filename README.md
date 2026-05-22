# claude-code-github-auth-shiyoka-skill

> [!NOTE]
> **v0.6.0** — 開発初期段階です。主要な認証フローは動作確認済みですが、環境によって想定外の問題が発生する可能性があります。不具合・改善点は Issue でお知らせください。

Claude Code 用 GitHub 認証スキル — GitHubの認証ばセッティングしよか！

「しよか（しようか）」で、GitHub 認証をサクッと完了させるスキルです。  
Fine-grained PAT（リポジトリ単位の権限制御）を最優先とし、SSH 方式（Deploy Key / 1Password SSH Agent）や GCM にも対応する。

> [!TIP]
> **Agentic Skill 対応** — デフォルトでは `/github-auth-shiyoka` と入力して起動します。  
> 「GitHub認証」「PAT設定」「git clone で認証エラー」などのキーワードを会話中に検出して **自動起動させたい場合** は、`~/.claude/skills/github-auth-shiyoka/SKILL.md` の frontmatter を変更してください：
> ```yaml
> disable-model-invocation: false
> ```

## 前提条件

- **Windows 11 + WSL2**（Ubuntu など）/ **macOS** / **Linux**
- WSL2 の場合：WSL interop が有効であること（`ssh.exe` が WSL から呼び出せる状態）
- 1Password 8.10 以降推奨（SSH Agent 機能が必要）

参考：[1Password SSH Agent - WSL2 integration](https://developer.1password.com/docs/ssh/integrations/wsl/)

## 動作環境

- Claude Code
- Git
- 以下のいずれか：
  - 1Password CLI (`op`)（Fine-grained PAT 方式・最優先）
  - 1Password デスクトップアプリ（Deploy Key / SSH 方式）
  - GCM（Git Credential Manager）（op なしの HTTPS 方式）

## インストール

```bash
git clone https://github.com/ambasad/claude-code-github-auth-shiyoka-skill.git ~/.claude/skills/github-auth-shiyoka
```

インストール後、Claude Code を再起動するとスキルが認識されます。認識されているか確認するには：

```
/github-auth-shiyoka
```

と入力してスキルが起動すれば成功です。

## 使い方

Claude Code のプロンプトで以下を入力：

```
/github-auth-shiyoka
```

起動時に **セキュリティ監査** を自動実行してからセットアップに進む：

**セキュリティ監査モード（常に最初に実行）**

1. 現在の GitHub 認証状態を診断（`gh auth status`）
2. PAT 種別を確認（Fine-grained か Classic か）
3. 有効期限を確認（期限切れ・30日以内の警告）
4. 権限スコープを精査（過剰権限があれば最小権限推奨を表示）
5. 1Password Vault 連携状況を確認
6. 総合セキュリティレポートを出力（問題点・即時対応策を箇点で）
7. 必要ならトークン回転手順を提案

その後、以下を自動チェックして環境に応じた設定を行う：

- OS 種別（WSL2 / macOS / Linux）
- SSH 疎通（`ssh.exe -T git@github.com`）
- SSH config の設定状態
- 1Password CLI (`op`) の有無とバージョン
- 既存の credential helper 設定
- 1Password SSH Agent の鍵一覧

すでに完了しているステップは自動的にスキップして案内する。

### 認証方式（優先順位順）

| 優先順位 | 方式 | 権限スコープ | 秘密情報の保存場所 |
|---|---|---|---|
| 1（最優先） | Fine-grained PAT + 1Password CLI | リポジトリ単位・操作種別 | 1Password Vault |
| 2 | Deploy Key | リポジトリ単位（SSH） | 1Password Vault（ディスク非保存） |
| 3 | SSH + 1Password SSH Agent | アカウント全体 | 1Password Vault（ディスク非保存） |
| 4 | GCM | アカウント全体 | OS キーチェーン |

### プロジェクトごとの認証設定

- **SSH 方式**：remote URL（`git@github-<alias>:org/repo.git`）に設定が記録される
- **HTTPS + PAT 方式**：`op.env` + `git config --local` でプロジェクト単位に設定

`op.env.example` をプロジェクトにコミットしておくと、他のメンバーが `cp op.env.example op.env` で即座に使い始められる：

```bash
# op.env.example（コミットする）
GITHUB_USERNAME=<GitHubユーザー名>
GITHUB_TOKEN=op://<Vault名>/<アイテム名>/credential
```

> **注意：** `op.env`（実際の値）は `.gitignore` に追加してコミットしないこと。

### SSH 方式（WSL2）の設定内容

- Linux 側 `~/.ssh/config` と Windows 側 `%USERPROFILE%\.ssh\config` の両方に Host ブロックを追記
  - `ssh.exe`（git が使用）は Windows 側 config を読むため両側への設定が必要
- Deploy Key・複数アカウントは Host エイリアス（`github-<名前>`）で使い分ける

## 設定後にできること

| 機能 | Step 2a (PAT+op) | Step 2b (GCM) | Step 3 (Deploy Key) | Step 4/4b (SSH Agent) |
|---|:---:|:---:|:---:|:---:|
| `git clone / push / pull` 自動認証 | ✅ | ✅ | ✅ | ✅ |
| リポジトリ単位の権限制限 | ✅ | - | ✅ | - |
| 秘密情報をディスクに保存しない | ✅ | - | ✅ | ✅ |
| PAT を 1Password で管理 | ✅ | - | - | - |
| SSH 鍵を 1Password で管理 | - | - | ✅ | ✅ |
| 複数 GitHub アカウントの使い分け | - | - | ✅ | ✅ (Step 4c) |

> SSH config テンプレート（`templates/ssh_config_github`）はリポジトリに含まれており、Step 4b 実行時に自動で Linux・Windows 両側に適用されます。

## 解除方法

**HTTPS + PAT 方式の場合：**

```bash
git config --global --unset credential.https://github.com.helper
# プロジェクトローカル設定を解除する場合
git config --local --unset credential.https://github.com.helper
```

**Deploy Key / SSH エイリアスの場合：**

```bash
ALIAS="github-<名前>"
WIN_USERNAME=$(cmd.exe /c 'echo %USERNAME%' 2>/dev/null | tr -d '\r')
sed -i "/^Host $ALIAS/,/^$/d" ~/.ssh/config 2>/dev/null || true
sed -i "/^Host $ALIAS/,/^$/d" "/mnt/c/Users/$WIN_USERNAME/.ssh/config" 2>/dev/null || true
```

> GitHub 側の Deploy Key も Settings → Deploy keys から削除すること。

**SSH + 1Password Agent の場合：**

```bash
# Linux・Windows 両側の SSH config から github.com ブロックを削除
sed -i '/^Host github\.com/,/^$/d' ~/.ssh/config 2>/dev/null || true
WIN_USERNAME=$(cmd.exe /c 'echo %USERNAME%' 2>/dev/null | tr -d '\r')
sed -i '/^Host github\.com/,/^$/d' "/mnt/c/Users/$WIN_USERNAME/.ssh/config" 2>/dev/null || true
git config --global --unset core.sshCommand
```

## 更新

```bash
cd ~/.claude/skills/github-auth-shiyoka
git pull
```

## テスト

[bats](https://github.com/bats-core/bats-core) が必要です。

```bash
# インストール（未インストールの場合）
npm install -g bats

# テスト実行
bats tests/github-auth-shiyoka.bats
```

## ライセンス

MIT
