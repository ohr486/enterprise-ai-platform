# MCP Gateway サブシステム 要件定義書

- **ステータス**: Draft (要承認)
- **起票日**: 2026-05-13
- **最終更新**: 2026-05-13
- **対象**: enterprise-ai-platform に同居する **MCP アグリゲーター・ゲートウェイ** サブシステム
- **関連ドキュメント**:
  - [MCP_GATEWAY_PLAN.md](./MCP_GATEWAY_PLAN.md)（実装プラン、8 フェーズ）
  - [OPENCLAW_PLATFORM_REQUIREMENTS.md](./OPENCLAW_PLATFORM_REQUIREMENTS.md)（本体要件定義）
  - [OPENCLAW_PLATFORM_PLAN.md](./OPENCLAW_PLATFORM_PLAN.md)（本体実装プラン）
  - [KNOWLEDGE_DB_REQUIREMENTS.md](./KNOWLEDGE_DB_REQUIREMENTS.md)（横断サブシステム）
  - [ADR 0001: Python 統一バックエンド](./adr/0001-python-unified-backend.md)

---

## 0. ドキュメント概要

### 0.1 目的

本ドキュメントは、`enterprise-ai-platform` の独立サブシステムである **MCP Gateway**（MCP アグリゲーター・ゲートウェイ）の **要件**（What / Why / 制約 / 受け入れ基準）を定義する。

実装プラン（How）・コード・Terraform スニペットは `docs/MCP_GATEWAY_PLAN.md` に分離する。本書は決定事項・制約条件・受け入れ基準・Open Questions を要件レベルで集約し、ユーザーの意思決定を最短経路で取れる粒度に整理することを目的とする。

### 0.2 想定読者

- プロダクトオーナー / アーキテクト（スコープ承認・意思決定）
- セキュリティ / コンプライアンス担当（3 軸 ACL・上流クレデンシャル・データ越境）
- 後続フェーズの実装プラン執筆者
- 運用 / SRE（観測性・コスト・SLO・上流障害ハンドリング）
- 社内エンジニア（Claude Desktop / Code 利用者）

### 0.3 関連ドキュメント

- 実装プラン（8 フェーズ、約 3.3 人月）: `docs/MCP_GATEWAY_PLAN.md`
- 本体要件定義: `docs/OPENCLAW_PLATFORM_REQUIREMENTS.md`
- 本体実装プラン（10 フェーズ）: `docs/OPENCLAW_PLATFORM_PLAN.md`
- KDB 要件定義（横断サブシステム）: `docs/KNOWLEDGE_DB_REQUIREMENTS.md`
- ADR 0001: バックエンドサービスを Python 3.12 に統一する

### 0.4 用語定義

| 用語 | 定義 |
|---|---|
| MCP | Model Context Protocol。Anthropic 提唱のクライアント ⇄ サーバー間プロトコル |
| MCP Gateway | 本書で要件定義するサブシステム名。複数の上流 MCP サーバーを集約し、単一エンドポイントで配信 |
| 上流 MCP サーバー（Upstream） | Gateway がプロキシする実 MCP サーバー（GitHub / Jira / Slack / 社内 DB 等） |
| Inbound / Downstream | Gateway → クライアント（Claude Desktop / Code / Agent Container）方向 |
| Upstream | Gateway → 上流 MCP サーバー方向 |
| 名前空間（Namespace） | `{upstreamId}__{originalName}` 形式でツール / リソース / プロンプトを一意化 |
| 3 軸 ACL | user × upstream × tool の 3 軸で許可判定を行うアクセス制御 |
| Streamable HTTP | MCP 標準の HTTP ストリーミングトランスポート（本番標準） |
| HTTP+SSE | MCP レガシートランスポート（上流互換のみサポート） |
| stdio | プロセス間通信。ローカル開発と同一コンテナ内サブプロセス上流のみサポート |
| Circuit Breaker | 上流障害伝播を防ぐ Closed/Open/Half-Open 状態機械 |
| AuditRepository(tableName) | `packages/audit-events/` の抽象。本体 / MCP / KDB の 3 系統テーブルに同一スキーマで書き込む |

---

## 1. 背景と目的

### 1.1 背景

社内では複数の MCP サーバー（GitHub MCP / Jira MCP / Slack MCP / 社内 DB MCP 等）が並列に提供される見通しがある。一方、クライアント側（Claude Desktop / Claude Code / `enterprise-ai-platform` の Agent Container / 社内エージェント）が個別に各上流 MCP を直接接続する構成には以下の課題がある:

| 課題 | 影響 |
|---|---|
| クライアント設定の散在 | 利用者ごとに N 個の接続設定をメンテ。新規上流追加でクライアント全配布が必要 |
| 認証情報の分散 | 各クライアントに上流クレデンシャルを配布する必要があり漏洩リスク増 |
| 3 軸 ACL の欠如 | 「誰が・どの上流の・どのツール」を呼べるかを一元管理できない |
| 監査の不在 | 上流ごとにログ形式・場所が異なり、横断検索不可 |
| 障害伝播 | 1 つの上流障害がクライアント全体のハングを誘発 |
| 名前衝突 | `create_issue` などツール名が複数上流で重複 |
| レート制限の分散 | 各上流個別に制限がかかり、全社一貫した利用統制が困難 |

`docs/OPENCLAW_PLATFORM_REQUIREMENTS.md` で定義する OPENCLAW プラットフォーム本体は、上記課題の影響を強く受ける（Agent Container がクライアントの 1 つ）。共通インフラ（VPC / Cognito / 監査基盤）を共有しつつ、独立サブシステムとして MCP Gateway を配置することで、本体スコープを膨張させずに横断課題を解く。

### 1.2 目的

以下の 5 点を満たすサブシステムを定義する:

1. **単一エンドポイント集約**: 複数の上流 MCP サーバーを単一の MCP エンドポイントで配信
2. **名前空間化**: `{upstreamId}__{toolName}` で衝突回避し、起動時に Fail Fast 検証
3. **3 軸 ACL**: user × upstream × tool で許可判定、デフォルト deny
4. **監査の一元化**: 全 RPC を `AUDIT#mcp-gw#` で永続化、Athena 経由で横断検索
5. **障害分離**: 上流ごとの Circuit Breaker / 接続プール / レート制限で 1 上流障害が全体に伝播しない

### 1.3 既存類似サービスとの差分

| 観点 | 単純な MCP リバースプロキシ | **本サブシステム（MCP Gateway）** |
|---|---|---|
| 名前空間 | なし（上流のツール名そのまま） | `{upstreamId}__{toolName}` で集約、起動時 Fail Fast |
| ACL | なし / IP 単位 | **3 軸 ACL**（user × upstream × tool） |
| 認証 | Pass-through | **API キー（HMAC） / JWT（Cognito） / Azure AD/OIDC** の段階導入 |
| 監査 | アクセスログのみ | **全 RPC の DynamoDB + S3 Object Lock 監査**、Athena 横断検索 |
| 障害分離 | なし | **Circuit Breaker / 接続プール / レート制限** |
| 上流クレデンシャル | クライアント保持 | **Secrets Manager 一元化** |
| PII フィルタ | なし | **Bedrock Guardrails 連携**（M2 以降） |
| 動的リロード | 再起動必須 | **`/admin/upstreams` API + DynamoDB ポーリング 30s** |

### 1.4 横断サブシステムとの関係

| サブシステム | 関係 |
|---|---|
| OPENCLAW 本体 | Agent Container は **クライアント側として利用しない**（本体は独自にツールを実行）。共通インフラ（VPC / Cognito / Admin Console / 監査基盤）を共有 |
| KDB | KDB Service を **上流 MCP の 1 つ** として登録（`knowledge__search`, `knowledge__fetch`, `knowledge__list_kbs`）。MCP Gateway は粗フィルタ、KDB 内 ACL が精密フィルタ |

### 1.5 ユースケース駆動の必要性

「単一エンドポイント集約」が必要な理由:

1. **クライアント設定の一元化**: 新規上流追加時、Gateway 側で完結（クライアント側の設定変更不要）
2. **クレデンシャル漏洩リスク低減**: 上流トークンは Gateway の Secrets Manager にのみ存在
3. **3 軸 ACL の一貫適用**: 「Manager だけが `jira__create_ticket` を呼べる」「全員に `github__search_code` を許可」など細粒度ポリシーを一元管理
4. **監査の横断検索**: 「誰が・いつ・どの上流の・どのツールを呼んだか」を Athena で一元検索

