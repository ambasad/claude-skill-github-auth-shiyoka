---
name: github-auth-shiyoka
description: GitHubの認証ばセッティングしよか（1Password CLIがあれば自動取得、なければ手動入力）
version: "0.2.0"
---

# github-auth-shiyoka

## スキルの入手方法

`~/.claude/skills/` 配下に直接クローンする。
クローン先を `github-auth-shiyoka` に指定することで、そのまま Claude Code に認識される。

```bash
git clone https://github.com/ambasad/claude-skill-github-auth-shiyoka.git ~/.claude/skills/github-auth-shiyoka
```

### スキルを最新に更新する場合

```bash
cd ~/.claude/skills/github-auth-shiyoka
git pull
```

---

## 使い方

Claude Code のプロンプトで以下のように入力する：

```
/github-auth-shiyoka
```

実行すると Claude が 1Password CLI (`op`) の有無を確認し、
環境に応じた方法で git の GitHub 認証を設定する。

### 事前準備（1Password CLI を使う場合）

プロジェクトルートに `op.env` を作成しておく（なければスキル実行時に自動作成）：

```bash
GITHUB_USERNAME=<GitHubユーザー名>
GITHUB_TOKEN=op://<Vault名>/<アイテム名>/credential
```

> **注意：** `op.env` には実際のトークンではなく 1Password の参照パスを記載する。
> `.gitignore` に `op.env` を追加してリポジトリにコミットしないこと。

### 設定後にできること

- `git clone https://github.com/<組織>/<リポジトリ>.git` がそのまま動く
- `git push` / `git pull` も認証なしで動く
- 解除したい場合は再度 `/github-auth-shiyoka` を実行して解除手順を依頼する

---

> **前提条件：** Windows 11 + WSL2 環境を想定している。WSL interop が有効で `ssh.exe` が WSL から呼び出せること。
> macOS / Linux ネイティブ環境では Step 1b は対象外となり、Step 1 または Step 2〜4 を使う。

git credential helper を設定し、以降の `git clone` / `git push` / `git pull` で
GitHub 認証を自動的に行えるようにする。

このスキルは以下の方式を扱う。リポジトリ単位の権限制御を重視した優先順位：

| 優先順位 | 方式 | 権限スコープ | 特徴 |
|---|---|---|---|
| 1（最高） | Fine-grained PAT + 1Password CLI | リポジトリ単位・操作種別 | 最小権限・PAT を 1Password で管理 |
| 2 | Deploy Key | リポジトリ単位 | SSH・特定リポジトリ専用鍵・CI/CD 向き |
| 3 | SSH + 1Password SSH Agent | アカウント全体 | PAT 不要・秘密鍵ディスク非保存 |
| 4 | GCM（Git Credential Manager） | アカウント全体 | OAuth 使用・OS キーチェーンに保存 |

スキル起動時に、まず **事前チェック** を実行して現在の設定状態を把握してから、全体手順を案内する。

## 事前チェック（スキル起動時に必ず実行）

> **重要：** `~/.ssh/config` はセキュリティポリシーにより Claude が直接読み取れない。
> 以下の手順で Claude が実行できるチェックとユーザーに依頼するチェックを分けて行う。

### 1. Claude が直接実行するチェック

以下をまとめて実行し、結果を解釈する：

```bash
# OS 判定
if grep -qi microsoft /proc/version 2>/dev/null; then
  OS_TYPE="WSL2"
elif [[ "$(uname)" == "Darwin" ]]; then
  OS_TYPE="macOS"
else
  OS_TYPE="Linux"
fi
echo "OS: $OS_TYPE"

# OS に応じた SSH / ssh-add コマンドを決定
if [[ "$OS_TYPE" == "WSL2" ]]; then
  SSH_CMD="ssh.exe"
  SSH_ADD_CMD="ssh-add.exe"
  # WSL2: ssh.exe の存在確認
  command -v ssh.exe >/dev/null 2>&1 && echo "ssh.exe: OK" || echo "⚠️ ssh.exe が見つかりません（WSL interop が無効の可能性あり）"
else
  SSH_CMD="ssh"
  SSH_ADD_CMD="ssh-add"
fi

# SSH 疎通確認
$SSH_CMD -T git@github.com 2>&1 || true

# SSH config の github.com 設定確認
echo "--- SSH config (github.com) ---"
$SSH_CMD -G github.com 2>/dev/null | grep -E "^(hostname|user) "

if [[ "$OS_TYPE" == "WSL2" ]]; then
  # WSL2: Windows 側 config も確認
  echo "--- Windows SSH config ---"
  ssh.exe -G github.com 2>/dev/null | grep -E "^(hostname|user) "
  if ssh.exe -G github.com 2>/dev/null | grep -q "^user git$"; then
    echo "✅ Windows SSH config: 設定済み (user git)"
  else
    echo "🔧 Windows SSH config: 未設定（Step 4b ### 3 が必要）"
  fi
fi

# git の SSH コマンド設定
git config --global core.sshCommand 2>/dev/null || echo "(未設定)"

# 1Password CLI の有無とバージョン確認
if command -v op >/dev/null 2>&1; then
  echo "op: $(op --version 2>/dev/null || echo 'バージョン取得失敗')"
else
  echo "op: not found"
fi

# 1Password SSH Agent の鍵一覧
$SSH_ADD_CMD -l 2>/dev/null || echo "SSH Agent 未接続"

# 既存の github-* エイリアス確認（マルチアカウント）
echo "--- github-* エイリアス ---"
grep "^Host github-" ~/.ssh/config 2>/dev/null || echo "(なし)"
if [[ "$OS_TYPE" == "WSL2" ]]; then
  WIN_USERNAME=$(cmd.exe /c 'echo %USERNAME%' 2>/dev/null | tr -d '\r' || echo "$USER")
  echo "--- github-* エイリアス (Windows) ---"
  grep "^Host github-" "/mnt/c/Users/$WIN_USERNAME/.ssh/config" 2>/dev/null || echo "(なし)"
fi

# credential helper の確認
git config --global credential.helper 2>/dev/null || echo "(未設定)"
git config --global credential.https://github.com.helper 2>/dev/null || echo "(未設定)"
```

