# Knowledge DB サブシステム 要件定義書

- **ステータス**: Final Draft (要承認)
- **起票日**: 2026-05-12
- **最終更新**: 2026-05-12（OQ-03 / 05 / 06 / 07 / 15 確定反映、その後 OQ-06 を「統合 → 共存」に再修正）
- **対象**: enterprise-ai-platform / MCP Gateway 双方から参照される汎用ナレッジ DB（RAG 系）サブシステム
- **関連ドキュメント**:
  - [OPENCLAW_PLATFORM_REQUIREMENTS.md](./OPENCLAW_PLATFORM_REQUIREMENTS.md)（本体要件定義 — IAM 境界テスト AC-S-11、3 層 SOUL、4 ティアの詳細はこちら）
  - [OPENCLAW_PLATFORM_PLAN.md](./OPENCLAW_PLATFORM_PLAN.md)
  - [MCP_GATEWAY_REQUIREMENTS.md](./MCP_GATEWAY_REQUIREMENTS.md)（MCP Gateway 要件定義）
  - [MCP_GATEWAY_PLAN.md](./MCP_GATEWAY_PLAN.md)
  - [ADR 0001: Python 統一バックエンド](./adr/0001-python-unified-backend.md)
  - 起票予定: ADR 0002 / 0003 / 0004（§ 11.1 参照）

---

## 0. ドキュメント概要

### 0.1 目的

本ドキュメントは、エンタープライズ OpenClaw クラウドサービス（以下「本体」）と MCP Gateway の両サブシステムから共通利用される **汎用ナレッジ DB（RAG 系）** の **要件**（What / Why / 制約 / 受け入れ基準）を定義する。

実装プラン（How）・コード・Terraform スニペットは本書のスコープ外。決定事項を Open Questions と分離し、ユーザーの意思決定を最短経路で取れる粒度に整理する。

### 0.2 想定読者

- プロダクトオーナー / アーキテクト（スコープ承認）
- セキュリティ / コンプライアンス担当（多層防御・データ越境）
- 後続フェーズの実装プラン執筆者
- 運用 / SRE（観測性・コスト）

### 0.3 関連ドキュメント

- 本体プラン（10 フェーズ）: `docs/OPENCLAW_PLATFORM_PLAN.md`
- MCP Gateway プラン（8 フェーズ）: `docs/MCP_GATEWAY_PLAN.md`
- ADR 0001: バックエンドサービスを Python 3.12 に統一する
- 起票予定 ADR: 0002（Bedrock KB 採用） / 0003（独立サービス配置） / 0004（DynamoDB テーブル分離）

### 0.4 用語定義

| 用語 | 定義 |
|---|---|
| Knowledge DB (KDB) | 本書で定義するサブシステム名。社内文書を取り込み、セマンティック / ハイブリッド検索を提供する |
| KB（Knowledge Base） | KDB 内の論理単位。1 KB = 1 つのデータソース + 1 つの ACL ポリシー集合 |
| Document | KB に取り込まれる元ファイル単位（PDF / Markdown / Word / HTML 等） |
| Chunk | Document を埋め込み単位に分割した断片 |
| RAG | Retrieval-Augmented Generation |
| Workspace Assembler | 本体 Phase 3 Step 2 の SOUL 組み立てモジュール |
| 3 層 SOUL | Global / Position / Personal の Markdown マージ機構 |
| 4 ティアランタイム | Standard / Restricted / Engineering / Executive |
| directory_kb | Phase 9 Step 5 の組織ディレクトリ KB（小規模 Markdown 自動注入、KDB と共存する独立機構） |

---

## 1. 背景と目的

### 1.1 背景

現状の Source of Truth では、ナレッジ系の機能として以下が定義されている:

| 既存機能 | 出典 | 性質 |
|---|---|---|
| Bedrock Knowledge Bases (RAG) | OPENCLAW プラン § 2.1 AI Layer | 単に「使う」とのみ明記、運用 IF は未定義 |
| S3 `knowledge/` プレフィックス | OPENCLAW プラン § 7, § 8 | データ配置のみ定義、取り込み / 検索 API なし |
| `directory_kb.py`（Phase 9 Step 5） | OPENCLAW プラン Phase 9 | 組織ディレクトリ（部門 / 役職 / 連絡先）を Markdown で生成し、全エージェントの SOUL に **自動注入** する小規模機構 |
| MCP Gateway の上流 MCP | MCP プラン全体 | 既存 MCP サーバー（GitHub / Jira 等）の透過プロキシ。**社内文書 RAG は提供範囲外** |

つまり「**汎用 RAG**」「**OPENCLAW から動的検索できる KB**」「**MCP クライアント（Claude Desktop / Code）からも検索できる KB**」を担うサブシステムが未定義である。

### 1.2 目的

以下の 3 点を満たす横断サブシステムを定義する:

1. **OPENCLAW Agent Container のランタイム**から検索ツールとして呼び出せる
2. **MCP Gateway のクライアント（Claude Desktop / Claude Code / 自社エージェント）**から MCP ツールとして呼び出せる
3. **管理・運用・監査**を Admin Console と既存監査基盤に統合する

### 1.3 既存 `directory_kb.py` 構想との差分（OQ-06 確定: 共存する）

`directory_kb.py`（本体プラン Phase 9 Step 5）は **KDB と共存**させる。役割が重複しないため統合は行わず、両者は独立して並存する。

| 観点 | `directory_kb`（既存構想 / 存続） | **本サブシステム（KDB）** |
|---|---|---|
| 役割 | 組織情報の **コールドスタート時自動注入** | 汎用 RAG（**ランタイム検索**） |
| データ形式 | 単一の Markdown を生成 | PDF / Markdown / Word / HTML 等を取り込み・チャンク化 |
| データ量 | 小規模（数 KB〜数百 KB） | 中〜大規模（〜数 GB / 数十万チャンク） |
| 取得タイミング | Workspace Assembler が S3 fetch | エージェント実行中の **検索 API コール** |
| 検索方式 | なし（全文埋め込み） | セマンティック + キーワード + ハイブリッド |
| ACL | Global SOUL 経由（全員に注入） | KB / ドキュメント / メタデータ単位 |
| MCP 露出 | なし | あり（`knowledge__search`, `knowledge__fetch`） |
| 位置付け | **KDB と共存**。組織情報のみ専用機構として残す | 新規追加 |

> **整理（OQ-06 確定: 2026-05-12 共存）**: `directory_kb.py` は廃止せず「組織情報 → SOUL 注入」用途として残す。KDB は別経路（ランタイム検索）。両者は **役割が重複しない**（注入 vs ランタイム検索、小規模単一文書 vs 大規模文書集合）。組織情報を KDB から検索したい要求が将来発生した場合は、別途要件追加で再検討する。

### 1.4 ユースケース駆動の必要性

「OPENCLAW と MCP 双方から参照」が必要な理由:

1. **同一の文書資産を二重メンテしたくない**: 社内規程 / 製品仕様書 / SOP は OPENCLAW エージェントも Claude Desktop ユーザーも参照する
2. **ACL の一貫性**: 「Manager だけが見られる文書」は、Portal 経由でも Claude Desktop 経由でも同じ判定であるべき
3. **監査の一元化**: 「誰が・いつ・どの文書を検索したか」は監査パイプラインに統一すべき
4. **コスト最適化**: 埋め込み計算は最も高コスト要素。1 つの KDB に集約することで重複埋め込みを回避

### 1.5 ビジネス価値（KPI 候補）

| KPI | 測定方法 | 目標値（500 ユーザー時） |
|---|---|---|
| 検索ヒット率（人手評価） | 月次 100 クエリのゴールデンセット | 80%+ |
| エージェント回答に占める KDB 引用率 | 監査ログから集計 | 30%+ |
| 1 クエリあたりレイテンシ p95 | CloudWatch | < 1.5s |
| 月間検索クエリ数 | 監査ログ | 50,000+ (KPI、上限ではない) |
| ドキュメント取り込み成功率 | 取り込みパイプラインメトリクス | 99%+ |
| KDB 起因の PII / 機密漏洩インシデント | セキュリティレビュー | 0 件 |

---

## 2. ユースケース / アクター

### 2.1 主要アクター

