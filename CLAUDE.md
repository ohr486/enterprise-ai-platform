# CLAUDE.md

このファイルは、Claude Code (claude.ai/code) が本リポジトリで作業する際の指針を提供します。

## リポジトリの状態

**本リポジトリは要件定義フェーズです。** OPENCLAW プラットフォームの要件定義書のみが存在し、実装コード・実装プラン・ADR はまだ存在しません。

現時点では以下が存在します:

- `README.md` — リポジトリ概要
- `CLAUDE.md` — 本ファイル
- `.gitignore` — `sample/` / `ecc/` シンボリックリンクを除外
- `docs/OPENCLAW_PLATFORM_REQUIREMENTS.md` — OPENCLAW プラットフォーム要件定義書（Draft、要承認）
- `sample/` → シンボリックリンク（gitignored、AWS Samples の参考実装）
- `ecc/` → シンボリックリンク（gitignored、ローカル Claude plugin リポジトリ）

アプリケーションコード・テスト・インフラ・実装プラン・ADR は何も書かれていません。ビルド・lint・テストの実行コマンドも存在しません。

## 作業時の指針

- **要件定義書の編集**: `docs/OPENCLAW_PLATFORM_REQUIREMENTS.md` が現時点での唯一の設計ドキュメントです。要件の追加・変更はユーザーと合意の上で本書に集約してください。
- **設計ドキュメントの新規起票**: 実装プラン・ADR などを書き起こす際は、まずユーザーと「何を / どの粒度で / どのファイル構成で」起こすかを合意してから着手してください。
- **コードの追加**: 選定した言語・ツールに応じた実コマンド（ビルド・lint・テスト・実行）を本ファイルに追記してください。
- **`sample/` からのコードコピー禁止**: `sample/` シンボリックリンク（→ `../sample-OpenClaw-on-AWS-with-Bedrock`）は AWS Samples のリファレンス実装です。設計参考としてのみ使用し、独立して実装してください。
- **言語**: ユーザーの母語は日本語です。設計ドキュメントの議論・コメント・コミットメッセージなどは日本語を優先してください。

## リポジトリ構成

```
.
├── docs/
│   ├── .gitkeep
│   └── OPENCLAW_PLATFORM_REQUIREMENTS.md  # 本体プラットフォーム要件定義書（Draft）
├── README.md
├── CLAUDE.md                    # 本ファイル
├── .gitignore
├── sample/   → シンボリックリンク（gitignored、AWS Samples リファレンス、コピー禁止）
└── ecc/      → シンボリックリンク（gitignored、Claude plugin）
```
