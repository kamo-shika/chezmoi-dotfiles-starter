---
name: skill-management
description: Claude Code スキルの作成・追加・更新・削除・配置に関する運用ルールを提供する。スキルを新規作成する、他人の公開スキルを install する、既存スキルを update する、スキルをどこに置くべきか迷う、`~/.claude/skills/` / chezmoi / `skills` CLI / dotfiles 連携・使い分け、プロジェクト固有スキルの配置、二重管理の回避方法などスキル管理に関わるあらゆる操作で必ず使用する。「スキル作って」「スキル追加して」「このリポジトリのスキル入れて」「スキル更新」「skills add」「skills update」「案件用のスキル」「chezmoi でスキル」などの発言、および `~/.claude/skills/` / `~/.agents/.skill-lock.json` / `dot_claude/skills/` への言及があれば必ずトリガーする。
---

# Skill Management

Claude Code スキルの出自を 3 種類に分けて、それぞれに対応するツールで管理する。出自を混ぜると `chezmoi apply` で更新が巻き戻る等の事故が起きるので、必ず分類してから作業する。

## 判定フロー

ユーザーのリクエストが来たらまず以下で分類する。

```
スキル作成/追加/更新/削除の依頼
│
├─ ユーザー自身が author する新規スキル / 既存の自作スキル編集
│  → 【1. 自作スキル】chezmoi source を触る
│
├─ 他人が公開している GitHub repo のスキルを入れたい/更新したい
│  → 【2. 他人のスキル】skills CLI を使う
│
└─ 特定プロジェクト固有のスキル（案件用ワークフロー等）
   → 【3. 案件スキル】プロジェクト repo の .claude/skills/ に直接置く
```

判定が曖昧な場合は必ずユーザーに確認する。分類を間違えると後で再分類のコストが重い。

---

## 1. 自作スキル（chezmoi 管理）

ユーザー自身が author するスキル。chezmoi の dotfiles repo で source-of-truth を管理し、社内 GHE に push するだけで同僚へ配布可能。

**dotfiles repo の remote 構成（例）:**
- `origin` → `github.com/<your-gh-user>/dotfiles`（個人用・public）
- `ghe` → `<your-ghe-host>/<your-ghe-user>/<your-ghe-user>-dotfiles`（**社内配布用・他のメンバーはここから install**）
- `upstream` → `github.com/kamo-shika/chezmoi-dotfiles-starter`（スターター、fetch のみ）

remote 名は好みで変えて構わないが、以降の手順は上記を前提に書いている。

### 配置

| 役割 | パス |
|---|---|
| 編集対象（source） | `~/dotfiles/dot_claude/skills/<name>/SKILL.md` |
| 実体（target） | `~/.claude/skills/<name>/SKILL.md` |

`dot_` プレフィックスは chezmoi 慣習で `.` に変換される。**編集は必ず source 側で行う**。target を直接編集しても次回 `chezmoi apply` で上書きされる。

### 新規作成の手順

```bash
# 1. source 側にディレクトリを作る
mkdir -p ~/dotfiles/dot_claude/skills/<name>

# 2. SKILL.md を書く（最低限 frontmatter に name/description が必要）

# 3. 反映
chezmoi apply

# 4. Claude Code で動作確認

# 5. dotfiles repo にコミット
cd ~/dotfiles && git add dot_claude/skills/<name> && git commit -m "feat: add <name> skill"
```

### 既存の自作スキルを編集

```bash
# source を直接編集（推奨）
$EDITOR ~/dotfiles/dot_claude/skills/<name>/SKILL.md
chezmoi apply

# または chezmoi 経由で編集（source を自動で開く）
chezmoi edit ~/.claude/skills/<name>/SKILL.md
chezmoi apply
```

### 公開

同僚向けの配布先は社内 GHE。両方に push する場合は remote を明示:

```bash
git push ghe main      # 同僚配布用（必須）
git push origin main   # 個人用バックアップ（任意）
```

同僚側のインストール手順:

```bash
# GHE には事前に gh auth login が必要
gh auth login --hostname <your-ghe-host>

# URL 直指定で install（GHE は owner/repo 形式では解決できない）
skills add https://<your-ghe-host>/<your-ghe-user>/<your-ghe-user>-dotfiles -s <name> -g
```

skills CLI は repo 内を再帰探索するので `dot_claude/skills/<name>/SKILL.md` という変則パスでも検出できる。

### GHE install の更新挙動（重要）

`skills add` は URL ホストで分岐する:

- `github.com` → `sourceType: "github"`、`api.github.com` から folder hash を取得・記録
- `github.com` 以外（GHE 含む）→ `sourceType: "git"`、**hash は空のまま**

この結果、`skills update -g` は GHE install を **silently skip する**（内部で `skillFolderHash` 空のものを除外し、理由表示すらされない）。原因は CLI が `api.github.com` ハードコードで、GHE エンドポイントに問い合わせできないため。

**手動更新手段:** 同じ URL で `skills add` を再実行すると上書きインストールされる。これが GHE スキルの唯一のアップデート手段。

```bash
# 同僚側の手動更新手順
export GH_TOKEN=$(gh auth token -h <your-ghe-host>)
skills add https://<your-ghe-host>/<your-ghe-user>/<your-ghe-user>-dotfiles -s <name> -g -y
```

`gh auth token` のホスト指定が skills CLI 側で効かないケースがあるため、`GH_TOKEN` 環境変数で明示渡しすると確実。

---

## 2. 他人のスキル（skills CLI 管理）

同僚や vercel-labs など、他者が公開しているスキル。`skills` CLI がバージョン管理・自動更新を担う。

### インストール

