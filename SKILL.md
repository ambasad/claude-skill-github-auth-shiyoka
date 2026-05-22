---
name: github-auth-shiyoka
description: GitHub認証設定が必要な場合（git clone/push/pullで認証エラー時、または「GitHub認証」「PAT設定」「1Password GitHub」「GitHubセキュリティチェック」「認証の安全確認」などと言及されたとき）に自動で環境診断＋最適セキュア設定を行い、さらに既存トークンの有効期限・権限精査・セキュリティレポートを出力します。1Password Vault優先・Fine-grained PATを最優先にし、ディスク非保存を徹底。
disable-model-invocation: true
version: "0.6.0"
---

# github-auth-shiyoka

> **Agentic Skill** — デフォルトでは `disable-model-invocation: true` のため、`/github-auth-shiyoka` と明示的に入力して起動します。  
> キーワード（「GitHub認証」「PAT設定」「git clone で認証エラー」など）を会話中に検出して **自動起動させたい場合** は、このファイルの frontmatter を以下のように変更してください：
>
> ```yaml
> disable-model-invocation: false
> ```

git credential helper を設定し、以降の `git clone` / `git push` / `git pull` で
GitHub 認証を自動的に行えるようにする。

このスキルは以下の方式を扱う。リポジトリ単位の権限制御を重視した優先順位：

| 優先順位 | 方式 | 権限スコープ | 特徴 |
|---|---|---|---|
| 1（最高） | Fine-grained PAT + 1Password CLI | リポジトリ単位・操作種別 | 最小権限・PAT を 1Password で管理 |
| 2 | Deploy Key | リポジトリ単位 | SSH・特定リポジトリ専用鍵・CI/CD 向き |
| 3 | SSH + 1Password SSH Agent | アカウント全体 | PAT 不要・秘密鍵ディスク非保存 |
| 4 | GCM（Git Credential Manager） | アカウント全体 | OAuth 使用・OS キーチェーンに保存 |

---

## セキュリティ監査モード（スキル起動時に常に最初に実行）

スキル起動直後に以下の監査フローを実行し、**セキュリティレポートを出力**してから設定手順に進む。

### 1. 現在の GitHub 認証状態を診断

```bash
# GitHub CLI による認証状態確認
gh auth status 2>&1 || echo "gh コマンドが見つからないか未認証"

# 現在の credential helper 確認
echo "=== credential helper ==="
git config --global credential.helper 2>/dev/null || echo "(グローバル未設定)"
git config --local credential.helper 2>/dev/null || echo "(ローカル未設定)"
git config --global credential.https://github.com.helper 2>/dev/null || echo "(github.com 専用未設定)"

# OS 判定
if grep -qi microsoft /proc/version 2>/dev/null; then
  OS_TYPE="WSL2"
elif [[ "$(uname)" == "Darwin" ]]; then
  OS_TYPE="macOS"
else
  OS_TYPE="Linux"
fi
echo "OS: $OS_TYPE"
```

### 2. PAT 種別を確認（Fine-grained か Classic か）

`gh auth status` の出力または以下のコマンドでトークン種別を確認する：

```bash
# 現在のトークンを gh から取得して種別判定
gh auth token 2>/dev/null | head -c 20 || echo "(トークン取得不可)"
# Fine-grained PAT は "github_pat_" で始まる
# Classic PAT は "ghp_" で始まる
# OAuth トークンは "gho_" で始まる
```

判定基準：
- `github_pat_` → Fine-grained PAT（最推奨）
- `ghp_` → Classic PAT（過剰権限の可能性あり → Fine-grained PAT への移行を推奨）
- `gho_` → OAuth / GCM（アカウント全体アクセス）
- 取得不可 → 未認証またはトークン管理外

### 3. 有効期限を確認（期限切れ・30日以内の警告）

