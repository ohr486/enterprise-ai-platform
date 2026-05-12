# CLAUDE.md

このファイルは、Claude Code (claude.ai/code) が本リポジトリで作業する際の指針を提供します。

## リポジトリの状態

**本リポジトリは実装前の設計フェーズにあります。** 成果物は `docs/` 配下の **要件定義書（REQUIREMENTS）** と **実装プラン（PLAN）** のみです。アプリケーションコード・テスト・インフラはまだ何も書かれていないため、ビルド・lint・テストの実行コマンドは存在しません。

コードを追加する際は、選定した言語・ツールに応じた実コマンドを本ファイルに追記してください。

## Source of Truth: 要件定義書と実装プラン

設計ドキュメントは **要件定義書（What / Why / 制約 / 受け入れ基準）** と **実装プラン（How / フェーズ / コード / Terraform）** に分離されています。アーキテクチャ・スコープ・技術選定に関わる変更を提案する前に、該当サブシステムの両ドキュメントを必ず読んでください（日本語で記述されています）。

| サブシステム | 要件定義書（What/Why） | 実装プラン（How） |
|---|---|---|
| OPENCLAW 本体 | `docs/OPENCLAW_PLATFORM_REQUIREMENTS.md` | `docs/OPENCLAW_PLATFORM_PLAN.md`（10 フェーズ、約 10.25 人月） |
| MCP Gateway | `docs/MCP_GATEWAY_REQUIREMENTS.md` | `docs/MCP_GATEWAY_PLAN.md`（8 フェーズ、約 3.3 人月） |
| Knowledge DB (KDB) | `docs/KNOWLEDGE_DB_REQUIREMENTS.md` | 未着手（要件定義のみ、実装プラン起票予定） |

本体プランは以下を定義しています:

- 10 の実装フェーズ（基盤インフラ → ゲートウェイ → Agent Container → AgentCore → Always-On → 管理コンソール/ポータル → Slack → ガバナンス → デジタルツイン → テスト/ドキュメント）
- ターゲット規模、選定技術スタック、モノレポ構成、AWS サービスマップ、マイルストーンロードマップ（M1〜M4）

MCP Gateway と Knowledge DB (KDB) は独立サブシステムとして、本体と共通インフラ（VPC / Cognito / 監査基盤）を共有しつつ配置されます。

要望がプランと矛盾する場合は、黙って逸脱せずユーザーに矛盾を提示してください。要件レベルの変更（What / Why / 制約）が必要なら REQUIREMENTS を、実装レベルの変更（How / フェーズ）が必要なら PLAN を更新してください。

## 厳格なスコープ制約

これらの制約は意図的に選択されたものです。ユーザーの明示的な承認なしにスコープを拡大しないでください:

- **最大 500 ユーザー、単一リージョン強制**。アーキテクチャは 50（MVP）→ 500（最終）規模で設計されています。EKS 採用、マルチリージョン、5,000 ユーザー超のスケール、EU / APAC のデータ主権要件（AgentCore 非対応リージョン内処理必須）は明示的にスコープ外。全データを AgentCore 対応リージョン（us-east-1 / us-west-2）に配置できる組織のみを対象とします。クロスリージョンの仕掛け（`region_override`, Global Table 等）は禁止です。
- **IM プラットフォームは Slack のみ**。Teams / Telegram / Discord / Feishu / WhatsApp アダプタはスコープ外です。`services/im-adapter/` の構成には *将来の* 拡張に備えた薄い `core/` インタフェースを含みますが、実装するのは `slack/` のみです。
- **OpenClaw へのゼロ侵襲**。エージェントの挙動はワークスペースファイル（SOUL.md, IDENTITY.md, SESSION_CONTEXT.md, TOOLS.md など）を `workspace_assembler` でアセンブルすることでのみ制御します。OpenClaw ソースをフォーク・パッチしないでください。OpenClaw のバージョンは Agent Container の Dockerfile で `2026.3.24` に固定 — アップグレード禁止です。
- **`sample/` からのコードコピー禁止**。`sample/` シンボリックリンク（→ `../sample-OpenClaw-on-AWS-with-Bedrock`）は AWS Samples のリファレンス実装です。設計参考としてのみ使用し、独立して実装してください。