| アクター | 役割 | KDB との関わり |
|---|---|---|
| 従業員（Employee） | Portal / Slack から AI に質問 | エージェント経由で間接的に KDB を検索 |
| 部門管理者（Manager） | 部門 KB のオーナー | KB 作成、ドキュメント取り込み、ACL 設定 |
| IT 管理者 | 全社 KB のオーナー、コンプライアンス | グローバル KB 管理、KMS 鍵運用、削除権 |
| エンジニア（Claude Desktop / Code 利用者） | MCP クライアント経由で KDB 検索 | `knowledge__search` ツール直接呼び出し |
| Agent Container（ランタイム） | OpenClaw 実行プロセス | 検索ツールとして呼び出し、引用付き回答生成 |
| MCP Gateway | アグリゲーター | KDB を上流 MCP として登録、3 軸 ACL 適用 |
| 監査担当 | コンプライアンス検証 | 監査ログから「誰が何を検索したか」を抽出 |

### 2.2 主要ユースケース（合計 8 件）

| # | ユースケース | アクター | 主経路 |
|---|---|---|---|
| UC-01 | 社内規程 Q&A | 従業員 → Portal → エージェント | OPENCLAW ランタイム検索 |
| UC-02 | 部門 SOP 参照 | 従業員（同部門） → Slack → エージェント | OPENCLAW ランタイム検索（部門 ACL） |
| UC-03 | コード規約検索 | エンジニア → Claude Code | MCP `knowledge__search` |
| UC-04 | 製品仕様書のセマンティック検索 | エンジニア → Claude Desktop | MCP `knowledge__search` |
| UC-05 | 経営層限定 KB の参照 | 役員 → Portal → Executive ティアエージェント | OPENCLAW ランタイム検索（Executive ACL） |
| UC-06 | ドキュメントの定期同期 | IT 管理者（バッチ） | 取り込みパイプライン |
| UC-07 | 監査: 誰が機密文書を検索したか | 監査担当 → Admin Console | 監査ログ検索 |
| UC-08 | 削除権（GDPR）の行使 | IT 管理者 → Admin Console | 該当文書 + ベクター + 監査ログ匿名化 |

### 2.3 非ユースケース（明示的スコープ外）

- **生成系タスク**: 要約 / 翻訳 / コード生成は KDB の責務外（呼び出し側エージェントの責務）
- **構造化データ Q&A**: BI / DWH への自然言語クエリは別サブシステム
- **画像 / 動画の埋め込み検索**: 初期版はテキストのみ
- **オフライン同期 / モバイルキャッシュ**: 単一リージョン制約下、サポートしない
- **外部公開**: KDB はテナント内部のみ。デジタルツインからの外部検索は許可しない
- **マルチテナント（組織間）**: 500 ユーザー上限・単一組織前提、組織間共有 KB は将来検討
- **チャットボット UI**: 検索結果の閲覧 UI は Admin Console の「検索プレビュー」のみ。エンドユーザー向けの専用 UI は提供しない（Portal Chat 経由でアクセス）

---

## 3. 機能要件

### 3.1 ドキュメント取込

| 要件 ID | 要件 | 決定 / 未決定 |
|---|---|---|
| FR-ING-01 | バッチアップロード（管理者が ZIP / 個別ファイルを Admin Console から投入） | 決定 |
| FR-ING-02 | 定期同期（S3 prefix 監視、または外部 SaaS への定期 pull） | **未決定**: 初期版は S3 監視のみ、外部 SaaS pull は将来 |
| FR-ING-03 | 対応フォーマット: PDF / Markdown / 平文 / HTML | 決定 |
| FR-ING-04 | 対応フォーマット: Word (.docx) / Excel (.xlsx) / PowerPoint (.pptx) | **未決定**: MVP に含めるか |
| FR-ING-05 | 対応フォーマット: Confluence エクスポート / Notion エクスポート | **未決定**: 外部 SaaS 連携の延長として後続フェーズで検討 |
| FR-ING-06 | OCR（スキャン PDF / 画像 PDF） | **未決定**: Textract 採用可否、コスト影響大 |
| FR-ING-07 | 上限: 1 ファイル 50 MB / 1 KB あたり 10,000 ドキュメント / 全 KB 計 100 GB | **未決定**: 数値の妥当性（500 ユーザー規模） |

### 3.2 取り込みパイプライン

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-PIPE-01 | パイプライン: アップロード → 検証 → テキスト抽出 → チャンク → 埋め込み → インデックス | 決定 |
| FR-PIPE-02 | 全工程の状態管理（DynamoDB に `DOC#{kbId}#{docId}` 状態を持つ） | 決定 |
| FR-PIPE-03 | 失敗時のリトライ（指数バックオフ、最大 3 回） | 決定 |
| FR-PIPE-04 | 再インデックス API（KB 単位で全ドキュメント再埋め込み） | 決定 |
| FR-PIPE-05 | 取り込み中の検索: 旧バージョンを返す（無停止再インデックス） | 決定 |
| FR-PIPE-06 | チャンク戦略: 既定 = トークンベース 500 token / オーバーラップ 50 token | 既定値は決定、KB 単位で上書き可能 |
| FR-PIPE-07 | チャンク戦略の選択肢: 固定 / セマンティック / 階層 / Markdown 構造 | **未決定**: MVP は固定のみで開始するか |
| FR-PIPE-08 | 埋め込みモデル: **MVP は Titan v2 固定**（§ A OQ-07 確定）。最終版で KB 単位選択を検討 | 決定（MVP） |
| FR-PIPE-09 | 日本語 / 英語混在の前処理（言語判定・正規化） | 決定（必須）、ライブラリは未決定 |

### 3.3 検索 API

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-SRCH-01 | セマンティック検索（ベクター類似度） | 決定 |
| FR-SRCH-02 | キーワード検索（BM25 ないし類似） | 決定 |
| FR-SRCH-03 | ハイブリッド検索（セマンティック + キーワードのスコア統合） | 決定 |
| FR-SRCH-04 | メタデータフィルタ（`classification`, `department`, `tier` 等） | 決定 |
| FR-SRCH-05 | リランキング | **MVP では不採用**（§ A OQ-03 確定）。M2 以降で再評価 |
| FR-SRCH-06 | top_k 既定 = 5、最大 = 20 | 決定 |
| FR-SRCH-07 | 結果に含めるフィールド: chunk_text, score, doc_id, doc_title, source_uri, page, metadata | 決定 |
| FR-SRCH-08 | レスポンス形式: JSON、`{ success, data, error, meta }` エンベロープ | 決定（API 共通パターン） |
| FR-SRCH-09 | クエリ拡張（HyDE / Multi-Query） | **未決定**: 採用可否 |
| FR-SRCH-10 | 検索範囲限定（特定 KB / 複数 KB / 全アクセス可能 KB） | 決定 |

### 3.4 アクセス制御

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-ACL-01 | KB 単位の ACL（誰がその KB を検索 / 管理できるか） | 決定 |
| FR-ACL-02 | ドキュメント単位の ACL（KB 内の特定文書のみ制限） | 決定 |
| FR-ACL-03 | メタデータベース ACL（`classification: confidential` は Executive のみ等） | 決定 |
| FR-ACL-04 | 3 層 SOUL 階層との対応: Global KB / Position KB / Personal KB の概念をサポート | 決定 |
| FR-ACL-05 | 4 ティアランタイムとの対応: Restricted は限定 KB のみアクセス可 | 決定 |
| FR-ACL-06 | MCP 経由アクセス時の 3 軸 ACL 統合（user × upstream(="knowledge") × tool） | 決定 |
| FR-ACL-07 | デフォルト deny（明示許可がない KB は不可視） | 決定 |
| FR-ACL-08 | ACL 評価レイテンシ目標 p95 < 20ms（検索全体 1.5s 予算のうち） | 決定 |

### 3.5 OPENCLAW からの利用 IF

| 要件 ID | 要件 | 決定 / 未決定 |
|---|---|---|
| FR-OC-01 | Workspace Assembler 経由のプリフェッチ（コールドスタート時に小規模 KB を SOUL に埋め込む） | **未決定（OQ-14）**: ランタイム検索のみか両方提供か |
| FR-OC-02 | ランタイム検索ツール（OpenClaw のツール呼び出しとして `knowledge_search` を提供） | 決定（主経路） |
| FR-OC-03 | 検索結果の OpenClaw への返却形式（JSON / Markdown） | **未決定**: OpenClaw のツール戻り値慣習に合わせる |
| FR-OC-04 | エージェントの所属ポジション / ティアに応じた自動 KB スコープ絞り込み | 決定（Tenant Router から渡される `emp_id` / `pos_id` で自動適用） |
| FR-OC-05 | ストリーミング応答内での引用ハイライト | **未決定**: UI 仕様未確定 |