### 1.6 ビジネス価値（KPI 候補）

| KPI | 測定方法 | 目標値（500 ユーザー時） |
|---|---|---|
| 月間 MCP RPC 数 | 監査ログから集計 | 100,000+（KPI、上限ではない） |
| `tools/list` p95 | CloudWatch | < 200ms |
| `tools/call` 透過オーバーヘッド p95 | CloudWatch | < 50ms |
| 上流障害時の他上流成功率 | カオステスト | 99%+ |
| 上流クレデンシャル漏洩インシデント | セキュリティレビュー | **0 件**（必須） |
| 名前空間衝突 | 起動ログ | **0 件**（起動時 Fail Fast） |
| 監査ログ書き込み成功率 | 監査メトリクス | 99.9%+ |
| 月次 Active クライアント数（Claude Desktop / Code 等） | 監査ログ | 100+ |

---

## 2. ユースケース / アクター

### 2.1 主要アクター

| アクター | 役割 | MCP Gateway との関わり |
|---|---|---|
| 社内エンジニア | Claude Desktop / Claude Code / Cursor で MCP 経由のツール利用 | 個別 API キー / JWT で Gateway へ接続、3 軸 ACL で許可された範囲を利用 |
| AI エージェント | Agent Container / 社内エージェント | プログラマティックに Gateway へ接続、自動化ワークフロー実行 |
| 業務システム | RPA / バッチ / Workflow | サービスアカウントの API キーで Gateway へ接続 |
| IT 管理者 | 上流 MCP の登録 / クレデンシャル管理 / ACL 設定 | Admin Console から上流定義 / API キー発行 / ACL 編集 |
| 監査担当 | コンプライアンス検証 | Admin Console + Athena で「誰が何の RPC を呼んだか」を抽出 |
| KDB Service | 上流 MCP の 1 つ | KDB を上流登録、`knowledge__*` を公開 |

### 2.2 主要ユースケース（合計 10 件）

| # | ユースケース | アクター | 主経路 |
|---|---|---|---|
| UC-01 | Claude Desktop からの GitHub Issue 作成 | エンジニア | Claude Desktop → Gateway → GitHub MCP |
| UC-02 | Claude Code からの社内 DB クエリ | エンジニア | Claude Code → Gateway → 社内 DB MCP |
| UC-03 | Agent Container からの Jira ステータス更新 | AI エージェント | Agent Container → Gateway → Jira MCP |
| UC-04 | KDB 検索の MCP 経由呼び出し | エンジニア（Claude Desktop） | Claude Desktop → Gateway → KDB MCP |
| UC-05 | 複数上流の `tools/list` 統合表示 | エンジニア | Gateway が複数上流から集約、名前空間付きで返却 |
| UC-06 | 上流障害時の他上流継続利用 | 任意のクライアント | Circuit Breaker が障害上流を Open、他上流は継続 |
| UC-07 | 上流追加 / 設定変更の動的反映 | IT 管理者 | Admin Console → `/admin/upstreams` → 30 秒以内に全クライアントへ伝播 |
| UC-08 | API キー発行 / 失効 | IT 管理者 | Admin Console → `/admin/api-keys` |
| UC-09 | ACL 設定（user × upstream × tool） | IT 管理者 | Admin Console → `/admin/acls` |
| UC-10 | 監査検索（特定ユーザーの過去 24h RPC） | 監査担当 | Admin Console → DynamoDB GSI（1 秒以内）or Athena |

### 2.3 非ユースケース（明示的スコープ外）

- **Gateway 独自ツール**: Gateway 自身がツールを定義・提供しない（純粋なアグリゲーター）
- **MCP プロトコル拡張**: Anthropic 仕様への独自拡張は行わない
- **組織間マルチテナント分離**: 単一組織前提（OPENCLAW 本体と同方針）
- **オンプレミス逆プロキシ**: AWS 上の Gateway のみ。オンプレ MCP のリレーは対象外
- **クライアント SDK の提供**: クライアントは MCP 標準 SDK で接続（Gateway 専用 SDK を作らない）
- **REST ↔ MCP 変換**: REST API を MCP として露出する変換層は対象外
- **GraphQL / gRPC 上流**: MCP プロトコル準拠の上流のみサポート
- **クライアント向け UI**: Gateway は API のみ。Admin Console は管理者向け
- **Slack 以外の IM 通知**: 上流アラート / オンコール通知は CloudWatch + PagerDuty のみ

---

## 3. 機能要件

### 3.1 MCP プロトコル対応

| 要件 ID | 要件 | 決定 / 未決定 |
|---|---|---|
| FR-PROTO-01 | MCP プロトコル準拠（公式仕様、Anthropic 提唱版） | 決定 |
| FR-PROTO-02 | `initialize` / `tools/list` / `tools/call` / `resources/*` / `prompts/*` / `ping` の RPC をサポート | 決定 |
| FR-PROTO-03 | 公式 Python `mcp` SDK を採用 | 決定 |
| FR-PROTO-04 | SDK バージョン固定（四半期アップグレード PR ポリシー） | 決定 |
| FR-PROTO-05 | MCP プロトコル拡張・独自 RPC の追加 | 決定（**禁止**）|

### 3.2 トランスポート

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-TRANS-01 | Streamable HTTP（本番標準、Inbound / Upstream 両対応） | 決定 |
| FR-TRANS-02 | HTTP+SSE（上流互換のみ、Inbound は非対応） | 決定 |
| FR-TRANS-03 | stdio（ローカル開発 + 同一コンテナ内サブプロセス上流のみ） | 決定（MVP オプション、Phase 6 後に再着手検討） |
| FR-TRANS-04 | WebSocket | 決定（**対象外**） |
| FR-TRANS-05 | TLS 1.3 強制（Inbound 外向け） | 決定 |
| FR-TRANS-06 | mTLS（クライアント認証）| **未決定（OQ-01）**: 内部クライアント向けに採用するか |

### 3.3 名前空間化と集約

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-NS-01 | 名前空間形式: `{upstreamId}__{originalName}`（例: `github__create_issue`） | 決定 |
| FR-NS-02 | 起動時に重複チェック、衝突時は **Fail Fast**（コンテナ起動失敗） | 決定 |
| FR-NS-03 | 区切り文字 `__`（ダブルアンダースコア）の厳格検証 | 決定 |
| FR-NS-04 | `upstreamId` は `[a-z][a-z0-9-]{0,31}` の正規表現で制約 | 決定 |
| FR-NS-05 | `tools/list` 応答のキャッシュ（TTL 30s） | 決定 |
| FR-NS-06 | `resources/list` / `prompts/list` も同一名前空間規約 | 決定 |
| FR-NS-07 | ACL でユーザーごとに見える名前空間を絞り込み | 決定 |

### 3.4 ルーティング

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-ROUTE-01 | フロー: Inbound 受信 → Namespace 抽出 → AuthZ → Rate Limit → 接続プール取得 → 上流転送 → ストリーミング中継 → 監査 → 応答 | 決定 |
| FR-ROUTE-02 | 名前空間から upstreamId 抽出、未知 upstreamId は `-32601 method not found` | 決定 |
| FR-ROUTE-03 | ストリーミング応答の双方向中継（`asyncio.CancelledError` 確実伝播） | 決定 |
| FR-ROUTE-04 | クライアント切断時の上流接続即時解放 | 決定 |
| FR-ROUTE-05 | 上流タイムアウト: 既定 30 秒、上流別にコンフィグで上書き可 | 決定 |

### 3.5 上流接続管理

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-UP-01 | `IUpstreamClient` 抽象で Streamable HTTP / HTTP+SSE / stdio を統一インタフェース化 | 決定 |
| FR-UP-02 | 接続プール（上流ごとに最大同時接続数を制限、既定 20） | 決定 |
| FR-UP-03 | 接続再利用（Keep-Alive、Streamable HTTP セッション維持） | 決定 |
| FR-UP-04 | 上流ヘルスチェック（30 秒間隔、`/healthz` 等を上流定義に記載） | 決定 |
| FR-UP-05 | 上流追加時の再起動不要（動的リロード、Phase 7） | 決定 |
| FR-UP-06 | stdio サブプロセス上流のプロセスマネージャ（自動再起動、メモリ上限） | 決定（採用時） |