## 大局アーキテクチャ（ターゲット）

ターゲットシステムは、7 層構成のマルチテナント AWS ネイティブ AI エージェントプラットフォームです:

```
Presentation (Next.js Portal + Admin Console + Slack adapter)
    ↓
Edge (CloudFront + WAF + API Gateway)
    ↓
Control Plane (FastAPI on ECS Fargate — 組織 CRUD, SOUL エディタ, 監査, RBAC)
    ↓
Gateway Plane (Tenant Router + Bedrock H2 Proxy + MCP Gateway — 3 ティアルーティング, SigV4)
    ↓
Data Plane (Bedrock AgentCore microVM もしくは ECS Fargate Always-On)
    ↓
State (DynamoDB シングルテーブル + S3 ワークスペース/KB/監査 + SSM/Secrets)
    ↓
AI (Bedrock + Guardrails + Knowledge Bases)
```

コンポーネントを横断する 3 つの load-bearing なコンセプトがあります:

1. **3 層 SOUL** — エージェントアイデンティティはコールドスタート時に Global（IT がロック）→ Position（部門管理者）→ Personal（従業員）の Markdown ファイルをマージして組み立てられます。下位レイヤーは上位レイヤーを上書きできず、上位レイヤーは `CRITICAL IDENTITY OVERRIDE` ヘッダ付きで先頭に追加されます。
2. **4 ティアランタイムモデル** — Standard / Restricted / Engineering / Executive。各ティアは独自の Docker イメージ、IAM ロール、Bedrock Guardrail を持ちます。Tenant Router は毎リクエストで position → tier を解決します。
3. **5 層多層防御** — L1 SOUL ルール → L2 ツール許可リスト（DynamoDB `PERM#{posId}`）→ L3 IAM（ティアごとのロール）→ L4 コンピュート分離（Firecracker / Fargate タスク）→ L5 Bedrock Guardrails。L3〜L5 はインフラストラクチャ境界であり、プロンプトインジェクションでバイパスできません。

**Tenant Router** はマルチテナントの心臓部です。ルーティング優先順位: (1) Always-On オーバーライド（SSM `/tenants/{empId}/always-on-agent`）、(2) ポジションルール（DynamoDB `CONFIG#routing`）、(3) デフォルト AgentCore Runtime。セッション ID のプレフィックス（`emp__`, `pgnd__`, `twin__`, `admin__`）がアクセスパスをエンコードしセッションごとの挙動を駆動します。`SESSION_CONTEXT.md` はコールドスタートごとに書き換えられます。

## MCP Gateway サブシステム

`docs/MCP_GATEWAY_REQUIREMENTS.md` および `docs/MCP_GATEWAY_PLAN.md` で定義される独立サブシステム。単一エンドポイントで複数の上流 MCP サーバー（GitHub / Jira / Slack / 社内 DB など）へプロキシするアグリゲーター・ゲートウェイ。`{upstreamId}__{toolName}` 形式で名前空間化し、ユーザー × 上流 × ツールの 3 軸 ACL で認可を行います。

本体プランとは VPC / DynamoDB / S3 / Secrets Manager / Cognito / Admin Console を共有し、ECS タスク・ALB ターゲットグループのみ分離されます。

## Knowledge DB (KDB) サブシステム

`docs/KNOWLEDGE_DB_REQUIREMENTS.md` で定義される横断サブシステム。本体（OPENCLAW Agent Container）と MCP Gateway クライアント（Claude Desktop / Code / 自社エージェント）の **双方から** 利用可能な汎用 RAG / ナレッジ検索基盤です。Bedrock Knowledge Bases をストレージ層、Titan v2 を埋め込みモデル（MVP 固定）として採用予定。

