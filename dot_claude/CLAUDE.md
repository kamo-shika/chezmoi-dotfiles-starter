# Global Instructions

Claude Code のグローバル指示。プロジェクト横断で常に適用される。

## Language
- ユーザーとの会話は常に日本語
- コード内のコメントは日本語、コミットメッセージは英語

## Workflow
- 指示されたタスクはすぐに着手する。過剰な計画やリサーチに時間をかけない
- 不明点は探索を続けるより先にユーザーに質問する

## Coding Preferences
- Python のパッケージマネージャは `uv`
- Node.js のパッケージマネージャは `pnpm`

## Do NOT
- 依頼されていない README.md / CHANGELOG を勝手に生成しない
- 既存コードに不要なコメント・型アノテーション・docstring を追加しない