### 2. チェック結果の解釈と次のアクション

| チェック結果 | 意味 | 次のアクション |
|---|---|---|
| `ssh -T` → `Hi <user>! You've successfully authenticated` | SSH 認証完了 | Step 5 へ（動作確認のみ） |
| `ssh -T` → `Permission denied (publickey)` | SSH 未設定または config 不足 | Step 4 の設定が必要 |
| `user` = `git`（SSH config） | SSH config 設定済み | そのステップはスキップ |
| `user` = OS ユーザー名（SSH config） | SSH config 未設定 | Step 4 / 4b ### 3 が必要 |
| `ssh-add -l` → 鍵が表示される | SSH Agent は動作中 | SSH config のみ設定すれば OK |
| `ssh-add -l` → 複数の鍵が表示される | 複数アカウントの可能性 | Step 4c で追加アカウントを設定 |
| `ssh-add -l` → `Could not open connection` | SSH Agent 未起動 | 1Password の SSH Agent を有効化 |
| `core.sshCommand` = `ssh.exe`（WSL2） | git の SSH コマンド設定済み | そのステップはスキップ |
| HTTPS credential helper 設定済み | HTTPS 方式は動作中 | SSH への移行は任意 |
| `github-*` エイリアスが表示される | マルチアカウント設定済み | Step 4c はスキップ可 |

チェック結果をもとに、**すでに完了しているステップは明示的にスキップ**して案内する。

---

スキル起動時に、まず以下の全体手順を案内してからユーザーに確認を取り、Step 1 から進める：

```
【github-auth-shiyoka 全体手順】

Step 1: Fine-grained PAT の準備（最優先・リポジトリ単位の権限制御）
  - GitHub で Fine-grained PAT を作成（リポジトリ・操作種別を限定）
  - 1Password に保存

Step 2: 1Password CLI (op) の有無を確認
  - ある場合 → Step 2a へ
  - ない場合 → Step 2b へ

Step 2a: op がある場合
  - op signin
  - op.env を作成（または既存を読み取り）
  - git credential helper を設定
  → 完了したら Step 5 へ

Step 2b: op がない場合
  - GCM（Git Credential Manager）を使って設定
  - OS のキーチェーンに安全に保存
  → 完了したら Step 5 へ

Step 3: Deploy Key（リポジトリ単位の SSH アクセスが必要な場合）
  - 1Password で SSH 鍵を生成
  - リポジトリの Settings → Deploy keys に登録
  - SSH config にリポジトリ専用エイリアスを設定

Step 4: SSH + 1Password SSH Agent（アカウント全体のアクセスが必要な場合）
  - 1Password アプリ内で SSH キーを生成（秘密鍵はディスク非保存）
  - GitHub に公開鍵を登録
  - SSH config を設定
  ※ WSL2 の場合は Step 4b（ssh.exe 経由・ブリッジ不要）を使う

Step 4c: 追加の GitHub アカウントを設定する（複数アカウント）
  - 1Password で追加アカウント用の SSH 鍵を生成
  - Host エイリアス（例: github-<ユーザー名>）を SSH config に追加

Step 5: 動作確認
  - SSH 方式：ssh -T git@github.com（WSL2 は ssh.exe）
  - HTTPS 方式：git ls-remote で認証テスト

Fine-grained PAT（Step 1）を使いますか？（使わない → Step 3 か Step 4 へ）
```

## Step 1: Fine-grained PAT の準備（最優先）