### 3.6 Circuit Breaker

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-CB-01 | Closed / Open / Half-Open の 3 状態 | 決定 |
| FR-CB-02 | エラー率閾値: 50%（60 秒移動窓 / 最低 10 リクエスト） | 決定 |
| FR-CB-03 | Open 状態の保持時間: 30 秒 | 決定 |
| FR-CB-04 | Half-Open で 3 リクエスト成功 → Closed 復帰 | 決定 |
| FR-CB-05 | Open 時のエラーコード: `-32001 upstream timeout` または `-32603 internal error`（要確定） | **未決定**: 専用コード新設か既存流用か |
| FR-CB-06 | Circuit Breaker 状態を Admin Console から可視化 | 決定 |

### 3.7 認証（AuthN）

| 要件 ID | 要件 | 決定 / 未決定 |
|---|---|---|
| FR-AN-01 | M1: API キー認証（HMAC + DynamoDB） | 決定 |
| FR-AN-02 | M2: JWT 認証（Cognito 統合） | 決定 |
| FR-AN-03 | M3: Azure AD / OIDC | 決定 |
| FR-AN-04 | API キー発行 API（`POST /admin/api-keys`） | 決定 |
| FR-AN-05 | API キーの有効期限（既定 90 日、ローテーション必須） | 決定 |
| FR-AN-06 | API キーの即時失効 | 決定 |
| FR-AN-07 | API キーのスコープ（user / service-account の区別） | 決定 |
| FR-AN-08 | mTLS（クライアント証明書） | **未決定（OQ-01）** |

### 3.8 認可（AuthZ）/ 3 軸 ACL

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-AZ-01 | 3 軸 ACL: user × upstream × tool | 決定 |
| FR-AZ-02 | デフォルト deny（明示許可のない組み合わせは拒否） | 決定 |
| FR-AZ-03 | グループ ACL（user グループ × upstream × tool） | 決定 |
| FR-AZ-04 | ワイルドカード（`upstream:github__*` で github 上流の全ツール許可） | 決定 |
| FR-AZ-05 | ACL 評価レイテンシ p95 < 20ms | 決定 |
| FR-AZ-06 | ACL 変更の即時反映（DynamoDB Streams + キャッシュ無効化、5 秒以内） | 決定 |
| FR-AZ-07 | KDB 経由アクセス時の二段階フィルタ（Gateway 粗 → KDB 内 ACL 精密） | 決定 |

### 3.9 レート制限

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-RL-01 | トークンバケット型、デフォルト **60 req/分/ユーザー** | 決定 |
| FR-RL-02 | 上流別の上限をコンフィグで指定可 | 決定 |
| FR-RL-03 | ツール別の上限をコンフィグで指定可（高コスト LLM ツール向け） | 決定 |
| FR-RL-04 | レート制限超過時のエラーコード: `-32004 rate limit` | 決定 |
| FR-RL-05 | レート制限応答 p95 < 50ms | 決定 |
| FR-RL-06 | 管理者向け上限のバイパス（既定無効、明示有効化のみ） | 決定 |

### 3.10 PII フィルタ / Guardrails

| 要件 ID | 要件 | 決定 / 未決定 |
|---|---|---|
| FR-PII-01 | Bedrock Guardrails との連携 | 決定（Phase 6 / M2 以降） |
| FR-PII-02 | フィルタ適用方向: Inbound / Upstream / 双方 | **未決定**: 既定は双方か |
| FR-PII-03 | PII 検出時のエラーコード: `-32005 guardrails block` | 決定 |
| FR-PII-04 | フィルタ誤検知率: < 5%（100 ケースベンチマーク） | 決定 |
| FR-PII-05 | 管理者向けバイパス（既定無効） | 決定 |

### 3.11 クレデンシャル管理

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-CRED-01 | 上流クレデンシャルを Secrets Manager に集中保管 | 決定 |
| FR-CRED-02 | 上流定義 YAML には Secrets Manager の ARN のみを記載（生トークン禁止） | 決定 |
| FR-CRED-03 | クレデンシャル取得の権限を Gateway タスクロールに限定(最小権限) | 決定 |
| FR-CRED-04 | クレデンシャルローテーション（既定 90 日、SecretsManager 自動ローテーション設定） | 決定 |
| FR-CRED-05 | 上流接続時のシークレット注入はメモリ上のみ（ディスク書込禁止） | 決定 |
| FR-CRED-06 | IAM Access Analyzer でクレデンシャル系の過剰権限ゼロ | 決定 |

### 3.12 監査

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-AUD-01 | 全 RPC を `AUDIT#mcp-gw#` 名前空間で DynamoDB に書込（**MCP 専用テーブル**） | 決定 |
| FR-AUD-02 | 記録項目: timestamp, user_id, upstream_id, tool_name, request_id, latency_ms, status, error_code | 決定 |
| FR-AUD-03 | 監査ログ書き込み成功率 99.9%+（失敗時 CloudWatch Logs にフォールバック） | 決定 |
| FR-AUD-04 | 直近 24h の特定ユーザー RPC 抽出 GSI で 1 秒以内 | 決定 |
| FR-AUD-05 | S3 Object Lock + Athena で長期保管・横断検索 | 決定 |
| FR-AUD-06 | `packages/audit-events/AuditRepository(tableName)` 抽象で **MCP 専用テーブルに書込** | 決定 |
| FR-AUD-07 | 機密ペイロードのマスキング（既定: 上流クレデンシャルは記録しない） | 決定 |

### 3.13 観測性

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-OBS-01 | CloudWatch メトリクス（EMF）: `RequestCount` / `Latency p50/p95/p99` / `UpstreamErrors` / `RateLimitHits` / `ActiveSessions` | 決定 |
| FR-OBS-02 | X-Ray 分散トレーシング（Inbound → AuthZ → Upstream → 応答） | 決定 |
| FR-OBS-03 | 構造化ログ（`structlog`、JSON 形式、`request_id` 必須） | 決定 |
| FR-OBS-04 | ヘルスチェック: `/healthz`（Gateway 自体）/ `/healthz/upstreams`（各上流） | 決定 |
| FR-OBS-05 | メトリクスカバレッジ率 90%+（全公開 RPC が Metrics + Tracing 双方で計測） | 決定 |

### 3.14 管理 API / 動的リロード

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-ADM-01 | `/admin/upstreams` 上流定義 CRUD | 決定 |
| FR-ADM-02 | `/admin/acls` ACL CRUD | 決定 |
| FR-ADM-03 | `/admin/api-keys` API キー発行・失効 | 決定 |
| FR-ADM-04 | `CONFIG#mcp-gw#version` を 30 秒ポーリングで動的リロード | 決定 |
| FR-ADM-05 | 設定変更の監査ログ（誰がいつ何を変更したか） | 決定 |
| FR-ADM-06 | Admin Console 統合（`apps/admin-console/app/mcp-gateway/`、本体 Phase 6b 後） | 決定 |

### 3.15 上流登録

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-UPREG-01 | 上流定義は YAML on S3（静的）+ DynamoDB（動的上書き）| 決定 |
| FR-UPREG-02 | 必須属性: upstreamId, transport, endpoint, healthCheck, credentialsArn, timeoutSec, ratelimit | 決定 |
| FR-UPREG-03 | 任意属性: tags, owner, description | 決定 |
| FR-UPREG-04 | 起動時バリデーション（必須属性欠落 / 名前空間衝突で Fail Fast） | 決定 |
| FR-UPREG-05 | 上流定義の差分プレビュー（管理者が編集前に確認） | 決定 |
| FR-UPREG-06 | 上流定義のバージョン履歴（最新 10 世代、ロールバック可） | 決定 |

---

## 4. 非機能要件

### 4.1 性能

| 指標 | 目標値 | 計測方法 |
|---|---|---|
| `tools/list` p95 | < 200ms | CloudWatch カスタムメトリクス（500 ユーザー時） |
| `tools/list` p99 | < 500ms | 同上 |
| `tools/call` 透過オーバーヘッド p95 | < 50ms | Gateway 内部処理時間（上流応答時間を除く） |
| ACL 評価 p95 | < 20ms | 同上 |
| レート制限応答 p95 | < 50ms | 同上 |
| 監査ログ書込 p95 | < 30ms | 同上（非同期化されていれば inbound 影響なし） |
| `/healthz` 応答 | < 50ms | 同上 |
| 同時接続数 | 500（500 ユーザー時） | k6 負荷試験 |

