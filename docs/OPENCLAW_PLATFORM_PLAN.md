# 実装プラン: Enterprise OpenClaw Cloud Service

## 概要

エンタープライズ向けに、組織駆動型のAIエージェント・ガバナンス基盤を AWS 上にゼロから構築するクラウドサービスです。`sample/` の参考実装（OpenClaw on AWS with Bedrock — Enterprise）の設計思想（3層SOUL、4ティアランタイム、5層セキュリティ、サーバーレス＋常時稼働ハイブリッド）を踏襲しつつ、**コードはコピーせずに独自実装**します。MVPでは「50ユーザー / 1部門 / シングルランタイム / Portalチャットのみ」から始め、フェーズを重ねて最終規模 500 ユーザーまで段階的にスケールさせます。

---

## 1. 要件の再記述

### 1.1 サービスの定義

「エンタープライズ OpenClaw クラウドサービス」とは、以下を満たす SaaS / 自社ホスト両対応の AI エージェント基盤です：

- **役割固有のAIエージェント** — 全社員に同じBotではなく、ポジション（職種）ごとに固有のアイデンティティ・ツール権限・知識を持ったエージェントを配布
- **組織駆動の自動プロビジョニング** — 入社・異動・退職を組織図と連動させ、エージェントが自動再構成
- **IT統制可能** — グローバルポリシー（CISO管理）/ 部門ポリシー（部門長管理）/ 個人設定（本人）を3層でマージ
- **マルチテナント分離** — Firecracker microVM レベルのコンピュート分離 + IAM境界 + ガードレール
- **マルチチャネル** — Web Portal / IM (Slack のみ) / 公開デジタルツインリンク
- **完全な監査証跡** — 全ツールコール、SOUL変更、ガードレールブロックを永続化

### 1.2 ターゲット規模

| ティア | ユーザー数 | 部門数 | エージェント | 主な構成 |
|-------|----------|-------|-----------|---------|
| **Small (MVP)** | 50 | 1〜3 | 50（=従業員数） | シングルRuntime、Portalのみ、ECS未使用 |
| **Medium (最終目標)** | 500 | 5〜10 | 500 | 4ティアRuntime、IM (Slack のみ)、常時稼働一部、フルガバナンス |

> 本プロジェクトの上限は 500 ユーザーです。EKS 併用やマルチリージョン展開は、本スコープ外（将来要望時に別プロジェクトとして検討）。

### 1.3 必須機能（MVP / 拡張の分離）

#### MVP（Phase 1〜4 + Phase 6 の最小実装）
- 単一 AWS アカウント / シングルリージョンで動作
- 組織CRUD（部門・ポジション・従業員）
- 3層SOULマージ、Bedrock 経由のチャット
- Web Portal（チャットのみ）と Admin Console（最小：組織管理 + 監査閲覧）
- 認証は Employee ID + Password のみ（Azure AD は Phase 8 で導入）
- シングルランタイム（Standard ティアのみ）を AgentCore Firecracker microVM で運用
- DynamoDB シングルテーブル + S3 ワークスペース + 監査ログ（DynamoDB Streams → Firehose → S3 一次系を含む）

#### 拡張機能（Phase 5 以降）
- 常時稼働 ECS Fargate モード（Phase 5）
- IM チャネル統合（Slack のみ）（Phase 7）
- ガバナンス機能（監査検索 UI・Insights・スキル統制・Azure AD SSO・Bedrock Guardrails ティア割当）（Phase 8）
- デジタルツイン公開リンク・プレイグラウンド・IT 管理者アシスタント（Phase 9）
- 負荷試験・一発デプロイ・ドキュメント整備（Phase 10）

> **スコープ外:** EKS 対応、マルチリージョン、5,000ユーザー超のスケール、**EU / APAC のデータ主権要件があり AgentCore 非対応リージョン内処理が必須のテナント** は本プロジェクトの対象外。

### 1.4 成功基準

- [ ] 50 ユーザー想定で `bash deploy.sh` 相当の一発デプロイが 30 分以内に完了
- [ ] 平均レスポンスレイテンシ：コールドスタート 中央値 < 10s / p95 < 15s、ウォーム 中央値 < 3s / p95 < 5s
- [ ] テストカバレッジ 80%+（unit / integration / e2e）
- [ ] **マルチテナント漏洩テスト**: テナント ID 改ざん・偽プレフィックス注入・並行リクエスト混線の 100 ケース以上を CI 必須化
- [ ] OWASP Top 10 への対策が CI で検証される
- [ ] DynamoDB / S3 / Bedrock すべてが IAM 最小権限で構成
- [ ] 1 ユーザー / 月 あたりの AWS コストが 500 ユーザー規模で $5 以下
- [ ] 全コンポーネントが単一リージョン配置（クロスリージョン構成禁止）

---

## 2. アーキテクチャ概要

### 2.1 レイヤー構成（独自設計）