```bash
# グローバル（推奨、全プロジェクトから使える）
skills add <owner>/<repo> -g

# プロジェクトローカル
skills add <owner>/<repo>

# 特定スキルだけ
skills add <owner>/<repo> -s <skill-name> -g

# 中身を見るだけ（install せず一覧）
skills add <owner>/<repo> -l

# 確認スキップ
skills add <owner>/<repo> -g -y
```

### 更新・削除・一覧

```bash
skills update -g              # グローバル全部を更新
skills update -g <name>       # 単体
skills ls -g                  # 一覧
skills rm -g <name>           # 削除（実体もロックからも消える）
skills find <query>           # skills.sh から検索
```

### ⚠ 絶対禁止

**他人のスキルを chezmoi 管理下に入れてはいけない**。具体的には:

- `~/.claude/skills/<name>/` を `chezmoi add` しない
- `~/dotfiles/dot_claude/skills/` に他人のスキルを手でコピーしない

理由: `skills update -g` が上流から更新した内容を、次の `chezmoi apply` が chezmoi source の古い内容で巻き戻す。二重管理は必ず事故る。

`skills` CLI が管理するファイル群:
- 実体: `~/.claude/skills/<name>/`（chezmoi 管理外として並存）
- ロック: `~/.agents/.skill-lock.json`（sourceUrl / folderHash / updatedAt を記録）

ロックファイルも chezmoi 管理下に入れない。マシンごとに状態が違って当然のファイル。

---

## 3. 案件スキル（プロジェクト repo 管理）

特定案件のワークフローや規約を codify したスキル。個人の dotfiles とは独立。

### 配置

```
<project-root>/
└── .claude/
    └── skills/
        └── <name>/
            └── SKILL.md
```

プロジェクトの git repo に一緒にコミットする。他のメンバーがその repo をクローンしたら自動的に使えるようになる（Claude Code は `.claude/skills/` をプロジェクトスコープとして読む）。

### やってはいけない

- 案件スキルを `~/dotfiles` にコピーしない（プロジェクト外で動いても意味がない）
- 案件スキルを `skills add` でグローバル install しない（プロジェクト脱出で不要なゴミになる）

---

## 主要コマンド早見表

### skills CLI

| コマンド | 用途 |
|---|---|
| `skills add <repo> -g` | 他人のスキルをグローバル install |
| `skills update -g` | グローバル全スキル更新 |
| `skills ls -g` | グローバルスキル一覧 |
| `skills rm <name> -g` | 削除 |
| `skills find <query>` | skills.sh で検索 |
| `skills init <name>` | SKILL.md テンプレ生成（※自作スキルは chezmoi source に作る方が良い） |

フラグ:
- `-g, --global`: ユーザー全体（`~/.claude/skills/`）
- `-a claude-code`: エージェント指定（claude-code / codex / cursor / gemini / windsurf 等）
- `-s <name>`: 特定スキルだけ操作
- `-y`: 確認スキップ
- `--all`: `-s '*' -a '*' -y` の短縮

### chezmoi

| コマンド | 用途 |
|---|---|
| `chezmoi apply` | source → target 反映 |
| `chezmoi edit <target>` | source を開く |
| `chezmoi source-path` | source ディレクトリの場所を表示 |
| `chezmoi add <file>` | 既存ファイルを chezmoi 管理下に取り込む（**他人のスキルには使わない**） |

---

## トラブルシューティング

### 「スキルを更新したのに戻ってしまった」

そのスキルが chezmoi 管理下なのに `skills add`/`update` で上書きした可能性。確認:

```bash
ls ~/dotfiles/dot_claude/skills/<name>/  # あれば chezmoi 管理対象
```

chezmoi 管理対象なら chezmoi source 側を編集する。そうでなければ `skills update` が効く。

### 「ロックに残骸が残っている」

`skills rm -g <name>` で実体とロックを同時削除できる。chezmoi 管理のスキルを誤って `skills add` してロックに残ってしまった場合も、`skills rm -g <name>` でロックだけ綺麗にできる（実体の ~/.claude/skills/<name>/ は chezmoi apply で再生成される）。

### 「新マシンで自作スキルが使えるようにしたい」

dotfiles repo を clone → `chezmoi init --apply` で自動的に配置される。skills CLI は不要。他人のスキルは別途 `skills add` で入れ直し（ロックは各マシンで独立管理が正しい）。

### 「他人のスキルを改変して自作扱いにしたい（移行）」

`skills add` で入れた他人のスキルをカスタマイズして自分の管理下に置くケース。chezmoi に直接 `chezmoi add` するのは禁則（skills CLI のロックと chezmoi source の両方に同名が残って事故る）。代わりに **2 段階で完全に所有権を切り替える**:

```bash
# 1. 現状ファイルを chezmoi source にコピー（別名推奨、将来の元 repo 更新と衝突回避）
cp -R ~/.claude/skills/<orig-name> ~/dotfiles/dot_claude/skills/<new-name>

# 2. frontmatter の name: を新名に変更、自分好みに改変

# 3. 元の skills CLI install を削除（実体とロックを同時除去）
skills rm -g <orig-name>

# 4. chezmoi で反映
chezmoi apply

# 5. dotfiles にコミット
cd ~/dotfiles && git add dot_claude/skills/<new-name> && git commit -m "feat: fork <orig-name> as <new-name>"
```

別解: 元 repo を fork して GitHub に置き、引き続き `skills add <your-fork>` で管理する方法もある。ただし社内 GHE 配布に乗せたいなら上記の dotfiles 取り込みの方が remote 構成と噛み合う。

---

## 迷ったら

- スキルの source-of-truth はどこか？ → それに対応するツールだけで操作する
- 自分で書いた → chezmoi
- 他人から取ってきた → skills CLI
- プロジェクトで使う → プロジェクト repo

混ぜない。迷ったら必ずユーザーに確認してから進める。
