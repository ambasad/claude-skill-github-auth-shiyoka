# github-auth-shiyoka

> [!NOTE]
> このスキルは現在テスト中です。不具合・改善点があれば Issue でお知らせください。

Claude Code スキル — GitHubの認証ばセッティングしよか！

Fine-grained PAT（リポジトリ単位の権限制御）を最優先とし、SSH 方式（Deploy Key / 1Password SSH Agent）や GCM にも対応する。

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
git clone https://github.com/ambasad/claude-skill-github-auth-shiyoka.git ~/.claude/skills/github-auth-shiyoka
```

## 使い方

Claude Code のプロンプトで以下を入力：

```
/github-auth-shiyoka
```

起動時に以下を自動チェックしてから、環境に応じた方法で git の GitHub 認証を設定する：

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

- `git clone` / `git push` / `git pull` が追加入力なしで動く
- Fine-grained PAT でリポジトリ単位・操作種別まで権限を絞れる
- Deploy Key でリポジトリ専用の SSH 鍵を設定できる
- 複数の GitHub アカウントを Host エイリアスで使い分けられる
- 秘密鍵・PAT はすべて 1Password に保存され、ディスクに平文保存されない

> SSH config テンプレート（`templates/ssh_config_github`）はリポジトリに含まれているため、インストール後に手動作成する必要はありません。

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