### 4.2 可用性

| 指標 | 目標値 |
|---|---|
| Gateway SLO | 99.9%（500 ユーザー時、Multi-AZ） |
| Multi-AZ | 必須（ECS Fargate 2 AZ × 2 task 以上） |
| RTO | < 30 分 |
| RPO | < 1 時間 |
| Graceful Shutdown | SIGTERM 受信後 30 秒以内に進行中 RPC 完了 |
| 1 上流障害時の他上流成功率 | 99%+（Circuit Breaker による分離） |

### 4.3 スケーラビリティ

| 指標 | MVP（50 ユーザー） | M2（200 ユーザー） | 最終目標（500 ユーザー） |
|---|---|---|---|
| 同時接続数 | 50 | 250 | 500 |
| 上流 MCP サーバー数 | 2〜5 | 10〜15 | 20〜30 |
| ツール総数 | 30〜80 | 150〜300 | 300〜600 |
| 月間 RPC 数 | 10,000 | 50,000 | 100,000+ |
| API キー数 | 50 | 200 | 500 |

### 4.4 セキュリティ（多層防御マッピング）

> 本体 OPENCLAW の 5 層多層防御と整合させつつ、MCP Gateway 固有の層を定義する。

| 層 | MCP Gateway での対策 | Trust Boundary |
|---|---|---|
| L1 上流仕様 | 上流 MCP の `tools/list` を信頼する（プロトコル準拠） | ❌ |
| L2 名前空間 | `{upstreamId}__` の厳格検証、起動時 Fail Fast | △ |
| L3 3 軸 ACL | user × upstream × tool、デフォルト deny | ✅（アプリケーション境界） |
| L4 IAM（Gateway タスクロール） | 上流 Secrets Manager / DynamoDB / S3 への最小権限 | ✅ |
| L5 ネットワーク境界 | パブリックポート ALB のみ、上流接続は Private Subnet / VPC Endpoint | ✅ |

| 要件 ID | 要件 | 決定 |
|---|---|---|
| NFR-SEC-01 | 保存時暗号化（DynamoDB / S3 / Secrets Manager） | 決定 |
| NFR-SEC-02 | 転送時暗号化（TLS 1.3、上流接続もすべて TLS） | 決定 |
| NFR-SEC-03 | API キーは HMAC ハッシュで保存（生キー保存禁止） | 決定 |
| NFR-SEC-04 | JWT 検証時の `aud` / `iss` / `exp` 厳格チェック | 決定 |
| NFR-SEC-05 | OWASP Top 10 対策の CI 検証（Semgrep / Trivy / gitleaks） | 決定 |
| NFR-SEC-06 | IAM Access Analyzer で過剰権限ゼロ | 決定 |
| NFR-SEC-07 | WAF ルール（OWASP CRS + IP レート制限） | 決定 |
| NFR-SEC-08 | パブリックポートは ALB のみ、ECS タスクは Private Subnet | 決定 |
| NFR-SEC-09 | 上流クレデンシャルのメモリ常駐期間制限（リクエスト終了で wipe） | 決定 |

### 4.5 マルチテナント分離（テーブル分離）

> 本サブシステムは単一組織前提だが、**本体 / KDB との物理分離**は必須（DynamoDB I/O 競合・PITR コスト結合・DDL 影響を回避）。

| 要件 ID | 要件 | 決定 |
|---|---|---|
| NFR-TEN-01 | DynamoDB テーブル `enterprise-ai-platform-mcp-gw-{env}` を新設 | 決定 |
| NFR-TEN-02 | PITR / TTL / On-Demand 容量を独立設定 | 決定 |
| NFR-TEN-03 | Gateway タスクロールは MCP テーブルのみ Read/Write 可、本体 / KDB テーブルは拒否（Resource ARN 厳格指定） | 決定 |
| NFR-TEN-04 | IAM 越権防止: AuditRepository IAM 境界テスト 3 系統に拡張（本体 / MCP / KDB） | 決定（本体 § 10.3 **AC-S-11** と同方針） |
| NFR-TEN-05 | スキーマは 3 系統共通（`AuditRepository(tableName)` 抽象で吸収） | 決定 |

### 4.6 監査

| 要件 ID | 要件 | 決定 |
|---|---|---|
| NFR-AUD-01 | 全 RPC を 99.9%+ で監査ログに書込 | 決定 |
| NFR-AUD-02 | 記録内容: timestamp, user_id, upstream_id, tool_name, request_id, latency_ms, status, error_code | 決定 |
| NFR-AUD-03 | 生ペイロード保管 vs ハッシュ保管 | **未決定（OQ-02）**: プライバシーと監査有用性の両立 |
| NFR-AUD-04 | 保持期間: DynamoDB 90 日 + S3 Object Lock 7 年 | 決定 |
| NFR-AUD-05 | Athena で過去 90 日横断検索 < 30 秒 | 決定 |
| NFR-AUD-06 | 監査ログのアクセス監査（誰が監査を閲覧したか） | 決定 |

### 4.7 コンプライアンス

| 要件 ID | 要件 | 決定 / 未決定 |
|---|---|---|
| NFR-CMP-01 | 単一リージョン強制（本体と同一リージョン） | 決定 |
| NFR-CMP-02 | クロスリージョン構成禁止 | 決定 |
| NFR-CMP-03 | 上流クラウド MCP 利用時のデータ越境 | **未決定（OQ-03）**: 上流別に法務確認、初期は社内ホスト or 同一リージョン上流のみ |
| NFR-CMP-04 | MCP プロトコル MIT、SDK MIT のライセンス問題なし | 決定 |
| NFR-CMP-05 | GDPR 削除権の SLA | **未決定（OQ-04）**: 本体と整合（30 日想定） |

### 4.8 観測性

| 要件 ID | 要件 | 決定 |
|---|---|---|
| NFR-OBS-01 | CloudWatch ダッシュボード（5 指標: RequestCount / Latency / UpstreamErrors / RateLimitHits / ActiveSessions） | 決定 |
| NFR-OBS-02 | X-Ray トレース（Inbound → AuthZ → Upstream → 応答） | 決定 |
| NFR-OBS-03 | エラー予算: 99.9% SLO に対し月間 < 43 分 | 決定 |
| NFR-OBS-04 | PagerDuty 連携（CRITICAL アラート） | 決定 |
| NFR-OBS-05 | コストダッシュボード（MCP Gateway 単独タグ） | 決定 |

### 4.9 コスト

> 本表は MCP Gateway 単体のサマリ。詳細内訳は § 9.3 を参照。3 書合算は **本体要件書 § 4.9 / § 9.3 を一次出典**とし、本書は MCP 単体の金額のみを担当する。

| 項目 | MVP / 月 | 500 ユーザー / 月 |
|---|---|---|
| ECS Fargate（2 AZ × 2 task） | $40 | $80 |
| ALB | $20 | $25 |
| DynamoDB（On-Demand + Streams） | $10 | $50 |
| S3 + CloudFront | $5 | $20 |
| Secrets Manager | $5 | $15 |
| ElastiCache（任意、`tools/list` キャッシュ強化用） | $0 | $30 |
| **小計（MCP Gateway 単独）** | **$80** | **$220** |

3 書合算（本体要件書 § 9.3 と整合）:

| マイルストーン | 本体（§ 9.3 小計） | KDB | MCP Gateway | 合計 | 成功基準目標（$5/user × N） |
|---|---|---|---|---|---|
| MVP（M1: 50 ユーザー） | $255〜$435 | $80〜$130 | $80 | **$415〜$645** | $250 |
| 最終（M3: 500 ユーザー） | $2,090〜$3,290 | $300〜$640 | $220 | **$2,610〜$4,150** | **$2,500** |

> 500 ユーザー時の最大想定が目標 $2,500 を超過しうる点は本体要件書 § 9.3 と同認識。Bedrock 推論コストの最適化で吸収。

---

## 5. アーキテクチャ方針（要件レベルの選定比較）

> 本セクションは「採用方針の比較と決定理由」のみ。実装プランは別途。

### 5.1 配置パターン