> **リポジトリ単位・操作種別（Read-only / Read-write）まで権限を絞れる最小権限方式。**
> PAT を 1Password で管理することで、ディスクへの平文保存なしに安全に利用できる。

1Password に GitHub Fine-grained PAT が未登録の場合は以下の手順で作成・保存する。
登録済みの場合は Step 2 へ進む。

> **注意：** Fine-grained PAT は GitHub の Web UI でのみ作成可能。コマンドラインからの作成は非対応。

### PAT に新しいリポジトリを追加する（PAT 作成済みの場合）

すでに PAT がある場合は、新規作成せずにリポジトリを追加できる。

以下の URL をブラウザで開く：

`https://github.com/settings/personal-access-tokens`

手順：
1. 対象の PAT の **Edit** をクリック
2. **Repository access** の **Only select repositories** で追加したいリポジトリを選択
3. **Update token** をクリック

> トークンの値は変わらないため、1Password の再保存は不要。
> 編集後は Step 2 へ進む。

---

### GitHub で PAT を作成する（新規の場合）

以下の URL をブラウザで開いてから、各入力項目を案内する：

`https://github.com/settings/personal-access-tokens/new`

URL を開いたらユーザーに以下をまとめて案内する：

```
以下の項目を入力・選択してください：

  Token name        : 用途がわかる名前（例：my-org-aws-access）
  Expiration        : 有効期限（推奨：90 days）
  Resource owner    : 組織名またはユーザー名（例：my-org）
  Repository access : "Only select repositories" → 対象リポジトリを選択
  Permissions
    └ Contents      : clone / pull のみ → "Read-only"
                      push も必要    → "Read and write"
    └ Metadata      : "Read-only"（必須・未設定だと 403 エラーになる）

すべて入力したら "Generate token" をクリックしてください。
表示されたトークン（github_pat_ で始まる）をコピーしてください。
```

> **注意：** トークンはこの画面でしか表示されない。必ずすぐに保存すること。
> 組織が Fine-grained PAT を未許可の場合は組織オーナーに有効化を依頼すること。

### 1Password に保存する

`op` コマンドで保存する（推奨）：

```bash
op item create \
  --category="API Credential" \
  --title="<アイテム名>" \
  --vault="<Vault名>" \
  "credential[concealed]=<コピーしたトークン>" \
  "expires[date]=<有効期限（例：2027-01-01）>"
```

> 有効期限は GitHub の PAT Expiration と合わせること。形式：`YYYY-MM-DD`

または 1Password アプリから手動で保存する：
1. 1Password を開く → **新規アイテム** → **API Credential**
2. 以下を入力して保存：

   | フィールド | 値 |
   |---|---|
   | タイトル | `op.env` の `GITHUB_TOKEN` に指定するアイテム名 |
   | credential | コピーしたトークン |

> アイテム名にスペースが含まれる場合は `op.env` の値をダブルクォートで囲むこと。
> 例：`GITHUB_TOKEN="op://Private/My GitHub PAT/credential"`

---

## Step 2: 1Password CLI の有無を確認する

```bash
which op
```

結果に応じて Step 2a または Step 2b に進む。

`op` が見つからない場合はインストールする：

```bash
# macOS
brew install 1password-cli

# Linux / WSL（公式ドキュメント参照）
# https://developer.1password.com/docs/cli/get-started/
```

---

## Step 2a: op がある場合

### 1. サインイン

```bash
eval "$(op signin)"
```

### 2. op.env を確認・作成する

プロジェクトルートの `op.env` を読み取る。存在しない場合は以下を自動検出してまとめて1回で確認し、`op.env` を作成する：

```bash
# git の表示名を参考情報として取得（GitHub ユーザー名と異なる場合あり）
git config --global user.name 2>/dev/null

# 1Password の GitHub 関連アイテムを候補として取得（サインイン済みの場合）
op item list --format=json 2>/dev/null | grep -i github
```

取得した候補をもとに、以下をまとめて1回で確認する：

```
以下の設定で op.env を作成してよいですか？

GITHUB_USERNAME=<自動検出した値>
GITHUB_TOKEN=op://<Vault名>/<アイテム名>/credential

変更がある場合は修正してください。
```

確認後、`op.env` を作成し `.gitignore` に追記する：

```bash
# op.env を作成
cat > op.env << EOF
GITHUB_USERNAME=<GitHubユーザー名>
GITHUB_TOKEN=op://<Vault名>/<アイテム名>/credential
EOF

# 他のユーザーから読めないようにする
chmod 600 op.env

# .gitignore に追加（未記載の場合のみ）
grep -qxF 'op.env' .gitignore 2>/dev/null || echo 'op.env' >> .gitignore
```

### 3. 既存の credential helper を確認する

```bash
git config --global credential.helper
```