```bash
# 1Password に保存された PAT の有効期限を取得（op がある場合）
if command -v op >/dev/null 2>&1; then
  echo "=== 1Password: GitHub 関連アイテムの有効期限 ==="
  op item list --categories "API Credential" --format=json 2>/dev/null \
    | python3 -c "
import json, sys, datetime
items = json.load(sys.stdin)
today = datetime.date.today()
warn_days = 30
for item in items:
  title = item.get('title', '')
  if 'github' in title.lower():
    for field in item.get('fields', []):
      if field.get('id') == 'expires' or field.get('label', '').lower() == 'expires':
        val = field.get('value', '')
        if val:
          try:
            exp = datetime.date.fromisoformat(val)
            delta = (exp - today).days
            if delta < 0:
              print(f'🔴 期限切れ: {title} (期限: {val})')
            elif delta <= warn_days:
              print(f'⚠️  {delta}日後に期限切れ: {title} (期限: {val})')
            else:
              print(f'✅ 有効期限OK: {title} (期限: {val}, 残り{delta}日)')
          except:
            print(f'ℹ️  {title}: 期限値={val}')
" 2>/dev/null || echo "(有効期限の自動取得失敗 - 手動で確認してください)"
fi

# gh CLI でトークン有効期限確認
gh auth status 2>&1 | grep -i "expir\|token expir\|期限" || true
```

### 4. 権限スコープを精査（過剰権限の検出）

```bash
# gh CLI でスコープ確認
gh auth status 2>&1 | grep -i "scope\|権限" || true

# API で直接スコープ確認（Classic PAT / OAuth の場合）
SCOPES=$(curl -sI -H "Authorization: token $(gh auth token 2>/dev/null)" \
  https://api.github.com/user 2>/dev/null \
  | grep -i "^x-oauth-scopes:" | cut -d: -f2- | tr -d ' \r')
if [ -n "$SCOPES" ]; then
  echo "現在のスコープ: $SCOPES"
else
  echo "(スコープ取得不可 - Fine-grained PAT またはトークン未設定の可能性)"
fi
```

過剰権限の判定基準：
- `repo`（フルアクセス）が設定されていて、Read のみで十分な場合 → Fine-grained PAT への移行推奨
- `admin:org`・`delete_repo`・`workflow` が不要なのに含まれている → 即時削除推奨
- Classic PAT でスコープが広い → Fine-grained PAT に移行して最小権限化

### 5. 1Password Vault 連携状況を確認

```bash
# op CLI の有無とサインイン状態
if command -v op >/dev/null 2>&1; then
  echo "op version: $(op --version 2>/dev/null)"
  op account list 2>/dev/null | head -5 || echo "(未サインイン)"
  echo "=== GitHub 関連アイテム ==="
  op item list --categories "API Credential" --format=json 2>/dev/null \
    | python3 -c "
import json, sys
items = json.load(sys.stdin)
found = [i['title'] for i in items if 'github' in i.get('title','').lower()]
print('\n'.join(found) if found else '(GitHub関連アイテムなし)')
" 2>/dev/null || echo "(取得失敗)"
else
  echo "op: not found（1Password CLI 未インストール）"
fi

# credential helper が op を使っているか確認
git config --global credential.https://github.com.helper 2>/dev/null | grep -q "op read" \
  && echo "✅ credential helper: 1Password CLI 連携済み" \
  || echo "ℹ️  credential helper: 1Password CLI 未連携"
```

### 6. 総合セキュリティレポートを出力

上記 1〜5 のチェック結果をもとに、以下の形式でセキュリティレポートを出力する：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔐 GitHub 認証 セキュリティレポート
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

【現在の認証状態】
  - 認証方式: <Fine-grained PAT / Classic PAT / OAuth / 未認証>
  - トークン種別: <github_pat_ / ghp_ / gho_ / なし>
  - 1Password 連携: <済 / 未連携>

【有効期限チェック】
  - <✅ 有効期限OK / ⚠️ XX日後に期限切れ / 🔴 期限切れ>

【権限スコープチェック】
  - <✅ 最小権限 / ⚠️ 過剰権限あり（詳細） / ℹ️ スコープ不明>

【問題点と即時対応策】
  • <問題点があれば箇点で記載。なければ「問題なし」>
  • <例: Classic PAT を使用中 → Fine-grained PAT への移行を推奨>
  • <例: トークンが30日以内に期限切れ → Regenerate 手順を案内>
  • <例: 1Password 未連携 → op.env 設定を推奨>

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 7. 必要ならトークン回転手順を提案

以下の場合に、対応する回転手順を案内する：