| 選択肢 | 評価 |
|---|---|
| **A. 独立 ECS Fargate サービス（共通インフラ共有）** | **採用**。本体 / KDB と同パターン、運用ノウハウ共有。ECS タスクと DynamoDB テーブルのみ分離 |
| B. 本体 ECS タスクに同居 | 責務肥大、本体スコープ侵食 |
| C. Lambda + API Gateway | ストリーミング維持と長時間接続に不向き |
| D. EKS | 500 ユーザー前提でオーバースペック |

**決定: A**。独立 ECS Fargate（Multi-AZ、ARM64 Graviton）。

### 5.2 MCP SDK

| 選択肢 | 評価 |
|---|---|
| **A. Python `mcp` 公式 SDK** | **採用**。ADR 0001 に整合、Agent Container と運用ノウハウ共有可 |
| B. TypeScript `@modelcontextprotocol/sdk` | 言語分散、運用負債 |
| C. Go / Rust（自前実装） | プロトコル進化追従コスト過大 |

**決定: A**。`enterprise-ai-platform` 主要バックエンドと言語統一。

### 5.3 認証方式（段階導入）

| 選択肢 | M1 | M2 | M3 |
|---|---|---|---|
| **A. API キー（HMAC） → JWT（Cognito） → Azure AD/OIDC** | **採用** | 追加 | 追加 |
| B. JWT 一本 | M1 立ち上げが遅延 | - | - |
| C. mTLS のみ | 配布運用負債大 | - | - |

**決定: A**。段階導入で MVP を最速化、エンタープライズ要件を順次満たす。

### 5.4 ACL ストア

| 選択肢 | 評価 |
|---|---|
| **A. DynamoDB（動的）+ YAML キャッシュ** | **採用**。動的反映 + 30s ポーリング、本体パターンと一致 |
| B. ファイルベース（再起動必須） | 動的反映不可 |
| C. 専用 OPA / Cedar サーバー | 運用負債過大、500 ユーザーで不要 |

**決定: A**。

### 5.5 名前空間衝突対応

| 選択肢 | 評価 |
|---|---|
| **A. 起動時 Fail Fast** | **採用**。設定誤りを最早期に検出、本番中の名前変化を排除 |
| B. 警告 + 後勝ち / 先勝ち | 静かなバグの温床 |
| C. 自動サフィックス付与（`_2`, `_3`） | クライアントが追跡困難 |

**決定: A**。

### 5.6 DynamoDB テーブル分離

| 選択肢 | 評価 |
|---|---|
| **A. MCP 専用テーブル `enterprise-ai-platform-mcp-gw-{env}`** | **採用**。I/O 競合・PITR コスト結合・DDL 影響を回避 |
| B. 本体テーブルに `MCP#` 名前空間で同居 | I/O 競合・容量設定干渉のリスク（KDB と同論点） |

**決定: A**。本体 OPENCLAW § 4.5 NFR-TEN-05 / KDB § 6.5 と同方針。

### 5.7 上流クレデンシャル管理

| 選択肢 | 評価 |
|---|---|
| **A. Secrets Manager 一元化、Gateway タスクロールのみ取得可** | **採用**。漏洩リスク最小化、ローテーションも自動化可 |
| B. SSM Parameter Store（SecureString） | ローテーション機能弱 |
| C. クライアント側保持 | 漏洩リスク・運用負債大 |

**決定: A**。

---

## 6. データモデル要件

> 詳細スキーマは実装プラン § 8 で定義。要件レベルで属性のみ列挙。

### 6.1 DynamoDB テーブル（MCP 専用）

テーブル名: `enterprise-ai-platform-mcp-gw-{env}`

| パターン | PK | SK | 用途 / GSI |
|---|---|---|---|
| Upstream 定義 | `UPSTREAM#{upstreamId}` | `META` | 上流メタ |
| ACL（user × upstream × tool） | `ACL#{userId}` | `RULE#{upstreamId}#{toolPattern}` | GSI1: upstreamId |
| ACL（グループ） | `ACL#group#{groupId}` | `RULE#{upstreamId}#{toolPattern}` | 同上 |
| API キー | `APIKEY#{keyHash}` | `META` | GSI1: userId, GSI2: expiresAt |
| Config Version | `CONFIG#mcp-gw#version` | `META` | 動的リロード用バージョン |
| Audit Event | `AUDIT#mcp-gw#{ts}` | `RPC#{requestId}` | GSI1: userId, GSI2: upstreamId（90 日 TTL） |
| Rate Limit Counter | `RL#{userId}` | `WINDOW#{minute}` | 1 分 TTL |

### 6.2 上流定義（YAML スキーマ）

```yaml
upstreams:
  - id: github                  # FR-NS-04 制約に従う
    transport: streamable_http  # streamable_http / http_sse / stdio
    endpoint: https://...
    healthCheck: /healthz
    credentialsArn: arn:aws:secretsmanager:...
    timeoutSec: 30
    rateLimit:
      perUser: 60               # 60 req/分/ユーザー
    tags: [code, devops]
    owner: it-team@example.com
```

### 6.3 ACL 表現

| 観点 | 表現 |
|---|---|
| user × upstream × tool | `ACL#{userId}` レコードに `allowed: [github__create_issue, jira__*]` |
| グループ | user に複数 group を割り当て、グループ ACL を OR で評価 |
| ワイルドカード | `{upstreamId}__*` で上流全許可、`*__{tool}` は禁止（誤許可防止） |
| デフォルト deny | 該当ルールなし → 拒否 |

### 6.4 監査イベント属性

| 属性 | 型 | 必須 |
|---|---|---|
| timestamp | ISO 8601 | ✓ |
| user_id | string | ✓ |
| upstream_id | string | ✓ |
| tool_name | string | ✓（`tools/call` の場合） |
| request_id | UUID | ✓ |
| method | string | ✓（`tools/list`, `tools/call`, ...） |
| latency_ms | int | ✓ |
| status | enum | ✓（success / denied / upstream_error / rate_limited / guardrails_blocked） |
| error_code | int | （JSON-RPC エラーコード） |
| client_id | string | ✓（API キー識別子のハッシュ） |
| transport | enum | ✓（streamable_http / http_sse / stdio） |

### 6.5 JSON-RPC エラーコード規約

| エラー | コード |
|---|---|
| 上流タイムアウト | -32001 |
| 認証失敗 | -32002 |
| 認可拒否 | -32003 |
| レート制限 | -32004 |
| Guardrails ブロック | -32005 |

---

## 7. インテグレーション要件

### 7.1 共有インフラ（本体との関係）

| 共有対象 | 共有方法 |
|---|---|
| VPC / Subnet / IAM 基盤 | 同一 Terraform ステート |
| Cognito User Pool | 本体 Phase 6a Step 3 構築済みを共有（**MCP Phase 4 着手要件**） |
| Secrets Manager | 同一 KMS CMK でクレデンシャル管理 |
| S3 audit-archive プレフィックス | 共有（テーブルは分離） |
| Admin Console | 本体 Phase 6b 後に `app/mcp-gateway/` 追加 |
| CI/CD | 同一 GitHub Actions / Turborepo |

### 7.2 独立部分（本体から分離）

| 独立対象 | 理由 |
|---|---|
| ECS Fargate サービス + ALB ターゲットグループ | 障害分離・スケール独立 |
| DynamoDB テーブル `enterprise-ai-platform-mcp-gw-{env}` | I/O 競合・PITR コスト・DDL 影響の分離（本体 § 4.5 / KDB § 6.5 と同方針） |

### 7.3 上流サブシステム（KDB）との統合

| 観点 | 内容 |
|---|---|
| KDB を上流 MCP として登録 | `config/upstreams.yaml` に `knowledge` upstream 登録 |
| 公開ツール | `knowledge__search`, `knowledge__fetch`, `knowledge__list_kbs` |
| AuthN | Gateway が Cognito JWT を取得 → KDB Service へ internal SigV4 / IAM 認証 |
| AuthZ 二段階 | Gateway 3 軸 ACL（粗フィルタ）+ KDB 内 ACL（精密フィルタ） |
| ストリーミング | KDB の検索は一括レスポンス、ストリーミング不要 |

### 7.4 クライアント側統合

