---
name: example-skill
description: スキルの書き方サンプル。「スキルのサンプルを見せて」「example-skillを呼び出して」で呼び出し。実運用では delete して自分のスキルに置き換える。
---

# example-skill

このファイルは `dot_claude/skills/<skill-name>/SKILL.md` の雛形。

## スキルの基本構造

- ディレクトリ名がスキル名と一致する（`name` frontmatter と揃える）
- frontmatter に `name` と `description` は必須
- `description` は Claude がトリガー判定に使う。**いつ呼び出すべきか** を具体的に書く
  - ❌ "便利なスキル"
  - ✅ "Markdown の目次を生成する。『目次を作って』『TOCを作成』で呼び出し"

## 本文の書き方

Claude がこのスキルを呼び出したときに読むドキュメント。
必要に応じて手順、禁止事項、参照資料へのリンクを書く。

- 手順はチェックリスト形式にすると Claude が TodoWrite に写しやすい
- 厳密に従わせたい制約は明確に書く（「〜してはならない」）
- 長すぎると読まれないので、詳細は `references/` 以下のサブファイルに分けて
  必要時だけ Read させる構成にするとよい

## 追加ファイルの置き方

```
skills/example-skill/
├── SKILL.md
├── references/
│   └── detailed-guide.md
└── scripts/
    └── helper.py
```

SKILL.md 内から `references/detailed-guide.md` を Read するよう指示できる。
