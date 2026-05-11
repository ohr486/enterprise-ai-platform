# 実装プラン: Enterprise MCP Gateway Server

## 概要

社内のクライアント（Claude Desktop / Claude Code / 自社製エージェント / `enterprise-ai-platform` の Agent Container）から、単一のエンドポイントを介して複数の上流 MCP サーバー（GitHub / Jira / Slack / 社内 DB など）へアクセス可能にする MCP アグリゲーター・ゲートウェイ。Gateway 自身が MCP サーバーとして振る舞い、上流 MCP の `tools` / `resources` / `prompts` を名前空間化して集約・配信する。既存の `enterprise-ai-platform` とは独立サブシステムとして配置しつつ、認証基盤・監査基盤・IaC を共有する。

## 1. 要件の再記述

### 1.1 ターゲット利用者

- 社内エンジニア (Claude Desktop / Claude Code / Cursor)
- AI エージェント (Agent Container, 社内エージェント)
- 業務システム (RPA / バッチ / Workflow)

### 1.2 想定規模

| 指標 | MVP | 最終目標 |
|---|---|---|
| 同時接続数 | 50 | 500 |
| 上流 MCP サーバー数 | 2〜5 | 20〜30 |
| ツール総数 | 30〜80 | 300〜600 |

### 1.3 セキュリティ要件

- AuthN: API キー (HMAC) → JWT (Cognito) → Azure AD/OIDC
- AuthZ: ユーザー × 上流 × ツールの3軸 ACL
- 監査: 全 RPC を DynamoDB `AUDIT#mcp-gw#`
- PII フィルタ: Bedrock Guardrails 連携
- クレデンシャル: Secrets Manager に集中

### 1.4 運用要件

- 可用性 99.9% (Multi-AZ)
- `tools/list` p95 < 200ms
- `tools/call` 透過オーバーヘッド p95 < 50ms

### 1.5 スコープ外

- Gateway 独自ツール、MCP プロトコル拡張
- 組織間マルチテナント分離
- オンプレ逆プロキシ、クライアント SDK
- REST↔MCP 変換

## 2. アーキテクチャ概要

```
Clients (Claude Desktop / Code, Agent Container, 社内システム)
   │ Streamable HTTP (MCP)
   ↓
Edge: ALB (TLS 1.3) + WAF
   ↓
MCP Gateway (ECS Fargate, ARM64 Graviton)
  ├─ Inbound: Streamable HTTP / stdio (dev) / JSON-RPC dispatcher
  ├─ Auth & Policy: AuthN, AuthZ (3軸 ACL), Rate Limit, Audit
  ├─ Aggregation: Namespace, Tool/Resource/Prompt Registry, Router
  └─ Upstream Pool: stdio / HTTP+SSE / Streamable HTTP, Circuit Breaker
   │
   ├──→ GitHub MCP / Jira MCP / Slack MCP / 社内DB MCP / ...
   │
Shared Infra: DynamoDB / S3 / Secrets Manager / SSM / Cognito / CloudWatch
```

### 2.1 トランスポート対応

- Streamable HTTP: 本番標準 (Inbound/Upstream 両対応)
- HTTP+SSE: 上流互換のみ
- stdio: ローカル開発と同一コンテナ内サブプロセス上流のみ

### 2.2 名前空間化

`{upstreamId}__{originalName}` 形式。例: `github__create_issue`。重複登録は起動時 Fail Fast。

### 2.3 ルーティングフロー

1. Inbound 受信 → 2. Namespace で upstreamId 抽出 → 3. AuthZ 検査 → 4. Rate Limit → 5. 接続プール取得 → 6. 上流転送 → 7. ストリーミング中継 → 8. 監査記録 → 9. 応答

### 2.4 エラーコード規約

| エラー | JSON-RPC code |
|---|---|
| 上流タイムアウト | -32001 |
| 認証失敗 | -32002 |
| 認可拒否 | -32003 |
| レート制限 | -32004 |
| Guardrails ブロック | -32005 |