すでに別の設定（`manager` など）が入っている場合は上書きになる。必要であればバックアップしておく：

```bash
# バックアップ（任意）
git config --global credential.helper >> ~/credential-helper.bak
```

### 4. credential helper を設定する

```bash
set -a && source op.env && set +a
git config --global credential.https://github.com.helper \
  "!f() { echo username=$GITHUB_USERNAME; echo password=\$(op read \"$GITHUB_TOKEN\"); }; f"
```

> `credential.https://github.com.helper` とすることで github.com 専用の設定になり、GitLab など他のホストに影響しない。

---

## Step 2b: op がない場合

GCM（Git Credential Manager）を使って認証を設定する。

1. GCM の有無を確認する：

   ```bash
   # WSL / Linux
   git-credential-manager --version 2>/dev/null || echo "not found"

   # Windows 側の GCM を WSL から使う場合
   /mnt/c/Program\ Files/Git/mingw64/bin/git-credential-manager.exe --version 2>/dev/null || echo "not found"
   ```

2. GCM がある場合は credential helper に設定する：

   ```bash
   # WSL から Windows 側の GCM を使う場合（スペース対策済み）
   git config --global credential.helper \
     "/mnt/c/Program\ Files/Git/mingw64/bin/git-credential-manager.exe"

   # Linux / Mac でネイティブ GCM がある場合
   git config --global credential.helper manager
   ```

3. GCM が見つからない場合はインストールを案内する：

   ```
   GCM のインストール方法：
   https://github.com/git-ecosystem/git-credential-manager/releases

   - Windows: Git for Windows に同梱（推奨）
   - macOS: brew install git-credential-manager
   - Linux: パッケージまたはバイナリをインストール
   ```

4. 設定後、初回の git 操作時にブラウザまたはダイアログで認証が求められる。
   以降は OS のキーチェーン（Windows Credential Manager / macOS Keychain）に安全に保存される。

---

## Step 3: Deploy Key（リポジトリ単位の SSH アクセス）

> **特定リポジトリのみにアクセスできる SSH 鍵を設定する。**
> CI/CD や自動化スクリプトで最小権限の SSH アクセスが必要な場合に使う。
> 1 つの Deploy Key は 1 つのリポジトリにのみ登録できる（複数リポジトリには別々の鍵が必要）。

### 1. 1Password で Deploy Key 用の SSH 鍵を生成する

1. 1Password アプリを開く → Vault を選択
2. **新規アイテム → SSH Key**
3. **秘密鍵を追加 → 新しい鍵を生成**
4. Key type：**Ed25519**
5. タイトル例：`<リポジトリ名> - Deploy Key`（例：`my-repo - Deploy Key`）
6. 保存

### 2. 公開鍵をリポジトリの Deploy keys に登録する

1. 1Password で作成したアイテムを開く
2. **Public key** フィールドの「コピー」ボタンをクリック
3. ブラウザで対象リポジトリの Settings を開く：
   `https://github.com/<org>/<repo>/settings/keys/new`
4. 以下を入力して **Add deploy key** をクリック：

   | 項目 | 値 |
   |---|---|
   | Title | 用途がわかる名前（例：`WSL2 Deploy Key`） |
   | Key | コピーした公開鍵を貼り付け |
   | Allow write access | push も必要な場合はチェック（デフォルトは Read-only） |

### 3. SSH config にリポジトリ専用エイリアスを設定する

複数の Deploy Key を使い分けるため、リポジトリごとに Host エイリアスを作成する。

```bash
ALIAS="github-<リポジトリ名>"  # 例: github-my-repo

BLOCK="
Host $ALIAS
    HostName github.com
    User git"

add_host_block() {
  local cfg="$1"
  mkdir -p "$(dirname "$cfg")"
  sed -i "/^Host $ALIAS/,/^$/d" "$cfg" 2>/dev/null || true
  printf '%s\n' "$BLOCK" >> "$cfg"
  chmod 600 "$cfg"
  echo "✅ 追記完了: $cfg"
}

# macOS / Linux: ~/.ssh/config のみ
add_host_block "$HOME/.ssh/config"

# WSL2 のみ: Windows 側にも追記
if grep -qi microsoft /proc/version 2>/dev/null; then
  WIN_USERNAME=$(cmd.exe /c 'echo %USERNAME%' 2>/dev/null | tr -d '\r' || echo "$USER")
  add_host_block "/mnt/c/Users/$WIN_USERNAME/.ssh/config"
fi
```

### 4. 動作確認

```bash
# macOS / Linux
ssh -T git@github-<リポジトリ名>

# WSL2
ssh.exe -T git@github-<リポジトリ名>

# 成功時: Hi <org>/<repo>! You've successfully authenticated...
#（Deploy Key の場合はリポジトリ名が表示される）
```

### 使い方