### 3.6 MCP からの利用 IF

| 要件 ID | 要件 | 決定 / 未決定 |
|---|---|---|
| FR-MCP-01 | KDB を MCP Gateway の上流 MCP サーバーとして登録 | 決定 |
| FR-MCP-02 | 名前空間: `knowledge__search`, `knowledge__fetch`, `knowledge__list_kbs` | 提案、要承認 |
| FR-MCP-03 | MCP `tools` として実装（`resources` / `prompts` ではない） | 決定 |
| FR-MCP-04 | MCP Gateway の 3 軸 ACL と KDB 内 ACL の関係（**二段階フィルタ**: Gateway で粗フィルタ、KDB で精密フィルタ） | 決定 |
| FR-MCP-05 | MCP クライアントへの結果整形（Markdown 引用ブロック） | 決定 |
| FR-MCP-06 | MCP Resource として KB 一覧を露出するか | **未決定**: Tool のみで十分か |

### 3.7 管理機能

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-ADM-01 | KB CRUD（作成 / 名前変更 / 削除 / 所有者変更） | 決定 |
| FR-ADM-02 | ドキュメント CRUD（一覧 / 詳細 / 削除 / 再取込） | 決定 |
| FR-ADM-03 | 検索プレビュー（管理者が任意クエリで検索結果を確認） | 決定 |
| FR-ADM-04 | メトリクス UI（取り込み成功率 / 検索 QPS / レイテンシ / コスト） | 決定 |
| FR-ADM-05 | 再インデックス UI（KB / 全社単位、進捗表示） | 決定 |
| FR-ADM-06 | ACL 設定 UI | 決定 |
| FR-ADM-07 | 監査ログ UI（KDB クエリの一覧、本体 Audit UI に統合） | 決定 |

### 3.8 バージョニング / 鮮度管理 / 削除

| 要件 ID | 要件 | 決定 |
|---|---|---|
| FR-VER-01 | ドキュメントバージョン履歴（最新 N 世代を保持） | **未決定**: N = 3 / 5 / 10 |
| FR-VER-02 | TTL（古いチャンクを自動失効） | **未決定**: 既定値、KB 単位設定可否 |
| FR-VER-03 | 鮮度メタデータ（`last_indexed_at`, `source_modified_at`）を検索結果に含む | 決定 |
| FR-VER-04 | 削除と忘却権（GDPR 系）: 該当文書 + ベクター + 監査ログ匿名化を 30 日以内に実施 | 決定 |
| FR-VER-05 | 削除後の検索結果からの即時除外（インデックス削除を 5 分以内） | 決定 |
| FR-VER-06 | バックアップ（S3 バージョニング + DynamoDB PITR の継承） | 決定 |

---

## 4. 非機能要件

### 4.1 性能

| 指標 | 目標値 | 計測方法 |
|---|---|---|
| 検索 p50 | < 500ms | CloudWatch カスタムメトリクス |
| 検索 p95 | < 1,500ms | 同上 |
| 検索 p99 | < 3,000ms | 同上 |
| 取り込みスループット | 100 ドキュメント / 分 | バッチ取込ベンチ |
| 同時クエリ | 50 QPS | k6 負荷試験 |
| ACL 評価 p95 | < 20ms | 同上 |
| 取り込み: 1 ドキュメント（10 ページ PDF）の処理時間 | < 60s | パイプラインメトリクス |

### 4.2 可用性

| 指標 | 目標値 |
|---|---|
| 検索 API SLO | 99.5%（500 ユーザー時） |
| 取り込み API SLO | 99.0% |
| Multi-AZ | 必須（検索フロントのみ。Bedrock KB はマネージド） |
| RTO | < 1 時間 |
| RPO | < 24 時間（S3 + DynamoDB PITR で達成） |

### 4.3 スケーラビリティ（500 ユーザー上限内）

| 指標 | **MVP（厳格）** | MVP（当初案 / 参考） | 最終目標 |
|---|---|---|---|
| KB 数 | **5** | 10 | 50 |
| ドキュメント数（全 KB 合計） | **3,000** | 10,000 | 100,000 |
| チャンク / ベクター数 | **30,000** | 100,000 | 1,000,000 |
| 月間検索クエリ数 | **3,000** | 5,000 | 50,000 |
| 埋め込みモデル選択肢 | **Titan v2 固定** | 選択可 | KB 単位選択可（最終） |
| リランキング | **不採用** | 任意 | M2 以降で再評価 |

> § A OQ-07 確定により、MVP は本体予算（500〜1,000 USD/月）に吸収するため厳格化した規模で開始する。

### 4.4 セキュリティ（多層防御マッピング）

| 層 | KDB での対策 |
|---|---|
| L1 SOUL ルール | エージェントの SOUL に「機密 KB 引用時の取扱い」記載（ガバナンス層、Trust Boundary ではない） |
| L2 ツール許可リスト | `knowledge_search` ツールをポジション別に許可 / ブロック |
| L3 IAM | KDB の S3 prefix / DynamoDB へのアクセスを 4 ティア IAM ロールで分離。KDB タスクロールは KDB 専用テーブルのみアクセス可（§ 4.5 参照） |
| L4 コンピュート分離 | KDB の検索サービスを ECS Fargate 単一タスク（マルチテナント前提だが、組織内マルチユーザーのみ） |
| L5 Bedrock Guardrails | 検索結果の PII / 機密フィルタ |

| 要件 ID | 要件 | 決定 |
|---|---|---|
| NFR-SEC-01 | 保存時暗号化（S3 SSE-KMS、DynamoDB 暗号化、ベクターストア暗号化） | 決定 |
| NFR-SEC-02 | 転送時暗号化（TLS 1.3） | 決定 |
| NFR-SEC-03 | KMS 鍵分離（KDB 専用 CMK） | **未決定**: 既存 CMK 流用 vs 新規 |
| NFR-SEC-04 | PII フィルタ（取り込み時 / 検索結果） | 決定 |
| NFR-SEC-05 | Guardrails 統合（検索結果を Bedrock Guardrails に通過させる） | **未決定**: 取り込み時 vs 検索時 vs 双方 |
| NFR-SEC-06 | デフォルト deny ACL | 決定 |
| NFR-SEC-07 | クレデンシャル管理（外部 SaaS 接続トークンを Secrets Manager） | 決定 |

### 4.5 マルチテナント分離

> 本プロジェクトは単一組織前提（CLAUDE.md 制約）だが、**部門間・ティア間の論理分離**は必須。

| 要件 ID | 要件 | 決定 |
|---|---|---|
| NFR-TEN-01 | KB は所有部門 / ティアでラベル付け | 決定 |
| NFR-TEN-02 | 検索時に呼び出し元のティア / 部門を必ず ACL に渡す | 決定 |
| NFR-TEN-03 | クロスティア参照（Restricted → Executive KB）は明示的に拒否 | 決定 |
| NFR-TEN-04 | テナント ID 改ざんテスト（本体 Phase 2 完了基準と同様、CI 必須） | 決定 |
| NFR-TEN-05 | **KDB テーブル分離による I/O 競合・容量設定干渉・PITR コスト結合の回避**: KDB 専用 DynamoDB テーブル `enterprise-ai-platform-kdb-{env}` を新設し、本体 / MCP Gateway テーブルから物理分離する（§ 6.5、§ A OQ-05） | 決定 |

### 4.6 監査

| 要件 ID | 要件 | 決定 |
|---|---|---|
| NFR-AUD-01 | 全検索クエリを監査ログに記録 | 決定 |
| NFR-AUD-02 | 記録内容: timestamp, requester_id, kb_id, query (生クエリ or ハッシュ), hit_doc_ids, latency_ms, source (openclaw / mcp) | 決定（生クエリ vs ハッシュは下記） |
| NFR-AUD-03 | 生クエリ保管 vs ハッシュ保管: **未決定**（プライバシーと検索品質改善の両立） | **未決定** |
| NFR-AUD-04 | 監査ログ保持期間: 本体監査基盤の方針継承（DynamoDB 72h ホット + S3 Object Lock 長期） | 決定 |
| NFR-AUD-05 | 監査ログを `packages/audit-events/AuditRepository(tableName)` 経由で **KDB 専用テーブル**に書き込む（本体 / MCP テーブルとは物理分離、S3 長期保管プレフィックスのみ共有） | 決定 |

### 4.7 コンプライアンス

