OpenClaw Platform 要件定義書

- **ステータス**: Draft (要承認)
- **起票日**: 2026-05-13
- **最終更新**: 2026-05-29
- **対象**: enterprise-ai-platform（エンタープライズ OpenClaw クラウドサービス本体）
- **関連ドキュメント**: なし（本書はスタンドアロン）

---

## 0. ドキュメント概要

### 0.1 目的

本ドキュメントは、**エンタープライズ OpenClaw クラウドサービス本体**（以下「本体」または「OPENCLAW プラットフォーム」）の **要件**（What / Why / 制約 / 受け入れ基準）を定義する。

本書は要件レベル（What / Why / 制約 / 受け入れ基準）に閉じた **スタンドアロン要件定義書**である。

### 0.2 想定読者

- プロダクトオーナー / アーキテクト（スコープ承認・意思決定）
- セキュリティ / コンプライアンス担当（多層防御・データ越境・監査要件）
- 実装プラン執筆者・実装エンジニア
- 運用 / SRE（観測性・コスト・SLO）
- 経営層（プロジェクトスコープと予算の確認）

### 0.3 関連ドキュメント

- OpenClaw OSS 本体ソース（参照のみ、フォーク・パッチ禁止）: `openclaw/` シンボリックリンク先 → [github.com/openclaw/openclaw](https://github.com/openclaw/openclaw)
- 参考実装（コピー禁止・設計参考のみ）: `sample/` シンボリックリンク先（AWS Samples）
- リポジトリ運用指針: [`CLAUDE.md`](../CLAUDE.md)

### 0.4 用語定義

| 用語 | 定義 |
|---|---|
| OPENCLAW | 本プラットフォームに内包される AI エージェントランタイム（OpenClaw `2026.3.24` 固定） |
| プラットフォーム | 本書で要件定義する OPENCLAW を中核としたエンタープライズ AI エージェント基盤全体 |
| Tenant Router | マルチテナントルーティングの心臓部。`channel + raw_user_id` → `emp_id` → runtime を解決 |
| Bedrock H2 Proxy | Bedrock API への HTTP/2 ストリーミングプロキシ（SigV4 署名 + Guardrails 介入） |
| Agent Container | OpenClaw を内包する Docker コンテナ。AgentCore / ECS Fargate の両方で実行 |
| Workspace Assembler | S3 から Global / Position / Personal の Markdown を取得しマージするモジュール |
| AgentCore | AWS Bedrock AgentCore Runtime。Firecracker microVM ベースのサーバーレス実行環境 |
| 3 層 SOUL | Global（IT がロック）/ Position（部門管理者）/ Personal（従業員）の Markdown マージ機構 |
| 4 ティアランタイム | Standard / Restricted / Engineering / Executive。ティアごとに ECR イメージ・IAM ロール・Guardrails が異なる |
| 5 層多層防御 | L1 SOUL ルール / L2 ツール許可 / L3 IAM / L4 コンピュート分離 / L5 Bedrock Guardrails |
| Always-On | ECS Fargate で常時稼働するエージェントモード（コールドスタートゼロ） |
| デジタルツイン | 本人不在時に応答する公開リンク型エージェント |

---

## 1. 背景と目的

### 1.1 背景

エンタープライズ組織は「全社員に同じ Bot」ではなく、ポジション（職種）ごとに固有のアイデンティティ・ツール権限・知識を持った AI エージェントを必要としている。一方、汎用 SaaS（ChatGPT Enterprise / Claude for Work 等）には以下の制約がある:

| 制約 | 影響 |
|---|---|
| IT 統制の限界（グローバル / 部門 / 個人の 3 層分離が不完全） | コンプライアンス・ガバナンス要件を満たせない |
| マルチテナント分離が論理層のみ（コンピュート境界なし） | Firecracker / Fargate Task レベルの分離を要求する業界（金融・医療等）に不適合 |
| 監査証跡の粒度不足 | SOC2 / ISO 27001 監査要件を満たせない |
| ベンダーロックイン | モデル選定・データ越境制御に制約 |

`sample/` の参考実装（OpenClaw on AWS with Bedrock — Enterprise）は、これらの課題を解く設計思想（3 層 SOUL / 4 ティアランタイム / 5 層セキュリティ / サーバーレス + 常時稼働ハイブリッド）を実装している。本プロジェクトは **コードをコピーせず**、その設計思想を独自実装する。

### 1.2 目的

以下の 6 点を満たすエンタープライズ AI エージェントクラウドサービスを定義する:

1. **役割固有の AI エージェント**: ポジションごとに固有のアイデンティティ・ツール権限・知識を持つエージェントを配布
2. **組織駆動の自動プロビジョニング**: 入社・異動・退職を組織図と連動させ、エージェントが自動再構成
3. **IT 統制可能**: グローバル / 部門 / 個人の 3 層 SOUL でポリシーをマージし、上位レイヤーは下位レイヤーを上書きできる
4. **マルチテナント分離**: Firecracker microVM / Fargate Task レベルのコンピュート分離 + IAM 境界 + Guardrails
5. **マルチチャネル**: Web Portal / Slack / 公開デジタルツインリンク
6. **完全な監査証跡**: 全ツールコール・SOUL 変更・Guardrails ブロックを永続化

### 1.3 既存類似サービスとの差分

| 観点 | ChatGPT Enterprise / Claude for Work | **本プラットフォーム** |
|---|---|---|
| エージェント単位 | 全社員に同一 Bot（カスタム GPT 等は個人裁量） | 組織図駆動で **ポジション × 個人** にエージェントを 1:1 バインディング |
| IT 統制 | 管理画面でモデル選定・データ保持制御 | **3 層 SOUL** で IT / 部門 / 個人のポリシーを階層マージ |
| マルチテナント | 論理層分離（テナント ID） | **Firecracker microVM / Fargate Task** によるコンピュート境界 |
| ツール権限 | プラグイン単位（ユーザーが ON/OFF） | **ポジション × ティア** で IT が事前制御（`PERM#{posId}`） |
| 監査 | ログ閲覧 | DynamoDB + S3 Object Lock + Athena で **永続化 + 全文検索** |
| 公開リンク | なし / 限定的 | **デジタルツイン**（本人不在時の AI 代理応答） |
| デプロイ | SaaS のみ | **自社 AWS アカウント**で完結（VPC 内、データ越境ゼロ） |

### 1.4 ビジネス価値（KPI 候補）

| KPI | 測定方法 | 目標値（500 ユーザー時） |
|---|---|---|
| 1 ユーザーあたり AWS コスト | CloudWatch Billing | **< $5 / 月** |
| ウォーム応答 p95 | CloudWatch | < 5s |
| コールドスタート p95 | CloudWatch | < 15s |
| 自動プロビジョニング成功率 | 監査ログ | 99.5%+ |
| マルチテナント漏洩インシデント | セキュリティレビュー | **0 件**（必須） |
| 月次 Active Employee 数 | 監査ログ | 500 上限 |
| Portal Chat NPS | 内部アンケート | +20 以上 |
| 監査クエリ応答時間 p95 | Admin Console | < 2 秒 |
| デプロイ自動化（`bash deploy.sh` 相当） | 実測 | < 30 分 |

---

## 2. ユースケース / アクター

### 2.1 主要アクター

| アクター | 役割 | プラットフォームとの関わり |
|---|---|---|
| 従業員（Employee） | Portal / Slack から AI に質問・依頼 | エージェントを介してツール実行・知識検索 |
| 部門管理者（Manager） | 自部門のエージェントポリシー所有者 | Position SOUL 編集、ツール許可リスト管理、自部門監査閲覧 |
| IT 管理者（Admin） | 全社ポリシー所有者、コンプライアンス責任 | Global SOUL 編集、ティア定義、Guardrails 設定、全社監査閲覧 |
| 経営層（Executive） | Executive ティアエージェント利用者 | 機密度の高い分析・意思決定支援 |
| エンジニア（Engineering ティア） | コード生成・実行を伴うエージェント利用者 | コードインタプリタ・shell ツールを許可されたエージェント利用 |
| デジタルツイン参照者（外部） | 本人不在時に公開リンクを訪れる外部関係者 | 公開ツインエージェントとの会話 |
| 監査担当 | コンプライアンス検証 | 監査ログ全文検索、Insights 閲覧 |
| プラットフォーム運用者（SRE） | システム運用 | CloudWatch / X-Ray / コスト監視 |

### 2.2 主要ユースケース（合計 12 件）

| # | ユースケース | アクター | 主経路 |
|---|---|---|---|
| UC-01 | Portal Chat（汎用 Q&A・業務依頼） | 従業員 → Portal | Tenant Router → AgentCore Standard |
| UC-02 | Slack DM での AI 依頼 | 従業員 → Slack | IM Adapter → Tenant Router → Runtime |
| UC-03 | 経営層分析（機密データ参照） | 役員 → Portal | Tenant Router → Always-On Executive |
| UC-04 | エンジニアのコード生成・実行 | エンジニア → Portal | Tenant Router → Engineering ティア（shell 許可） |
| UC-05 | コンプライアンス制限部門の利用 | 法務 / 監査部門 → Portal | Tenant Router → Restricted（ツール最小限） |
| UC-06 | 新入社員の自動エージェント生成 | IT 管理者 → 組織 API（または HRIS 連携） | Auto-Provisioning → S3 ワークスペース生成 + DynamoDB エントリ |
| UC-07 | 異動時のエージェント再構成 | 部門管理者 → Admin Console | Position 切替 → Workspace Assembler が次回コールドスタートで反映 |
| UC-08 | 退職時のエージェント停止 | IT 管理者 → Admin Console | DynamoDB エントリ deactivation + S3 ワークスペース冷却 |
| UC-09 | SOUL 編集と即時反映 | 部門管理者 → Admin Console | SOUL Editor → `CONFIG#global-version` bump → 5 分以内に全コンテナ反映 |
| UC-10 | 監査検索（誰がいつ何を実行したか） | 監査担当 → Admin Console | DynamoDB `AUDIT#` + Athena 長期保管 |
| UC-11 | デジタルツイン公開 | 従業員 → Portal でツイン有効化 → 外部 | API Gateway → twin プレフィックス Runtime |
| UC-12 | Always-On 切替（特定エージェントを常駐化） | 部門管理者 → Admin Console | `PATCH /agents/{id}/runtime-mode` → ECS Service + Cloud Map |

### 2.3 非ユースケース（明示的スコープ外）

- **EU / APAC のデータ主権要件**: 当該リージョン内処理が必須のテナントは対象外（AgentCore 非対応リージョン）
- **EKS 採用**: ECS Fargate に統一。EKS の運用負債は本規模で正当化されない
- **マルチリージョン展開**: 単一リージョン強制。`region_override` / Global Table 禁止
- **5,000 ユーザー超のスケール**: 上限 500 ユーザー
- **Slack 以外の IM**: Teams / Telegram / Discord / Feishu / WhatsApp はスコープ外
- **エージェント自身のセルフトレーニング / Fine-tuning**: Bedrock の標準モデル + SOUL/RAG で完結
- **オンプレミス対応**: AWS 専用。クラウドフォーミングへの依存を許容
- **モバイルアプリ**: Portal Web のみ。レスポンシブ対応はする
- **公開 API（B2B SaaS としての販売）**: 自社ホスト前提

---

## 3. 機能要件

### 3.1 組織モデル管理

| 要件 ID | 要件 | 決定 / 未決定 |
|---|---|---|
| FR-ORG-01 | 組織（Organization）/ 部門（Department）/ ポジション（Position）/ 従業員（Employee）の CRUD | 決定 |
| FR-ORG-02 | 階層構造（部門 → サブ部門 → ポジション → 従業員）の表現 | 決定 |
| FR-ORG-03 | 従業員 1:1 でエージェントを自動生成（Auto-Provisioning Hook） | 決定 |
| FR-ORG-04 | 異動時に Position 切替で SOUL を再構成（コードコピーなし） | 決定 |
| FR-ORG-05 | 退職時のエージェント soft-delete（30 日後 hard-delete） | 決定 |
| FR-ORG-06 | HRIS（人事システム）連携での自動同期 | **未決定（OQ-01）**: MVP に含めるか / 連携先 SaaS |
| FR-ORG-07 | 組織変更履歴の保持 | 決定（監査ログとして） |

### 3.2 3 層 SOUL

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-SOUL-01 | Global SOUL（IT がロック、全エージェントの先頭に強制プリペンド） | 決定 |
| FR-SOUL-02 | Position SOUL（部門管理者が編集、Position 単位） | 決定 |
| FR-SOUL-03 | Personal SOUL（従業員が編集、個人カスタマイズ） | 決定 |
| FR-SOUL-04 | マージ規則: Global > Position > Personal。下位は上位を **上書き不可** | 決定 |
| FR-SOUL-05 | `CRITICAL IDENTITY OVERRIDE` ヘッダで上位プリペンドを明示 | 決定 |
| FR-SOUL-06 | SOUL 変更の即時反映（再デプロイ不要、`CONFIG#global-version` bump で 5 分以内反映） | 決定 |
| FR-SOUL-07 | SOUL 変更の差分プレビュー（管理者が編集前に確認） | 決定 |
| FR-SOUL-08 | SOUL のバージョン履歴（最新 20 世代保持、ロールバック可） | 決定 |
| FR-SOUL-09 | SOUL マージ結果のスナップショットテスト（ゴールデン 20 件以上） | 決定（CI 必須） |

### 3.3 4 ティアランタイム

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-TIER-01 | Standard / Restricted / Engineering / Executive の 4 ティア | 決定 |
| FR-TIER-02 | ティアごとに独立した ECR イメージ | 決定 |
| FR-TIER-03 | ティアごとに独立した IAM ロール（最小権限） | 決定 |
| FR-TIER-04 | ティアごとに独立した Bedrock Guardrails 割当 | 決定 |
| FR-TIER-05 | Position → Tier のマッピング（DynamoDB `CONFIG#routing`） | 決定 |
| FR-TIER-06 | クロスティア参照禁止（Restricted → Executive KB / ツールは拒否） | 決定 |
| FR-TIER-07 | 1 設定変更でティア切替可能（Position の `tier` 属性更新のみ） | 決定 |

### 3.4 ツール許可リスト（Permission Profile）

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-PERM-01 | Position 単位のツール許可リスト（DynamoDB `PERM#{posId}`） | 決定 |
| FR-PERM-02 | `allowedRoles` / `blockedRoles` の両方をサポート | 決定 |
| FR-PERM-03 | デフォルト deny（明示許可がないツールは実行不可） | 決定 |
| FR-PERM-04 | 許可リスト変更の即時反映（5 分以内） | 決定 |
| FR-PERM-05 | 既定ツールセット（Standard）: 検索 / メモ / 汎用 RAG ツール | 決定 |
| FR-PERM-06 | Engineering 追加: shell / コードインタプリタ / Git 操作 | 決定 |
| FR-PERM-07 | Restricted 制限: 外部 HTTP / ファイル書込を禁止 | 決定 |
| FR-PERM-08 | Executive 追加: 全社 KB / 機密分類アクセス | 決定 |

### 3.5 ゲートウェイ / Tenant Router

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-GW-01 | `/route` エンドポイントで `channel + raw_user_id` → `emp_id` 解決 | 決定 |
| FR-GW-02 | セッション ID プレフィックス: `emp__`, `pgnd__`, `twin__`, `admin__` を発行 | 決定 |
| FR-GW-03 | 3 ティアルーティング: (1) Always-On オーバーライド (2) ポジションルール (3) デフォルト AgentCore | 決定 |
| FR-GW-04 | テナント解決の 5 分キャッシュ | 決定 |
| FR-GW-05 | Bedrock H2 Proxy: HTTP/2 ストリーミング + SigV4 署名し直し | 決定 |
| FR-GW-06 | Guardrails 介入レイヤー（Input/Output で `apply_guardrail`） | 決定 |
| FR-GW-07 | 解決失敗時の安全側フェイル（不明テナントは拒否） | 決定 |

### 3.6 Agent Container（OpenClaw 実行基盤）

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-AC-01 | OpenClaw `2026.3.24` 固定（アップグレード禁止） | 決定 |
| FR-AC-02 | ARM64 Graviton ベースイメージ | 決定 |
| FR-AC-03 | コールドスタート時に Workspace Assembler が S3 から SOUL/KB/Skills を取得 | 決定 |
| FR-AC-04 | セッション ID プレフィックスで `SESSION_CONTEXT.md` を書き分け | 決定 |
| FR-AC-05 | Memory 同期: 60 秒ウォッチドッグで `memory/` を S3 ライトバック | 決定 |
| FR-AC-06 | 危険パターン拒否（`rm -rf`, SQL injection 等） | 決定 |
| FR-AC-07 | 全ツールコール / 拒否を `AUDIT#` に書込 | 決定 |
| FR-AC-08 | `/invocations` HTTP エンドポイントを AgentCore / ECS 両対応 | 決定 |
| FR-AC-09 | OpenClaw ソースのフォーク・パッチ禁止（ゼロ侵襲） | 決定 |

### 3.7 ランタイム実行モード

| 要件 ID | 要件 | 決定 / 未決定 |
|---|---|---|
| FR-RT-01 | AgentCore Runtime（Firecracker microVM、サーバーレス）が既定 | 決定 |
| FR-RT-02 | ECS Fargate 常時稼働モード（M2 で導入） | 決定 |
| FR-RT-03 | Always-On 切替 API（`PATCH /agents/{id}/runtime-mode`） | 決定 |
| FR-RT-04 | Service Discovery: AWS Cloud Map（DNS TTL 10 秒） | 決定 |
| FR-RT-05 | EFS マウントで永続ワークスペース | 決定 |
| FR-RT-06 | コールドスタート短縮: AgentCore Session Storage で 2〜3 秒 | 決定 |
| FR-RT-07 | Provisioned Concurrency の採用可否 | **未決定（OQ-02）**: AgentCore のコールドスタートが p95 < 15s を満たさない場合の対応 |

### 3.8 認証・認可

| 要件 ID | 要件 | 決定 / 未決定 |
|---|---|---|
| FR-AUTH-01 | MVP: Cognito User Pool（Employee ID + Password） | 決定 |
| FR-AUTH-02 | M3: Azure AD / SAML 連携 | 決定 |
| FR-AUTH-03 | MFA 必須（管理者・Executive ティア） | 決定 |
| FR-AUTH-04 | 3-role RBAC（Admin / Manager / Employee） | 決定 |
| FR-AUTH-05 | Manager は自部門のみ操作可（BFS ロールアップ） | 決定 |
| FR-AUTH-06 | JWT セッション（有効期限 1 時間、Refresh Token 24 時間） | 決定 |
| FR-AUTH-07 | Slack OAuth（IM 統合時） | 決定 |
| FR-AUTH-08 | 外部 IdP（Okta / Google Workspace）対応 | **未決定（OQ-03）**: Azure AD 以外のサポート範囲 |

### 3.9 IM 統合（Slack のみ）

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-IM-01 | Slack Bolt SDK（Python）ベースの IM Adapter | 決定 |
| FR-IM-02 | DM / メンションでのトリガー | 決定 |
| FR-IM-03 | スレッド継続性の維持（Slack ts → セッション） | 決定 |
| FR-IM-04 | Slack ユーザー ID → emp_id マッピング（`MAPPING#slack__{userId}`） | 決定 |
| FR-IM-05 | Teams / Telegram / Discord / Feishu / WhatsApp はスコープ外 | 決定（明示） |
| FR-IM-06 | 将来拡張に備えた `core/` インタフェース（薄い抽象、実装は `slack/` のみ） | 決定 |

### 3.10 デジタルツイン

| 要件 ID | 要件 | 決定 / 未決定 |
|---|---|---|
| FR-TW-01 | 従業員が Portal から公開リンクを発行 / 失効 | 決定 |
| FR-TW-02 | 公開リンクは API Gateway 経由でレート制限 | 決定 |
| FR-TW-03 | twin プレフィックス Runtime はツール権限を絞った Restricted 相当 | 決定 |
| FR-TW-04 | ツインアクセスのメタデータ取得（IP / UA / 訪問頻度） | 決定 |
| FR-TW-05 | 外部公開時のデータ越境制御（外部から内部 KB / 社員情報を引き出せない） | 決定 |
| FR-TW-06 | キャプチャ / 録画リンク機能 | **未決定**: スコープに含めるか |

### 3.11 管理機能（Admin Console）

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-ADM-01 | 組織図エディタ（部門 / ポジション / 従業員） | 決定 |
| FR-ADM-02 | SOUL エディタ（Global / Position / Personal） | 決定 |
| FR-ADM-03 | ツール許可リスト管理 UI | 決定 |
| FR-ADM-04 | ランタイムモード切替（Serverless ↔ Always-On） | 決定 |
| FR-ADM-05 | 監査検索（時系列 / テナント / イベントタイプ / 全文検索） | 決定 |
| FR-ADM-06 | 使用量集計（ティア別 / Position 別 / 月次トークン消費） | 決定 |
| FR-ADM-07 | コストダッシュボード（CloudWatch Billing 統合） | 決定 |
| FR-ADM-08 | Guardrails 設定 UI | 決定（M3） |
| FR-ADM-09 | ページ構成: Admin Console 8 ページ + Portal 5 ページ（合計 13 ページ） | 決定 |

### 3.12 Portal（従業員向け UI）

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-PRT-01 | MVP: Portal Chat 1 画面のみ | 決定 |
| FR-PRT-02 | M2 以降: マルチエージェント切替 / 履歴 / プロファイル / ツイン管理 | 決定 |
| FR-PRT-03 | ストリーミング応答表示 | 決定 |
| FR-PRT-04 | ファイル添付（PDF / 画像、サイズ上限要確定） | **未決定（OQ-04）**: サイズ上限・MIME 範囲 |
| FR-PRT-05 | スレッド継続性（過去 30 日間） | 決定 |
| FR-PRT-06 | 多言語対応（日本語 / 英語） | 決定 |

### 3.13 自動プロビジョニング

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-PROV-01 | 従業員作成時に (1) DynamoDB Employee エントリ (2) DynamoDB Agent + 1:1 バインディングエントリ (3) S3 ワークスペース (4) DynamoDB 監査エントリ の 4 リソースを一括作成 | 決定 |
| FR-PROV-02 | 部分失敗時の補償トランザクション（DynamoDB Streams ベース、収束 SLA: 5 分以内、CI で自動検証） | 決定 |
| FR-PROV-03 | プロビジョニング完了通知（管理者へメール / Slack） | 決定 |
| FR-PROV-04 | バッチプロビジョニング（CSV インポート） | 決定 |

### 3.14 監査

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-AUD-01 | DynamoDB `AUDIT#` で直近 24〜72h のホット参照 | 決定 |
| FR-AUD-02 | DynamoDB Streams → Firehose → S3 Object Lock の長期保管（MVP から構築） | 決定 |
| FR-AUD-03 | Athena テーブルで S3 監査ログを全文検索 | 決定 |
| FR-AUD-04 | OpenSearch 連携（拡張、M3 以降） | 決定 |
| FR-AUD-05 | 記録対象: ツールコール / SOUL 変更 / Guardrails ブロック / 権限拒否 / プロビジョニング / ログイン | 決定 |
| FR-AUD-06 | 監査書込はリポジトリパターン抽象でテーブル名を注入可能にし、複数の書込元から同一スキーマで書き込めるよう設計 | 決定 |

---

## 4. 非機能要件

### 4.1 性能

| 指標 | 目標値 | 計測方法 |
|---|---|---|
| コールドスタート 中央値 | < 10s | 10 連続実測 |
| コールドスタート p95 | < 15s | 同上 |
| ウォーム応答 中央値 | < 3s | 同上 |
| ウォーム応答 p95 | < 5s | 同上 |
| Bedrock H2 Proxy 透過オーバーヘッド p95 | < 50ms | 100 リクエスト実測 |
| Tenant Router 解決 p95 | < 50ms | 同上 |
| Always-On 切替後の応答 p95 | < 2s | 同上 |
| Always-On Cloud Map register p95 | < 30s | 切替トリガから実測 |
| Always-On Cloud Map register → DNS 安定 p95 | < 45s | 切替トリガから実測 |
| Always-On deregister → IP 不返却 p95 | < 20s | 同上 |
| 監査検索（直近 24h、単一テナント） p95 | < 2s | Admin Console 実測 |
| 監査検索（Athena、過去 90 日） p95 | < 30s | 同上 |

### 4.2 可用性

| 指標 | 目標値 |
|---|---|
| Portal API SLO | 99.5%（500 ユーザー時） |
| Tenant Router SLO | 99.9%（クリティカルパス） |
| Bedrock H2 Proxy SLO | 99.5% |
| Always-On 成功率 | 99.5%+（10 分 1 RPS 継続負荷） |
| Multi-AZ | 必須（全 ECS サービス、3 AZ デプロイ） |
| RTO | < 1 時間 |
| RPO | < 24 時間（S3 + DynamoDB PITR で達成） |

### 4.3 スケーラビリティ（500 ユーザー上限内）

| 指標 | **MVP（M1: 50 ユーザー）** | M2（200〜300 ユーザー） | 最終目標（M3: 500 ユーザー） |
|---|---|---|---|
| ユーザー数 | 50 | 300 | 500 |
| 部門数 | 1〜3 | 3〜7 | 5〜10 |
| エージェント数 | 50 | 300 | 500 |
| Concurrent Sessions（ピーク） | 10 | 50 | 100 |
| 月間メッセージ数 | 10,000 | 50,000 | 200,000 |
| ティア | Standard のみ | Standard + Engineering + Executive | 4 ティア全部 |
| 常時稼働エージェント数 | 0 | 5〜10 | 20〜30 |
| 監査イベント / 月 | 100K | 500K | 2M |

### 4.4 セキュリティ（5 層多層防御）

| 層 | 対策 | Trust Boundary |
|---|---|---|
| L1 SOUL ルール | Markdown プロンプト規約。**ガバナンス層、Trust Boundary ではない**（プロンプトインジェクションで破られうる） | ❌ |
| L2 ツール許可リスト | DynamoDB `PERM#{posId}` | △（モデル遵守に依存） |
| L3 IAM（ティア別） | 4 つの IAM ロール、Resource ARN 厳格指定 | ✅ |
| L4 コンピュート分離 | Firecracker microVM / Fargate Task 単位 | ✅ |
| L5 Bedrock Guardrails | ティアごと割当（PII / Topic / Injection フィルタ） | ✅ |

> **重要**: L1 はガバナンス層であり Trust Boundary ではない。SLA・セキュリティ評価は L3 以降のみを保証範囲とする。

| 要件 ID | 要件 | 決定 / 未決定 |
|---|---|---|
| NFR-SEC-01 | 保存時暗号化（S3 SSE-KMS、DynamoDB 暗号化、EFS 暗号化） | 決定 |
| NFR-SEC-02 | 転送時暗号化（TLS 1.3、ALB ↔ ECS は内部 TLS） | 決定 |
| NFR-SEC-03 | KMS CMK のキー管理（テナント / システムで分離） | 決定 |
| NFR-SEC-04 | パブリックポートゼロ（ALB のみ Public、ECS / DynamoDB / S3 は Private Subnet + VPC Endpoint） | 決定 |
| NFR-SEC-05 | Secrets Manager / SSM のローテーション（90 日） | 決定 |
| NFR-SEC-06 | OWASP Top 10 対策の CI 検証（Semgrep / Checkov / Trivy） | 決定 |
| NFR-SEC-07 | IAM Access Analyzer で過剰権限ゼロ | 決定 |
| NFR-SEC-08 | gitleaks による秘密漏洩検出 | 決定 |
| NFR-SEC-09 | WAF ルール（OWASP Core Rule Set + IP レート制限） | 決定 |
| NFR-SEC-10 | DDoS 保護（CloudFront + AWS Shield Standard） | 決定 |

### 4.5 マルチテナント分離

> 本プロジェクトは単一組織前提だが、**部門間・ティア間の論理分離 + コンピュート分離**は必須。

| 要件 ID | 要件 | 決定 |
|---|---|---|
| NFR-TEN-01 | テナント ID 改ざんテスト 100 ケース以上 を CI 必須 | 決定 |
| NFR-TEN-02 | 偽プレフィックス注入テスト（`twin__` 偽装による越権） | 決定 |
| NFR-TEN-03 | 並行リクエスト混線テスト（プロパティベース fuzz） | 決定 |
| NFR-TEN-04 | クロスティア参照拒否テスト（Restricted → Executive KB） | 決定 |
| NFR-TEN-05 | テナントごと S3 prefix 分離（`workspaces/{empId}/`） | 決定 |
| NFR-TEN-06 | テナントごと IAM Session Policy（AgentCore コールドスタート時） | 決定 |

### 4.6 監査

| 要件 ID | 要件 | 決定 |
|---|---|---|
| NFR-AUD-01 | 全イベントを 99.9%+ で `AUDIT#` に書込（失敗時はリトライ + DLQ） | 決定 |
| NFR-AUD-02 | 記録項目: timestamp, actor_id, action, resource, before/after, request_id, session_id | 決定 |
| NFR-AUD-03 | 監査ログ改ざん防止（S3 Object Lock + KMS） | 決定 |
| NFR-AUD-04 | 保持期間: DynamoDB 72h ホット + S3 Object Lock 7 年（コンプライアンス要件） | 決定 |
| NFR-AUD-05 | Athena 経由のテナント別検索が 30 秒以内 | 決定 |
| NFR-AUD-06 | 監査ログのアクセス監査（誰が監査ログを閲覧したか） | 決定 |

### 4.7 コンプライアンス

| 要件 ID | 要件 | 決定 / 未決定 |
|---|---|---|
| NFR-CMP-01 | 単一リージョン強制（AgentCore 対応の us-east-1 or us-west-2） | 決定 |
| NFR-CMP-02 | クロスリージョン構成禁止（Global Table / レプリケーション禁止） | 決定 |
| NFR-CMP-03 | GDPR 削除権の SLA | **未決定（OQ-05）**: 30 日 vs 即時 |
| NFR-CMP-04 | SOC2 Type II 準拠（M4 で取得目標） | **未決定**: 認証取得スコープ |
| NFR-CMP-05 | ISO 27001 準拠 | **未決定**: 同上 |
| NFR-CMP-06 | 個人情報保護法（日本）への準拠 | 決定 |
| NFR-CMP-07 | 監査ログ提出 API（規制当局向け） | **未決定**: 要否 |

### 4.8 観測性

| 要件 ID | 要件 | 決定 |
|---|---|---|
| NFR-OBS-01 | CloudWatch メトリクス（QPS / レイテンシ / エラー率 / コスト） | 決定 |
| NFR-OBS-02 | X-Ray 分散トレーシング（Portal → Tenant Router → Runtime → Bedrock） | 決定 |
| NFR-OBS-03 | 構造化ログ（`structlog`、JSON 形式） | 決定 |
| NFR-OBS-04 | エラー予算: 99.5% SLO に対し月間 < 3.65 時間 | 決定 |
| NFR-OBS-05 | フロントエンドエラー（Sentry） | 決定 |
| NFR-OBS-06 | PagerDuty 連携（CRITICAL アラート） | 決定 |
| NFR-OBS-07 | コストダッシュボード（ティア別 / 部門別） | 決定 |

### 4.9 コスト

> 本表はサマリ。詳細内訳は § 9.3 を参照。

| 指標 | MVP（M1: 50 ユーザー） | 最終（M3: 500 ユーザー） |
|---|---|---|
| 1 ユーザーあたり月額 | < $10 | **< $5**（成功基準、500 × $5 = $2,500） |
| 月次合計 | $255〜$435 | $2,090〜$3,290 |
| 成功基準目標 | — | **$2,500**（500 × $5、M3 のみ） |

> 500 ユーザー時の上振れ（$3,290）は目標 $2,500 を超える可能性がある。詳細リスクと吸収方針は § 9.3 を参照。

---

## 5. アーキテクチャ方針（要件レベルの選定比較）

> 本セクションは「採用方針の比較と決定理由」のみ。実装プランは別途。

### 5.1 ランタイムプラットフォーム

| 選択肢 | スケール | 運用負債 | コールドスタート | 評価 |
|---|---|---|---|---|
| **A. Bedrock AgentCore + ECS Fargate（ハイブリッド）** | 500 ユーザーに十分 | 中（マネージド寄り） | AgentCore: 数秒〜10s / Fargate: 0（常時稼働） | **採用** |
| B. ECS Fargate のみ | 中 | 中 | 0（ただし常時コスト） | コスト過大 |
| C. EKS | 大 | 高 | カスタム可 | スコープ外（500 ユーザー前提） |
| D. Lambda のみ | 中 | 低 | 中 | OpenClaw の長時間実行と相性悪 |

**決定: A**。AgentCore（Firecracker microVM、サーバーレス）を既定、Always-On 要件のみ ECS Fargate を併用。

### 5.2 マルチテナント分離戦略

| 選択肢 | 分離レベル | 評価 |
|---|---|---|
| A. 論理分離のみ（テナント ID + IAM Session Policy） | 中 | 金融・医療要件に不適合 |
| **B. コンピュート分離（Firecracker / Fargate Task 単位）+ IAM + Guardrails** | 高 | **採用** |
| C. アカウント分離（テナントごと AWS アカウント） | 最高 | 500 ユーザー前提では過剰、運用負債大 |

**決定: B**。`sample/` の参考実装の思想を継承。Firecracker / Fargate Task が L4 Trust Boundary。

### 5.3 SOUL マージ機構

| 選択肢 | 評価 |
|---|---|
| A. テンプレートエンジン（Jinja / Handlebars） | 上書き禁止の制約表現が複雑 |
| **B. プリペンド方式（上位レイヤーを先頭に追加、`CRITICAL IDENTITY OVERRIDE` ヘッダ）** | **採用**。シンプル、上書き不可が自然 |
| C. JSON Merge Patch / JSON Patch | Markdown には不向き |

**決定: B**。

### 5.4 ゲートウェイ実装言語

| 選択肢 | 性能 | 運用負債 | 評価 |
|---|---|---|---|
| A. Node.js / Hono | 高（HTTP/2 ネイティブ） | 低 | 性能優位だが言語が分散 |
| **B. Python 3.12 + httpx + h2 + anyio** | 中〜高 | 低 | **採用**。Bedrock Proxy も SigV4 が boto3 と同言語で揃う |
| C. Go | 高 | 中（学習コスト） | スタック分散 |

**決定: B**。バックエンドサービスは Python 3.12 に統一する。

### 5.5 監査長期保管

| 選択肢 | 検索性能 | コスト | 改ざん耐性 | 評価 |
|---|---|---|---|---|
| **A. DynamoDB（72h ホット）+ S3 Object Lock + Athena** | 中（Athena） | 低 | 高 | **採用** |
| B. OpenSearch 永続 | 高 | 高 | 中 | コスト過大 |
| C. CloudWatch Logs Insights | 中 | 中 | 低 | 改ざん耐性不足 |

**決定: A**。M3 以降で OpenSearch を全文検索拡張として追加し B を補完する。

### 5.6 認証 IdP

| 選択肢 | エンタープライズ親和 | 評価 |
|---|---|---|
| **A. Cognito（MVP）→ Azure AD/SAML（M3）** | 高 | **採用** |
| B. Auth0 / Okta（マネージド） | 高 | コスト・ベンダー依存 |
| C. Keycloak（セルフホスト） | 中 | 運用負債大 |

**決定: A**。M1 は Cognito で迅速立ち上げ、M3 で Azure AD/SAML 連携。

---

## 6. データモデル要件

> 要件レベルで主要属性のみ列挙する。詳細スキーマ・GSI 設計・アクセスパターン仕様は実装プランで定義する。

### 6.1 DynamoDB シングルテーブル（本体）

テーブル名: `enterprise-ai-platform-{env}`

| パターン | PK | SK | 用途 |
|---|---|---|---|
| Organization | `ORG#{orgId}` | `META` | 組織メタ |
| Department | `ORG#{orgId}` | `DEPT#{deptId}` | 部門 |
| Position | `DEPT#{deptId}` | `POS#{posId}` | ポジション |
| Employee | `POS#{posId}` | `EMP#{empId}` | 従業員 |
| Agent | `EMP#{empId}` | `AGENT` | 1:1 エージェントバインディング |
| Permission Profile | `PERM#{posId}` | `META` | ツール許可リスト |
| Routing Config | `CONFIG#routing` | `RULE#{ruleId}` | 3 ティアルーティングルール |
| Global Version | `CONFIG#global-version` | `META` | SOUL 反映タイマー |
| Tenant Mapping | `MAPPING#{channel}__{userId}` | `META` | チャネル ID → emp_id |
| Audit Event | `AUDIT#{tenantId}#{ts}` | `EVENT#{eventId}` | 監査イベント（72h TTL） |

GSI 設計（アクセスパターン別の検索キー）は実装プランで定義する。

### 6.2 S3 プレフィックス構造

```
s3://enterprise-ai-platform-{env}/
├── workspaces/{empId}/           # 従業員別ワークスペース
│   ├── SOUL.md
│   ├── IDENTITY.md
│   ├── SESSION_CONTEXT.md
│   ├── memory/
│   ├── skills/
│   └── tools/
├── _shared/
│   ├── soul/
│   │   ├── global/                # Global SOUL（IT 管理）
│   │   └── positions/{posId}/     # Position SOUL（部門管理者）
│   └── skills/                    # 全社共通スキル
├── frontend-assets/               # 静的アセット
└── audit-archive/                 # 監査長期保管（Object Lock）
```

### 6.3 ティア定義

| ティア | ECR イメージ | IAM ロール | Guardrails | 主な許可ツール |
|---|---|---|---|---|
| Standard | `agent-container-standard` | `role-standard` | `gr-standard` | 検索 / メモ / 汎用 RAG ツール |
| Restricted | `agent-container-restricted` | `role-restricted` | `gr-restricted` | 検索 / メモ（外部 HTTP / ファイル書込禁止） |
| Engineering | `agent-container-engineering` | `role-engineering` | `gr-engineering` | + shell / コード実行 / Git |
| Executive | `agent-container-executive` | `role-executive` | `gr-executive` | + 全社 KB / 機密分類 |

### 6.4 セッション ID プレフィックス

| プレフィックス | 用途 | アクセスパス |
|---|---|---|
| `emp__` | 従業員通常セッション | Portal / Slack |
| `pgnd__` | プレイグラウンド（テスト用） | Admin Console |
| `twin__` | デジタルツイン公開リンク | API Gateway 公開 |
| `admin__` | 管理者操作 | Admin Console |

---

## 7. インテグレーション要件

### 7.1 Bedrock 関連サービス連携

| 接続先 | 接続方法 | 備考 |
|---|---|---|
| Bedrock Foundation Models | Bedrock H2 Proxy 経由（SigV4 + ストリーミング） | Nova / Claude などのモデル選定は実装プランで確定 |
| Bedrock Guardrails | Bedrock H2 Proxy ミドルウェアで `apply_guardrail` を呼び出し | 本格稼働は後フェーズ、初期は最小ルールセット |

> Bedrock Knowledge Bases は本書スコープ外（本体エージェントは直接呼び出さない）。

### 7.2 外部 SaaS / 認証連携

| 接続先 | 接続方法 | 段階 |
|---|---|---|
| Cognito User Pool | MVP の認証基盤、JWT 発行 | MVP |
| Azure AD / SAML | M3 で SSO 連携 | M3 |
| Slack Workspace | OAuth + Bolt SDK | M2 |
| HRIS（人事システム） | **未決定（OQ-01）** | M2 以降 |

### 7.3 Admin Console / Portal 統合

| UI | 構成 | 配置 |
|---|---|---|
| Admin Console | Next.js 15 (App Router) + React 19 + Tailwind 4 + shadcn/ui、8 ページ | `apps/admin-console/` |
| Portal | Next.js 15 + Tailwind、MVP は 1 ページ、M2 で 5 ページ | `apps/portal/` |
| 共通 | CloudFront + WAF + ALB 経由 | エッジ層 |

### 7.4 CI/CD

| 観点 | 採用 |
|---|---|
| パイプライン | GitHub Actions + OIDC（AWS IAM Role） |
| IaC | Terraform Cloud（state） / Atlantis（plan PR） |
| カバレッジゲート | 80%（Codecov） |
| セキュリティスキャン | Trivy / Semgrep / Checkov / gitleaks |
| デプロイ自動化 | `bash deploy.sh` 相当で 30 分以内 |

---

## 8. マイルストーン

> 実装フェーズ分割は実装プランで定義する。本節はマイルストーン粒度のみを示す。

| マイルストーン | 含む要件 |
|---|---|
| **MVP (M1)** | 組織管理 / 3 層 SOUL / Standard ティア / Cognito 認証 / Portal Chat 最小 / 監査一次系 |
| **M2** | + 常時稼働（Always-On）/ Slack 統合 / Engineering + Executive ティア |
| **M3** | + フルガバナンス / デジタルツイン / Azure AD / 500 ユーザー対応 |
| **M4** | + SOC2 準拠基盤（NFR-CMP-04 の認証取得スコープ確定後）/ 運用安定化 |

---

## 9. リスクと制約

### 9.1 技術リスク

| リスク | 影響 | 緩和策 |
|---|---|---|
| マルチテナント情報漏洩 | コンプライアンス重大違反 | コンピュート分離（L4）+ IAM Session Policy + 100+ ケース漏洩テスト CI 必須 |
| プロンプトインジェクションによる L2 突破 | 権限境界突破 | L3 以降のインフラ境界で防御、L1/L2 は補助的 |
| コールドスタートが p95 < 15s を満たさない | UX 低下 | AgentCore Session Storage、Provisioned Concurrency 検討（OQ-02） |
| Bedrock H2 Proxy のストリーミング維持と SigV4 の両立 | レイテンシ未達 | バックプレッシャ試験 + p95 < 50ms オーバーヘッド検証 |
| 3 層 SOUL マージのバグ | アイデンティティ崩壊 | スナップショットテスト 20+ ケース CI 必須 |
| 自動プロビジョニングの部分失敗 | データ不整合 | 補償トランザクション（DynamoDB Streams） |
| Cloud Map DNS TTL の伝播遅延 | Always-On 切替時の旧 IP 参照 | TTL 10 秒、p95 < 45s で安定 |
| Bedrock リージョン制約 | サービス利用不可 | us-east-1 / us-west-2 の AgentCore 対応状況を MVP 着手前に確定 |

### 9.2 運用リスク

| リスク | 緩和策 |
|---|---|
| IAM 過剰権限の蓄積 | IAM Access Analyzer を CI に組み込み、過剰権限ゼロを完了基準化 |
| 監査ログ改ざん | S3 Object Lock + KMS、監査ログのアクセス監査も記録 |
| デプロイ事故 | OIDC + plan PR レビュー必須、`bash deploy.sh` 30 分以内 |
| SOUL の不正な変更 | 変更前差分プレビュー必須、20 世代履歴、ロールバック |
| 監査ストリーム障害 | DLQ + リトライ、5 分以上滞留で PagerDuty |
| 公開デジタルツインの濫用 | API Gateway throttling + WAF レート制限 + アクセスメタデータ監視 |

### 9.3 コスト見積もり（500 ユーザー時）

| 項目 | MVP（M1: 50 ユーザー） | 最終（M3: 500 ユーザー） |
|---|---|---|
| Bedrock（Nova / Claude 推論） | $50〜$150 | $800〜$1,500 |
| AgentCore Runtime | $30〜$80 | $300〜$600 |
| ECS Fargate（Tenant Router / Bedrock Proxy / Control Plane） | $80 | $200 |
| ECS Fargate（Always-On エージェント） | $0 | $150〜$300 |
| ALB / API Gateway / CloudFront / WAF | $30 | $80 |
| DynamoDB（On-Demand + Streams） | $20 | $100 |
| S3 + Object Lock + Athena | $15 | $80 |
| EFS（Always-On） | $0 | $30 |
| CloudWatch / X-Ray / Sentry | $20 | $80 |
| Cognito | $0 | $20 |
| SSM / Secrets Manager / KMS | $5 | $20 |
| Firehose | $5 | $30 |
| **月次合計** | **$255〜$435** | **$2,090〜$3,290** |
| **目標（$5 / ユーザー / 月、M3 のみ適用）** | — | **$2,500** |

> 500 ユーザー時の上振れ（$3,290）が目標 $2,500 を超える可能性がある。Bedrock 推論コスト（最大費目、$800〜$1,500）の最適化が一次的な吸収手段。MVP 規模（50 ユーザー）では単価が高くなることは合理的範囲とし、$5/ユーザー目標は M3 達成時の成功基準とする。

### 9.4 法的・データ越境

| 制約 | 詳細 |
|---|---|
| 単一リージョン強制 | 全データ（ワークスペース / 監査 / KB）が単一リージョンに留まることを Terraform で強制 |
| クロスリージョンレプリケーション禁止 | Global Table / S3 Cross-Region Replication 禁止 |
| EU / APAC データ主権要件のテナント | **スコープ外**（AgentCore 対応リージョンに全データ配置可能な組織のみ対象） |
| GDPR 削除権 | **未決定（OQ-05）**: SLA を 30 日 / 即時のいずれにするか |
| 個人情報保護法（日本） | 遵守（NFR-CMP-06） |
| `sample/` ライセンス | AWS Samples / Apache 2.0 想定、コードコピー禁止 |
| 本リポジトリライセンス | MIT 予定 |

#### 9.4.1 GDPR 削除権 SLA と監査保持期間の対立

OQ-05 で扱う論点。以下のように要求が衝突するため、コンプライアンス判断者による方針確定が必要。

| 観点 | 要求 |
|---|---|
| GDPR 削除権 SLA | 30 日以内に個人データを完全削除（GDPR Art.17） |
| 監査長期保管 | DynamoDB 72h + S3 Object Lock **7 年**（§ 4.6 NFR-AUD-04） |
| 両立の困難点 | 監査ログに含まれる `actor_id` / `request_id` / 操作ログ本体が個人データに該当しうる。S3 Object Lock は保持期間中の削除を物理的にブロック |

想定される解（OQ-05 確定時に選定）:

- **案 A**: 監査ログ内の個人識別子を擬名化（pseudonymization）し、7 年保持と両立
- **案 B**: 監査保持期間を 30 日に短縮し、SOC2 / 個人情報保護法側の要請と再交渉
- **案 C**: 個人データと監査メタデータを分離保管し、個人データのみ 30 日削除可能とする

---

## 10. 受け入れ基準（要件達成の判定）

> 本書の `AC-F-xx` / `AC-P-xx` / `AC-S-xx` / `AC-O-xx` は本書内で完結する ID 体系である。

### 10.1 機能受け入れ基準

| ID | 基準 |
|---|---|
| AC-F-01 | 従業員が Portal / Slack から AI に質問し、3 層 SOUL がマージされた応答を受信できる |
| AC-F-02 | 部門管理者が Admin Console で Position SOUL を編集すると、5 分以内に全エージェントで反映される |
| AC-F-03 | IT 管理者が Global SOUL を編集すると、Position SOUL が上書きできないことを CI で検証 |
| AC-F-04 | 新入従業員作成時に Auto-Provisioning が (1) DynamoDB Employee エントリ (2) DynamoDB Agent + 1:1 バインディングエントリ (3) S3 ワークスペース (4) DynamoDB 監査エントリ の **4 リソース**（§ 3.13 FR-PROV-01 の列挙と一致）を全件作成して成功とする。途中失敗時は FR-PROV-02 の補償トランザクションが 5 分以内に整合状態（全件作成 or 全件ロールバック）に収束させる。CI で「途中で 1 リソース失敗 → 5 分以内に整合」の自動テスト必須 |
| AC-F-05 | 異動時に Position 切替で次回コールドスタートから新 SOUL が適用される |
| AC-F-06 | 退職時にエージェントが soft-delete され、30 日後に hard-delete される |
| AC-F-07 | ツール許可リスト変更が 5 分以内に反映される |
| AC-F-08 | Always-On 切替後、Cloud Map register が 30 秒以内、DNS 安定が p95 < 45s |
| AC-F-09 | Slack DM / メンションでスレッド継続性を保ったまま応答 |
| AC-F-10 | デジタルツイン公開リンクから外部関係者がレート制限内で会話可能 |
| AC-F-11 | 監査検索で「特定従業員の過去 24h 全操作」を Admin Console から 2 秒以内に取得 |
| AC-F-12 | Athena 経由で「過去 90 日の Guardrails ブロック一覧」を 30 秒以内に取得 |
| AC-F-13 | SOUL ロールバックが直近 20 世代から 1 クリックで実行可能 |
| AC-F-14 | `bash deploy.sh` 相当で 50 ユーザー想定環境のデプロイが 30 分以内に完了 |

### 10.2 性能受け入れ基準

| ID | 基準 |
|---|---|
| AC-P-01 | コールドスタート 中央値 < 10s / p95 < 15s（10 連続実測） |
| AC-P-02 | ウォーム応答 中央値 < 3s / p95 < 5s |
| AC-P-03 | Bedrock H2 Proxy 透過オーバーヘッド p95 < 50ms |
| AC-P-04 | Tenant Router 解決 p95 < 50ms |
| AC-P-05 | Always-On 切替後の応答 p95 < 2s |
| AC-P-06 | Always-On Cloud Map register → DNS 安定 p95 < 45s |
| AC-P-07 | 500 ユーザー時の同時 100 セッションを成功率 99.5%+ で処理 |

### 10.3 セキュリティ受け入れ基準

| ID | 基準 |
|---|---|
| AC-S-01 | マルチテナント漏洩テスト 100+ ケースが CI で全パス（テナント ID 改ざん・偽プレフィックス・並行混線） |
| AC-S-02 | クロスティア参照テスト（Restricted → Executive KB / ツール）が全拒否 |
| AC-S-03 | 危険パターン拒否ケース（`rm -rf`、SQL injection 等）が 100% ブロック |
| AC-S-04 | IAM Access Analyzer で過剰権限ゼロ |
| AC-S-05 | 全 S3 / DynamoDB / EFS に SSE-KMS / 暗号化が適用（Terraform で強制） |
| AC-S-06 | OWASP Top 10 全カテゴリの自動検査（Semgrep / Checkov）が PASS |
| AC-S-07 | gitleaks でハードコードシークレット検出ゼロ |
| AC-S-08 | パブリックポートが ALB / CloudFront のみであることを Terraform で強制 |
| AC-S-09 | 監査ログが 99.9%+ で書き込まれる |
| AC-S-10 | デジタルツイン公開リンクが API Gateway throttling + WAF で保護される |
| AC-S-11 | **監査書込 IAM 境界テスト**: 本体タスクロールが許可されたテーブル以外に書込できないことを (a) IAM Simulator API による宣言的検証 + (b) CI 環境での実 STS AssumeRole 試行による実アクセス検証 の **二段階** で確認 |

### 10.4 観測性受け入れ基準

| ID | 基準 |
|---|---|
| AC-O-01 | CloudWatch ダッシュボードに QPS / レイテンシ p50/p95/p99 / エラー率 / コストが表示 |
| AC-O-02 | X-Ray トレースで Portal → Tenant Router → Runtime → Bedrock の内訳が見える |
| AC-O-03 | CRITICAL アラートが PagerDuty に 1 分以内に通知される |
| AC-O-04 | 1 ユーザー / 月コストが Admin Console のコストダッシュボードで表示される |
| AC-O-05 | 監査一次系（Firehose → S3 Object Lock → Athena）の動作検証が CI で日次実施される |

---

## 11. 未決事項と意思決定が必要な論点（Open Questions）

更新後 Open Questions 残: **5 件**（OQ-01 〜 OQ-05）。

| ID | 論点 | 判断者 | 期限 |
|---|---|---|---|
| OQ-01 | HRIS（人事システム）連携を MVP に含めるか / 連携先 SaaS（Workday / SAP SuccessFactors / Sansan 等） | プロダクトオーナー + IT 管理者 | M2 設計時 |
| OQ-02 | AgentCore コールドスタートが p95 < 15s を満たさない場合の対応（Provisioned Concurrency / 専用 Lambda Warmer / Always-On 標準化） | アーキテクト | MVP ランタイム検証完了時 |
| OQ-03 | Azure AD 以外の外部 IdP（Okta / Google Workspace 等）対応範囲 | プロダクトオーナー | M3 設計時 |
| OQ-04 | Portal Chat のファイル添付サイズ上限・MIME 範囲（業務文書 / 画像 / コード） | プロダクトオーナー | M2 Portal 拡張設計時 |
| OQ-05 | GDPR 削除権 SLA（30 日 vs 即時）と監査ログとの整合（監査は 7 年保持要件） | コンプライアンス | M3 設計時 |

---

## 12. 用語集

| 用語 | 定義 |
|---|---|
| OPENCLAW プラットフォーム | 本書で要件定義するエンタープライズ AI エージェント基盤全体 |
| OpenClaw | プラットフォームに内包される AI エージェントランタイム（`2026.3.24` 固定） |
| 3 層 SOUL | Global / Position / Personal の Markdown を上位優先でマージするアイデンティティ機構 |
| 4 ティアランタイム | Standard / Restricted / Engineering / Executive |
| 5 層多層防御 | L1 SOUL ルール / L2 ツール許可 / L3 IAM / L4 コンピュート分離 / L5 Guardrails |
| Tenant Router | チャネル ID → emp_id → runtime の解決を担うゲートウェイ |
| Bedrock H2 Proxy | Bedrock API への HTTP/2 ストリーミングプロキシ |
| AgentCore | AWS Bedrock AgentCore Runtime（Firecracker microVM） |
| Always-On | ECS Fargate での常時稼働モード |
| Workspace Assembler | S3 から 3 層 SOUL をマージするモジュール |
| Permission Profile | DynamoDB `PERM#{posId}` のツール許可リスト |
| デジタルツイン | 本人不在時の公開リンク型エージェント |
| `CRITICAL IDENTITY OVERRIDE` | 上位 SOUL レイヤーが下位を上書きできないことを示す先頭ヘッダ |
| Firecracker microVM | AgentCore の実行単位、L4 コンピュート境界 |
| Bedrock Guardrails | PII / Topic / Injection フィルタを提供するマネージドガードレール |

---

