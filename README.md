# AIMS v0.2（`aims-engine`）— 完全自律メンテナンス＆自己再生エンジン

[![Spec](https://img.shields.io/badge/Spec-AIMS--SPEC--002-blue.svg)](docs/AIMS_SPEC_v0.2_PUBLIC.md)
[![Dependencies](https://img.shields.io/badge/External_Crates-0_(Pure_Rust_std)-brightgreen.svg)](evidence/verification-report.json)
[![External_API](https://img.shields.io/badge/External_LLM_API-0_(Offline_Autonomous)-brightgreen.svg)](docs/AIMS_SPEC_v0.2_PUBLIC.md)
[![Tests](https://img.shields.io/badge/Tests-9%2F9_Passed-success.svg)](evidence/cargo-test-output.txt)

---

## 概要・コンセプト

**AIMS v0.2（`aims-engine`）** は、ミッションクリカルな Web アプリケーションおよび基幹 SQLite データベースシステムを「専任開発者の不在後・長期放置環境・閉域網」においても自律維持するために設計された **Pure Rust（`std` のみ・外部クレート依存ゼロ・外部 LLM / 外部 API 非依存）の単一バイナリ型自律保守エンジン** です。

多くの従来型 AI 運用自動化はクラウド LLM API に依存するため、以下の致命的な運用リスクを抱えていました。

- 数年単位の長期運用における **モデル廃止・API 仕様変更・API キー失効**
- 障害発生ループ時における **従量課金の暴走やクラウド障害の連鎖影響**
- LLM 出力の不確定性による **認証ロジック・DB スキーマ・業務データの破壊リスク**

`aims-engine` はこれらの課題を構造的に解決するため、**外部通信・外部依存を完全に排除した 4 段階エスカレーション推論** と **三権分立型の独立審査機構（`IndependentJudge`）**、さらに **隔離領域からの自己修復機構（`selfheal-vault`）** を単一バイナリ（約 550 KB）へ統合しています。

詳細な技術仕様と全体アーキテクチャ図は [公開用技術仕様書（AIMS_SPEC_v0.2_PUBLIC.md）](docs/AIMS_SPEC_v0.2_PUBLIC.md) を参照してください。

---

## 4段階自律エスカレーション構造

異常検知時、システムは軽量な処置から構造クリーン再構築へと段階的（各段階最大 2 回試行）にエスカレーションします。

1. **Stage 1: `LightMlp`（`12 -> 16 -> 4` / 無停止軽量修復）**
   - 12 次元のリアルタイム観測特徴量（ヘルスチェック異常、直近エラー数、5xx 比率、DB エラー比率、SQLite WAL サイズ比、期限切れエラーログ、孤立一時ファイル、メモリ・ディスク圧迫率、必須ファイル欠損等）から瞬時に障害種別を分類し、サービスを停止せずに修復します。
   - **実行アクション**：`NoOp`, `FlushWalCheckpoint`, `CleanExpiredLogsAndTemp`, `LightCombinedCleanup`

2. **Stage 2: `HighMlp`（`24 -> 32 -> 24 -> 4` / LayerNorm ＋ GELU 近似活性化）**
   - Stage 1 で解消しない場合、もしくは複合障害に対して自動昇格します。時系列のエラー増減勾配や WAL 増加率、機能領域別エラー偏り、静的アセット欠損指数を含む 24 次元特徴量から多層推論を行います。
   - **実行アクション**：`RestartAppService`, `DatabaseOptimizeAndBackup`, `SyncStandaloneAssets`, `ServiceAndProxyRecovery`

3. **Stage 3: `DqhmCleanRebuild`（360-DQHM 連想記憶による OS クリーンインストール方式の再構築）**
   - Stage 2 でも回復できないコード破損・設定破壊・ファイル改ざん・欠損に対して昇格します。
   - **360 次元複素位相連想記憶（360-DQHM）**：正常稼働時のアプリケーション各モジュールの構造を 360 次元単位複素位相ベクトルとして黄金記憶（Golden Memory / `golden-dqhm.json`）に保持し、位相共鳴類似度（`O(1)`）によって破損層を特定します。
   - **基幹 DB 保護 ＋ 構造クリーン再展開**：業務基幹データベース（`operations.db` 等）を整合確認の上で安全に退避保護（物理破損時のみ事前の整合検証済みバックアップから復旧）し、アプリケーション本体を黄金構造アーカイブから OS クリーンインストール方式で再構築します。
   - **想起不変・3 層分離追記ログ**：想起処理（Recall）による黄金記憶の上書きを禁止し、経験（`episodes.jsonl`）・活動（`actions.jsonl`）・学習改訂（`revisions.jsonl`）を追記型（Append-only）で厳格に分離保存します。

4. **Stage 4: `SafeHaltWithReport`（安全停止と診断レポート出力）**
   - Stage 3 でも解決しないハードウェア・物理環境レベルの異常時は、任意の外部拡張スロット（非 LLM 標準運用では無効化済み）を経由後、基幹データベースと黄金状態を保護して安全停止（終了コード `3`）し、詳細診断レポート（`safe-halt-report.json`）を保存します。

---

## 三権分立の安全機構（`IndependentJudge`）

修復を設計・実行する推論 AI と、その正当性を審査する判定機構を完全に分離しています。

- **別経路での基準共有**：最上位窓口（Dispatcher）が外部判定基準定義（サンプル：[config/aims-criteria.sample.json](config/aims-criteria.sample.json)）を読み込み、修復 AI とは独立した経路で `IndependentJudge` へ渡します。
- **実行前設計審査（Pre-execution Gate）**：`forbiddenOperations`、保護対象パス（認証ロジック、スキーマ、マイグレーション、テスト、基準ファイル等）への変更、および当該 Stage で許可されていない操作を含む修復計画を即座に拒否します。
- **Phase 事後審査（Post-phase Gate）**：修復処置ごとに以下を機械的に検証します。
  - SQLite `PRAGMA integrity_check;` が `"ok"` であること
  - SQLite `PRAGMA foreign_key_check;` 違反が `0` 件であること
  - 主要業務テーブルのレコード件数が処置前を下回らないこと
  - 保護対象ファイルの SHA-256 ハッシュが黄金記憶と一致すること
  - ヘルスチェックエンドポイント（`/api/health`）が HTTP 200 を返すこと
- **サーキットブレーカー**：1 段階あたり最大 2 回失敗で自動昇格し、1 日の最大自律修復実行回数を `6` 回に制限して暴走ループを防ぎます。

---

## 隔離自己修復機構（`selfheal-vault`）

業務システム側を監視・修復する 5 分周期タイマーとは **完全に独立した 15 分周期タイマー** により、エンジン自身の健全性を検証します。

- 隔離保管庫（`selfheal-vault`）に `aims-engine` の実行バイナリ、初期重みファイル、判定基準 JSON、および黄金構造記憶の SHA-256 マニフェストとバックアップを隔離保管。
- `aims-engine self-heal` の実行時、エンジン構成ファイルや自身のファイルに破損・欠損が検知された場合、業務アプリケーションや基幹 DB に触れずに Vault から自動原状復帰させます。

---

## CLI コマンド仕様

```bash
# 正常稼働状態の黄金構造記憶 (360-DQHM) と隔離自己修復 Vault を初期化
aims-engine init-golden --app-root /opt/app --data-dir /var/lib/app --criteria /opt/app/config/aims-criteria.json

# 現在のヘルス状態・DB整合性・360-DQHM類似度・実行履歴をJSON表示
aims-engine status

# 定期監視および段階エスカレーション自律修復を実行 (5分周期タイマー推奨)
aims-engine check-and-heal

# エンジン自身のバイナリ・重み・基準・記憶を隔離VaultとSHA-256照合し自己修復 (15分周期タイマー推奨)
aims-engine self-heal
```

利用可能なオプション一覧：

```text
使い方: aims-engine <init-golden|check-and-heal|self-heal|status> [--app-root DIR] [--data-dir DIR] [--db-path FILE] [--error-log-dir DIR] [--aims-dir DIR] [--criteria FILE] [--health-url URL] [--health-host HOST] [--service-name UNIT]
```

---

## 検証実績とリリースバイナリ情報

| 検証・成果物項目 | 実測値 / 確認結果 | 参照証拠 |
| :--- | :--- | :--- |
| **リリースバイナリ配置** | `releases/aims-engine.exe`（Windows x86_64 MSVC） | [releases/aims-engine.exe](releases/aims-engine.exe) |
| **ファイルサイズ** | `562,688` bytes | [verification-report.json](evidence/verification-report.json) |
| **SHA-256 ハッシュ** | `964b1a93d2112cc1346cbe1d281f4fb74fc8145174990206426355908d740331` | [verification-report.json](evidence/verification-report.json) |
| **ビルド保護設定** | `strip = "symbols"`, `lto = true`, `codegen-units = 1`, `panic = "abort"`, `opt-level = 3`（COFF シンボル 0） | [verification-report.json](evidence/verification-report.json) |
| **外部クレート依存数** | **`0`**（`Cargo.toml` `[dependencies]` 空 / Pure Rust `std` のみ） | [verification-report.json](evidence/verification-report.json) |
| **単体・結合テスト** | **全 9 件合格 / 失敗 0 件**（`cargo test` / 所要時間 19.70s） | [cargo-test-output.txt](evidence/cargo-test-output.txt) |
| **生ソース・DB・設定非同梱** | `.rs`, `.db`, `.env` ファイル **0 件**（ソース非公開） | [LICENSE_AND_POLICY.md](LICENSE_AND_POLICY.md) |

---

## パッケージ構成

```text
dist-aims-public-showcase/
├── LICENSE_AND_POLICY.md
├── README.md
├── config/
│   └── aims-criteria.sample.json
├── docs/
│   └── AIMS_SPEC_v0.2_PUBLIC.md
├── evidence/
│   ├── cargo-test-output.txt
│   └── verification-report.json
└── releases/
    └── aims-engine.exe
```

ライセンス・知的財産保護・OEM 導入に関する方針は [LICENSE_AND_POLICY.md](LICENSE_AND_POLICY.md) をご覧ください。