| 要件 ID | 要件 | 決定 |
|---|---|---|
| NFR-CMP-01 | 単一リージョン強制（AgentCore 対応の us-east-1 or us-west-2） | 決定 |
| NFR-CMP-02 | データ越境禁止（埋め込みモデル呼び出し含めて単一リージョン） | 決定 |
| NFR-CMP-03 | S3 Object Lock（取り込み元原本の改ざん防止）の適用範囲 | **未決定**: 取り込み済み Doc にも適用するか |
| NFR-CMP-04 | GDPR 削除権の SLA: 30 日以内 | 決定 |

### 4.8 観測性

| 要件 ID | 要件 | 決定 |
|---|---|---|
| NFR-OBS-01 | CloudWatch メトリクス（検索 QPS / レイテンシ p50/p95/p99 / 取り込み成功率 / ヒット率） | 決定 |
| NFR-OBS-02 | X-Ray 分散トレーシング（API → ベクター検索 → リランキング） | 決定 |
| NFR-OBS-03 | 構造化ログ（`structlog`、本体共通） | 決定 |
| NFR-OBS-04 | エラー予算: 検索 SLO 99.5% に対し月間ダウンタイム < 3.65 時間 | 決定 |
| NFR-OBS-05 | **月次コストレポート**: CloudWatch Billing Alarm で KDB タグ付けされたコストを日次トラッキング、Admin Console に表示 | 決定 |

---

## 5. アーキテクチャ方針（要件レベルの選定比較）

> 本セクションは「採用方針の比較と決定理由」のみ。実装プランは別途。

### 5.1 ストレージ層（ベクター + メタデータ）

| 選択肢 | コスト | 運用 | 単一リージョン | 既存 S3 連携 | スケール | 評価 |
|---|---|---|---|---|---|---|
| **A. Bedrock Knowledge Bases (managed)** | 中（モデル + ベクトル DB 一体） | 低（マネージド） | OK（us-east-1 / us-west-2） | ネイティブ | 100 万チャンク級まで実績 | **採用（ADR 0002 起票予定）** |
| B. OpenSearch Serverless (vector engine) | 高（最低 OCU 課金） | 中 | OK | 自前 | 大規模可 | 500 ユーザー規模ではオーバースペック |
| C. Aurora PostgreSQL + pgvector | 中〜高 | 中（運用負荷） | OK | 自前 | 中規模まで | 既存に Aurora 採用なし → 新規導入負債 |
| D. DynamoDB + ベクター手書き | 低 | 高 | OK | 同テーブル統合可 | 検索性能限界 | 不採用 |

**決定: A**。理由:
- 既存 AI Layer に明記済み（OPENCLAW プラン § 2.1）
- マネージドで運用コスト最小、500 ユーザー規模に十分
- Bedrock 一系統で埋め込み / 検索 / Guardrails が完結

**Open Question OQ-01**: Bedrock KB の単一リージョン制約と AgentCore リージョン制約の整合（双方が us-east-1 / us-west-2 をサポートする前提を再確認）

### 5.2 埋め込みモデル

| 選択肢 | 言語性能 | コスト（1M token） | 次元 | 評価 |
|---|---|---|---|---|
| **A. Bedrock Titan Embeddings v2** | 日英中の多言語、可変次元（256/512/1024） | 約 $0.02 | 1024 | **MVP 固定採用**（§ A OQ-07） |
| B. Cohere Embed Multilingual v3 | 100+ 言語、日本語性能評価高い | 約 $0.10 | 1024 | 日本語性能が要件未達なら最終版で切替検討 |
| C. OSS（multilingual-e5-large 等）on Sagemaker | 自前運用、コスト分離 | インスタンス課金 | 1024 | 運用負債大、初期不採用 |

**決定（MVP）**: Titan v2 固定。最終版で KB 単位選択を検討（OQ-02 で日本語ベンチマーク要）

**Open Question OQ-02**: 日本語ベンチマーク（社内 100 クエリ）で Titan v2 が許容ヒット率（例: 70%+）を出すか、事前検証する

### 5.3 リトリーバ実装

| 選択肢 | 評価 |
|---|---|
| **A. Bedrock KB の Retrieve API を直接呼ぶ** | マネージド、シンプル。**採用** |
| B. カスタム検索層（自前ハイブリッド + リランキング） | 柔軟性高いが運用負債大 |
| C. ハイブリッド: Bedrock KB + 上位リランキング層 | 中庸、品質要件未達時に追加 |

**決定**: A で開始、品質未達なら C に拡張（最終版で再評価）

### 5.4 リランキング（§ A OQ-03 確定: MVP 不採用）

| 選択肢 | コスト | 品質改善期待値 | 評価 |
|---|---|---|---|
| **A. なし（MVP）** | 0 | - | **MVP 採用確定** |
| B. Bedrock Rerank | 中 | 中 | M2 以降の再評価対象 |
| C. Cohere Rerank | 中 | 高 | M2 以降の再評価対象 |

**決定**: MVP は A（不採用）。M2 以降で品質指標を元に再評価。

### 5.5 アクセス制御の実装方式

| 選択肢 | 評価 |
|---|---|
| **A. メタデータフィルタ（KB 内で classification / tier / department で絞り込み）** | Bedrock KB のメタデータフィルタを活用、シンプル |
| B. KB 物理分割（ティア / 部門ごとに KB を分ける） | 強分離だがメンテ負債 |
| **C. 二段階フィルタ（KB 物理分割 + メタデータフィルタ）** | グローバル KB + 部門 KB 等のハイブリッド、推奨 |

**決定: C**。グローバル / 部門 / 個人 KB は物理分割（A の SOUL 階層と一致）、KB 内の更に細かい分類はメタデータフィルタ

### 5.6 共有方法（OPENCLAW と MCP の双方からアクセス）

これは **本要件定義の最重要設計判断**。

| 選択肢 | 説明 | 利点 | 欠点 |
|---|---|---|---|
| **A. 共有 API レイヤー（KDB Service）を切る** | KDB を独立 ECS Fargate サービスとしてデプロイ、OPENCLAW と MCP Gateway は HTTP API で呼ぶ | 単一の権威 / 監査一元化 / ACL 一貫性 | サービス数増加、ホップ増 |
| B. OPENCLAW 経由 + MCP は OPENCLAW を上流 MCP として登録 | OPENCLAW Agent Container が MCP サーバーも兼ねる | サービス数最小 | OPENCLAW 起動コストが MCP クエリにも乗る、責務肥大 |
| C. Bedrock KB を OPENCLAW / MCP Gateway 双方から直接呼ぶ | KDB 専用サービスを置かない | 最薄 | ACL / 監査ロジックが二重実装になる |

**決定: A**。KDB を独立 ECS Fargate サービス（`services/knowledge-db/`）として配置。OPENCLAW Agent Container はランタイム検索ツール経由で HTTP 呼び出し、MCP Gateway は KDB を上流 MCP として登録（**ADR 0003 起票予定**）。理由:

- 既存パターン（Tenant Router / Bedrock Proxy / Agent Container と同様の ECS Fargate）に従う
- 監査・ACL・Guardrails 統合が単一箇所
- MCP Gateway 側は薄いプロキシで済む

**Open Question OQ-04**: KDB Service と MCP Gateway 経由 KDB アクセスの認証境界（KDB Service 直接アクセスは禁止し MCP / OPENCLAW のみ許可するか）

---

## 6. データモデル要件

> 詳細スキーマは実装プランで定義。要件レベルで属性のみ列挙。

### 6.1 KB（Knowledge Base）属性

| 属性 | 型 | 必須 | 説明 |
|---|---|---|---|
| kb_id | string | ✓ | UUID（予約 ID: `directory`） |
| name | string | ✓ | 人間可読名 |
| owner_type | enum | ✓ | global / position / personal / reserved |
| owner_id | string | ✓ | position_id, employee_id 等 |
| classification | enum | ✓ | public / internal / confidential / executive |
| allowed_tiers | array | ✓ | [Standard, Restricted, Engineering, Executive] のサブセット |
| region | string | ✓ | us-east-1 等（単一リージョン強制で固定） |
| embedding_model | string | ✓ | MVP は titan-v2 固定 |
| chunking_strategy | object | ✓ | size / overlap / strategy |
| retention_days | int |  | 文書保持期間 |
| ttl_policy | object |  | 削除ポリシー |
| created_at / updated_at | timestamp | ✓ |  |

### 6.2 Document 属性