```bash
# Deploy Key でクローン
git clone git@github-<リポジトリ名>:<org>/<repo>.git

# 既存リポジトリの remote URL を変更
git remote set-url origin git@github-<リポジトリ名>:<org>/<repo>.git
```

> Deploy Key を使う場合、`ssh -T` の応答は `Hi <org>/<repo>!` の形式になる（ユーザー名ではなくリポジトリ名）。

---

## Step 4: SSH + 1Password SSH Agent（アカウント全体のアクセスが必要な場合）

> **秘密鍵は 1Password Vault 内にのみ保存され、ディスクには残らない。**
> PAT の作成・管理が不要になり、セッションタイムアウトの問題もない。
> ただし権限スコープはアカウント全体（リポジトリ単位の制限不可）。

> **WSL2 の場合は Step 4b（後述）を参照。**
> macOS / Linux は本 Step の手順で設定する。

### 1. 1Password SSH Agent を有効化

1Password デスクトップアプリで：
- **Settings → Developer**
- 「Use the SSH agent」をオン
- 「Display SSH key names when authorizing connections」をオン（推奨）

### 2. SSH キーを 1Password 内で生成（秘密鍵ディスク非保存）

1. 1Password アプリを開く → Vault を選択
2. **New Item → SSH Key**
3. **Add Private Key → Generate New Key**
4. Key type：**Ed25519**（推奨）
5. タイトルをつけて保存

> 秘密鍵は 1Password 内にのみ保存される。アプリ外にエクスポートしない限りディスクに残らない。

### 3. 公開鍵を GitHub に登録

1. 1Password アプリで作成した SSH Key アイテムを開く
2. **Public key** フィールドの右にある「コピー」ボタンをクリック
3. GitHub → Settings → SSH and GPG keys → **New SSH key**
4. Title：用途がわかる名前（例：`MyPC - GitHub`）
5. Key type：**Authentication Key**
6. Key：コピーした公開鍵を貼り付けて **Add SSH key** をクリック

### 4. SSH config を設定（macOS / Linux）

```bash
mkdir -p ~/.ssh
if ! grep -q "^Host github.com" ~/.ssh/config 2>/dev/null; then
  cat >> ~/.ssh/config << 'EOF'

Host github.com
    HostName github.com
    User git
    IdentityAgent ~/.1password/agent.sock
    IdentitiesOnly yes
EOF
fi
chmod 600 ~/.ssh/config
```

> `~/.1password/agent.sock` は 1Password デスクトップアプリが自動作成するソケットファイル。
> 1Password の Settings → Developer に表示されるパスと一致しない場合はそちらを使うこと。
> このコマンドは何度実行しても重複しない（冪等）。

### 5. SSH Agent の鍵を確認

```bash
ssh-add -l
# 1Password に保存されている鍵の一覧が表示されれば成功
```

### 6. 動作確認 → Step 5 へ

```bash
ssh -T git@github.com
# 成功時: Hi <username>! You've successfully authenticated...
```

成功したら Step 5 へ進む。以降のクローンは HTTPS ではなく SSH URL を使う：

```bash
git clone git@github.com:<組織名>/<リポジトリ名>.git
```

---

## Step 4b: WSL2 から 1Password SSH Agent を使う（2026年最新・公式推奨）

> `npiperelay` / `socat` のような複雑なブリッジ設定は不要。
> Windows 側の 1Password SSH Agent を有効化するだけで `ssh.exe` 経由で WSL2 から利用可能。
> 参考：
> - https://www.1password.dev/ssh/integrations/wsl

### 1. Windows 側の設定

1. 1Password for Windows を起動
2. **設定 → 開発者** →「**SSHエージェントを使用**」をオン（「実行中」と表示されれば OK）
3. 使用したい SSH 鍵（秘密鍵）を 1Password に保存・インポートしておく
4. GitHub への公開鍵登録は Step 1 の手順 3 と同様

> Windows Hello（生体認証 / PIN）が有効である必要がある。

### 2. WSL2 側の動作確認

```bash
ssh-add.exe -l
# 1Password に保存されている鍵の一覧が表示されれば成功
```

### 3. SSH config を設定

スキルフォルダのテンプレートを使って Linux・Windows 両側の config に追記する。
`ssh.exe`（git が使用）は Windows 側 config を、WSL native `ssh` は Linux 側 config を読むため両方に設定する。

