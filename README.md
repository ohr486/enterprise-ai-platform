# enterprise-ai-platform

エンタープライズ向け AI エージェントプラットフォーム（要件定義フェーズ）。

> **本リポジトリは要件定義フェーズです。** OPENCLAW プラットフォーム要件定義書のみが存在し、実装コード・実装プラン・ADR はまだ存在しません。

## 構想（最小限の方向性メモ）

エンタープライズ組織向けに、役割固有の AI エージェントを社内ユーザーに提供することを想定。AWS Bedrock と OpenClaw を中核として組み立てる。具体的なスコープ・規模・技術選定は要件定義書（[`docs/OPENCLAW_PLATFORM_REQUIREMENTS.md`](./docs/OPENCLAW_PLATFORM_REQUIREMENTS.md)）を参照。

## リポジトリ構成

```
.
├── docs/
│   └── OPENCLAW_PLATFORM_REQUIREMENTS.md  # 要件定義書（Draft）
├── README.md                    # 本ファイル
├── CLAUDE.md                    # Claude Code 向けガイダンス
├── .gitignore
├── sample/    → シンボリックリンク（gitignored、AWS Samples リファレンス、コピー禁止）
├── openclaw/  → シンボリックリンク（gitignored、OpenClaw OSS 本体ソース、参照のみ）
└── ecc/       → シンボリックリンク（gitignored、Claude plugin）
```

## 参考リポジトリ

- [OpenClaw OSS](https://github.com/openclaw/openclaw) — 本プラットフォームが内包する AI エージェントランタイム。`openclaw/` シンボリックリンクで参照可能（フォーク・パッチ禁止、ソース読解のみ）。
- AWS Samples: OpenClaw on AWS with Bedrock — 設計参考のサンプル実装。`sample/` シンボリックリンクで参照可能（コードコピー禁止）。

## ローカルセットアップ

リポジトリ直下の `sample/` / `openclaw/` / `ecc/` は `.gitignore` 除外のシンボリックリンクで、いずれもクローン直後には存在しません。各リポジトリは本リポジトリと **同じ親ディレクトリ** に配置されていることを前提とし、以下のコマンドでリンクを作成します（**任意・参照用途のみ**、本プラットフォームのビルドには必須ではありません）。

```bash
# 親ディレクトリに以下のリポジトリがチェックアウト済みであること:
#   ../openclaw                              （https://github.com/openclaw/openclaw）
#   ../sample-OpenClaw-on-AWS-with-Bedrock   （AWS Samples）
#   ../everything-claude-code                （ローカル Claude plugin）

ln -s ../openclaw openclaw
ln -s ../sample-OpenClaw-on-AWS-with-Bedrock sample
ln -s ../everything-claude-code ecc
```

リンク先リポジトリが存在しない場合はリンク作成をスキップしても本プラットフォームの作業に支障はありません。

## 開発状況

要件定義フェーズ。実装プラン・ADR・実装コードはこれからユーザーと方針を合意した上で進める。

## ライセンス

未定。