| 属性 | 型 | 必須 |
|---|---|---|
| doc_id | string | ✓ |
| kb_id | string | ✓ |
| title | string | ✓ |
| source_type | enum | ✓ (upload / s3_sync / saas_pull / directory_render) |
| source_uri | string | ✓ |
| mime_type | string | ✓ |
| size_bytes | int | ✓ |
| version | int | ✓ |
| status | enum | ✓ (pending / ingesting / indexed / failed / deleted) |
| metadata | object |  | 任意の K/V |
| ingested_at | timestamp | ✓ |
| source_modified_at | timestamp |  |

### 6.3 Chunk / ベクター属性

Bedrock KB がマネージドで管理する範囲だが、論理属性として:
- chunk_id, doc_id, kb_id, text, embedding_vector, position, page (PDF), token_count, metadata

### 6.4 ACL 表現

| 観点 | 表現 |
|---|---|
| KB ACL | `KB#{kbId}` レコード内に allowed_subjects（role / position_id / employee_id のリスト） |
| Doc ACL | `DOC#{kbId}#{docId}` レコード内に override allowed_subjects（KB ACL を上書き） |
| メタデータ ACL | KB の classification + 検索時のティア確認で評価 |
| MCP 3 軸 ACL との結合 | MCP Gateway 側で粗フィルタ（KDB 全体への到達可否） + KDB 側で精密フィルタ |

### 6.5 DynamoDB テーブル分離（§ A OQ-05 確定）

KDB 専用の DynamoDB テーブル `enterprise-ai-platform-kdb-{env}` を新設し、本体（`enterprise-ai-platform-{env}`）/ MCP Gateway（`enterprise-ai-platform-mcp-gw-{env}`）から物理分離する。

#### 6.5.1 分離方針

| 観点 | 方針 |
|---|---|
| テーブル名 | `enterprise-ai-platform-kdb-{env}`（env: dev/stg/prod） |
| PITR | KDB 専用設定（本体・MCP のコストと結合しない） |
| TTL | KDB 専用設定（KB / Doc / 監査でそれぞれ異なる TTL を独自に設定可） |
| Streams | KDB 専用に有効化（取り込み完了通知、再インデックストリガ用） |
| 抽象化 | `packages/audit-events/AuditRepository(tableName)` 抽象でテーブル名注入。3 つのテーブル間でスキーマ共通（本体 Phase 10 Step 3a の互換性テストで担保） |
| IAM 境界 | KDB タスクロールは KDB テーブルのみ、本体タスクロールは本体テーブルのみ、MCP タスクロールは MCP テーブルのみアクセス可（Resource ARN を厳格指定、ワイルドカード禁止） |

#### 6.5.2 シングルテーブル設計（KDB 専用テーブル内）

| パターン | PK | SK | GSI |
|---|---|---|---|
| KB メタ | `KB#{kbId}` | `META` | GSI1: classification |
| Doc メタ | `KB#{kbId}` | `DOC#{docId}` | GSI1: status |
| 取込ジョブ | `INGEST#{jobId}` | `STATE#{ts}` | GSI1: kb_id |
| 検索監査 | `AUDIT#kdb#{ts}` | `QUERY#{queryId}` | GSI1: requester_id, GSI2: kb_id |
| `directory` スナップショット | `KB#directory` | `SNAPSHOT#{version}` | GSI1: version |

#### 6.5.3 分離による効果（NFR-TEN-05）

- **I/O 競合の回避**: 検索クエリのバースト（KDB）と組織 CRUD のスパイク（本体）が互いの読み書きスロットルを侵食しない
- **容量設定の独立**: KDB はベクター取込時に大量書き込みが発生するため On-Demand を採用しやすい
- **PITR コストの分離**: 本体・MCP・KDB それぞれの保持コストを別計上できる（§ 9.3 のコスト管理に直結）
- **DDL 変更の影響局所化**: KDB 側のスキーマ進化が本体監査検索 GSI を壊さない
- **IAM 越権防止**: テーブル名注入による越境を、ARN ベース IAM ポリシーで物理的にブロック（§ 10.3 AC-S-06 で検証）

---

## 7. インテグレーション要件

### 7.1 OPENCLAW Agent Container からの呼び出し

| 要件 | 内容 |
|---|---|
| 露出方法 | OpenClaw のツール定義として `knowledge_search` / `knowledge_fetch` を登録 |
| 配置 | Workspace Assembler が `_shared/tools/knowledge/*.json` から Tool 定義を `tools/` に配置 |
| Tool 定義スキーマ | OpenClaw の Tool スキーマ（要 OpenClaw 仕様確認） |
| ティア別の利用可否 | Permission Profile（`PERM#{posId}`）で `knowledge_search` の許可可否を制御 |
| 呼び出し時に渡すコンテキスト | requester_id (emp_id), tier, position_id, session_id（KDB が ACL 適用に使う） |

### 7.2 MCP Gateway からの呼び出し

| 要件 | 内容 |
|---|---|
| 上流 MCP 登録 | KDB Service を MCP Gateway の `config/upstreams.yaml` に `knowledge` upstream として登録 |
| ツール定義 | 最小ツールセット: `knowledge__search`, `knowledge__list_kbs`, `knowledge__fetch` |
| AuthN | MCP Gateway が Cognito JWT を取得 → KDB Service への internal SigV4 / IAM 認証 |
| AuthZ | MCP Gateway 3 軸 ACL（user × `knowledge` × tool 名）+ KDB 内 ACL の二段階 |
| ストリーミング | 検索は一括レスポンスで OK、ストリーミング不要 |

### 7.3 Admin Console 統合

| 要件 | 内容 |
|---|---|
| ページ | `apps/admin-console/app/knowledge/`（KB 一覧 / KB 詳細 / 取込 / 検索プレビュー / 監査） |
| 既存統合 | Audit ページに KDB クエリログを統合表示 |
| 権限 | Admin / Manager（自部門 KB のみ）/ Employee（不可）の 3-role RBAC |

### 7.4 Audit Pipeline / Guardrails / Cognito / Azure AD

| 接続先 | 接続方法 |
|---|---|
| Audit Pipeline | `packages/audit-events/AuditRepository(tableName="enterprise-ai-platform-kdb-{env}")` 経由で **KDB 専用テーブル**に書き込み（本体・MCP テーブルとは物理分離）。**S3 長期保管プレフィックス（`audit-archive/kdb/`）のみ既存パイプラインと共有**。DynamoDB Streams → Firehose → S3 Object Lock の二次系は KDB 専用ストリームを構成 |
| Bedrock Guardrails | 検索結果を ティアごとの Guardrails に通過させる（Phase 8 完了後に有効化、それまではバイパス） |
| Cognito | OPENCLAW Phase 6a Step 3 で構築済み User Pool を共有 |
| Azure AD | OPENCLAW Phase 8 完了後に SAML 連携 |

---

## 8. 既存プランとの整合

### 8.1 フェーズ組み込み提案

KDB を新規 Phase として既存プランに挿入するのではなく、**横断サブシステム**（MCP Gateway と同じ位置付け）として `docs/KNOWLEDGE_DB_PLAN.md`（後続成果物）を新設する案を提案。

ただし、**横断依存**としては:

| KDB マイルストーン | 依存する本体 / MCP フェーズ | 提供価値 |
|---|---|---|
| KDB-M1（最小検索） | 本体 Phase 1（DynamoDB / S3 / IAM）、Phase 4（AgentCore） | OPENCLAW から `knowledge_search` ツール呼び出し可能 |
| KDB-M2（MCP 統合） | MCP Gateway Phase 2 / 3 完了 | Claude Desktop / Code から検索可能 |
| KDB-M3（Admin UI） | 本体 Phase 6b（Admin Console 基盤） | 管理者が KB を操作可能 |
| KDB-M4（Guardrails / 監査強化） | 本体 Phase 8 | 機密分類 + 長期保管 |

### 8.2 既存 Phase 番号との横断依存

```
本体 Phase 1 ──→ KDB-M1（最小検索）─┐
本体 Phase 4 ──→ ──────────────────┤
                                   ├─→ KDB-M2 ←── MCP Phase 3
MCP Phase 2/3 ─→ ─────────────────┘
本体 Phase 6b ─→ KDB-M3（Admin UI）
本体 Phase 8 ──→ KDB-M4（強化）
```

### 8.3 既存 `directory_kb.py` の位置付け再整理（§ A OQ-06 確定: 共存）

`directory_kb.py` は **KDB と共存**させる。既存プラン Phase 9 Step 5 のまま独立実装として残し、KDB は別経路（ランタイム検索）を提供する。

