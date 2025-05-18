# DynamicExitStrategyImplementation ドキュメント索引

このドキュメントは、DynamicExitStrategyImplementationプロジェクトに関連するすべてのドキュメントへのリンクを提供します。

## 入門ガイド

- [**クイックスタートガイド**](./guides/quickstart-guide.md) - プロジェクトの基本的な使用方法と設定
- [**実装ガイド**](./implementation-guide.md) - コンポーネントの実装方法とパターン
- [**トラブルシューティングガイド**](./guides/troubleshooting-guide.md) - 一般的な問題と解決策
- [**バージョン管理と変更履歴ガイド**](./guides/versioning-changelog-guide.md) - バージョン付けとCHANGELOG管理

## アーキテクチャ文書

- [**プロジェクト全体ドキュメント**](./project-documentation.md) - プロジェクトの概要と全体像
- [**アーキテクチャ概要**](./architecture-overview.md) - システムアーキテクチャの詳細

### モジュール別アーキテクチャ文書

- [**アーキテクチャ概要図**](./architecture/architecture-diagram.md) - モジュール間の関係と依存関係の視覚的表現
- [**Coreモジュールアーキテクチャ**](./architecture/core-architecture-patterns.md) - 基盤となる共通パターンと依存関係
- [**Data/Buffersモジュールアーキテクチャ**](./architecture/data-buffers-architecture.md) - データ構造とバッファ管理の実装
- [**EntryLogic/ExitLogicモジュールアーキテクチャ**](./architecture/entry-exit-strategy-architecture.md) - トレーディング戦略の実装
- [**Indicators/Fibonacciモジュールアーキテクチャ**](./architecture/indicators-fibonacci-architecture.md) - テクニカル指標とフィボナッチツール
- [**スレッド安全パターン**](./architecture/thread-safety-patterns.md) - 同期メカニズムと並行処理

## 開発者ガイド

- [**テストガイド**](./guides/testing-guide.md) - コンポーネントのテスト方法と検証手順
- [**共通ユーティリティ使用ガイド**](../common-utilities-usage-guide.md) - ユーティリティクラスとヘルパー関数の使用方法
- [**エラーハンドリングベストプラクティス**](../error-handling-best-practices.md) - エラー処理とリカバリの推奨方法

## 分析と改善提案

- [**コード重複と統合機会**](./analysis/code-duplication-consolidation.md) - 重複パターンとリファクタリング提案
- [**プロジェクト概要と改善提案**](./analysis/project-summary-and-improvements.md) - 総合的な分析と改善計画

## リファクタリングと移行

- [**名前空間移行計画**](../namespace-migration-plan.md) - ScalpingStrategyProjectからの移行
- [**リファクタリング遵守規約**](../リファクタリング遵守規約.TXT) - リファクタリングのガイドライン
- [**メソッド統合移行状況**](../メソッド統合移行状況.md) - メソッド統合の進捗状況

## 機能別ドキュメント

- [**入口/出口戦略アーキテクチャ**](./architecture/entry-exit-strategy-architecture.md) - トレーディング戦略の設計と実装
- [**入口/出口戦略パターン**](./strategies/entry-exit-strategy-patterns.md) - 戦略設計と実装パターン
- [**指標とフィボナッチツール**](./architecture/indicators-fibonacci-architecture.md) - テクニカル分析コンポーネント
- [**バッファ管理**](./architecture/data-buffers-architecture.md) - 時系列データの効率的な管理

## 統合とテスト

- [**統合テスト実行計画**](../統合テスト実行計画.md) - 統合テストの計画と手順
- [**テストガイド**](./guides/testing-guide.md) - テスト方法と検証手順

## TradingPlatform / Quantower API 統合

- [**Quantower API ガイド**](./integration/quantower-api-guide.md) - Quantower APIとの整合性を保つためのガイドライン
- [**TradingPlatform 統合ガイド**](./integration/tradingplatform-integration-guide.md) - TradingPlatform.BusinessLayerとの統合方法
- [**Quantower Core コードインデックス**](./integration/quantower-core-code-index.md) - Core APIの主要コンポーネント
- [**Quantower サンプルコード集**](./integration/quantower-sample-code.md) - 一般的なユースケースのコード例

## まとめドキュメント

- [**リファクタリング概要（最終）**](../refactoring_summary_final.md) - リファクタリングの最終概要
- [**出口戦略リファクタリング概要**](../exit-strategy-refactoring-summary.md) - 出口戦略のリファクタリング概要
- [**出口戦略マージ結果**](../exit-strategy-merge-results.md) - 出口戦略の統合結果

## 補助ツールとリソース

- [**ロギング標準**](../logging-standards.md) - ロギングの標準とベストプラクティス
- [**エラーハンドリング例**](../error-handling-examples.md) - エラー処理の具体例