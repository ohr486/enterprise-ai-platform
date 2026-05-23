# ADR 0001: バックエンドサービスを Python 3.12 に統一する

- **ステータス**: Accepted
- **決定日**: 2026-05-12
- **関連プラン**: [OPENCLAW_PLATFORM_PLAN.md § 8](../OPENCLAW_PLATFORM_PLAN.md) / [MCP_GATEWAY_PLAN.md § 7](../MCP_GATEWAY_PLAN.md)

## コンテキスト

`enterprise-ai-platform` の初期設計案では、バックエンドサービスの言語選定で以下の分散を許容していた:

- **Python 3.12**: Control Plane API / Tenant Router / Agent Container / MCP Gateway
- **Node.js 22 LTS + Hono**: Bedrock H2 Proxy（HTTP/2 ストリーミング性能を理由に）

Bedrock H2 Proxy だけ別言語にする運用負債が懸念されたため、再評価した。

## 検討した選択肢

### 選択肢 A: Node.js / Hono を Bedrock H2 Proxy に採用（当初案）

- **利点**: HTTP/2 + ストリーミング性能の参考実装が豊富。Hono は軽量
- **欠点**:
  - SigV4 署名・監査スキーマ・shared types を Python と Node.js で二重メンテ
  - CI の静的解析（Ruff vs Biome）と依存管理（uv vs pnpm）が二系統
  - 1 サービスのみのために運用人材の言語スキルが分散
  - Agent Container（OpenClaw 内蔵）が Python であるため、ストリーミング層を分けると Bedrock 呼び出しのトレーシングが不連続化

### 選択肢 B: Python 3.12 + `httpx[http2]` + `anyio` に統一（採用）

- **利点**:
  - 全バックエンドサービスが単一言語、共通ライブラリ（`packages/shared-python/`）を共有
  - `httpx` は HTTP/2 ネイティブサポート、`anyio` でストリーミング・キャンセル伝播
  - `botocore.auth.SigV4Auth` で署名処理が Python 側に集約
  - Phase 10 の静的解析（Ruff, Trivy, Semgrep）が単一系統
  - Agent Container との運用ノウハウ・shared types がそのまま共有可能
- **欠点**:
  - Hono と比較した素のスループットは劣る可能性（要ベンチマーク）
  - Python の asyncio ストリーミングは Node.js より学習コストが高い実装者がいる場合あり

## 決定

**選択肢 B（Python 3.12 統一）を採用する。**

Bedrock H2 Proxy は `FastAPI + httpx[http2] + anyio` で実装し、Phase 2 完了基準に「p95 レイテンシ閾値 + バックプレッシャ試験」を含めて性能を担保する。

## 影響

- **言語選定（OPENCLAW § 8）**: バックエンド層は Python 3.12 のみを採用。Node.js 22 LTS は不採用
- **モノレポ構成**: `services/gateway/bedrock-proxy/` は Python パッケージとして配置（pyproject.toml）
- **CI/CD**: 静的解析は Ruff に統一、Biome は TypeScript（フロントエンド）のみ
- **共通ライブラリ**: `packages/shared-python/` で SigV4 ヘルパー・httpx クライアントラッパー・監査ロガーを共有

## リスクと緩和

| リスク | 緩和策 |
|---|---|
| Python ストリーミング性能が要件未達 | Phase 2 Step 4 完了基準に p95 < 50ms (透過オーバーヘッド) を含める。未達なら Hono 採用を再検討 |
| `httpx[http2]` の安定性 | 公式リリース版（>=0.27）をピン留め、CI で接続再利用テストを必須化 |
| 学習コスト | `packages/shared-python/` でストリーミング・SigV4 のラッパーを提供し、各サービスで再実装させない |

## 関連

- 旧 ADR: なし（初版）
- 後続検討事項: Bedrock H2 Proxy のベンチマーク結果（Phase 2 完了時点で本 ADR に追記）