| 機能 | 担当 | 備考 |
|---|---|---|
| 組織情報を SOUL に **コールドスタート時注入** | `directory_kb.py`（Phase 9 Step 5、既存構想のまま） | 真実の源 = HR / 組織テーブル |
| 組織情報も含めた **ランタイム検索** | KDB（新規） | 別経路。組織情報を KDB に投入したい要求が将来発生した場合は別途要件追加で再検討 |
| 両者の整合性 | 各自が HR / 組織テーブルを参照（真実の源は同一） | 二重投入の必要があれば、組織情報を KDB に「定期同期」する形で対応（OQ 化候補） |

**既存プラン Phase 9 Step 5 への影響**:

なし。既存計画どおり `services/control-plane/api/services/directory_kb.py` として独立実装する。KDB-M1 は Phase 9 Step 5 と独立して進行可能。

### 8.4 整合性確認チェック（既存プランとの矛盾検出）

| 観点 | 既存プラン記載 | KDB 要件 | 矛盾の有無 |
|---|---|---|---|
| Bedrock KB の採用 | OPENCLAW § 2.1 AI Layer に「Knowledge Bases (RAG)」と明記 | 採用（ADR 0002 起票予定） | **矛盾なし** |
| S3 `knowledge/` prefix | OPENCLAW § 7, § 8 に既存 | 流用 | **矛盾なし** |
| DynamoDB シングルテーブル | OPENCLAW § 8 | KDB 専用テーブルに分離（§ 6.5） | **矛盾なし**（MCP Gateway と同方式で揃える） |
| 単一リージョン | OPENCLAW § 5.4 | 遵守 | **矛盾なし** |
| Python 3.12 統一 | ADR 0001 | 遵守 | **矛盾なし** |
| MCP Gateway 別テーブル | MCP § 4.2 | KDB も同様に分離（ADR 0004 起票予定） | **矛盾なし**（同方針で統一） |
| 4 ティアランタイム | OPENCLAW § 2.3 | ACL に反映 | **矛盾なし** |
| 監査 DynamoDB Streams → Firehose → S3 | OPENCLAW Phase 1 Step 5 | KDB 専用ストリーム + 共通 S3 プレフィックス | **矛盾なし** |
| 500 ユーザー上限 | CLAUDE.md | スケール要件で遵守 | **矛盾なし** |
| Audit IAM 境界テスト | OPENCLAW Phase 10 Step 3b（本体 / MCP 2 系統） | KDB を加えて 3 系統に拡張（§ 10.3 AC-S-06） | **拡張**（既存テストの自然延長） |
| MVP コスト厳格化 | OPENCLAW § 1.4 で「1 ユーザー / 月 $5 以下」 | KDB MVP を 100〜200 USD/月に厳格化（§ 9.3、§ 4.3） | **矛盾なし**（本体予算 500〜1,000 USD/月に吸収） |
| `directory_kb.py` の扱い | OPENCLAW Phase 9 Step 5 | 既存構想のまま独立実装として残し、KDB と共存（OQ-06 確定: 共存） | **矛盾なし** |

### 8.5 既存プラン修正提案（`docs/OPENCLAW_PLATFORM_PLAN.md` への差分）

本要件定義の承認後、以下の差分を本体プランに反映する提案。**実際の編集は別タスク**として扱い、本書では提案として記載する。OQ-06 が「共存」に確定したため、Phase 9 Step 5 への修正は不要。

#### 修正提案 (a): § 2.1 AI Layer の追記

「Bedrock Knowledge Bases (RAG)」の項目に「KDB サブシステム（`docs/KNOWLEDGE_DB_REQUIREMENTS.md`）を介してアクセスする」と注記を追加。

#### 修正提案 (b): § 8 技術スタック「データストア戦略」表の追記

| データ | ストア | 形式 |
|---|---|---|
| **KDB メタ / 監査** | **DynamoDB `enterprise-ai-platform-kdb-{env}`** | **シングルテーブル（KDB 専用）** |
| **KDB ベクター / Bedrock KB** | **Bedrock Knowledge Bases (managed)** | **マネージド** |

#### 修正提案 (c): § 6 コストリスク表に KDB 行追加

| 想定 | 500 ユーザー規模 / 月 |
|---|---|
| **KDB（Bedrock KB + 埋め込み + ECS）** | **$200〜$400** |

合計が約 $700〜$1,400 / 月になるため、本体目標「1 ユーザー / 月 $5 以下」（500 ユーザーで $2,500）の枠内に収まる。

---

## 9. リスクと制約

### 9.1 技術リスク

| リスク | 影響 | 緩和策 |
|---|---|---|
| 埋め込みモデルドリフト（モデル更新で過去ベクターが古くなる） | 検索品質低下 | バージョン固定、再インデックス API、KB 単位の選択（最終版） |
| PII リーク（埋め込み済みチャンクから PII が検索結果に） | コンプライアンス違反 | 取り込み時 PII マスキング + 検索結果 Guardrails |
| 検索品質（ヒット率 50% 以下） | 利用低迷 | ゴールデンセット評価、リランキング追加（M2 以降）、HyDE 検討 |
| **コスト爆発（Bedrock KB の従量課金）** | **予算超過（本体予算を逼迫）** | **月次予算アラート閾値: KDB 単独 250 USD/月で警告、400 USD/月でハードストップ（取り込み停止）**、月次 KPI 監視、KB 単位の上限、長期未使用 KB の冷却、§ 4.3 MVP 厳格化 |
| Bedrock KB のリージョン制約 | サービス利用不可 | AgentCore 同一リージョン強制、リージョン選定を Phase 1 で確定（OQ-01） |
| 取り込みパイプライン障害 | 鮮度低下 | リトライ、デッドレターキュー、Admin Console 可視化 |
| 検索レイテンシ未達 | UX 低下 | キャッシュ層、リランキング無効化（MVP は元から無効）、KB 物理分割 |

### 9.2 運用リスク

| リスク | 緩和策 |
|---|---|
| ACL 設定漏れ | デフォルト deny、検索プレビューで管理者が事前確認、CI でテンプレ ACL の妥当性検証 |
| 再インデックスダウンタイム | バージョン世代管理、新インデックス完成後に切替（無停止） |
| ドキュメント取込失敗が放置 | 失敗時 SNS 通知、Admin Console に失敗一覧、TTL で自動再試行 |
| 管理者の KB 削除事故 | 削除前に 7 日 soft-delete、Admin だけが hard-delete 可能 |
| KDB テーブルへの越権アクセス | IAM Resource ARN 厳格指定、IAM Simulator API テスト（§ 10.3 AC-S-06） |

### 9.3 コスト見積もり（本体予算 500〜1,000 USD/月に吸収 / § A OQ-07 確定）

> § 4.3 で MVP 規模を厳格化した結果、KDB 単独で MVP 100〜200 USD/月、最終目標 200〜400 USD/月に抑える。本体予算 500〜1,000 USD/月の枠内で吸収する運用基準とする。

| 項目 | **MVP（厳格: 5 KB / 3,000 doc / 月 3,000 クエリ）** | 最終（50 KB / 100,000 doc / 月 50,000 クエリ） |
|---|---|---|
| Bedrock KB ベクター保存 | $10〜$25 | $80〜$200 |
| 埋め込み計算（Titan v2 固定、初回 + 月次更新） | $5〜$15 | $50〜$120 |
| Bedrock Retrieve API | $3〜$10 | $30〜$100 |
| ECS Fargate (KDB Service, 2AZ × 2 task) | $40 | $80 |
| ALB（既存共有可で実質 $0〜$10） | $10 | $15 |
| S3（原本保管） | $3 | $20 |
| DynamoDB（KDB 専用テーブル On-Demand） | $5〜$15 | $30〜$80 |
| CloudWatch / X-Ray | $5 | $20 |
| **小計（KDB 単独）** | **約 $80〜$130** | **約 $300〜$640** |
| **運用基準上限** | **MVP 200 USD/月** | **最終 400 USD/月** |
| **予算アラート閾値** | 250 USD/月で警告、400 USD/月でハードストップ | 同左 |

**コスト管理ポリシー**:
- 本体予算 500〜1,000 USD/月 + MCP 220 USD/月 + KDB 200〜400 USD/月 = **合計 920〜1,620 USD/月**（本体 § 1.4 の「1 ユーザー / 月 $5 以下」目標を維持）
- CloudWatch Billing Alarm で KDB タグ付けされたコストを日次トラッキング（NFR-OBS-05）
- ハードストップ発火時: 取り込みパイプライン停止、検索は継続、Admin Console に通知