- 専用 DynamoDB テーブル `enterprise-ai-platform-kdb-{env}` を新設し、本体 / MCP Gateway テーブルから **物理分離**（I/O 競合 / PITR コスト結合の回避）
- 本体 Phase 9 Step 5 の `directory_kb.py`（組織ディレクトリ Markdown 自動注入）とは **共存**（統合せず両者並存）
- 実装プラン (`docs/KNOWLEDGE_DB_PLAN.md`) は未起票。実装着手前に起票が必要
- 起票予定 ADR: 0002（Bedrock KB 採用）/ 0003（独立サービス配置）/ 0004（DynamoDB テーブル分離）

## リポジトリ構成

```
.
├── docs/
│   ├── OPENCLAW_PLATFORM_REQUIREMENTS.md  # 本体要件定義（What/Why/制約/AC）
│   ├── OPENCLAW_PLATFORM_PLAN.md          # 本体実装プラン（10 フェーズ）
│   ├── MCP_GATEWAY_REQUIREMENTS.md        # MCP Gateway 要件定義
│   ├── MCP_GATEWAY_PLAN.md                # MCP Gateway 実装プラン（8 フェーズ）
│   ├── KNOWLEDGE_DB_REQUIREMENTS.md       # Knowledge DB 要件定義（実装プラン未起票）
│   └── adr/                               # Architecture Decision Records
│       └── 0001-python-unified-backend.md
├── README.md                    # 概要・設計ドキュメント目次
├── CLAUDE.md                    # 本ファイル
├── .gitignore                   # sample/ と ecc/ シンボリックリンクを除外
├── sample/   → シンボリックリンク（gitignored）  AWS Samples リファレンス、コピー禁止
└── ecc/      → シンボリックリンク（gitignored）  ローカル Claude plugin リポジトリ
```

予定されているモノレポ構成（`apps/`, `services/`, `packages/`, `infra/`, `tests/`, `scripts/`）は `docs/OPENCLAW_PLATFORM_PLAN.md` § 7 に記載されています。フェーズ実行時に必要に応じてディレクトリを作成してください — 空のスキャフォールディングを先に作らないでください。

## プラン運用上の注意

- MVP は Phase 1〜4 + Phase 6a（API 層の最小実装、最小限の Portal Chat UI のみ）。本格的な Admin Console / Portal UI（Phase 6b）は M2 以降。後続フェーズは前フェーズの完了に依存します（依存関係マップは `docs/OPENCLAW_PLATFORM_PLAN.md` § 4、横断依存も同節を参照）。MCP Gateway Phase 4 は本体 Phase 6a Step 3 完了で着手可能。
- 要件と実装の二層構成: 要件（What / Why / 制約 / 受け入れ基準）は `*_REQUIREMENTS.md`、実装（How / フェーズ / コード / Terraform）は `*_PLAN.md` に分離されています。スコープ・制約・AC の変更は REQUIREMENTS で、フェーズ分割・実装手段の変更は PLAN で行ってください。
- 重要な設計判断は `docs/adr/` の Architecture Decision Records として記録します（例: [ADR 0001: Python 統一バックエンド](docs/adr/0001-python-unified-backend.md)）。KDB 関連で ADR 0002 / 0003 / 0004 の起票が予定されています（`docs/KNOWLEDGE_DB_REQUIREMENTS.md` § 11.1）。設計変更を提案する際は該当 ADR を更新するか、新規 ADR を起こしてください。
- プランは意図的に 2 回スコープ縮小されています: (1) 5,000 → 500 ユーザー、(2) 5 IM プラットフォーム → Slack のみ。さらなるスコープ拡大要望は、ルーチンな進化ではなく、改めてユーザー確認が必要な事項として扱ってください。
- ユーザーの母語は日本語です。設計ドキュメントの議論・コメント・コミットメッセージなどは日本語を優先してください。