| クライアント | 統合方法 |
|---|---|
| Claude Desktop | 公式 MCP 設定（`mcpServers` セクション）に Gateway エンドポイント + API キー登録 |
| Claude Code | 同上 |
| Cursor | 同上（MCP サポート版） |
| Agent Container（OPENCLAW 本体） | **クライアントとして利用しない**（本体は独立にツールを実行）。OPENCLAW 内のツールが KDB を呼ぶ際は OpenClaw Tool 定義経由（Gateway を通らない） |
| 社内 RPA / バッチ | サービスアカウントの API キーで接続 |

### 7.5 監査基盤統合

| 接続先 | 接続方法 |
|---|---|
| `packages/audit-events/AuditRepository(tableName)` | テーブル名 `enterprise-ai-platform-mcp-gw-{env}` を注入 |
| DynamoDB Streams → Firehose → S3 Object Lock | MCP 専用ストリーム + 共通 S3 プレフィックス（`audit-archive/mcp-gw/`） |
| Athena テーブル | MCP 専用パーティション、本体 / KDB と JOIN 可能（共通スキーマ） |

### 7.6 CI/CD

| 観点 | 採用 |
|---|---|
| パイプライン | GitHub Actions + OIDC |
| IaC | Terraform Cloud（state） / Atlantis（plan PR） |
| コンテナビルド | Buildx（ARM64 cross-build） |
| 静的解析 | Ruff / Semgrep / Trivy / gitleaks |
| テスト | pytest + pytest-asyncio + pytest-cov、80% カバレッジゲート |
| MCP プロトコル準拠テスト | MCP Inspector を CI に組込 |
| 負荷試験 | k6（50 / 250 / 500 同時接続） |

---

## 8. 既存プランとの整合

### 8.1 実装プラン（MCP_GATEWAY_PLAN.md）との対応

本書の要件は実装プランの 8 フェーズに分散実装される:

| 要件カテゴリ | 主実装フェーズ |
|---|---|
| MCP プロトコル対応（§ 3.1） | Phase 1 |
| トランスポート（§ 3.2） | Phase 1, 2 |
| 名前空間化と集約（§ 3.3） | Phase 3 |
| ルーティング（§ 3.4） | Phase 3 |
| 上流接続管理（§ 3.5） | Phase 2 |
| Circuit Breaker（§ 3.6） | Phase 6 |
| 認証（§ 3.7） | Phase 4（API キー / JWT）+ M3（Azure AD） |
| 認可（§ 3.8） | Phase 4 |
| レート制限（§ 3.9） | Phase 6 |
| PII フィルタ（§ 3.10） | Phase 6 |
| クレデンシャル管理（§ 3.11） | Phase 4 |
| 監査（§ 3.12） | Phase 5 |
| 観測性（§ 3.13） | Phase 5 |
| 管理 API / 動的リロード（§ 3.14） | Phase 7 |
| 上流登録（§ 3.15） | Phase 2, 7 |

### 8.2 マイルストーンとの対応

| マイルストーン | 含むフェーズ | 本書の対応要件 |
|---|---|---|
| **M1 MVP** | Phase 1〜3 + Phase 4（API キー）+ Phase 5 最小 | プロトコル基本 / 名前空間 / API キー認証 / 監査最小 |
| **M2** | + Phase 4（JWT）+ Phase 5 全 + Phase 6 | Cognito JWT / 全監査 / Multi-AZ / Circuit Breaker / PII フィルタ / 200 ユーザー |
| **M3** | + Phase 7 + Phase 8 | 動的リロード / Admin Console / 500 ユーザー / e2e + 負荷試験 |

### 8.3 横断依存（本体プラン / KDB との結合）

| MCP Gateway Phase | 依存する本体 / KDB Phase | 提供価値 |
|---|---|---|
| Phase 1 | なし（並行可能） | Walking Skeleton 確立 |
| Phase 4（Cognito JWT） | 本体 **Phase 6a Step 3**（RBAC + Cognito）— 6a 全体完了不要 | JWT 認証統合 |
| Phase 6（PII / Guardrails） | 本体 Phase 8（Guardrails ティア定義） | PII フィルタ統合 |
| Phase 7（Admin Console 統合） | 本体 **Phase 6b**（UI 基盤） | 管理 UI |
| KDB を上流登録 | KDB-M1 完了 + MCP Phase 3 完了 | `knowledge__*` 公開 |

並行 2 名体制では本体 Phase 6a Step 3 完了時点で MCP Phase 4 が着手可能。本体 6b（UI）と MCP Phase 4〜7 を並行進行することで、Phase 6 全体待ちより 0.5〜1 人月短縮できる。

### 8.4 整合性確認チェック（本体 / KDB / プランとの矛盾検出）

| 観点 | 既存記載 | 本書 | 矛盾の有無 |
|---|---|---|---|
| 単一リージョン強制 | CLAUDE.md / 本体プラン § 5.4 | § 4.7 NFR-CMP-01 で遵守 | **矛盾なし** |
| Python 3.12 統一 | ADR 0001 | § 5.2 で遵守 | **矛盾なし** |
| MCP 独自テーブル分離 | MCP プラン § 4.2 | § 4.5 / § 6.1 で再掲 | **矛盾なし** |
| 監査スキーマ共通 | 本体 / KDB 要件 | § 7.5 / § 6.4 で共通 | **矛盾なし** |
| KDB を上流として登録 | KDB 要件 § 7.2 | § 7.3 で整合 | **矛盾なし** |
| Cognito User Pool 共有 | MCP プラン § 7.5 | § 7.1 で再掲 | **矛盾なし** |
| Admin Console 統合は本体 Phase 6b 後 | MCP プラン § 7.5 | § 7.1 で再掲 | **矛盾なし** |
| ARM64 Graviton | MCP プラン § 7.4 | § 5.1 で遵守 | **矛盾なし** |
| 500 ユーザー上限 | CLAUDE.md / 本体プラン § 1.2 | § 4.3 で遵守 | **矛盾なし** |
| AuditRepository(tableName) 抽象 | KDB § 4.5 / 本体 § 3.14 FR-AUD-06 | § 3.12 FR-AUD-06 で遵守 | **矛盾なし** |
| L1 SOUL は Trust Boundary でない | 本体 § 4.4 | § 4.4 で MCP 独自層を定義（本体 5 層とは別軸） | **矛盾なし**（独立サブシステム固有） |

### 8.5 既存プラン修正提案

本要件定義の承認後、以下の差分を `docs/MCP_GATEWAY_PLAN.md` に反映する提案。**実際の編集は別タスク**。

#### 修正提案 (a): § 1 末尾に「要件定義書」リンク追記

「詳細な要件は `docs/MCP_GATEWAY_REQUIREMENTS.md` を参照」と注記。

#### 修正提案 (b): 各 Phase 完了基準と本書 § 10 受け入れ基準のクロスリンク

各 Phase の完了基準節に「対応する受け入れ基準: AC-F-xx, AC-P-xx, AC-S-xx」を追記。

---

## 9. リスクと制約

### 9.1 技術リスク

| リスク | 影響 | 緩和策 |
|---|---|---|
| MCP プロトコル進化 | クライアント断絶 | SDK バージョン固定、四半期アップグレード PR、Walking Skeleton で早期検出 |
| 上流障害伝播 | 全クライアント影響 | Circuit Breaker（50% / 60s / Open 30s / Half-Open 3 成功） + 上流別接続プール + `tools/list` キャッシュ継続 |
| 名前空間衝突 | ルーティング崩壊 | 起動時 Fail Fast、`__` 区切り厳格検証 |
| 上流クレデンシャル漏洩 | 全社情報漏洩 | Secrets Manager 一元化、IAM Access Analyzer、ローテーション 90 日 |
| stdio サブプロセスリーク | メモリ / FD 枯渇 | プロセスマネージャ、定期再起動、メモリ上限、MVP は採用延期可 |
| ストリーミング中断 | クライアントハング | `asyncio.CancelledError` 確実伝播、上流中断シグナル送信 |
| `tools/list` 肥大化 | 応答遅延 | ACL ユーザー別絞り込み、上流別ページング、TTL 30s キャッシュ |
| `tools/call` 透過オーバーヘッド未達 | UX 低下 | 内部処理の Python `asyncio` プロファイリング、ホットパスから boto3 / ロガーの同期 I/O を除外 |
| ACL 評価遅延 | 全リクエストに影響 | DynamoDB → メモリキャッシュ、ACL 変更時のみ無効化、p95 < 20ms 達成 |

### 9.2 運用リスク

