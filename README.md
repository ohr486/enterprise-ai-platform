# enterprise-ai-platform

エンタープライズ向け AI エージェントプラットフォーム。組織駆動型のガバナンスを備えた、役割固有の AI エージェントを社内 500 ユーザー規模に提供する。AWS Bedrock AgentCore + OpenClaw（ゼロ侵襲）+ Slack 連携で構築。

> **本リポジトリは現在、設計フェーズです。** 実装コード・インフラ・テストは未着手。設計ドキュメントのみを保持しています。

## 設計ドキュメント

| ドキュメント | 概要 |
|---|---|
| [docs/OPENCLAW_PLATFORM_PLAN.md](docs/OPENCLAW_PLATFORM_PLAN.md) | OpenClaw プラットフォーム本体の実装プラン（10 フェーズ、約 10.25 人月） |
| [docs/MCP_GATEWAY_PLAN.md](docs/MCP_GATEWAY_PLAN.md) | MCP Gateway サブシステムの実装プラン（8 フェーズ、約 3.3 人月） |
| [docs/adr/](docs/adr/) | Architecture Decision Records（設計判断の履歴） |
| └ [0001: Python 統一バックエンド](docs/adr/0001-python-unified-backend.md) | Bedrock H2 Proxy を含む全バックエンドを Python 3.12 に統一する判断 |
| [CLAUDE.md](CLAUDE.md) | Claude Code 向けのリポジトリ作業ガイダンス |

## 提供価値

ChatGPT Team / Microsoft Copilot との違い:

- **役割ごとに異なる AI エージェントアイデンティティ** — 全員に同じ Bot ではなく、ポジション（職種）ごとに固有の SOUL・ツール権限・知識を持つエージェントを配布
- **組織駆動の自動プロビジョニング** — 入社・異動・退職を組織図と連動させ、エージェントが自動再構成
- **IT 統制可能な 3 層 SOUL** — グローバル（CISO/IT）→ ポジション（部門長）→ パーソナル（本人）でマージ、下位は上位を上書き不可
- **マルチテナント分離** — Firecracker microVM 単位のコンピュート分離 + 4 ティア IAM 境界 + Bedrock Guardrails
- **完全な監査証跡** — 全ツールコール・SOUL 変更・ガードレールブロックを DynamoDB + S3 で永続化
- **MCP アグリゲーター・ゲートウェイ** — 単一エンドポイントで複数の上流 MCP サーバー（GitHub / Jira / 社内 DB など）を集約・配信

## ターゲット規模

| ティア | ユーザー数 | 部門数 | 主な構成 |
|---|---|---|---|
| Small (MVP) | 50 | 1〜3 | シングルランタイム、Portal チャットのみ |
| Medium (最終目標) | 500 | 5〜10 | 4 ティアランタイム、Slack 連携、常時稼働一部、フルガバナンス |

> EKS 採用、マルチリージョン、5,000 ユーザー超のスケール、EU / APAC のデータ主権要件（AgentCore 非対応リージョン内処理必須）は本プロジェクトのスコープ外。全コンポーネントを AgentCore 対応リージョン（us-east-1 / us-west-2）に単一配置することが前提です。

## アーキテクチャ概要

```
Presentation (Next.js Portal + Admin Console + Slack adapter)
    ↓
Edge (CloudFront + WAF + API Gateway)
    ↓
Control Plane (FastAPI on ECS Fargate)
    ↓
Gateway Plane (Tenant Router + Bedrock H2 Proxy + MCP Gateway)
    ↓
Data Plane (Bedrock AgentCore microVM ← MVP / ECS Fargate Always-On ← Phase 5 以降)
    ↓
State (DynamoDB シングルテーブル + S3 + SSM/Secrets)
    ↓
AI (Bedrock + Guardrails + Knowledge Bases)
```

詳細は [docs/OPENCLAW_PLATFORM_PLAN.md](docs/OPENCLAW_PLATFORM_PLAN.md) を参照。

## 技術スタック

| 層 | 採用 |
|---|---|
| フロントエンド | Next.js 15 + React 19 + Tailwind 4 + shadcn/ui |
| バックエンド | FastAPI (Python 3.12) + Pydantic v2 |
| Bedrock プロキシ | FastAPI + httpx + h2 (Python 3.12, Bedrock H2 Proxy も Python 統一) |
| IaC | Terraform 1.10+ |
| データ | DynamoDB シングルテーブル + S3 + Secrets Manager + SSM |
| AI | AWS Bedrock (Nova / Claude / DeepSeek) + AgentCore + Guardrails |
| IM | Slack のみ (Bolt SDK Python) |
| 認証 | Cognito (MVP) → Azure AD / SAML |
| デプロイ | ECS Fargate (ARM64 Graviton) Multi-AZ |
| CI/CD | GitHub Actions + Turborepo + pnpm workspaces |

## リポジトリ構成（現状）

```
.
├── docs/
│   ├── OPENCLAW_PLATFORM_PLAN.md  # OpenClaw 本体設計の Source of Truth
│   └── MCP_GATEWAY_PLAN.md        # MCP Gateway サブシステム設計
├── README.md                    # 本ファイル
├── CLAUDE.md                    # Claude Code 向けガイダンス
├── .gitignore
├── sample/   → シンボリックリンク（gitignored、AWS Samples リファレンス）
└── ecc/      → シンボリックリンク（gitignored、Claude plugin）
```

予定されているモノレポ構成（`apps/`, `services/`, `packages/`, `infra/`, `tests/`, `scripts/`）は実装フェーズ実行時に必要に応じて作成されます。

## マイルストーン

| マイルストーン | 含むフェーズ | 提供価値 |
|---|---|---|
| **M1 MVP** | Phase 1〜4 + Phase 6a（API 層の最小実装） | 50 ユーザー、Portal チャット（最小 UI のみ）、Standard ティアのみ。Phase 6b の本格的な Admin Console / Portal UI は M2 で追加 |
| **M2** | + Phase 5, 7 | 200〜300 ユーザー、常時稼働、Slack 連携 |
| **M3** | + Phase 8, 9 | 500 ユーザー、フルガバナンス、デジタルツイン、Azure AD |
| **M4** | + Phase 10 強化 | 500 ユーザー、SOC2 準拠基盤、運用安定化 |

## 工数見積もり

| サブシステム | 人月 | 期間（2〜3 名） |
|---|---|---|
| OpenClaw 本体 | 10.25 | 4〜5 ヶ月 |
| MCP Gateway | 3.3 | 2〜3 ヶ月 |
| **合計** | **13.55** | **5〜6 ヶ月** |

## 想定コスト（500 ユーザー規模）

| サブシステム | 月額 |
|---|---|
| 本体プラットフォーム | $500〜$1,000 |
| MCP Gateway | $220 |
| **合計** | **$720〜$1,220** |

1 ユーザーあたり $1.5〜$2.5 / 月。ChatGPT Team ($20/人/月) と比べ約 1/10 のコスト。

## 開発状況

実装は未着手。`docs/OPENCLAW_PLATFORM_PLAN.md` の Phase 1（基盤インフラ）から段階的に進める予定。各フェーズの完了基準と依存関係は同プラン § 4 を参照。

## ライセンス

未定（MIT を予定）。