### 2.5 設定管理

- 静的: YAML on S3 (上流定義)
- 動的: DynamoDB (ACL, レート制限)
- シークレット: Secrets Manager

## 3. 段階的実装フェーズ

### Phase 1: MCP プロトコル基本実装 (High / 1.5〜2 週)

1. プロジェクト雛形 (`pyproject.toml`, `Dockerfile`)
2. MCP サーバー骨格 (`src/server/app.py`)
3. JSON-RPC ディスパッチャ (`src/server/dispatcher.py`)
4. stdio トランスポート (開発用)
5. 構造化ロギング基盤
6. 単体テスト

完了基準: Claude Desktop で `initialize` 成功、空の `tools/list` 応答。

**Walking Skeleton マイルストーン**: Phase 1 完了時点で実 Claude Desktop / Claude Code に Gateway を登録し、`initialize` → `tools/list`（空配列）→ `ping` の往復が成立することを確認。プロトコル仕様準拠リスクを最早期に検出する。

### Phase 2: 上流接続層 (High / 2 週)

1. `IUpstreamClient` 抽象
2. Streamable HTTP クライアント（MVP 必須）
3. HTTP+SSE クライアント (legacy)（MVP 必須）
4. stdio サブプロセスクライアント (Risk: High) — **MVP オプション、リモート上流のみで開始可**
5. 接続プール
6. 設定ローダー (Pydantic v2)
7. 統合テスト (Mock 上流)

完了基準: Streamable HTTP 上流で `list_tools()` 成功（MVP 必須）。stdio 上流は MVP では延期可能（プロセスリーク・cgroup OOM・ファイルディスクリプタ枯渇のリスクを考慮し、Phase 6 の Multi-AZ 完了後に再着手することも検討）。

### Phase 3: 名前空間化と集約 (Medium / 1.5 週)

1. Namespace Manager
2. Tool Registry (TTL 30s キャッシュ)
3. Resource / Prompt Registry
4. Request Router
5. ストリーミング応答中継 (Risk: High)
6. 集約テスト

完了基準: 複数上流のツールが集約された `tools/list` が返り、`tools/call` が正しい上流に届く。

### Phase 4: 認証・認可 (High / 2 週)

1. API キー認証 (HMAC + DynamoDB)
2. API キー発行 API
3. ACL モデル (3軸: user × upstream × tool, Risk: High)
4. AuthZ ミドルウェア
5. JWT 認証 (Cognito 統合)
6. シークレット注入 (Risk: High)
7. AuthN/AuthZ テスト

完了基準: API キーごとに見える上流・ツールを ACL で絞れる。

### Phase 5: 監査・観測性 (Medium / 1〜1.5 週)

1. 監査イベントスキーマ (`packages/audit-events/`)
2. 監査ロガー (DynamoDB `AUDIT#mcp-gw#`)
3. CloudWatch メトリクス (EMF)
4. X-Ray 分散トレーシング
5. ヘルスチェック (`/healthz`, `/healthz/upstreams`)
6. CloudWatch ダッシュボード

完了基準: 上流別 RPS とエラー率がダッシュボードで確認可能。

### Phase 6: 運用機能 (Medium / 1.5 週)

1. トークンバケット型レート制限
2. Circuit Breaker (Closed/Open/Half-Open)
3. PII フィルタ (Bedrock Guardrails 連携, Risk: High)
4. Multi-AZ ECS Fargate デプロイ
5. Graceful Shutdown

完了基準: 上流 1 個停止しても他は応答継続、Burst が 429 でブロック。

### Phase 7: 設定管理 UI / 動的リロード (Medium / 1.5 週)

1. 設定 API (`/admin/upstreams`, `/admin/acls`, `/admin/api-keys`)
2. 動的リロード (`CONFIG#mcp-gw#version` ポーリング 30s)
3. Admin Console 統合 (`apps/admin-console/app/mcp-gateway/`)
4. 設定変更監査

完了基準: UI から上流追加 → 30 秒以内にクライアントの `tools/list` に反映。