```bash
TEMPLATE="$HOME/.claude/skills/github-auth-shiyoka/templates/ssh_config_github"
WIN_USERNAME=$(cmd.exe /c 'echo %USERNAME%' 2>/dev/null | tr -d '\r' || echo "$USER")
WIN_CONFIG="/mnt/c/Users/$WIN_USERNAME/.ssh/config"
LINUX_CONFIG="$HOME/.ssh/config"

# github.com ブロックを安全に追記する関数（重複削除 + 改行保証）
apply_ssh_config() {
  local cfg="$1"
  mkdir -p "$(dirname "$cfg")"
  if [ -f "$cfg" ]; then
    sed -i '/^Host github\.com/,/^$/d' "$cfg" 2>/dev/null || true
  fi
  echo "" >> "$cfg"
  cat "$TEMPLATE" >> "$cfg"
  echo "" >> "$cfg"
  chmod 600 "$cfg"
}

apply_ssh_config "$LINUX_CONFIG"   # WSL native ssh 用
apply_ssh_config "$WIN_CONFIG"     # ssh.exe (git) 用

echo "SSH config を Linux・Windows 両側に設定しました"
```

テンプレートの内容（`~/.claude/skills/github-auth-shiyoka/templates/ssh_config_github`）：

```
Host github.com
    HostName github.com
    User git
    IdentitiesOnly yes
```

> `ssh.exe` は Windows 側 `%USERPROFILE%\.ssh\config` を読む。WSL 側の `~/.ssh/config` は `ssh` コマンド用。両側に設定することで完全に対応する。

### 4. Git に ssh.exe を使うよう設定

```bash
git config --global core.sshCommand ssh.exe
```

### 5. エイリアスを設定（オプション）

`~/.bashrc` または `~/.zshrc` に追加：

```bash
alias ssh='ssh.exe'
alias ssh-add='ssh-add.exe'
```

設定を反映：

```bash
source ~/.bashrc   # または source ~/.zshrc
```

### 6. 動作確認 → Step 5 へ（Step 4b）

```bash
ssh.exe -T git@github.com
# 1Password のアクセスリクエストを承認 → Hi <username>! You've successfully authenticated...
```

### 7. Git コミット署名（SSH Signing）を設定したい場合

1. Windows 側の 1Password で使用する SSH 鍵を開く
2. **次のステップ：Gitのコミットに署名する** → **設定する** をクリック
3. 「**Linux用Windowsサブシステム(WSL)の設定**」にチェック → **スニペットをコピー**
4. WSL2 の `~/.gitconfig` に貼り付ける（`gpg.format`・`user.signingkey`・`gpg.ssh.program` が自動設定される）

### トラブルシューティング

- **`ssh.exe: command not found`** が出る場合：`/etc/wsl.conf` に以下を追加して WSL を再起動

  ```ini
  [interop]
  enabled = true
  ```

- 古い `SSH_AUTH_SOCK` 設定が残っている場合は削除する
- SSH config は WSL 側の `~/.ssh/config` を使用する（Step 1b の手順で設定済み）
- 1Password の承認は WSL セッションごとに行われる（セキュリティ仕様）

---

## Step 4c: 追加の GitHub アカウントを設定する（複数アカウント）

> すでに 1 つのアカウントが SSH で認証済みの状態で、別の GitHub アカウントを追加する場合の手順。
> 追加アカウントには Host エイリアス（例：`github-subaccount`）を使って使い分ける。

### IdentitiesOnly yes を使わない理由

1Password SSH Agent に複数の鍵が登録されている場合、Host ブロックに `IdentitiesOnly yes` を設定すると、特定の鍵が選ばれずに `Permission denied` になることがある。
追加アカウントの Host ブロックでは `IdentitiesOnly yes` を入れないこと。エージェントに登録されたすべての鍵が試されて、GitHub のアカウントに対応する鍵で認証される。

### 手順

#### 1. 1Password で追加アカウント用の SSH 鍵を生成する

1. 1Password アプリを開く → Vault を選択
2. **新規アイテム → SSH Key**
3. **秘密鍵を追加 → 新しい鍵を生成**
4. Key type：**Ed25519**
5. タイトル例：`<GitHubユーザー名> - GitHub`（例：`myuser - GitHub`）
6. 保存

#### 2. 公開鍵を追加アカウントの GitHub に登録する

1. 1Password で作成したアイテムを開く
2. **Public key** フィールドの「コピー」ボタンをクリック
3. 追加したい GitHub アカウントでログインした状態でブラウザを開く：
   `https://github.com/settings/ssh/new`
4. Title・Key type（Authentication Key）・Key を入力して **Add SSH key**

#### 3. SSH config に Host エイリアスを追加する

```bash
ALIAS="github-<ユーザー名>"  # 例: github-myuser

BLOCK="
Host $ALIAS
    HostName github.com
    User git"

add_host_block() {
  local cfg="$1"
  mkdir -p "$(dirname "$cfg")"
  sed -i "/^Host $ALIAS/,/^$/d" "$cfg" 2>/dev/null || true
  printf '%s\n' "$BLOCK" >> "$cfg"
  chmod 600 "$cfg"
  echo "✅ 追記完了: $cfg"
}

# macOS / Linux: ~/.ssh/config のみ
add_host_block "$HOME/.ssh/config"

# WSL2 のみ: Windows 側にも追記
if grep -qi microsoft /proc/version 2>/dev/null; then
  WIN_USERNAME=$(cmd.exe /c 'echo %USERNAME%' 2>/dev/null | tr -d '\r' || echo "$USER")
  add_host_block "/mnt/c/Users/$WIN_USERNAME/.ssh/config"
fi
```