### 9.4 法的・データ越境

- 取り込みファイルが個人情報を含む場合の取扱い（GDPR / 個人情報保護法）→ Open Question OQ-08
- 単一リージョン制約は守られる（Bedrock KB / 埋め込み / 検索すべて単一リージョン）
- 外部 SaaS 同期（Confluence / Notion）導入時のデータ越境リスク → Open Question OQ-09

---

## 10. 受け入れ基準（要件達成の判定）

> **AC ID の独立性**: 本書の `AC-F-xx` / `AC-P-xx` / `AC-S-xx` / `AC-O-xx` は **本書内で完結する ID 体系**であり、本体 OPENCLAW / MCP Gateway の同名 ID とは独立。横断参照する場合はファイル名込みで指定する（例: 本書 `AC-F-04`「3 層 SOUL 階層 ACL」≠ 本体 `AC-F-04`「Auto-Provisioning」）。書間で同一テストを指す場合のみ本文中で明示的にクロスリンクする（例: 本書 AC-S-06 ↔ 本体 AC-S-11）。

### 10.1 機能受け入れ基準

| ID | 基準 |
|---|---|
| AC-F-01 | OPENCLAW Agent Container から `knowledge_search` ツールで KB を検索し、引用付き回答を生成できる |
| AC-F-02 | Claude Desktop から MCP Gateway 経由で `knowledge__search` ツールが呼び出せ、結果を Markdown で取得できる |
| AC-F-03 | Admin Console から KB 作成 → ドキュメントアップロード → インデックス完了が 5 分以内に完了する |
| AC-F-04 | 3 層 SOUL 階層に対応した KB ACL（Global / Position / Personal）が正しく動作する |
| AC-F-05 | 4 ティアランタイム（Standard / Restricted / Engineering / Executive）に応じた KB 可視性が正しく適用される |
| AC-F-06 | デフォルト deny: ACL 未設定 KB は誰からも見えない |
| AC-F-07 | GDPR 削除権: 該当文書 + ベクター + 監査ログ匿名化が 30 日以内に完了 |
| AC-F-08 | 再インデックスがダウンタイムゼロで実行できる |
| AC-F-09 | 監査ログから「特定従業員の過去 24h の検索クエリ」を 2 秒以内に抽出できる |
| AC-F-10 | 取り込み失敗時に Admin Console に通知が表示され、ワンクリックでリトライできる |

### 10.2 性能受け入れ基準

| ID | 基準 |
|---|---|
| AC-P-01 | 検索 p95 < 1,500ms（500 ユーザー時、50 QPS） |
| AC-P-02 | ACL 評価 p95 < 20ms |
| AC-P-03 | 取り込み 1 ドキュメント < 60s（10 ページ PDF 基準） |
| AC-P-04 | 100,000 チャンクの KB で検索 p95 < 1,500ms |

### 10.3 セキュリティ受け入れ基準

| ID | 基準 |
|---|---|
| AC-S-01 | 4 ティア間のクロステナント漏洩テスト 50 ケース以上が全パス（本体 Phase 2 同様の CI 必須） |
| AC-S-02 | PII 検出テストセット 100 件で検出率 95%+ |
| AC-S-03 | 全 KB に SSE-KMS が適用されている（Terraform で強制） |
| AC-S-04 | IAM Access Analyzer で過剰権限ゼロ |
| AC-S-05 | 監査ログが全クエリで 99.9%+ の成功率で書き込まれる |
| AC-S-06 | **AuditRepository IAM 境界テスト（3 系統）**: KDB タスクロールが本体 / MCP テーブルへ越境アクセスできないことを二段階検証で全件拒否（本体 § 10.3 **AC-S-11** と同方針、3 書共通テストスイートとして実装）。検証手段の詳細（IAM Simulator + 実 STS AssumeRole）と 3 系統 × 越境先 2 系統 = 6 ケースの全網羅は本体 AC-S-11 注記を参照 |

### 10.4 観測性受け入れ基準

| ID | 基準 |
|---|---|
| AC-O-01 | CloudWatch ダッシュボードに検索 QPS / レイテンシ p50/p95/p99 / 取込成功率 / ヒット率 / コストが表示 |
| AC-O-02 | X-Ray トレースで API → ベクター検索 → リランキング（M2 以降）の内訳が見える |
| AC-O-03 | 月次ヒット率レポート（ゴールデンセット）が自動生成される |
| AC-O-04 | **月次コストレポート**: CloudWatch Billing Alarm で KDB タグ付けされたコストを日次トラッキング、250 USD で警告、400 USD でハードストップが発火し Admin Console に通知 |

---

## 11. 未決事項と意思決定が必要な論点（Open Questions）

更新後 Open Questions 残: **10 件**（OQ-01, 02, 04, 08, 09, 10, 11, 12, 13, 14）。OQ-03, 05, 06, 07, 15 は § A 確定済み事項に移動。OQ-16（旧 directory 統合時の取得経路）は OQ-06 の「共存」確定により消滅。

| ID | 論点 | 判断者 | 期限 |
|---|---|---|---|
| OQ-01 | Bedrock KB の単一リージョン制約と AgentCore リージョン制約の整合性事前確認（双方が us-east-1 / us-west-2 をサポートする最新状況） | アーキテクト | 要件確定前 |
| OQ-02 | 埋め込みモデルの最終版選定: Titan v2 を継続するか、Cohere Embed Multilingual に切替か。日本語 100 クエリのゴールデンセットでベンチマークを取る | アーキテクト + プロダクトオーナー | KDB-M1 完了時の振り返り |
| OQ-04 | KDB Service への直接アクセスは禁止し OPENCLAW / MCP Gateway 経由のみ許可するか（IAM Boundary 設計） | セキュリティ | KDB-M1 着手前 |
| OQ-08 | 取り込みファイルに含まれる個人情報の取扱い方針（マスキング基準、保持期間、削除権 SLA） | コンプライアンス | KDB-M1 設計時 |
| OQ-09 | 外部 SaaS 同期（Confluence / Notion 等）の MVP 取扱い（範囲 / セキュリティ / データ越境） | プロダクトオーナー | M2 設計時 |
| OQ-10 | チャンク戦略: MVP は固定サイズのみか、セマンティック / Markdown 構造も含めるか | アーキテクト | KDB-M1 設計時 |
| OQ-11 | 監査ログのクエリ生文保管 vs ハッシュ保管（プライバシー / 検索品質改善の両立） | セキュリティ + プロダクトオーナー | KDB-M1 着手前 |
| OQ-12 | OCR（スキャン PDF）対応: Textract 採用 / 採用しない / 後続 | プロダクトオーナー | KDB-M1 着手前 |
| OQ-13 | MCP `knowledge__*` 名前空間のツール最小セット承認（`search` / `list_kbs` / `fetch` で十分か） | アーキテクト | KDB-M2 着手前 |
| OQ-14 | OPENCLAW からの利用 IF: Workspace Assembler 経由プリフェッチ + ランタイム検索の両方提供か、ランタイム検索のみか | アーキテクト | KDB-M1 設計時 |

### 11.1 ADR 起票決定事項（§ A OQ-15 確定）

本要件定義の承認後、以下 3 件の ADR を確定タスクとして起票する。OQ-06 が「共存」に確定したため ADR 0005（`directory_kb` 統合）は不要となり起票しない。

#### ADR 0002: KDB のストレージに Bedrock Knowledge Bases を採用する

- **Context**: KDB のベクターストア選定で Bedrock KB / OpenSearch Serverless / Aurora pgvector / DynamoDB 自作の 4 候補を比較した。500 ユーザー規模・単一リージョン制約・既存 Bedrock 統合・運用負債最小化の観点でマネージドサービスが望ましい。
- **Decision**: Bedrock Knowledge Bases (managed) を採用する。埋め込みは Titan v2 を MVP で固定し、リトリーバは Bedrock KB の Retrieve API を直接利用。
- **Consequences**: us-east-1 / us-west-2 リージョン制約に従う必要があり、AgentCore と同一リージョンで統一できる。OpenSearch Serverless 採用の場合に比べて MVP コストが 50% 程度削減できる一方、検索アルゴリズム / インデックス構造のカスタマイズ余地は限定される（リランキングは別レイヤー追加で対応）。

#### ADR 0003: KDB を独立 ECS Fargate サービスとして配置する