### Phase 8: テスト・デプロイ (Medium / 1〜1.5 週 継続)

1. MCP プロトコル準拠テスト (MCP Inspector)
2. e2e スイート (Claude Desktop 相当)
3. 負荷試験 (k6, 50/250/500 同時接続)
4. セキュリティ静的解析 (Semgrep, Trivy, gitleaks)
5. `deploy.sh` 拡張
6. 運用ドキュメント

完了基準: e2e 全パス、500 同時接続で p95 < 200ms。

## 4. 依存関係・外部依存

### 4.1 MCP SDK

採用: **Python `mcp` 公式 SDK**。理由は `enterprise-ai-platform` 主要バックエンドと言語統一、Agent Container と運用ノウハウ共有可能。

### 4.2 既存 enterprise-ai-platform との関係

独立サブシステムだが共通基盤を共有:

| 共有 | 方法 |
|---|---|
| VPC / IAM / S3 / Secrets / Cognito | 同一 Terraform ステート |
| **DynamoDB（テーブル分離）** | **MCP Gateway 用に専用テーブル `enterprise-ai-platform-mcp-gw-{env}` を作成**。本体側との I/O 競合・容量設定干渉・PITR コスト結合・DDL 変更影響を回避するため、`PK="MCP#"` 名前空間分離ではなくテーブル分離方式に変更 |
| Admin Console | 既存 Next.js アプリに `app/mcp-gateway/` を追加 |
| 監査イベント | `packages/audit-events/` 拡張（ただし MCP Gateway は独自テーブルに書き込み、S3 長期保管プレフィックスのみ共有） |
| CI/CD | 同一 GitHub Actions / Turborepo |

独立部分: ECS Fargate サービス、ALB ターゲットグループ、**DynamoDB テーブル**。

**横断依存（OpenClaw 本体プランとの結合点）**:
| MCP Gateway Phase | 依存する OpenClaw Phase |
|---|---|
| Phase 4（Cognito JWT） | OpenClaw Phase 6（Cognito User Pool） |
| Phase 6（PII / Bedrock Guardrails） | OpenClaw Phase 8（Guardrails ティア定義） |
| Phase 7（Admin Console 統合） | OpenClaw Phase 6（Admin Console 基盤） |

並行 2 名体制では OpenClaw Phase 6 が両者のクリティカルパス。MCP Gateway 単独で Phase 7 へ進めない点に注意。

### 4.3 配置

`services/mcp-gateway/`、`infra/terraform/modules/mcp-gateway/`、`apps/admin-console/app/mcp-gateway/`、`docs/mcp-gateway/` を新規追加。

## 5. リスクと制約

### 5.1 技術リスク

| リスク | 緩和 |
|---|---|
| MCP プロトコル進化 | SDK バージョン固定、四半期アップグレード PR |
| 上流障害伝播 | Circuit Breaker、上流隔離、`tools/list` キャッシュ継続 |
| 名前空間衝突 | 起動時重複チェック、`__` 厳格検証 |
| 上流クレデンシャル漏洩 | Secrets Manager 一元化、IAM Access Analyzer |
| stdio サブプロセスリーク | プロセスマネージャ、定期再起動、メモリ上限監視 |
| ストリーミング中断 | `asyncio.CancelledError` 確実伝播、上流中断シグナル |
| `tools/list` 肥大化 | ACL ユーザー別絞り込み、上流別ページング |

### 5.2 運用リスク

- ACL 設定漏れ → デフォルト deny
- API キー流通 → 90 日有効期限 + ローテーション
- 監査ログ容量 → DynamoDB TTL (90 日) + S3 Object Lock

### 5.3 コスト見積もり

| 項目 | MVP/月 | 500 ユーザー/月 |
|---|---|---|
| ECS Fargate (2 AZ × 2) | $40 | $80 |
| ALB | $20 | $25 |
| DynamoDB | $10 | $50 |
| S3 + CloudFront | $5 | $20 |
| Secrets Manager | $5 | $15 |
| ElastiCache (任意) | $0 | $30 |
| **合計** | **$80** | **$220** |