#### 4. 動作確認

```bash
# macOS / Linux
ssh -T git@github-<ユーザー名>

# WSL2
ssh.exe -T git@github-<ユーザー名>

# 成功時: Hi <username>! You've successfully authenticated...
```

### 使い方

追加アカウントのリポジトリを操作する際は、SSH URL のホスト名を `github.com` の代わりにエイリアスに変更する：

```bash
# 追加アカウントでクローン
git clone git@github-<ユーザー名>:<リポジトリ名>.git

# 既存リポジトリの remote URL を変更
git remote set-url origin git@github-<ユーザー名>:<org>/<repo>.git
```

```bash
# 元のアカウント（github.com のまま）
git clone git@github.com:<org>/<repo>.git
```

---

## Step 5: 動作確認

**HTTPS 方式（Step 2a: op）の場合：**

op 方式はサインインしてから確認する：

```bash
eval "$(op signin)"
git ls-remote https://github.com/<組織名>/<リポジトリ名>.git
```

**HTTPS 方式（Step 2b: GCM）の場合：**

初回 clone 時にブラウザ認証が起動する。以降は自動認証される。

**Deploy Key（Step 3）の場合：**

```bash
ssh -T git@github-<リポジトリ名>          # macOS / Linux
ssh.exe -T git@github-<リポジトリ名>      # WSL2
# 成功時: Hi <org>/<repo>! You've successfully authenticated...
```

**SSH + 1Password Agent（Step 4）の場合：**

```bash
ssh -T git@github.com          # macOS / Linux
ssh.exe -T git@github.com      # WSL2
# 成功時: Hi <username>! You've successfully authenticated...
```

**複数アカウント（Step 4c）の場合：**

```bash
ssh -T git@github-<ユーザー名>          # macOS / Linux
ssh.exe -T git@github-<ユーザー名>      # WSL2
# 成功時: Hi <username>! You've successfully authenticated...
```

エラーがなければ設定完了。

### op のセッションタイムアウト時

`op` はデフォルト30分でセッションが切れ、git 操作時に以下のようなエラーが出ることがある：

```
error: could not read Username for 'https://github.com'
```

その場合は再サインインする：

```bash
eval "$(op signin)"
```

### PAT の有効期限が切れた場合

1. GitHub で同じ PAT を開いて **Regenerate token** をクリック（有効期限を延長）
2. 表示された新しいトークンをコピー
3. 1Password のアイテムを更新する：

   ```bash
   op item edit "<アイテム名>" --vault="<Vault名>" "credential[concealed]=<新しいトークン>"
   ```

   または 1Password アプリで該当アイテムの `credential` フィールドを直接更新する。

> credential helper の設定変更は不要。`op.env` の参照パスも変わらない。

以降は以下のコマンドでそのままクローンできる：

```bash
git clone https://github.com/<組織名>/<リポジトリ名>.git
```

---

## プロジェクトごとの認証設定を保存する

認証方式によって「設定の保存先」が異なる。

| 方式 | 保存先 | チーム共有 |
|---|---|---|
| SSH（Deploy Key / 複数アカウント） | remote URL（`.git/config`） | ✅ git remote に記録される |
| HTTPS + PAT（Step 2a） | `op.env` + `git config --local` | `op.env.example` をコミットして共有 |
| SSH + 1Password Agent（Step 4） | remote URL（`git@github.com:...`） | ✅ |

### SSH 方式：remote URL が設定を保持する

Deploy Key や複数アカウントで Host エイリアスを使っている場合、remote URL を正しく設定するだけで完結する。

```bash
git remote set-url origin git@github-<リポジトリ名 or ユーザー名>:<org>/<repo>.git
```

`.git/config` に記録されるため、`git push/pull` が常に正しい鍵を使う。追加設定不要。

### HTTPS + PAT 方式：op.env + git config --local

プロジェクトルートで以下を実行する：

```bash
# プロジェクトローカルの credential helper を設定（グローバル設定を上書き）
set -a && source op.env && set +a
git config --local credential.https://github.com.helper \
  "!f() { echo username=$GITHUB_USERNAME; echo password=\$(op read \"$GITHUB_TOKEN\"); }; f"
```

`--local` を使うことで `.git/config` に書き込まれ、このリポジトリだけに適用される。

### op.env.example をリポジトリにコミットする

実際の値は `.gitignore` で除外しつつ、テンプレートをコミットしておくと他のメンバーが `cp` するだけで使える：

