# CLAUDE.md

このファイルは、Claude Code (claude.ai/code) が本リポジトリで作業する際の指針を提供します。

## リポジトリの状態

**本リポジトリは実装前の設計フェーズにあります。** 唯一の成果物は `docs/OPENCLAW_PLATFORM_PLAN.md` および `docs/MCP_GATEWAY_PLAN.md` の実装プランです。アプリケーションコード・テスト・インフラはまだ何も書かれていないため、ビルド・lint・テストの実行コマンドは存在しません。

コードを追加する際は、選定した言語・ツールに応じた実コマンドを本ファイルに追記してください。

## Source of Truth: 実装プラン

`docs/OPENCLAW_PLATFORM_PLAN.md` は権威ある設計ドキュメントです（日本語で記述されています）。アーキテクチャ・スコープ・技術選定に関わる変更を提案する前に必ず読んでください。プランは以下を定義しています:

- 10 の実装フェーズ（基盤インフラ → ゲートウェイ → Agent Container → AgentCore → Always-On → 管理コンソール/ポータル → Slack → ガバナンス → デジタルツイン → テスト/ドキュメント）
- ターゲット規模、選定技術スタック、モノレポ構成、AWS サービスマップ、マイルストーンロードマップ（M1〜M4）

MCP Gateway サブシステムの設計は `docs/MCP_GATEWAY_PLAN.md` に分離されています（8 フェーズ、約 3.3 人月）。本体プランと共通インフラ・認証・監査基盤を共有し、独立サブシステムとして配置されます。

要望がプランと矛盾する場合は、黙って逸脱せずユーザーに矛盾を提示してください。

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

`docs/MCP_GATEWAY_PLAN.md` で定義される独立サブシステム。単一エンドポイントで複数の上流 MCP サーバー（GitHub / Jira / Slack / 社内 DB など）へプロキシするアグリゲーター・ゲートウェイ。`{upstreamId}__{toolName}` 形式で名前空間化し、ユーザー × 上流 × ツールの 3 軸 ACL で認可を行います。

本体プランとは VPC / DynamoDB / S3 / Secrets Manager / Cognito / Admin Console を共有し、ECS タスク・ALB ターゲットグループのみ分離されます。

## リポジトリ構成

```
.
├── docs/
│   ├── OPENCLAW_PLATFORM_PLAN.md  # OpenClaw 本体設計の Source of Truth（日本語）
│   ├── MCP_GATEWAY_PLAN.md        # MCP Gateway サブシステム設計（日本語）
│   └── adr/                       # Architecture Decision Records
│       └── 0001-python-unified-backend.md
├── README.md                    # スタブ
├── CLAUDE.md                    # 本ファイル
├── .gitignore                   # sample/ と ecc/ シンボリックリンクを除外
├── sample/   → シンボリックリンク（gitignored）  AWS Samples リファレンス、コピー禁止
└── ecc/      → シンボリックリンク（gitignored）  ローカル Claude plugin リポジトリ
```

予定されているモノレポ構成（`apps/`, `services/`, `packages/`, `infra/`, `tests/`, `scripts/`）は `docs/OPENCLAW_PLATFORM_PLAN.md` § 7 に記載されています。フェーズ実行時に必要に応じてディレクトリを作成してください — 空のスキャフォールディングを先に作らないでください。

## プラン運用上の注意

- MVP は Phase 1〜4 + Phase 6a（API 層の最小実装）。Phase 6b（UI）は M2 以降。後続フェーズは前フェーズの完了に依存します（依存関係マップは `docs/OPENCLAW_PLATFORM_PLAN.md` § 4、横断依存も同節を参照）。
- 重要な設計判断は `docs/adr/` の Architecture Decision Records として記録します（例: ADR 0001 でバックエンドの Python 統一）。設計変更を提案する際は該当 ADR を更新するか、新規 ADR を起こしてください。
- プランは意図的に 2 回スコープ縮小されています: (1) 5,000 → 500 ユーザー、(2) 5 IM プラットフォーム → Slack のみ。さらなるスコープ拡大要望は、ルーチンな進化ではなく、改めてユーザー確認が必要な事項として扱ってください。
- ユーザーの母語は日本語です。設計ドキュメントの議論・コメント・コミットメッセージなどは日本語を優先してください。