- **Context**: OPENCLAW Agent Container と MCP Gateway の双方から呼び出される KDB のアクセス方式で「共有 API レイヤー」「OPENCLAW 経由」「Bedrock KB 直接」の 3 候補を比較した。ACL / 監査 / Guardrails の単一権威を保ちつつ、既存パターンに整合させる必要があった。
- **Decision**: KDB を独立 ECS Fargate サービス（`services/knowledge-db/`）として配置し、OPENCLAW Agent Container はランタイム検索ツール経由で HTTP 呼び出し、MCP Gateway は KDB を上流 MCP として登録する。
- **Consequences**: サービス数が 1 つ増えるがホップ増は 1 段で抑えられる。監査・ACL・Guardrails のロジックを単一箇所に集約でき、MCP Gateway 側は薄いプロキシで済む。Tenant Router / Bedrock Proxy / Agent Container と同じ ECS Fargate / ALB パターンを再利用するため運用ノウハウ共有可。

#### ADR 0004: KDB の DynamoDB テーブルを本体 / MCP Gateway から分離する

- **Context**: 監査ログ・KB メタ・取込ジョブを格納する DynamoDB テーブルを、本体 `enterprise-ai-platform-{env}` に同居するか、MCP Gateway 同様に専用テーブルを分離するかを検討した。検索クエリのバーストと組織 CRUD のスパイクが互いの I/O スロットルを侵食すること、PITR コストの結合、DDL 変更影響の局所化が論点。
- **Decision**: KDB 専用テーブル `enterprise-ai-platform-kdb-{env}` を新設し、本体・MCP Gateway テーブルから物理分離する。`packages/audit-events/AuditRepository(tableName)` 抽象でテーブル切替し、スキーマは 3 系統で共通。S3 長期保管プレフィックスのみ既存パイプラインと共有する。
- **Consequences**: PITR / TTL / On-Demand 容量設定が独立し、コストとパフォーマンスを個別チューニングできる。IAM Resource ARN を厳格指定することで越権アクセスを物理的にブロックでき、3 系統間の境界テスト（§ 10.3 AC-S-06）が CI で担保される。テーブル分割管理の運用負担は AuditRepository 抽象で吸収。

---

## § A. 確定済み事項（OQ 解決ログ）

本セクションは Open Questions の確定履歴を集約する。日付は ISO 8601、決定者はユーザー（プロダクトオーナー）。

| OQ ID | 確定日 | 決定者 | 決定内容 | 根拠（1 行） | 影響を与えた本文セクション |
|---|---|---|---|---|---|
| OQ-03 | 2026-05-12 | ユーザー | リランキング MVP 不採用 / M2 以降で再評価 | MVP コスト厳格化（本体予算吸収）と性能要件を MVP では Bedrock KB Retrieve API のみで達成可能と判断 | § 3.3 (FR-SRCH-05), § 4.3, § 5.4 |
| OQ-05 | 2026-05-12 | ユーザー | KDB 専用 DynamoDB テーブル `enterprise-ai-platform-kdb-{env}` を新設 | I/O 競合・容量設定干渉・PITR コスト結合・DDL 変更影響の回避（MCP Gateway と同方針） | § 4.5 (NFR-TEN-05), § 6.5, § 7.4, § 8.4, § 10.3 (AC-S-06) |
| OQ-06 | 2026-05-12 | ユーザー | `directory_kb.py` は **KDB と共存**させる（当初「統合」と確定後、同日「共存」に再修正） | 役割が重複しない（SOUL 自動注入 vs ランタイム検索）。組織情報を KDB に投入する要求が将来発生したら別途要件追加で再検討 | § 1.3, § 8.3, § 8.4 |
| OQ-07 | 2026-05-12 | ユーザー | KDB コストは本体予算（500〜1,000 USD/月）に吸収、MVP 規模を厳格化 | コスト爆発リスクを MVP 段階で抑制、本体「1 ユーザー / 月 $5 以下」目標を維持 | § 3.2 (FR-PIPE-08), § 4.3, § 4.8 (NFR-OBS-05), § 9.1 (コスト爆発), § 9.3, § 10.4 (AC-O-04) |
| OQ-15 | 2026-05-12 | ユーザー | ADR 0002 / 0003 / 0004 を本要件定義承認後に起票（OQ-06「共存」確定により ADR 0005 は不要） | 重要設計判断（ストレージ / 配置 / テーブル分離）を ADR で恒久記録 | § 11.1 |

---

## 12. 用語集

| 用語 | 定義 |
|---|---|
| KDB | Knowledge DB。本ドキュメントで定義する汎用 RAG サブシステム |
| KB | Knowledge Base。KDB 内の論理単位 |
| `directory_kb` | 本体プラン Phase 9 Step 5 の組織ディレクトリ KB（小規模 Markdown 自動注入機構）。KDB とは独立して共存（OQ-06 確定） |
| Bedrock KB | AWS Bedrock Knowledge Bases（マネージド RAG サービス） |
| Chunk | Document を埋め込み単位に分割した断片 |
| Embedding | テキストをベクター空間に写像した数値表現 |
| RAG | Retrieval-Augmented Generation。検索結果を LLM のコンテキストに与える手法 |
| Reranking | 検索結果の二次並べ替え。クロスエンコーダ等で精度を上げる（KDB は MVP 不採用） |
| HyDE | Hypothetical Document Embeddings。仮の回答を生成して検索する手法 |
| ACL | Access Control List |
| 3 層 SOUL | OPENCLAW のアイデンティティ階層。Global（IT がロック）/ Position（部門管理者）/ Personal（従業員）の Markdown を、**上位が下位を上書きできない** マージ規則で結合する。マージ時、上位レイヤーは `CRITICAL IDENTITY OVERRIDE` ヘッダ付きで先頭にプリペンドされる。詳細は本体要件書 § 3.2 / 本体プラン § 2 |
| `CRITICAL IDENTITY OVERRIDE` | 上位 SOUL レイヤーが下位を上書きできないことを示す先頭ヘッダ。本体要件書 § 3.2 FR-SOUL-05 |
| 4 ティアランタイム | Standard / Restricted / Engineering / Executive |
| 5 層多層防御 | L1 SOUL ルール / L2 ツール許可 / L3 IAM / L4 コンピュート分離 / L5 Guardrails |
| MCP | Model Context Protocol |
| Walking Skeleton | 最小限のエンドツーエンド動作を最早期に確認するパターン |
| `AuditRepository(tableName)` | `packages/audit-events/` の抽象。テーブル名を引数に取り、本体 / MCP / KDB の 3 系統に同一スキーマで書き込む |

---

## 完了確認

- [x] 12 セクション + § A 確定済み事項すべて記述
- [x] OQ-03 / 05 / 06 / 07 / 15 の確定内容を本文に反映
- [x] § 4.3 スケーラビリティ表に「MVP（厳格）」列追加
- [x] § 4.5 に NFR-TEN-05（テーブル分離）追加
- [x] § 6.5 を「DynamoDB テーブル分離」に書き換え、IAM 境界方針を明示
- [x] § 7.4 を「KDB 専用テーブルに書き込み、S3 プレフィックスのみ共有」に修正
- [x] § 8.4 整合性チェック表に 3 行追加（IAM 境界拡張 / MVP コスト厳格化 / `directory_kb` 共存）
- [x] § 8.5 既存プラン修正提案ブロック新設（OQ-06 共存確定により Phase 9 Step 5 への修正は不要）
- [x] § 9.1 コスト爆発緩和策に予算アラート閾値追加
- [x] § 9.3 コスト見積もり表を MVP 厳格化版に書き換え、本体予算吸収方針を明示
- [x] § 10.3 に AC-S-06（3 系統 IAM 境界テスト）追加
- [x] § 10.4 に AC-O-04（月次コストレポート）追加
- [x] § 11 Open Questions を 10 件に更新（OQ-03 / 05 / 06 / 07 / 15 削除、OQ-16 は OQ-06「共存」確定により消滅）
- [x] § 11.1 を「ADR 起票決定事項」に格上げ、ADR 0002 / 0003 / 0004 の骨子を 3〜5 行ずつ記載（ADR 0005 は OQ-06 共存確定により不要）
- [x] § A 確定済み事項セクション新設、表で集約
- [x] § 12 用語集に `AuditRepository(tableName)` を追加
- [x] 既存パターン（DynamoDB シングルテーブル / Python 3.12 / Single region / 監査パイプライン / Guardrails）の再利用方針明示
- [x] スコープ縮小の精神尊重（500 ユーザー前提、MVP 厳格化、過剰機能なし）

---