```bash
# op.env.example（リポジトリにコミットする）
GITHUB_USERNAME=<GitHubユーザー名>
GITHUB_TOKEN=op://<Vault名>/<アイテム名>/credential
```

```bash
# .gitignore に追加
echo 'op.env' >> .gitignore
```

> `op.env.example` を見れば必要な値が一目でわかる。実際のトークンは 1Password にのみ保存されるため、誤ってコミットしても漏洩しない。

---

## 解除方法

**HTTPS 方式（Step 2a: op）の場合：**

```bash
git config --global --unset credential.https://github.com.helper
```

**HTTPS 方式（Step 2b: GCM）の場合：**

```bash
# キャッシュを削除
git credential reject <<EOF
protocol=https
host=github.com
EOF

# または GCM の logout コマンド
git-credential-manager github logout
```

**Deploy Key / 複数アカウント（Step 3 / 4c）の Host エイリアスを解除する場合：**

```bash
ALIAS="github-<名前>"  # 例: github-my-repo, github-myuser

# macOS / Linux
sed -i "/^Host $ALIAS/,/^$/d" ~/.ssh/config 2>/dev/null || true

# WSL2: Windows 側も削除
if grep -qi microsoft /proc/version 2>/dev/null; then
  WIN_USERNAME=$(cmd.exe /c 'echo %USERNAME%' 2>/dev/null | tr -d '\r' || echo "$USER")
  sed -i "/^Host $ALIAS/,/^$/d" "/mnt/c/Users/$WIN_USERNAME/.ssh/config" 2>/dev/null || true
fi
```

> Deploy Key の場合は GitHub 側の Settings → Deploy keys からも削除すること。

**SSH + 1Password Agent（Step 4 / macOS・Linux）の場合：**

```bash
cp ~/.ssh/config ~/.ssh/config.bak 2>/dev/null || true
sed -i '/^Host github\.com/,/^$/d' ~/.ssh/config 2>/dev/null || true
```

**SSH + 1Password Agent（Step 4b / WSL2）の場合：**

```bash
cp ~/.ssh/config ~/.ssh/config.bak 2>/dev/null || true

# Linux 側
sed -i '/^Host github\.com/,/^$/d' ~/.ssh/config 2>/dev/null || true

# Windows 側
WIN_USERNAME=$(cmd.exe /c 'echo %USERNAME%' 2>/dev/null | tr -d '\r' || echo "$USER")
sed -i '/^Host github\.com/,/^$/d' "/mnt/c/Users/$WIN_USERNAME/.ssh/config" 2>/dev/null || true

# git の SSH コマンド設定を解除
git config --global --unset core.sshCommand
```

> 1Password の SSH Key アイテム自体は削除不要（他の用途で再利用可能）。

---

## トラブルシューティング

### `ssh -T git@github.com` で `Permission denied (publickey)` が出る

原因と確認手順：

```bash
# SSH Agent に鍵が読み込まれているか確認
ssh-add.exe -l   # WSL2
ssh-add -l       # macOS / Linux
```

- **`Could not open connection to agent`** → 1Password の SSH Agent が無効。Settings → Developer → 「SSH エージェントを使用」をオン
- **`The agent has no identities`** → 鍵が Agent に読み込まれていない。1Password で SSH Key アイテムを開いて「認証済みの鍵として追加」を確認
- **鍵が表示されるが認証失敗** → GitHub に公開鍵が登録されていない。Step 4 の手順 3 で登録する

### `op: command not found` が出る

```bash
# macOS
brew install 1password-cli

# Linux / WSL（公式ドキュメント参照）
# https://developer.1password.com/docs/cli/get-started/
```

インストール後、`op --version` で動作確認してから再度スキルを実行する。

### WSL2 で Windows 側 SSH config が反映されない

`ssh.exe -G github.com | grep "^user "` で `user git` が返らない場合：

```bash
# Windows 側 config のパスを確認
WIN_USERNAME=$(cmd.exe /c 'echo %USERNAME%' 2>/dev/null | tr -d '\r')
echo "/mnt/c/Users/$WIN_USERNAME/.ssh/config"

# config の内容を確認
cat "/mnt/c/Users/$WIN_USERNAME/.ssh/config"
```

`Host github.com` ブロックがなければ Step 4b の手順 3 を再実行する。

### op のセッションタイムアウトで git 操作が失敗する

デフォルト 30 分でセッションが切れ、以下のようなエラーが出る：

```
error: could not read Username for 'https://github.com'
```

再サインインで解決する：

```bash
eval "$(op signin)"
```

### WSL2 で `ssh.exe: command not found` が出る

WSL interop が無効になっている。`/etc/wsl.conf` に以下を追加して WSL を再起動する：

```ini
[interop]
enabled = true
```

```bash
# WSL を再起動（PowerShell から）
wsl --shutdown
```