**PAT が期限切れ・30日以内の場合：**
```
【トークン回転手順】
1. https://github.com/settings/personal-access-tokens を開く
2. 対象 PAT の "Regenerate token" をクリック
3. 新しいトークンをコピー
4. 1Password を更新:
   op item edit "<アイテム名>" --vault="<Vault名>" "credential[concealed]=<新トークン>"
5. credential helper の再設定は不要（参照パスは変わらない）
```

**Classic PAT → Fine-grained PAT へ移行する場合：**
```
【移行手順】
1. Step 1 の手順で Fine-grained PAT を新規作成
2. 1Password に保存
3. op.env の参照パスを更新
4. 旧 Classic PAT を GitHub から削除:
   https://github.com/settings/tokens
```

---

セキュリティ監査完了後、続けて以下の **事前チェック** と **設定手順** を実行する。

---

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
  WIN_USERNAME=$(cmd.exe /c 'echo %USERNAME%' 2>/dev/null | tr -d '\r')
  if [ -z "$WIN_USERNAME" ]; then
    echo "⚠️ Windows ユーザー名取得失敗（WSL interop が無効の可能性あり）"
  else
    echo "--- github-* エイリアス (Windows) ---"
    grep "^Host github-" "/mnt/c/Users/$WIN_USERNAME/.ssh/config" 2>/dev/null || echo "(なし)"
  fi
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

認証方式を選んでください：

▶ HTTPS 方式（Fine-grained PAT を使う・最優先）
  Step 1: Fine-grained PAT を GitHub で作成し 1Password に保存
  Step 2: 1Password CLI (op) の有無を確認
    → op あり: Step 2a（op.env + credential helper を設定）→ Step 5 へ
    → op なし: Step 2b（GCM で設定）→ Step 5 へ

▶ SSH 方式（リポジトリ専用鍵 / アカウント全体）
  Step 3: Deploy Key（リポジトリ単位・SSH）
    - 1Password で SSH 鍵を生成 → リポジトリ Settings → Deploy keys に登録
    - SSH config にリポジトリ専用エイリアスを設定 → Step 5 へ

  Step 4: SSH + 1Password SSH Agent（アカウント全体）
    → macOS / Linux: Step 4（IdentityAgent を使う通常設定）
    → WSL2: Step 4b（ssh.exe 経由・ブリッジ不要）
    Step 4c: 複数 GitHub アカウントを使い分ける場合（Host エイリアスを追加）
    → Step 5 へ

Step 5: 動作確認
  - SSH 方式：ssh -T git@github.com（WSL2 は ssh.exe -T）
  - HTTPS 方式：git ls-remote で認証テスト

どちらの方式で設定しますか？（HTTPS 方式 / SSH 方式）
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
op item list --categories "API Credential" --format=json 2>/dev/null \
  | grep -i '"title"' | grep -i github

# 上記で候補が出ない場合は全アイテムから検索
op item list --format=json 2>/dev/null \
  | grep -i '"title"' | grep -i github
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

| 方式 | 保存先 | チーム共有 |
|---|---|---|
| SSH（Deploy Key / 複数アカウント） | remote URL（`.git/config`） | ✅ git remote に記録される |
| HTTPS + PAT（Step 2a） | `op.env` + `git config --local` | `op.env.example` をコミットして共有 |
| SSH + 1Password Agent（Step 4） | remote URL（`git@github.com:...`） | ✅ |

### SSH 方式：remote URL が設定を保持する

```bash
git remote set-url origin git@github-<リポジトリ名 or ユーザー名>:<org>/<repo>.git
```

### HTTPS + PAT 方式：op.env + git config --local

```bash
set -a && source op.env && set +a
git config --local credential.https://github.com.helper \
  "!f() { echo username=$GITHUB_USERNAME; echo password=\$(op read \"$GITHUB_TOKEN\"); }; f"
```

### op.env.example をリポジトリにコミットする

```bash
# op.env.example（リポジトリにコミットする）
GITHUB_USERNAME=<GitHubユーザー名>
GITHUB_TOKEN=op://<Vault名>/<アイテム名>/credential
```

```bash
echo 'op.env' >> .gitignore
```

---

## 解除方法

**HTTPS 方式（Step 2a: op）の場合：**