| リスク | 緩和策 |
|---|---|
| ACL 設定漏れ | デフォルト deny、Admin Console での設定プレビュー、CI で ACL テンプレ妥当性検証 |
| API キー流通 | 90 日有効期限 + ローテーション、即時失効、利用元 IP / UA 監査 |
| 監査ログ容量爆発 | DynamoDB TTL（90 日）+ S3 Object Lock、月次容量レポート |
| 動的リロード失敗 | YAML スキーマバリデーション、ロールバック機構、Admin Console に検証ボタン |
| 上流追加事故 | 差分プレビュー必須、起動時 Fail Fast、ロールバック 10 世代 |
| 認証バイパス | JWT 検証 `aud` / `iss` / `exp` の CI 必須テスト、HMAC 比較のタイミング攻撃対策 |
| 上流レート制限の不整合 | 上流別 rate limit を上流定義 YAML で明示、CloudWatch でカウント |

### 9.3 コスト見積もり

| 項目 | MVP / 月 | 500 ユーザー / 月 | 備考 |
|---|---|---|---|
| ECS Fargate（2 AZ × 2 task） | $40 | $80 | ARM64 Graviton |
| ALB | $20 | $25 | 既存共有可で実質減 |
| DynamoDB（On-Demand + Streams） | $10 | $50 | MCP 専用テーブル |
| S3 + CloudFront | $5 | $20 | 上流定義 YAML 配信 |
| Secrets Manager | $5 | $15 | 上流クレデンシャル |
| ElastiCache（任意） | $0 | $30 | `tools/list` キャッシュ強化用 |
| CloudWatch / X-Ray | $5 | $15 | メトリクス + トレース |
| **合計** | **$85** | **$235** | 当初見積もり $80 / $220 に観測性費を加算 |

> 本体予算 500〜1,000 USD / 月 + KDB 80〜400 USD / 月 + MCP Gateway 85〜235 USD / 月 = **合計 665〜1,635 USD / 月**。本体「1 ユーザー / 月 $5 以下」目標（500 ユーザーで $2,500）の枠内に収まる。

### 9.4 法的・データ越境

| 制約 | 詳細 |
|---|---|
| 上流クラウド MCP のデータ越境 | **OQ-03**: 上流別に法務確認。初期は社内ホスト or 同一リージョン上流のみ採用 |
| 単一リージョン強制 | 本体と同一リージョン、`region_override` 禁止 |
| MCP プロトコル MIT、SDK MIT | ライセンス問題なし |
| 監査ログ越境 | 単一リージョン内のみ保管 |

---

## 10. 受け入れ基準（要件達成の判定）

### 10.1 機能受け入れ基準

| ID | 基準 |
|---|---|
| AC-F-01 | Claude Desktop から Gateway を MCP サーバーとして登録し、`initialize` → `tools/list` → `tools/call` が往復成立する |
| AC-F-02 | 複数上流のツールが `{upstreamId}__{toolName}` 形式で集約された `tools/list` を返却 |
| AC-F-03 | 上流追加 / 設定変更が Admin Console から 30 秒以内にクライアントの `tools/list` に反映 |
| AC-F-04 | 名前空間衝突が起動時に Fail Fast（コンテナ起動失敗） |
| AC-F-05 | 3 軸 ACL で「特定ユーザー / グループ × 上流 × ツール」を許可・拒否できる |
| AC-F-06 | デフォルト deny: 明示許可のない組み合わせは `-32003 denied` |
| AC-F-07 | API キーが 90 日で自動失効し、ローテーション API で更新可能 |
| AC-F-08 | レート制限超過時に `-32004 rate limit` が p95 < 50ms で返却 |
| AC-F-09 | 1 上流を意図的に停止して 5 分間負荷を流し、他上流の成功率 99%+ を維持 |
| AC-F-10 | Circuit Breaker が Open → Half-Open → Closed の遷移を実機ログで確認、復旧時間 < 60s |
| AC-F-11 | 監査検索で「特定ユーザーの過去 24h RPC」を GSI で 1 秒以内に取得 |
| AC-F-12 | Athena で過去 90 日横断検索が 30 秒以内 |
| AC-F-13 | KDB を上流登録し、Claude Desktop から `knowledge__search` が動作 |
| AC-F-14 | 上流クレデンシャルがクライアントに直接配布されないことを CI で検証 |

### 10.2 性能受け入れ基準

| ID | 基準 |
|---|---|
| AC-P-01 | `tools/list` p95 < 200ms（500 ユーザー時） |
| AC-P-02 | `tools/call` 透過オーバーヘッド p95 < 50ms |
| AC-P-03 | ACL 評価 p95 < 20ms |
| AC-P-04 | レート制限応答 p95 < 50ms |
| AC-P-05 | `/healthz` 応答 < 50ms |
| AC-P-06 | **500 同時 MCP セッション**（クライアントごとに 1 セッション = 1 Streamable HTTP 長時間接続）で、その上を流れる全 RPC（`initialize` / `tools/list` / `tools/call` / `resources/*`）の成功率 99.9%+。上流タイムアウト由来の失敗は除外し、Gateway 内部起因のみで判定。本体 § 4.3「Concurrent Sessions 100」とは別軸（本体は AgentCore セッション、本書は MCP セッション） |
| AC-P-07 | Multi-AZ で 1 タスク強制終了しても 30 秒以内に ALB ターゲットから除外、Drained Connections ゼロ |

### 10.3 セキュリティ受け入れ基準

| ID | 基準 |
|---|---|
| AC-S-01 | API キーが HMAC ハッシュで保存されることを Terraform / コードレビューで確認 |
| AC-S-02 | JWT 検証の `aud` / `iss` / `exp` バイパステスト 50 ケース以上が全パス |
| AC-S-03 | 上流クレデンシャルが Secrets Manager にのみ存在、コードベース全文検索で生トークン検出ゼロ（gitleaks） |
| AC-S-04 | IAM Access Analyzer で過剰権限ゼロ |
| AC-S-05 | OWASP Top 10 全カテゴリの自動検査（Semgrep / Trivy）が PASS |
| AC-S-06 | **AuditRepository IAM 境界テスト 3 系統**: Gateway タスクロールが本体 / KDB テーブルへ Put / Get できないことを、(a) IAM Simulator API（宣言的検証）+ (b) CI 環境での実 STS AssumeRole 試行（実アクセス検証）の二段階で確認（本体 § 10.3 **AC-S-11** と同方針、3 書共通テストスイートとして実装） |
| AC-S-07 | パブリックポートが ALB のみであることを Terraform で強制 |
| AC-S-08 | ACL 設定変更が監査ログに 100% 記録 |
| AC-S-09 | PII フィルタ誤検知率 < 5%（100 ケースベンチマーク） |

### 10.4 観測性受け入れ基準

| ID | 基準 |
|---|---|
| AC-O-01 | CloudWatch ダッシュボードに `RequestCount` / `Latency p50/p95/p99` / `UpstreamErrors` / `RateLimitHits` / `ActiveSessions` の 5 指標が表示 |
| AC-O-02 | X-Ray トレースで Inbound → AuthZ → Upstream の内訳が見える |
| AC-O-03 | メトリクスカバレッジ率 90%+（全公開 RPC が Metrics + Tracing 双方で計測） |
| AC-O-04 | 監査ログ書き込み成功率 99.9%+（非同期キュー + CloudWatch Logs フォールバック含む） |
| AC-O-05 | CRITICAL アラート（上流障害 / 認証失敗バースト等）が PagerDuty に 1 分以内に通知 |

---

## 11. 未決事項と意思決定が必要な論点（Open Questions）

更新後 Open Questions 残: **5 件**（OQ-01 〜 OQ-05）。

| ID | 論点 | 判断者 | 期限 |
|---|---|---|---|
| OQ-01 | 内部クライアント向け mTLS（クライアント証明書認証）の採用可否 | セキュリティ + アーキテクト | Phase 4 設計時 |
| OQ-02 | 監査ログのペイロード保管方針（生 vs ハッシュ vs マスキング）。プライバシーと監査有用性の両立 | セキュリティ + コンプライアンス | Phase 5 着手前 |
| OQ-03 | 上流クラウド MCP（GitHub.com / Atlassian Cloud 等）のデータ越境リスク評価。初期に許可する上流の範囲 | 法務 + プロダクトオーナー | Phase 2 設計時（上流定義時） |
| OQ-04 | GDPR 削除権 SLA（30 日 vs 即時）と監査ログ保持要件の整合 | コンプライアンス | M2 設計時 |
| OQ-05 | Circuit Breaker Open 時のクライアント返却エラーコード（`-32001 upstream timeout` 流用 vs 専用コード新設） | アーキテクト | Phase 6 着手前 |