```
┌─────────────────────────────────────────────────────────────────┐
│ Presentation Layer                                              │
│   - Web Portal (Next.js 15 / React 19) ← 従業員向け             │
│   - Admin Console (Next.js 15 / React 19) ← IT・部門管理者向け  │
│   - IM Adapter  (Slack のみ)                                    │
└─────────────────────────────────────────────────────────────────┘
                               │
┌─────────────────────────────────────────────────────────────────┐
│ Edge / API Layer                                                │
│   - CloudFront + WAF                                            │
│   - API Gateway (REST/HTTP) → Lambda / ALB                      │
│   - 認証: Cognito (MVP) → Azure AD/SAML (Phase 8)               │
└─────────────────────────────────────────────────────────────────┘
                               │
┌─────────────────────────────────────────────────────────────────┐
│ Control Plane (Admin/Portal API)                                │
│   - FastAPI (Python 3.12) on ECS Fargate (ALB 配下)             │
│   - 組織CRUD / SOUL編集 / 監査検索 / 使用量集計 / RBAC強制       │
└─────────────────────────────────────────────────────────────────┘
                               │
┌─────────────────────────────────────────────────────────────────┐
│ Gateway Plane (Tenant Router + Bedrock H2 Proxy + MCP Gateway*) │
│   - Tenant Resolver: channel_user_id → emp_id → runtime         │
│   - 3ティアルーティング: 常時稼働 → ポジション → デフォルト       │
│   - Bedrock H2 Proxy: AWS SigV4 署名 + ストリーミング            │
│   * MCP Gateway は独立サブシステム — docs/MCP_GATEWAY_PLAN.md     │
└─────────────────────────────────────────────────────────────────┘
                               │
┌─────────────────────────────────────────────────────────────────┐
│ Data Plane (Agent Runtime)                                      │
│   ┌─────────────────────────┐  ┌─────────────────────────────┐ │
│   │ Serverless              │  │ Always-On                   │ │
│   │ Bedrock AgentCore       │  │ ECS Fargate (+EFS)          │ │
│   │ Firecracker microVM     │  │ Long-running container      │ │
│   └─────────────────────────┘  └─────────────────────────────┘ │
│        各コンテナ: workspace_assembler → OpenClaw → Bedrock     │
└─────────────────────────────────────────────────────────────────┘
                               │
┌─────────────────────────────────────────────────────────────────┐
│ State Layer                                                     │
│   - DynamoDB (シングルテーブル設計)                              │
│   - S3 (ワークスペース / KB / SOULテンプレート / 監査長期保管)    │
│   - SSM Parameter Store (ランタイムID / トークン)                │
│   - Secrets Manager (DB / Bot トークン)                          │
│   - OpenSearch (監査ログ全文検索 / 拡張機能)                     │
└─────────────────────────────────────────────────────────────────┘
                               │
┌─────────────────────────────────────────────────────────────────┐
│ AI Layer                                                        │
│   - AWS Bedrock (Nova / Claude / DeepSeek / Mistral)            │
│   - Bedrock Guardrails (PII / Topic / Injection フィルタ)        │
│   - Bedrock Knowledge Bases (RAG)                                │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 主要データフロー

```
[従業員Portal] → CloudFront → ALB → Control Plane API
                                    ↓ (auth + emp_id 解決)
                                    ↓ Tenant Router (gRPC/HTTP)
                                    ↓ → 3ティア解決 (DynamoDB CONFIG#routing)
                                    ↓
                       ┌────────────┴────────────┐
                       ↓                         ↓
                AgentCore Runtime         ECS Fargate Task
                (Firecracker microVM)     (常時稼働コンテナ)
                       ↓                         ↓
            workspace_assembler が S3 から SOUL/KB/Skills を取得
                       ↓
                   OpenClaw 実行 → Bedrock(Guardrails) → ストリーミング応答
                       ↓
                   監査イベント → DynamoDB (AUDIT#) + 非同期で S3
```

### 2.3 セキュリティ境界（5層多層防御）

| L# | 対策 | 実装 | Trust Boundary |
|----|------|------|---|
| L1 | SOUL ルール | Markdown プロンプト規約 | ❌ ガバナンス層（プロンプトインジェクションで破られうる） |
| L2 | ツール許可リスト | DynamoDB `PERM#{posId}` | △ モデル遵守に依存 |
| L3 | IAM ロール（ティア別） | 4 つの IAM ロール | ✅ インフラ境界 |
| L4 | コンピュート分離 | Firecracker / Fargate Task 単位 | ✅ ハードウェア境界 |
| L5 | Bedrock Guardrails | ティアごと割当 | ✅ インフラ境界 |

> **重要**: L1 はガバナンス層（行動規範）であり、Trust Boundary ではありません。L1 単独に依存せず、必ず L3〜L5 のインフラ境界で防御してください。SLA・セキュリティ評価は L3 以降のみを保証範囲とします。

---

## 3. 段階的実装フェーズ

各フェーズは独立してマージ可能・検証可能な単位で構成しています。

### Phase 1: 基盤インフラ + 監査一次系（複雑度: Medium / 工数: 4 週間 ≒ 1.0 人月）

> **工数注**: 旧見積もり 0.75 人月では監査一次系（Firehose / Athena / Object Lock）の前倒し構築を吸収しきれないため 1.0 人月に増。9 ステップのうち Step 5（DynamoDB + 監査一次系）が最も重い。

**目的：** AWS アカウントに、後続のフェーズが乗る土台を IaC で構築する。

1. **モノレポ初期化** （File: `package.json`, `pnpm-workspace.yaml`, `turbo.json`）
   - Action: pnpm workspaces + Turborepo セットアップ、共通 ESLint/Prettier/TypeScript 設定
   - Why: フロント / バック / IaC を同じツリーで管理しビルドキャッシュを共有
   - Dependencies: なし
   - Risk: Low

2. **Terraform ルートモジュール**（File: `infra/terraform/main.tf`, `infra/terraform/versions.tf`）
   - Action: `aws` provider、リモートステート (S3 + DynamoDB lock)、環境別 workspace (dev/stg/prod)
   - Why: CloudFormation より差分管理・モジュール化が容易、既存 AWS 知見と親和的
   - Dependencies: なし
   - Risk: Medium（state バックエンドの設計ミスは後で響く）

3. **ネットワークモジュール**（File: `infra/terraform/modules/network/`）
   - Action: VPC（10.0.0.0/16）、3 AZ、Public/Private/Isolated サブネット、NAT、VPC Endpoints (S3, DynamoDB, SSM, ECR, Bedrock)
   - Why: パブリックポートゼロ要件、Bedrock VPC Endpoint で egress 削減
   - Dependencies: Step 2
   - Risk: Medium

4. **IAM モジュール**（File: `infra/terraform/modules/iam/`）
   - Action: 4 ティア用ロール（Standard / Restricted / Engineering / Executive）、Control Plane タスクロール、Gateway タスクロール、Deploy ロール
   - Why: Phase 4 以降で参照、最小権限の境界を最初に定義
   - Dependencies: Step 2
   - Risk: High（過剰権限が後から残る）

5. **DynamoDB シングルテーブル + 監査一次系**（File: `infra/terraform/modules/dynamodb/`, `infra/terraform/modules/audit-pipeline/`）
   - Action:
     - テーブル `enterprise-ai-platform-{env}`、PK/SK、GSI1〜GSI3、PITR、TTL（90日）、Streams 有効化
     - **DynamoDB Streams → Kinesis Firehose → S3 (Object Lock) → Athena テーブル を Phase 1 から構築**（旧 Phase 8 Step 1 を前倒し）
     - DynamoDB 側の `AUDIT#` には直近 24〜72h のホット参照分のみ保持（TTL 自動削除）
   - Why: ホットパーティション/GSI 書き込みコスト爆発を回避。長期保管・全文検索は S3 + Athena で実現
   - Dependencies: Step 2
   - Risk: Medium（GSI 設計を間違えるとクエリパターンが破綻）

6. **S3 バケット群**（File: `infra/terraform/modules/storage/`）
   - Action: `workspaces/`, `knowledge/`, `skills/`, `audit-archive/`, `frontend-assets/` を SSE-KMS + バージョニング + Object Lock で
   - Why: ワークスペース / KB / 長期監査の分離、コンプライアンス対応
   - Dependencies: Step 2, Step 4 (KMS)
   - Risk: Low

7. **SSM / Secrets Manager**（File: `infra/terraform/modules/secrets/`）
   - Action: `JWT_SECRET`, `ADMIN_INITIAL_PASSWORD`, ボットトークン用パラメータパス定義
   - Why: ハードコード絶対禁止、初期セットアップで自動生成
   - Dependencies: Step 4
   - Risk: Medium

8. **Bedrock 有効化チェック**（File: `infra/terraform/modules/bedrock/`）
   - Action: モデルアクセス確認スクリプト、Bedrock Guardrails のスケルトン作成
   - Why: us-east-1 / us-west-2 限定、リージョン選定の自動検証
   - Dependencies: Step 2
   - Risk: Low

9. **CI/CD パイプライン**（File: `.github/workflows/infra-plan.yml`, `infra-apply.yml`）
   - Action: PR で `terraform plan`、main マージで `apply`、OIDC で AWS 認証
   - Why: 手動 apply の事故防止
   - Dependencies: Step 2
   - Risk: Medium

**完了基準:** `terraform apply` で空の VPC + DynamoDB + S3 + IAM が dev 環境に作成される。`pnpm build` がモノレポ全体で通る。

---

### Phase 2: ゲートウェイ層（複雑度: High / 工数: 2〜3週間）

**目的：** 受信リクエストをテナント解決し、Bedrock or Agent Runtime に振り分ける。

1. **Tenant Router (Python/FastAPI)**（File: `services/gateway/tenant-router/app/main.py`）
   - Action: `/route` エンドポイント、`channel + raw_user_id` から `emp_id` 解決、3ティアルーティング判定
   - Why: マルチテナントの心臓部。プレフィックス（`emp__`, `tg__`, `twin__` など）の発行も担当
   - Dependencies: Phase 1 完了
   - Risk: High（ルーティング欠陥はテナント間情報漏洩に直結）

2. **テナント解決リポジトリ**（File: `services/gateway/tenant-router/app/repositories/mapping_repo.py`）
   - Action: DynamoDB `MAPPING#{channel}__{userId}` の lookup、5分キャッシュ
   - Why: 1リクエストあたり1ms以内の解決が必要
   - Dependencies: Step 1
   - Risk: Medium

3. **ルーティング設定キャッシュ**（File: `services/gateway/tenant-router/app/services/routing_cache.py`）
   - Action: `CONFIG#routing` を 5 分キャッシュ、TTL 後にバックグラウンドリフレッシュ
   - Why: DynamoDB 読み取り削減
   - Dependencies: Step 1
   - Risk: Low

4. **Bedrock H2 Proxy (Python + FastAPI + httpx[http2])**（File: `services/gateway/bedrock-proxy/app/server.py`）
   - Action: HTTP/2 で受信、AWS SigV4 (`botocore.auth.SigV4Auth`) で Bedrock 署名し直し、`httpx.AsyncClient(http2=True)` でストリーミング転送
   - Why: OpenClaw 由来のクライアントは Bedrock API を直接叩く想定。プロキシで監査・ガードレール介入。**Python 統一でメンテコスト最小化**
   - Dependencies: Step 1
   - Risk: High（ストリーミング維持と SigV4 の両立、p95 レイテンシ閾値 + バックプレッシャ試験必須）

5. **ガードレール介入レイヤー**（File: `services/gateway/bedrock-proxy/app/middleware/guardrail.py`）
   - Action: Input/Output で `apply_guardrail` を呼び、ブロック時に監査イベント発行
   - Why: L5 セキュリティ
   - Dependencies: Step 4
   - Risk: Medium

6. **ECS Fargate デプロイ定義**（File: `infra/terraform/modules/gateway/`）
   - Action: ALB + 2 タスク（Tenant Router / H2 Proxy）、Health Check、Auto Scaling
   - Why: EC2 単一インスタンスより冗長
   - Dependencies: Phase 1, Step 1〜5
   - Risk: Medium

7. **ゲートウェイ単体・統合テスト + マルチテナント漏洩テスト**（File: `services/gateway/tenant-router/tests/`, `tests/security/tenant-isolation/`）
   - Action:
     - `moto` で DynamoDB をモック、解決パターン全網羅、ストリーミングテスト
     - **テナント ID 改ざん試行・偽プレフィックス注入・並行リクエストでのセッション混線を網羅する漏洩テストを CI 必須化**（§ 5.1 リスク表「マルチテナント情報漏洩」の緩和策）
     - プロパティベーステスト（`hypothesis`）で `channel + userId` 入力空間を fuzz
   - Why: ルーティング精度 100% + テナント間情報漏洩の確実な防止
   - Dependencies: Step 1〜5
   - Risk: Low

**完了基準:** `curl /route` が `emp-001` を含むテナント解決結果を返す。Bedrock Proxy 経由でローカルから Claude Haiku に到達できる。テナント漏洩テスト 100 ケースが全パス。

---

### Phase 3: Agent Container（複雑度: High / 工数: 5〜6 週間 ≒ 1.5 人月）

**目的：** OpenClaw を内包し、3層SOULマージ・スキルロード・権限制御を行う実行コンテナを作る。

1. **コンテナ基本構成**（File: `services/agent-container/Dockerfile`, `services/agent-container/entrypoint.sh`）
   - Action: ARM64 Graviton ベース、OpenClaw `2026.3.24` 固定、Python 3.12、起動時に `entrypoint.sh` がワークスペース sync → assembler → OpenClaw 起動
   - Why: バージョン固定は IM 統合互換性のため必須
   - Dependencies: Phase 1
   - Risk: High（バージョン依存の罠）

2. **Workspace Assembler**（File: `services/agent-container/src/workspace/assembler.py`）
   - Action: S3 から `_shared/soul/global/`、`_shared/soul/positions/{posId}/`、`{empId}/workspace/` を取得しマージ。`SOUL.md`、`IDENTITY.md`、`SESSION_CONTEXT.md` を生成
   - Why: 3層SOULの実体。**コア知財**
   - Dependencies: Step 1
   - Risk: High（マージロジックのバグはアイデンティティ崩壊）

3. **Skill Loader**（File: `services/agent-container/src/skills/loader.py`）
   - Action: ポジションの `allowedRoles/blockedRoles` でフィルタ、許可されたスキルのみ `skills/` にコピー
   - Why: ファイナンスに `shell` を渡さない
   - Dependencies: Step 2
   - Risk: Medium

4. **Permission Profile**（File: `services/agent-container/src/permissions/profile.py`）
   - Action: DynamoDB `PERM#{posId}` から許可リストを取得、SOUL 先頭にプリペンド (Plan A)
   - Why: 実行前ガード
   - Dependencies: Step 2
   - Risk: Medium

5. **Identity / Session Context**（File: `services/agent-container/src/session/context.py`）
   - Action: セッション ID プレフィックス（`emp__`, `pgnd__`, `twin__`, `admin__`）から SESSION_CONTEXT.md を書き分け
   - Why: ポータル / プレイグラウンド / ツインの挙動分岐
   - Dependencies: Step 2
   - Risk: Medium

6. **Memory 同期**（File: `services/agent-container/src/memory/sync.py`）
   - Action: 60 秒ウォッチドッグで `memory/` を S3 へライトバック、ターンごとのチェックポイント
   - Why: Admin Console から閲覧可能にする
   - Dependencies: Step 2
   - Risk: Medium

7. **Observability**（File: `services/agent-container/src/observability/logger.py`）
   - Action: 全呼び出し / ツールコール / 拒否を構造化ログ + DynamoDB `AUDIT#` に書き込み
   - Why: 監査要件
   - Dependencies: Step 1
   - Risk: Low

8. **Safety / 入力検証**（File: `services/agent-container/src/safety/validators.py`）
   - Action: メッセージ長、危険パターン（`rm -rf` 等）、HTML/SQL injection フィルター
   - Why: 入力境界バリデーション
   - Dependencies: Step 1
   - Risk: Medium

9. **Agent Container HTTP サーバ**（File: `services/agent-container/src/server.py`）
   - Action: `/invocations` エンドポイント、`openclaw agent --session-id` をサブプロセス呼び出し、Plan E 監査
   - Why: AgentCore Runtime / ECS の両方が叩く統一インタフェース
   - Dependencies: Step 1〜8
   - Risk: High

10. **コンテナ単体・統合テスト**（File: `services/agent-container/tests/`）
    - Action: `moto` + ローカル OpenClaw（モック）で 3層マージ・ティア切替・拒否ケースを網羅
    - Why: 80% カバレッジ
    - Dependencies: Step 1〜9
    - Risk: Low

**完了基準:**
- ローカル Docker で `POST /invocations` を叩くと 3 層 SOUL がマージされた応答が返る
- **マージ結果のスナップショットテスト** (`tests/golden/soul_merge/`) 全 20 ケース以上が一致（Global only / Position only / Personal override 試行 / `CRITICAL IDENTITY OVERRIDE` 順序検証など）
- ティア（Standard/Restricted/Engineering/Executive）切替が 1 設定変更で完結
- 危険パターン拒否ケース（`rm -rf`, SQL injection 等）が 100% ブロック
- 単体・統合テストカバレッジ 80%+

---

### Phase 4: AgentCore Runtime 統合（複雑度: High / 工数: 2週間）

**目的：** Phase 3 のコンテナを Bedrock AgentCore に登録し、Firecracker microVM で実行する。

1. **AgentCore Runtime 作成スクリプト**（File: `infra/scripts/runtime/create-runtime.sh`）
   - Action: `bedrock-agentcore-control create-agent-runtime` を 4 ティア（Standard/Restricted/Engineering/Executive）分、各 IAM ロール紐付け
   - Why: ティアごとに ECR イメージと IAM が異なる
   - Dependencies: Phase 1, 3
   - Risk: High

2. **Runtime ID を SSM へ保存**（File: `infra/terraform/modules/agentcore/`）
   - Action: 出力を `/openclaw/{env}/runtime/{tier}-id` に格納
   - Why: Tenant Router がここを参照
   - Dependencies: Step 1
   - Risk: Low

3. **ECR ライフサイクルポリシー**（File: `infra/terraform/modules/ecr/`）
   - Action: 直近 10 イメージのみ保持、古いタグは自動削除
   - Why: コスト
   - Dependencies: Phase 1
   - Risk: Low

4. **Session Storage 設定**（File: `services/agent-container/src/session/storage.py`）
   - Action: AgentCore セッションストレージで `workspace/` を永続化
   - Why: コールドスタートを 2〜3 秒に短縮
   - Dependencies: Step 1
   - Risk: Medium

5. **設定リフレッシュ機構**（File: `services/agent-container/src/config/refresh.py`）
   - Action: `CONFIG#global-version` を 5 分間隔でポーリング、変更時にアセンブリキャッシュをクリア
   - Why: 再デプロイなしで SOUL 更新を反映
   - Dependencies: Phase 3
   - Risk: Medium

6. **Runtime 統合 e2e テスト**（File: `tests/e2e/agentcore/`）
   - Action: 実 AgentCore に対しテストテナントでメッセージ往復
   - Why: 本物環境でのコールドスタート時間測定
   - Dependencies: Step 1〜5
   - Risk: Medium

**完了基準:**
- Tenant Router → AgentCore（Standard ティア）でエンド・ツー・エンド応答
- コールドスタート: **中央値 < 10s, p95 < 15s**（10 連続実測、Workspace Assembler の S3 fetch 含む）
- ウォーム応答: 中央値 < 3s, p95 < 5s
- Session Storage で 2 回目以降のセッション再開が 2〜3s
- 数値未達の場合は Provisioned Concurrency 検討タスクを Phase 5 と並列で計画

---

### Phase 5: ECS Fargate 常時稼働モード（複雑度: Medium / 工数: 2週間）

**目的：** カスタマーサービスやエグゼクティブ向けに、常時稼働コンテナを提供する。

1. **Fargate タスク定義**（File: `infra/terraform/modules/fargate-tier/`）
   - Action: 4 ティアそれぞれの Task Definition、`desiredCount=0` で作成
   - Why: 管理者が UI から起動
   - Dependencies: Phase 3, 4
   - Risk: Medium

2. **EFS マウント**（File: `infra/terraform/modules/efs/`）
   - Action: VPC 内 EFS、ティアごとにアクセスポイント、`/mnt/efs/{empId}/workspace`
   - Why: 永続ボリューム
   - Dependencies: Phase 1
   - Risk: Medium

3. **Service Discovery (AWS Cloud Map) 登録**（File: `infra/terraform/modules/cloudmap/`, ECS タスク定義側）
   - Action:
     - ECS Service の `serviceRegistries` で AWS Cloud Map (`{empId}.always-on.internal`) に自動登録
     - **DNS TTL を 10 秒に設定**（既定 60 秒は完了基準「30 秒以内に登録」と整合しないため）
     - Health Check に紐づき、停止時は自動で deregister
   - Why: **SSM Parameter Store はサービスディスカバリ用途ではない**。再起動・スケール時の古い IP 参照を防ぐにはマネージドの Service Discovery（Cloud Map）か ALB ターゲットグループが必須
   - Dependencies: Step 1
   - Risk: Low（マネージド）

4. **Tenant Router の Always-On 分岐**（File: `services/gateway/tenant-router/app/services/runtime_resolver.py`）
   - Action: 1 番目に Cloud Map の DNS lookup（`{empId}.always-on.internal`）、ヒットすれば直接 HTTP、NXDOMAIN ならポジションルールへフォールスルー
   - Why: 3ティアルーティングの 1 段目（IP 直接保持を廃止）
   - Dependencies: Step 3, Phase 2
   - Risk: Low

5. **Always-On 切替 API**（File: `services/control-plane/api/routes/agents.py` の追加）
   - Action: `PATCH /agents/{id}/runtime-mode` で `serverless` ↔ `always-on`
   - Why: 管理コンソールから操作
   - Dependencies: Step 1
   - Risk: Medium

**完了基準:**
- 管理者が UI から特定エージェントを Always-On に切替後、Cloud Map に 30 秒以内に登録
- 切替後の応答レイテンシ: 中央値 < 1s, p95 < 2s（コールドスタートゼロ）
- Always-On コンテナ再起動時にリクエスト失敗ゼロ（10 分間 1 RPS の継続負荷で計測）
- 成功率 99.5%+

---

### Phase 6: 管理コンソール / Portal API（複雑度: High / 工数: 10〜14 週間 ≒ 2.5〜3.5 人月）

> **6a**（API 層: Step 1〜7, 約 1.5 人月）と **6b**（UI 層: Step 8〜10, 約 1.0〜2.0 人月）に分割可能。6a 完了後は curl / Postman で全機能検証可能、6b は React/Next.js 経験者数に応じて伸縮。

**目的：** 組織管理・SOUL編集・監査閲覧の Web UI を提供する。

1. **FastAPI バックエンド骨格**（File: `services/control-plane/api/main.py`）
   - Action: FastAPI + Pydantic v2 + SQLModel(DynamoDB Repository) + JWT 認証
   - Why: 型安全 + OpenAPI 自動生成
   - Dependencies: Phase 1
   - Risk: Medium

2. **DynamoDB リポジトリ層**（File: `services/control-plane/api/repositories/`）
   - Action: Org / Position / Employee / Audit / Skill / SOUL の Repository インタフェース、シングルテーブル設計に追従
   - Why: ビジネスロジックを DynamoDB から分離
   - Dependencies: Phase 1
   - Risk: High

3. **3-role RBAC**（File: `services/control-plane/api/auth/rbac.py`）
   - Action: Admin / Manager / Employee、Manager は自部門のみ（BFS ロールアップ）
   - Why: 権限境界
   - Dependencies: Step 1
   - Risk: High

4. **Auto-Provisioning Hook**（File: `services/control-plane/api/services/provisioning.py`）
   - Action: 従業員作成時にエージェント、1:1 バインディング、S3 ワークスペースシード、監査エントリを一括作成
   - Why: 組織図 → エージェント自動生成
   - Dependencies: Step 2
   - Risk: High（部分失敗時の補償トランザクション）

5. **SOUL エディタ API**（File: `services/control-plane/api/routes/soul.py`）
   - Action: グローバル / ポジション / パーソナル の CRUD、変更時に `CONFIG#global-version` を bump
   - Why: 再デプロイなし反映の起点
   - Dependencies: Step 2
   - Risk: Medium

6. **監査検索 API**（File: `services/control-plane/api/routes/audit.py`）
   - Action: GSI を使った時系列・テナント・イベントタイプ検索、ページング
   - Why: ガバナンス必須
   - Dependencies: Step 2
   - Risk: Medium

7. **使用量・コスト集計**（File: `services/control-plane/api/services/usage.py`）
   - Action: DynamoDB Streams → Lambda → 日次・部門別・モデル別集計、$/1Mトークンの料金表
   - Why: コスト透明性
   - Dependencies: Step 2
   - Risk: Medium

8. **Admin Console フロントエンド**（File: `apps/admin-console/`）
   - Action: Next.js 15 + Tailwind 4 + shadcn/ui、ダッシュボード / Agent Factory / Security Center / Audit / Usage / Slack 連携管理（最低 8 ページ）
   - Why: IT・部門管理者の UI
   - Dependencies: Step 1〜7
   - Risk: Medium

9. **Portal フロントエンド**（File: `apps/portal/`）
   - Action: Next.js 15、Chat / Profile / Skills / Slack Connect / Digital Twin Toggle（最低 5 ページ）
   - Why: 従業員の UI
   - Dependencies: Step 1〜7
   - Risk: Medium

10. **API 統合テスト**（File: `services/control-plane/api/tests/`）
    - Action: pytest + httpx、RBAC 強制と Auto-Provisioning 全パターン
    - Why: 80% カバレッジ
    - Dependencies: Step 1〜7
    - Risk: Low

**完了基準:** 管理者がブラウザで部門・ポジション・従業員を作成 → 自動でエージェントが立ち上がり、Portal でチャット可能。

---

### Phase 7: IM チャネル統合（Slack のみ）（複雑度: Medium / 工数: 1.5〜2週間 ≒ 0.5 人月）

**目的：** Slack からエージェントに到達できるようにする。本プロジェクトでは Slack のみをサポート対象とし、他 IM プラットフォーム（Teams / Telegram / Discord / Feishu / WhatsApp）はスコープ外。

> **横断依存**: Step 4（Always-On 直接 Slack 接続）は **Phase 5** 完了が必須。Step 1〜3 は Phase 6 のみ依存。

1. **IM Adapter 共通インタフェース**（File: `services/im-adapter/core/`）
   - Action: `IIMAdapter` インタフェース（`receive`, `send`, `verifyWebhook`）と共通バリデータ。将来的な拡張に備えた抽象化のみ用意し、実装は Slack 1 種類
   - Why: 将来の他 IM 追加時の手戻り防止（過剰実装はしない）
   - Dependencies: Phase 2
   - Risk: Low

2. **Slack アダプタ**（File: `services/im-adapter/slack/`）
   - Action: Slack Bolt SDK、Events API、Slash コマンド、署名検証（`X-Slack-Signature`）、Block Kit メッセージ整形、スレッド対応
   - Why: 唯一サポートする IM プラットフォーム
   - Dependencies: Step 1
   - Risk: Medium

3. **ペアリングフロー API**（File: `services/control-plane/api/routes/im_pairing.py`）
   - Action: ワンタイムトークン発行 → 従業員が Slack DM で `/openclaw pair TOKEN` 実行 → DynamoDB `MAPPING#slack__{userId}` 書き込み
   - Why: Self-Service オンボーディング
   - Dependencies: Phase 6
   - Risk: Medium

4. **Always-On 直接 Slack 接続**（File: `services/agent-container/src/im/slack_direct.py`）
   - Action: 常時稼働コンテナ内に専用ボットトークン、Slack Webhook を直受け
   - Why: スケジュールタスクや CS bot 用途
   - Dependencies: Phase 5
   - Risk: Medium

5. **Slack アダプタ単体・統合テスト**（File: `services/im-adapter/slack/tests/`）
   - Action: 署名検証、Events API、ペアリング、エラーパスを網羅
   - Why: 80% カバレッジ
   - Dependencies: Step 1〜4
   - Risk: Low

**完了基準:** Portal からペアリングトークンを発行 → Slack DM 経由で 30 秒以内にペアリング → Slack とポータルでメモリ共有された応答が返る。

---

### Phase 8: ガバナンス機能（複雑度: Medium / 工数: 3週間）

**目的：** 監査・コスト・スキル統制・Bedrock Guardrails の運用画面を完備する。

1. **監査長期保管**（File: `infra/terraform/modules/audit-archive/`）
   - Action: DynamoDB Streams → Firehose → S3 (Object Lock)、Athena 検索
   - Why: 7 年保管要件想定
   - Dependencies: Phase 1, 6
   - Risk: Medium

2. **Insight Detector**（File: `services/control-plane/api/services/insights.py`）
   - Action: 異常パターン 5 種（拒否多発、PII 流出未遂、SOUL 改変、Slack 大量ペアリング、コスト跳ね上がり）を日次バッチで検出
   - Why: プロアクティブガバナンス
   - Dependencies: Step 1
   - Risk: Medium

3. **Bedrock Guardrails 統合**（File: `infra/terraform/modules/guardrails/`）
   - Action: ティアごとのガードレールを Terraform で定義、UI から付替え可能に
   - Why: L5 セキュリティ
   - Dependencies: Phase 1, 4
   - Risk: Medium

4. **コスト分析ダッシュボード**（File: `apps/admin-console/app/usage/`）
   - Action: 部門別 / モデル別 / 個人別、予算閾値アラート（SNS）
   - Why: 経営報告
   - Dependencies: Phase 6 Step 7
   - Risk: Low

5. **スキルガバナンス**（File: `services/control-plane/api/routes/skills.py`）
   - Action: スキル要求 → 管理者承認ワークフロー、`PERM#{posId}` への反映
   - Why: 統制
   - Dependencies: Phase 6
   - Risk: Medium

6. **Azure AD SSO**（File: `services/control-plane/api/auth/azure_ad.py`）
   - Action: SAML 2.0 / OIDC、`emp_id` クレームマッピング
   - Why: エンタープライズ標準
   - Dependencies: Phase 6
   - Risk: High

**完了基準:** 監査画面で 1 万件のイベントを検索 < 2 秒。Guardrails ブロックが UI 上で時系列に閲覧可能。

---

### Phase 9: デジタルツイン + 高度機能（複雑度: Medium / 工数: 2〜3週間）

**目的：** 公開リンク / IT 管理者アシスタント / プレイグラウンドなど差別化機能。

1. **デジタルツイン公開エンドポイント**（File: `services/control-plane/api/routes/twin.py`）
   - Action: `GET /twin/{token}` 公開ページ、`POST /public/twin/{token}/chat`
   - Why: 営業時間外 AI 可用性
   - Dependencies: Phase 6
   - Risk: High（認証なし公開エンドポイント、レート制限必須）

2. **ツイン専用ワークスペース**（File: `services/agent-container/src/twin/workspace.py`）
   - Action: 従業員メイン workspace と分離した `twin_workspace/`、メモリ書き戻しなし
   - Why: メイン会話汚染防止
   - Dependencies: Phase 3
   - Risk: Medium

3. **プレイグラウンド**（File: `apps/admin-console/app/playground/`）
   - Action: `pgnd__` プレフィックスで read-only セッション、SOUL 試験
   - Why: SOUL 編集後の動作確認
   - Dependencies: Phase 6
   - Risk: Low

4. **IT 管理者アシスタント**（File: `services/control-plane/api/routes/admin_assistant.py`）
   - Action: フローティングチャット、Claude Direct API、10 ホワイトリストツール
   - Why: 管理タスクの自動化
   - Dependencies: Phase 6
   - Risk: Medium

5. **組織ディレクトリ KB**（File: `services/control-plane/api/services/directory_kb.py`）
   - Action: 組織情報を Markdown で生成、全エージェントに自動注入
   - Why: 「誰に連絡すべき？」に回答
   - Dependencies: Phase 6
   - Risk: Low

**完了基準:**
- 公開 URL を別ブラウザで開き、ログインなしで該当従業員の AI と対話できる
- 公開エンドポイントのレート制限: 1 IP あたり 10 req/min, 1 トークンあたり 60 req/hour で動作確認
- ツイン専用ワークスペースが従業員メモリへ書き戻しゼロ（ログ検証）
- 従業員がトグルを OFF にすると 60 秒以内に URL が無効化
- 公開エンドポイントのレスポンス成功率 99%+（負荷試験 100 req）

---

### Phase 10: テスト・ドキュメント・デプロイ自動化（複雑度: Medium / 工数: 2週間 継続）

1. **e2e スイート**（File: `tests/e2e/`）
   - Action: Playwright で Portal / Admin Console、k6 で API 負荷
   - Why: ユーザージャーニー保証
   - Dependencies: Phase 6
   - Risk: Medium

2. **負荷試験**（File: `tests/load/`）
   - Action: 50 / 250 / 500 同時ユーザーシナリオ（最終目標 500 を上限とした段階的負荷試験）
   - Why: スケール検証
   - Dependencies: Phase 9
   - Risk: Medium

3. **セキュリティ静的解析**（File: `.github/workflows/security.yml`）
   - Action: Semgrep, Trivy, Checkov, gitleaks
   - Why: OWASP Top 10、IaC ミス検出
   - Dependencies: なし
   - Risk: Low

3a. **audit-events スキーマ互換性テスト**（File: `packages/audit-events/tests/test_schema_compat.py`）
   - Action: 本体テーブルと MCP Gateway テーブルの両方に `AuditRepository(tableName)` で同一スキーマ書込みを確認、Pydantic スキーマの破壊的変更を CI で検出（過去 N バージョンとの後方互換性チェック）
   - Why: 二系統書込みでスキーマがドリフトすると監査検索 / Athena クエリが壊れる
   - Dependencies: Phase 1 監査一次系、Phase 5 MCP 監査
   - Risk: Low

4. **`deploy.sh` 一発デプロイ**（File: `scripts/deploy.sh`）
   - Action: `terraform apply` → コンテナビルド・push → AgentCore Runtime 登録 → シード → 起動確認
   - Why: 30 分以内デプロイ要件
   - Dependencies: 全フェーズ
   - Risk: Medium

5. **ドキュメント整備**（File: `docs/`）
   - Action: アーキテクチャ図 / 運用ガイド / トラブルシュート / セキュリティドキュメント
   - Why: 運用引き継ぎ
   - Dependencies: 全フェーズ
   - Risk: Low

---

## 4. 依存関係マップ

### フェーズ間
```
Phase 1 (Infra) ───┬──→ Phase 2 (Gateway) ──┐
                   │                        ├──→ Phase 4 (AgentCore) ──┬─→ Phase 5 (Always-On) ─┐
                   ├──→ Phase 3 (Container)─┤                          │                        │
                   │                                                   ├─→ Phase 6 (Console) ───┼─→ Phase 7 (IM) ──┐
                   └──────────────────────────────────────────────────┘                         │      ↑           ├─→ Phase 9 (Twin)
                                                                                                │      │           │
                                                                                                │      └─ Step 4 のみ Phase 5 依存
                                                                                                ├─→ Phase 8 (Gov) ─┘
                                                                                                │
                                                                                                └────────────────────→ Phase 10
```

**横断依存（MCP Gateway サブシステムとの結合点）**:
- MCP Gateway Phase 4（Cognito JWT）← OpenClaw Phase 6（Cognito User Pool 構築）
- MCP Gateway Phase 6（PII フィルタ）← OpenClaw Phase 8（Bedrock Guardrails ティア定義）
- MCP Gateway Phase 7（Admin Console 統合）← OpenClaw Phase 6（Admin Console 基盤）

並行 2 名体制では OpenClaw Phase 6 が両者のクリティカルパスとなるためボトルネック化に注意。

### AWS サービス依存
- Bedrock AgentCore: us-east-1 / us-west-2 限定 → リージョン選定が全フェーズに影響
- DynamoDB / S3 / ECS Fargate / EFS / Cognito / Cloud Map: **AgentCore と同一リージョンに強制配置**（§ 5.4 リージョン戦略を参照、クロスリージョン構成は禁止）

### 外部 API 依存
| 機能 | 外部 API | フェーズ |
|------|---------|--------|
| Slack | Slack Web API / Events / Bolt SDK | 7 |
| Azure AD | Microsoft Graph / OIDC | 8 |

> **スコープ外:** Teams / Telegram / Discord / Feishu / WhatsApp など Slack 以外の IM プラットフォーム。

---

## 5. リスクと制約

### 技術リスク

| リスク | 影響 | 緩和策 |
|-------|------|--------|
| **OpenClaw バージョン非互換** | IM 統合崩壊 | Dockerfile で `2026.3.24` 固定、CI で検証 |
| **AgentCore リージョン制限** | Tokyo 等で使えない | 単一リージョン強制（§ 5.4）。AgentCore 対応リージョンに全データ配置できる組織のみが対象であることを README/契約で明示。EU/APAC データ主権要件があるテナントはスコープ外 |
| **Bedrock H2 Proxy のストリーミング** | レスポンス遅延 | FastAPI + `httpx[http2]` + `anyio`、p95 レイテンシ閾値 + バックプレッシャ試験を Phase 2 完了基準に必須化 |
| **DynamoDB ホットパーティション** | 書き込み制限 | パーティションキーに `ORG#` を含めない設計、suffix 分散 |
| **マルチテナント情報漏洩** | 重大インシデント | テナント ID 改ざんテストを CI 必須化 |
| **コールドスタート** | UX 低下 | Session Storage、必要なら Provisioned Concurrency |

### 組織リスク

| リスク | 緩和策 |
|-------|--------|
| 部門ごとに SOUL 運用ガバナンスが甘い | グローバル承認ワークフロー、変更は CISO/CTO 二段階 |
| 従業員が SOUL を編集して逸脱 | パーソナル層は上位を上書き不可、`CRITICAL IDENTITY OVERRIDE` ヘッダ強制 |
| Slack ワークスペース管理権限がない部門 | IT が単一の Slack App を発行・配布、Self-Service ペアリングのみ提供 |

### コストリスク

| 想定 | 500 ユーザー規模/月 |
|-------|--------------|
| Bedrock 推論（Nova 中心） | $200〜$400 |
| AgentCore microVM | $150〜$300 |
| ECS Fargate（常時稼働 5%） | $80〜$150 |
| DynamoDB On-Demand | $50〜$100 |
| S3 + CloudFront | $30〜$50 |
| **合計** | **約 $500〜$1,000 / 月** |

予算オーバー対策: 部門別予算アラート、従業員別月次上限、ティア降格ポリシー

### セキュリティリスク

- 公開デジタルツインのレート制限不足 → API Gateway throttling 必須
- 監査ログ改ざん → S3 Object Lock + KMS
- IAM 過剰権限 → IAM Access Analyzer を CI に組み込み

### 法的・コンプライアンス

- `sample/` のライセンス（AWS Samples / Apache 2.0 想定だが要確認） → コードはコピーせず参考のみ。本リポジトリのライセンスは MIT を予定
- **リージョン戦略（単一リージョン対応で完結）**:
  - 全コンポーネント（VPC / DynamoDB / S3 / AgentCore / ECS / Cognito / Cloud Map）を **単一リージョンに強制配置**する。`region_override` などのクロスリージョン仕掛けは導入しない
  - AgentCore は us-east-1 / us-west-2 のいずれかを選択。クロスリージョン構成は禁止（Workspace Assembler の S3 / DynamoDB 高頻度アクセスが p95 < 15s 要件と衝突するため）
  - 本プロジェクトは「AgentCore 利用可能リージョンに全データを配置できる組織」を対象とする。**EU / APAC のデータ主権要件があり当該リージョン内処理が必須のテナントは本プロジェクトのスコープ外**（別プロジェクトでマルチリージョン展開を検討）
- データ越境 → 当該テナントの全データ（ワークスペース・監査・KB）が単一リージョンに留まることを Terraform で強制（クロスリージョンレプリケーション・Global Table は禁止）

---

## 6. 複雑度・工数見積もり

| Phase | 複雑度 | 工数（人月） | 並行可能 |
|-------|--------|------------|----------|
| 1: 基盤インフラ + 監査一次系 | Medium | 1.0 | - |
| 2: ゲートウェイ + テナント漏洩テスト | High | 1.0 | Phase 3 と並行可 |
| 3: Agent Container | High | 1.5 | Phase 2 と並行可 |
| 4: AgentCore 統合 | High | 0.5 | - |
| 5: Always-On (Cloud Map) | Medium | 0.5 | Phase 6 と並行可 |
| **6a**: 管理コンソール API（FastAPI + DynamoDB Repository + RBAC + Auto-Provisioning + SOUL Editor + 監査検索 + 使用量集計） | High | 1.5 | Phase 5 と並行可 |
| **6b**: Admin Console + Portal フロントエンド（Next.js 15、13 ページ） | High | 1.5 | Phase 6a 完了後（6a で MCP Phase 4 結合解除） |
| 7: IM チャネル (Slack のみ) | Medium | 0.5 | Phase 5/6 完了後 |
| 8: ガバナンス | Medium | 0.75 | Phase 9 と並行可 |
| 9: デジタルツイン | Medium | 0.5 | Phase 8 と並行可 |
| 10: テスト・ドキュメント | Medium | 1.0（継続） | 全期間 |
| **合計** | - | **約 10.25 人月** | 2〜3 名で 4〜5 ヶ月 |

> 旧見積もり（7.0 人月）は Phase 1・3・6 が楽観的だったため改定。Phase 6a はクリティカルパス（MCP Gateway Phase 4 着手の前提）のため最優先で進めること。Phase 6b は UI 経験者を 1 名以上確保すること。

> **6a/6b 分割の効果**: Phase 6a 完了時点で全機能を curl / Postman で検証可能。MCP Gateway Phase 4 (Cognito JWT) を 6a 完了直後に着手できるため、並行効率が向上。

---

## 7. モノレポ構成案

```
enterprise-ai-platform/
├── apps/                                  # ユーザー向けフロントエンド
│   ├── admin-console/                    # Next.js 15 (IT/部門管理者)
│   └── portal/                           # Next.js 15 (従業員)
│
├── services/                              # バックエンドサービス
│   ├── control-plane/
│   │   └── api/                          # FastAPI (組織CRUD/SOUL/監査/使用量)
│   ├── gateway/
│   │   ├── tenant-router/                # FastAPI
│   │   └── bedrock-proxy/                # FastAPI + httpx + h2 (Python 統一)
│   ├── agent-container/                  # Python + OpenClaw コンテナ
│   │   ├── src/
│   │   │   ├── workspace/
│   │   │   ├── skills/
│   │   │   ├── permissions/
│   │   │   ├── session/
│   │   │   ├── memory/
│   │   │   ├── observability/
│   │   │   └── safety/
│   │   ├── Dockerfile
│   │   └── entrypoint.sh
│   ├── exec-agent/                       # Executive ティア専用イメージ
│   └── im-adapter/                       # IM アダプタ (Slack のみ)
│       ├── core/                         # 共通インタフェース (将来拡張用の薄い抽象)
│       └── slack/                        # Slack Bolt SDK ベース実装
│
├── packages/                              # 共有ライブラリ
│   ├── shared-types/                     # TypeScript 型定義
│   ├── shared-python/                    # Python 共通ユーティリティ
│   ├── soul-schema/                      # SOUL.md スキーマ + バリデータ
│   └── audit-events/                     # 監査イベント型・列挙 + AuditRepository(tableName) 抽象
│                                         #   本体は `enterprise-ai-platform-{env}` テーブル、
│                                         #   MCP Gateway は `enterprise-ai-platform-mcp-gw-{env}` テーブルへ書き込み（テーブル名注入）。
│                                         #   PITR/TTL 設定差異は Terraform 側で管理
│
├── infra/                                 # IaC
│   ├── terraform/
│   │   ├── envs/
│   │   │   ├── dev/
│   │   │   ├── stg/
│   │   │   └── prod/
│   │   └── modules/
│   │       ├── network/
│   │       ├── iam/
│   │       ├── dynamodb/
│   │       ├── storage/
│   │       ├── secrets/
│   │       ├── bedrock/
│   │       ├── agentcore/
│   │       ├── fargate-tier/
│   │       ├── efs/
│   │       ├── ecr/
│   │       ├── gateway/
│   │       ├── audit-archive/
│   │       └── guardrails/
│   └── scripts/
│       └── runtime/
│
├── tests/
│   ├── e2e/                              # Playwright + k6
│   ├── load/
│   └── security/
│
├── scripts/
│   ├── deploy.sh                         # 一発デプロイ
│   └── seed/                             # 組織データシード
│
├── docs/
│   ├── architecture/
│   ├── runbooks/
│   ├── security/
│   └── api/
│
├── .github/
│   └── workflows/                        # CI/CD
│
├── package.json
├── pnpm-workspace.yaml
├── turbo.json
├── .editorconfig
├── README.md
└── CLAUDE.md                             # AI 開発ガイド
```

---

## 8. 技術スタック

### 言語

| 用途 | 言語 | バージョン | 理由 |
|------|------|----------|------|
| フロントエンド | TypeScript | 5.6+ | 型安全 |
| 全バックエンドサービス (Control Plane / Tenant Router / Bedrock H2 Proxy / Agent Container) | Python | 3.12 | OpenClaw / boto3 連携、SigV4・監査スキーマ・共通ライブラリの単一メンテ |
| IaC | HCL (Terraform) | 1.10+ | モジュール化容易 |

> Bedrock H2 Proxy は当初 Node.js / Hono を検討したが、`httpx` + `h2` + `anyio` で Python でも同等の HTTP/2 ストリーミング性能が達成できるため、運用負債削減のため Python に統一。CI の静的解析（Ruff）も単一系統で済む。
>
> 決定の経緯と検討した代替案は [ADR 0001: バックエンドサービスを Python 3.12 に統一する](adr/0001-python-unified-backend.md) を参照。

### フレームワーク

| 層 | 採用 | 理由 |
|----|------|------|
| Web フロント | Next.js 15 (App Router) + React 19 + Tailwind 4 + shadcn/ui | エンタープライズ標準、SSR + RSC |
| バックエンド API | FastAPI + Pydantic v2 | 型安全 + OpenAPI 自動 |
| ストリーミングプロキシ | FastAPI + `httpx` + `h2` + `anyio` | Python 統一、HTTP/2 + SigV4 ストリーミング対応 |
| IM SDK | Slack Bolt SDK (Python) | 公式推奨、本プロジェクトは Slack のみサポート |

### AWS サービス

| カテゴリ | サービス |
|--------|---------|
| Compute | ECS Fargate, AWS Bedrock AgentCore, Lambda |
| Container | ECR |
| Network | VPC, ALB, CloudFront, WAF, API Gateway |
| Storage | S3, EFS |
| Database | DynamoDB（シングルテーブル） |
| Search | OpenSearch（監査全文検索 - Phase 8） |
| AI | Bedrock (Nova / Claude / DeepSeek) + Bedrock Guardrails + Knowledge Bases |
| Identity | Cognito (MVP) → Azure AD/SAML (Phase 8) |
| Secrets | SSM Parameter Store + Secrets Manager + KMS |
| Observability | CloudWatch Logs/Metrics, X-Ray, EventBridge |
| Streams | DynamoDB Streams + Firehose |

### データストア戦略

| データ | ストア | 形式 |
|------|------|------|
| 組織・割当・ルーティング・マッピング・監査 | DynamoDB | シングルテーブル `PK/SK + GSI1〜GSI3` |
| ワークスペース・KB・スキル・SOULテンプレ | S3 | プレフィックスで分離 |
| 監査長期保管 | S3 (Object Lock) | Athena 検索 |
| Runtime ID / トークン | SSM Parameter Store | SecureString |
| Bot トークン / DB 認証 | Secrets Manager | ローテーション対応 |
| 監査全文検索（拡張） | OpenSearch | Firehose 経由 |

### CI/CD

| 用途 | ツール |
|------|------|
| パイプライン | GitHub Actions |
| AWS 認証 | OIDC（IAM Role） |
| IaC | Terraform Cloud（state） / Atlantis（plan PR） |
| コンテナビルド | Buildx (ARM64 cross-build) |
| 静的解析 | Ruff (Python 統一) / Biome (TS) / Trivy / Semgrep / Checkov / gitleaks |
| テスト | pytest / Vitest / Playwright / k6 |
| カバレッジ | Codecov（80% gate） |

### モニタリング・運用

- CloudWatch ダッシュボード（フェーズ別）
- X-Ray 分散トレーシング
- Sentry（フロントエンドエラー）
- PagerDuty 連携（CRITICAL アラート）

---

## 補足: MVP からの拡張ロードマップ

| マイルストーン | 含むフェーズ | 提供価値 |
|--------------|-----------|---------|
| **MVP (M1)** | Phase 1〜4 + Phase 6a（最小実装） | 50 ユーザー、Portal チャット、Standard ティアのみ。Phase 6a 完了で curl/Postman 検証可、Phase 6b（UI）は M2 で追加 |
| **M2** | + Phase 5, 7（Slack） | 常時稼働、Slack 連携、200〜300 ユーザー |
| **M3** | + Phase 8, 9 | フルガバナンス、デジタルツイン、Azure AD、500 ユーザー対応 |
| **M4** | + Phase 10 強化 | 500 ユーザー、SOC2 準拠基盤、運用安定化 |

---

## 関連ファイル

参照した参考リポジトリのファイル（**コピー禁止、設計参考のみ**）:

- `sample/README_ENTERPRISE_JP.md`
- `sample/enterprise/README.md`
- `sample/enterprise/agent-container/server.py`
- `sample/enterprise/agent-container/workspace_assembler.py`
- `sample/enterprise/gateway/tenant_router.py`

新規作成予定のリポジトリルート:
- `enterprise-ai-platform/`