```bash
git config --global --unset credential.https://github.com.helper
```

> PAT 自体を無効化する場合：`https://github.com/settings/personal-access-tokens`

**HTTPS 方式（Step 2b: GCM）の場合：**

```bash
git credential reject <<EOF
protocol=https
host=github.com
EOF
git-credential-manager github logout
```

**Deploy Key / 複数アカウント（Step 3 / 4c）の Host エイリアスを解除する場合：**

```bash
ALIAS="github-<名前>"
sed -i "/^Host $ALIAS/,/^$/d" ~/.ssh/config 2>/dev/null || true

if grep -qi microsoft /proc/version 2>/dev/null; then
  WIN_USERNAME=$(cmd.exe /c 'echo %USERNAME%' 2>/dev/null | tr -d '\r' || echo "$USER")
  sed -i "/^Host $ALIAS/,/^$/d" "/mnt/c/Users/$WIN_USERNAME/.ssh/config" 2>/dev/null || true
fi
```

> Deploy Key の場合は GitHub 側からも削除：`https://github.com/<org>/<repo>/settings/keys`

**SSH + 1Password Agent（Step 4 / macOS・Linux）の場合：**

```bash
cp ~/.ssh/config ~/.ssh/config.bak 2>/dev/null || true
sed -i '/^Host github\.com/,/^$/d' ~/.ssh/config 2>/dev/null || true
```

**SSH + 1Password Agent（Step 4b / WSL2）の場合：**

```bash
cp ~/.ssh/config ~/.ssh/config.bak 2>/dev/null || true
sed -i '/^Host github\.com/,/^$/d' ~/.ssh/config 2>/dev/null || true
WIN_USERNAME=$(cmd.exe /c 'echo %USERNAME%' 2>/dev/null | tr -d '\r' || echo "$USER")
sed -i '/^Host github\.com/,/^$/d' "/mnt/c/Users/$WIN_USERNAME/.ssh/config" 2>/dev/null || true
git config --global --unset core.sshCommand
```

---

## トラブルシューティング

### `ssh -T git@github.com` で `Permission denied (publickey)` が出る

```bash
ssh-add.exe -l   # WSL2
ssh-add -l       # macOS / Linux
```

- **`Could not open connection to agent`** → 1Password Settings → Developer → SSH エージェントをオン
- **`The agent has no identities`** → 1Password で SSH Key アイテムを開いて「認証済みの鍵として追加」を確認
- **鍵が表示されるが認証失敗** → GitHub に公開鍵未登録。Step 4 の手順 3 で登録する

### `op: command not found` が出る

```bash
brew install 1password-cli          # macOS
# Linux / WSL: https://developer.1password.com/docs/cli/get-started/
```

### WSL2 で Windows 側 SSH config が反映されない

```bash
WIN_USERNAME=$(cmd.exe /c 'echo %USERNAME%' 2>/dev/null | tr -d '\r')
cat "/mnt/c/Users/$WIN_USERNAME/.ssh/config"
```

`Host github.com` ブロックがなければ Step 4b の手順 3 を再実行する。

### op のセッションタイムアウトで git 操作が失敗する

```bash
eval "$(op signin)"
```

### WSL2 で `ssh.exe: command not found` が出る

```ini
# /etc/wsl.conf に追加して wsl --shutdown で再起動
[interop]
enabled = true
```

---

## セキュリティ監査完了＋設定完了 まとめ

すべての手順が完了したら、以下のまとめを出力する：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ セキュリティ監査完了＋GitHub認証設定完了
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

【設定した認証方式】
  <設定した方式を記載>

【セキュリティ状態】
  - トークン種別: <Fine-grained PAT / Deploy Key / SSH / GCM>
  - 1Password Vault 連携: <済 / 未連携>
  - ディスク平文保存: なし（1Password 管理）
  - 権限スコープ: <リポジトリ単位 / アカウント全体>
  - 有効期限: <期限と残日数>

【次回の確認推奨事項】
  • PAT の有効期限が近づいたら Regenerate token → 1Password を更新
  • 不要になったトークンは https://github.com/settings/personal-access-tokens から削除
  • 定期的に /github-auth-shiyoka を実行してセキュリティ監査を継続

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