### 11.1 ADR 起票候補（要件確定後に起票判断）

本要件定義の承認後、以下の ADR を起票するか議論する:

| 候補 ADR | 仮タイトル | 主論点 |
|---|---|---|
| ADR 候補 D | MCP Gateway の DynamoDB テーブルを本体 / KDB から分離する | 3 系統テーブル分離方針の恒久化（本体・KDB の ADR と並列） |
| ADR 候補 E | MCP Gateway の認証を段階導入する（API キー → JWT → Azure AD） | 段階導入の理由・移行パスを恒久記録 |
| ADR 候補 F | 上流クレデンシャルを Secrets Manager に集中、Gateway タスクロール最小権限とする | 漏洩リスクとローテーション運用の根拠 |

---

## § A. 確定済み事項（既存定義より承継）

本セクションは Open Questions の確定履歴を集約する。本書の初版では、MCP Gateway について以下の事項は **CLAUDE.md / 既存 MCP プラン / ADR 0001 / 本体・KDB 要件で既に確定済み**として扱う。

| 確定事項 ID | 確定日 | 決定者 | 決定内容 | 根拠 | 影響セクション |
|---|---|---|---|---|---|
| FIX-MCP-01 | 既存 | プロダクトオーナー | 500 ユーザー上限、単一リージョン、Slack のみ（本体に整合） | CLAUDE.md 厳格スコープ制約 | § 2.3, § 4.3, § 4.7 |
| FIX-MCP-02 | 既存 | アーキテクト | Python 3.12 統一、公式 MCP SDK 採用 | ADR 0001 + MCP プラン § 4.1 | § 3.1, § 5.2 |
| FIX-MCP-03 | 既存 | アーキテクト | MCP 専用 DynamoDB テーブル `enterprise-ai-platform-mcp-gw-{env}` を新設 | MCP プラン § 4.2 + KDB § 6.5 / 本体 § 4.5 と同方針 | § 4.5, § 6.1, § 7.2 |
| FIX-MCP-04 | 既存 | アーキテクト | 名前空間 `{upstreamId}__{originalName}` 形式、起動時 Fail Fast | MCP プラン § 2.2 | § 3.3, § 5.5 |
| FIX-MCP-05 | 既存 | アーキテクト | 3 軸 ACL（user × upstream × tool）、デフォルト deny | MCP プラン § 1.3 | § 3.8 |
| FIX-MCP-06 | 既存 | アーキテクト | 認証段階導入: API キー（M1）→ JWT/Cognito（M2）→ Azure AD/OIDC（M3） | MCP プラン § 1.3 | § 3.7, § 5.3 |
| FIX-MCP-07 | 既存 | セキュリティ | 上流クレデンシャルを Secrets Manager 一元化、Gateway タスクロール最小権限 | MCP プラン § 1.3 / § 5.1 | § 3.11, § 5.7 |
| FIX-MCP-08 | 既存 | アーキテクト | デプロイは ECS Fargate（ARM64 Graviton）Multi-AZ、EKS / Lambda / EC2 / オンプレ不採用 | MCP プラン § 7.4 | § 4.2, § 5.1 |
| FIX-MCP-09 | 既存 | アーキテクト | Streamable HTTP が本番標準、HTTP+SSE は上流互換のみ、stdio は dev / 同一コンテナのみ | MCP プラン § 2.1 | § 3.2 |
| FIX-MCP-10 | 既存 | アーキテクト | Circuit Breaker パラメータ: 50% / 60s / Open 30s / Half-Open 3 成功 | MCP プラン Phase 6 完了基準 | § 3.6 |
| FIX-MCP-11 | 既存 | アーキテクト | レート制限既定 60 req/分/ユーザー、トークンバケット型 | MCP プラン Phase 6 | § 3.9 |
| FIX-MCP-12 | 既存 | アーキテクト | 本体 Phase 6a Step 3 完了で MCP Phase 4 着手可（6a 全体完了を待たない） | MCP プラン § 4.2 横断依存 | § 7.1, § 8.3 |

---

## 12. 用語集

| 用語 | 定義 |
|---|---|
| MCP | Model Context Protocol。Anthropic 提唱のクライアント ⇄ サーバー間プロトコル |
| MCP Gateway | 本書で要件定義するアグリゲーター・ゲートウェイサブシステム |
| 上流 MCP サーバー（Upstream） | Gateway がプロキシする実 MCP サーバー |
| Inbound | クライアント → Gateway 方向 |
| Upstream | Gateway → 上流 MCP 方向 |
| 名前空間 | `{upstreamId}__{originalName}` 形式 |
| 3 軸 ACL | user × upstream × tool で許可判定するアクセス制御 |
| Streamable HTTP | MCP 標準 HTTP ストリーミング（本番標準） |
| HTTP+SSE | MCP レガシートランスポート（上流互換のみ） |
| stdio | プロセス間通信（dev / 同一コンテナのみ） |
| Circuit Breaker | Closed / Open / Half-Open の 3 状態機械 |
| AuditRepository(tableName) | `packages/audit-events/` 抽象、3 系統テーブルに同一スキーマで書込 |
| Walking Skeleton | 最小限のエンドツーエンド動作を最早期に確認するパターン |
| KDB | Knowledge DB サブシステム（`docs/KNOWLEDGE_DB_REQUIREMENTS.md`） |
| OPENCLAW プラットフォーム | 本体（`docs/OPENCLAW_PLATFORM_REQUIREMENTS.md`） |
| 3 層 SOUL | OPENCLAW のアイデンティティ階層。Global（IT がロック）/ Position（部門管理者）/ Personal（従業員）の Markdown を、**上位が下位を上書きできない** マージ規則で結合する。マージ時、上位レイヤーは `CRITICAL IDENTITY OVERRIDE` ヘッダ付きで先頭にプリペンドされる。詳細は本体要件書 § 3.2 / 本体プラン § 2 |
| `CRITICAL IDENTITY OVERRIDE` | 上位 SOUL レイヤーが下位を上書きできないことを示す先頭ヘッダ。本体要件書 § 3.2 FR-SOUL-05 |

---

## 完了確認

- [x] 12 セクション + § A 確定済み事項を記述
- [x] § 1.4 で本体 OPENCLAW / KDB との関係を明示（Agent Container はクライアントとして利用しない、KDB を上流登録）
- [x] § 2 ユースケース 10 件 + 非ユースケース 9 件
- [x] § 3 機能要件を 15 サブカテゴリで網羅
- [x] § 4 非機能要件で性能 / 可用性 / スケール / セキュリティ / マルチテナント分離 / 監査 / コンプライアンス / 観測性 / コストをカバー
- [x] § 4.5 で MCP 専用テーブル分離方針を明示（本体 § 4.5 / KDB § 6.5 と整合）
- [x] § 5 で 7 個のアーキテクチャ選定比較を実施
- [x] § 6 でデータモデル要件（DynamoDB / YAML スキーマ / ACL 表現 / 監査属性 / JSON-RPC エラーコード）を列挙
- [x] § 7 で本体 / KDB / クライアント / 監査基盤 / CI/CD のインテグレーション要件を整理
- [x] § 8 で実装プランの 8 フェーズ・マイルストーン・横断依存・整合性チェックを実施
- [x] § 9 でリスク / コスト見積もり / 法的制約を整理
- [x] § 10 で機能 / 性能 / セキュリティ / 観測性の受け入れ基準を 35 件列挙
- [x] § 11 で Open Questions を 5 件に整理 + ADR 起票候補 3 件
- [x] § A で CLAUDE.md / MCP プラン / ADR 0001 / 本体・KDB 由来の既確定事項 12 件を集約
- [x] § 12 用語集
- [x] CLAUDE.md の厳格スコープ制約（500 ユーザー / 単一リージョン / OpenClaw ゼロ侵襲）を尊重
- [x] 既存ドキュメント（MCP プラン / 本体要件 / KDB 要件 / ADR 0001）との矛盾なし

---
