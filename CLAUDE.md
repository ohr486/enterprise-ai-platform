# CLAUDE.md

このファイルは、Claude Code (claude.ai/code) が本リポジトリで作業する際の指針を提供します。

## リポジトリの状態

**本リポジトリは要件定義フェーズです。** 過去に存在した実装プラン・ADR・横断サブシステム（MCP Gateway / Knowledge DB）の設計ドキュメントは、設計を一旦リセットするためにすべて削除されています。本体 OPENCLAW プラットフォームの要件定義書のみがスタンドアロン版として復活しています。

現時点では以下が存在します:

- `README.md` — リポジトリ概要
- `CLAUDE.md` — 本ファイル
- `.gitignore` — `sample/` / `ecc/` シンボリックリンクを除外
- `docs/OPENCLAW_PLATFORM_REQUIREMENTS.md` — 本体プラットフォームの要件定義書（Draft、要承認、スタンドアロン）
- `sample/` → シンボリックリンク（gitignored、AWS Samples の参考実装）
- `ecc/` → シンボリックリンク（gitignored、ローカル Claude plugin リポジトリ）

アプリケーションコード・テスト・インフラ・実装プラン・ADR・横断サブシステム要件（MCP Gateway / Knowledge DB）は何も書かれていません。ビルド・lint・テストの実行コマンドも存在しません。

## 作業時の指針

- **要件定義書の編集**: `docs/OPENCLAW_PLATFORM_REQUIREMENTS.md` はスタンドアロン版として書かれており、現時点では削除済みの実装プラン・ADR・横断サブシステム要件（MCP Gateway / KDB）への参照を持ちません。これらを復活させる場合は、要件定義書側のクロスリンクも合わせて更新してください。
- **設計ドキュメントの新規起票**: 実装プラン・ADR・横断サブシステム要件などを書き起こす際は、まずユーザーと「何を / どの粒度で / どのファイル構成で」起こすかを合意してから着手してください。過去の構成（`*_PLAN.md` / `docs/adr/` / `MCP_GATEWAY_PLAN.md` / `KNOWLEDGE_DB_REQUIREMENTS.md`）はリセット済みのため、必ずしも踏襲する必要はありません。
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