### 5.4 法的・コンプライアンス

- 上流クラウド MCP 利用時のデータ越境 → 法務確認
- MCP プロトコル MIT、SDK MIT、ライセンス問題なし

## 6. 複雑度・工数見積もり

| Phase | 複雑度 | 人月 |
|---|---|---|
| 1: MCP 基本 | High | 0.5 |
| 2: 上流接続 | High | 0.5 |
| 3: 名前空間集約 | Medium | 0.4 |
| 4: 認証認可 | High | 0.5 |
| 5: 監査観測 | Medium | 0.3 |
| 6: 運用 | Medium | 0.4 |
| 7: 設定 UI | Medium | 0.4 |
| 8: テスト | Medium | 0.3 |
| **合計** | - | **3.3** |

`enterprise-ai-platform` 本体 10.0 人月 + 本サブシステム 3.3 人月 = **約 13.3 人月**、2〜3 名で 5〜6 ヶ月。

## 7. 技術スタック

### 7.1 言語

- **Python 3.12**: 公式 MCP SDK 成熟、既存基盤統一

### 7.2 フレームワーク

- MCP プロトコル: `mcp` 公式 Python SDK
- HTTP: FastAPI + Pydantic v2
- 非同期 I/O: `asyncio` + `httpx`
- ロギング: `structlog`
- テスト: `pytest` + `pytest-asyncio` + `pytest-cov`

### 7.3 設定形式

- 静的: YAML
- 動的: DynamoDB
- シークレット: Secrets Manager

### 7.4 デプロイ

**ECS Fargate (ARM64 Graviton)** Multi-AZ。EKS / Lambda / EC2 / オンプレは不採用。

### 7.5 既存統合

- AWS インフラ完全統合
- Phase 4 で Cognito、Phase 8 で Azure AD
- Admin Console に 1 ページ追加
- 監査基盤再利用、ECS は別タスク

## 8. ディレクトリ構成案

```
services/mcp-gateway/
├── pyproject.toml
├── Dockerfile
├── README.md
├── config/
│   ├── upstreams.example.yaml
│   └── defaults.yaml
├── src/mcp_gateway/
│   ├── __init__.py
│   ├── __main__.py
│   ├── server/      (app, dispatcher, stdio, lifecycle)
│   ├── upstream/    (client, streamable_http, http_sse, stdio, pool, circuit_breaker)
│   ├── aggregation/ (namespace, tool_registry, resource_registry, prompt_registry, router, streaming)
│   ├── auth/        (api_key, jwt, acl, middleware, upstream_creds)
│   ├── limits/      (rate_limit)
│   ├── safety/      (filter)
│   ├── observability/ (logger, audit, metrics, tracing, health)
│   ├── admin/       (api_key, config_api, audit_hook)
│   └── config/      (loader, schema, refresh)
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── protocol/
│   └── fixtures/mock_upstream/
└── scripts/
    ├── dev_run.sh
    └── seed_acl.py
```

## 拡張ロードマップ

| マイルストーン | 内容 | 提供価値 |
|---|---|---|
| **M1 MVP** | Phase 1〜3 + Phase 4 (API キー) + Phase 5 最小 | Claude Desktop から 2〜3 個の上流 MCP を利用 |
| **M2** | + Phase 4 (JWT) + Phase 5 全 + Phase 6 | 監査/メトリクス、Multi-AZ、PII、200 ユーザー |
| **M3** | + Phase 7 + Phase 8 | 動的リロード、管理 UI、500 ユーザー、e2e/負荷試験 |

## 関連ファイル

- 参照: `docs/OPENCLAW_PLATFORM_PLAN.md` (整合性の基準)
- 新規予定: `services/mcp-gateway/`、`infra/terraform/modules/mcp-gateway/`、`apps/admin-console/app/mcp-gateway/`、`docs/mcp-gateway/`
