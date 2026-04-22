# dotfiles

[chezmoi](https://www.chezmoi.io/) で管理する macOS 向け dotfiles。主に Claude Code の
`CLAUDE.md` / `settings.json` / `skills/` を配布することを目的としている。

## 構成

```
.
├── .chezmoiignore         # chezmoi が無視するファイル
├── .gitignore
├── README.md
└── dot_claude/            # → ~/.claude/ に配布される
    ├── CLAUDE.md          # グローバルのユーザー指示
    ├── settings.json      # Claude Code の設定
    └── skills/            # ユーザースキル置き場
        ├── example-skill/
        │   └── SKILL.md
        └── skill-management/
            └── SKILL.md   # スキル管理の運用ルール（chezmoi / skills CLI 使い分け）
```

chezmoi の命名規則では `dot_foo` が `~/.foo` に展開される。
つまり `dot_claude/skills/` は `~/.claude/skills/` に配布される。

## インストール

chezmoi 未導入なら先に入れる。

```sh
brew install chezmoi
```

このリポジトリを chezmoi source として取り込んで apply する。

```sh
chezmoi init https://github.com/<your-username>/dotfiles.git
chezmoi diff        # 変更差分を確認
chezmoi apply       # 反映
```

以降、リポジトリ側を更新したら `chezmoi update` で pull & apply できる。

## スキルの追加

1. `dot_claude/skills/<skill-name>/SKILL.md` を作成する
2. frontmatter に `name` と `description` を書く（Claude Code が description からトリガー判定するため、**いつ呼び出すべきか** を具体的に書く）
3. `chezmoi apply` で `~/.claude/skills/` に配布される

最小の SKILL.md 例:

```markdown
---
name: my-skill
description: こういう場面で呼び出す。「こう言われたら」で呼び出し。
---

スキル本文。Claude が読むドキュメントとして書く。
```

自作スキルと他人のスキル（`skills` CLI 管理）、案件スキル（プロジェクト repo 管理）の棲み分けや、社内 GHE 配布時の注意点は `skill-management` スキルに codify してある。Claude Code 上で「スキルを作って」「スキル追加して」等と話しかけると自動で呼び出される。

## ローカルだけの差分を追加したい

コミットしたくないローカル変更は `~/.config/chezmoi/chezmoi.toml` に書くか、
`.chezmoiignore` に追加する。秘密情報は `chezmoi` の template 機能 + 1Password / age
を使うのが定石。

## 参考

- [chezmoi 公式ドキュメント](https://www.chezmoi.io/)
