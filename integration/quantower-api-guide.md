# Quantower API Alignment Guide

このドキュメントは、Quantower APIとの整合性を保つためのガイドラインとベストプラクティスを提供します。

## 主要な名前空間とクラス

### 主要な名前空間

- `TradingPlatform.BusinessLayer` - プラットフォームのコア名前空間
- `TradingPlatform.BusinessLayer.Symbol` - マーケットデータとシンボル操作用
- `TradingPlatform.BusinessLayer.Trading` - 注文とポジション用

### 重要なクラス

- `Symbol` - マーケットデータアクセスとクォート購読
- `Position` - ポジションの追跡と管理
- `IOrder` - 注文の作成と管理
- `Quote` と `Last` - マーケットデータ処理
- `Core.Instance` - グローバルプラットフォームサービスへのアクセス

## イベント購読とスレッド安全性のベストプラクティス

### イベント購読パターン

1. **SafeSubscribeパターンの使用**
   - `EventHandlerExtensions.SafeSubscribe`を使用
   - シンボルイベント: `symbol.SafeSubscribeNewQuote(handler, groupName)`
   - ポジションイベント: `position.SafeSubscribeUpdated(handler, groupName)`

2. **グループベースの購読管理**
   - 機能グループごとに購読を整理
   - `EnhancedEventManager`を使用して購読を追跡
   - 管理を容易にするための意味のあるグループ名を常に指定

3. **適切な購読解除**
   - 常に`Dispose`メソッドで購読解除
   - グループベースの購読解除でクリーンアップを簡素化

### スレッド安全性の実装

1. **LockHelper拡張の使用**
   - 読み取り操作: `_lock.WithReadLock(() => { /* 読み取りコード */ })`
   - 書き込み操作: `_lock.WithWriteLock(() => { /* 書き込みコード */ })`
   - 条件付き更新: `_lock.WithUpgradeableReadLock(() => { /* コード */ })`

2. **適切なロック選択**
   - 頻繁な読み取りと稀な書き込みのデータには`ReaderWriterLockSlim`を使用
   - より単純なシナリオには`WithLock()`での単純な`object`ロックを使用
   - 適切なタイムアウト設定（100-200msを推奨）

3. **エラー処理**
   - `ErrorHandling.SafeExecute`でイベントハンドラをラップ
   - より良いデバッグのために常に`operationName`を指定
   - 適切なロギングレベルを使用

## 名前空間の移行計画

### 名前空間移行の手順

1. **プロジェクトファイルの更新**
   - RootNamespaceを`ScalpingStrategyProject`から`DynamicExitStrategyImplementation`に変更

2. **名前空間宣言の更新**
   ```csharp
   // 変更前
   namespace ScalpingStrategyProject.Core.Threading
   
   // 変更後
   namespace DynamicExitStrategyImplementation.Core.Threading
   ```

3. **usingステートメントの更新**
   ```csharp
   // 変更前
   using ScalpingStrategyProject.Core;
   
   // 変更後
   using DynamicExitStrategyImplementation.Core;
   ```

4. **後方互換性のためのアダプターの作成**
   - 重要なコンポーネント用のアダプタークラスを実装
   - 必要に応じて名前空間エイリアスを使用

5. **検証プロセス**
   - 検証付きの自動検索と置換を実行
   - 変更後に包括的な単体テストを実行
   - すべてのコンポーネントが適切に読み込まれることを確認するランタイム検証テストの作成

## ポジション、シンボル、およびイベントの処理パターン

### ポジションとシンボルの管理

1. **SymbolPositionSubscriptionManagerの使用**
   - シンボルとポジションの購読を統一的に管理
   - `LastEventTime`監視によるイベント追跡の確保
   - 適切なクリーンアップによるメモリリークの防止

2. **シンボルとの相互作用**
   ```csharp
   // 適切なシンボル購読
   _symbolManager.SubscribeSymbolEvents(symbol);
   
   // イベント処理
   private void OnSymbolNewQuote(Symbol symbol, Quote quote)
   {
       ErrorHandling.SafeExecute(() => {
           // クォートデータの処理
       },
       operationName: "processing quote update");
   }
   ```

3. **ポジション追跡**
   ```csharp
   // ポジションイベント処理
   private void OnPositionUpdated(Position position)
   {
       ErrorHandling.SafeExecute(() => {
           // ポジション追跡状態の更新
           LastPnL = position.ProfitLoss;
           // 追加処理
       },
       operationName: "handling position update");
   }
   ```

### イベントシステムの統合

1. **EnhancedEventManager**
   - イベント購読に`TradingPlatform.BusinessLayer.Core.Instance.EnhancedManagers().EventManager`を使用
   - グループベースの購読管理を実装
   - 組み込みのヘルスチェックと診断を活用

2. **定期的なイベントヘルスチェック**
   - イベント購読の問題を検出するためにLastEventTimeを監視
   - 無効になった購読の再接続ロジックを実装

3. **スレッドセーフなイベント処理**
   ```csharp
   // スレッドセーフなイベント処理
   private void ProcessMarketUpdate(Symbol symbol, Quote quote)
   {
       _stateLock.WithUpgradeableReadLock(() => {
           // 現在の状態を読み取り
           var currentPrice = quote.Ask;
           
           if (NeedsStateUpdate(currentPrice)) {
               _stateLock.WithWriteLock(() => {
                   // 状態を更新
                   UpdateInternalState(currentPrice);
               });
           }
       });
   }
   ```

## 実装例：ベストプラクティスに従ったクラスの変換

### 変更前

```csharp
namespace ScalpingStrategyProject.Indicators
{
    public class SimpleIndicator
    {
        private Symbol _symbol;
        private object _lock = new object();
        private List<double> _values = new List<double>();
        
        public SimpleIndicator(Symbol symbol)
        {
            _symbol = symbol;
            _symbol.NewQuote += Symbol_NewQuote;
        }
        
        private void Symbol_NewQuote(Symbol symbol, Quote quote)
        {
            lock(_lock)
            {
                try
                {
                    _values.Add(quote.Ask);
                    if (_values.Count > 100)
                        _values.RemoveAt(0);
                        
                    CalculateIndicator();
                }
                catch (Exception ex)
                {
                    Core.Instance.Loggers.Log($"Error: {ex.Message}", LoggingLevel.Error);
                }
            }
        }
        
        private void CalculateIndicator()
        {
            // 計算コード...
        }
        
        public void Dispose()
        {
            _symbol.NewQuote -= Symbol_NewQuote;
        }
    }
}
```

### 変更後

```csharp
namespace DynamicExitStrategyImplementation.Indicators
{
    public class SimpleIndicator : IDisposable
    {
        private readonly Symbol _symbol;
        private readonly object _syncLock = new object();
        private readonly CircularBuffer<double> _values;
        private readonly string _eventGroup;
        private bool _disposed;
        
        public SimpleIndicator(Symbol symbol)
        {
            _symbol = symbol ?? throw new ArgumentNullException(nameof(symbol));
            _values = new CircularBuffer<double>(100);
            _eventGroup = $"{GetType().Name}_{Guid.NewGuid().ToString("N").Substring(0, 8)}";
            
            Initialize();
        }
        
        private void Initialize()
        {
            ErrorHandling.SafeExecute(() =>
            {
                // グループベースの購読を使用
                var eventManager = TradingPlatform.BusinessLayer.Core.Instance.EnhancedManagers().EventManager;
                eventManager.SubscribeSymbolNewQuote(_eventGroup, _symbol, OnSymbolNewQuote);
            }, "initializing indicator");
        }
        
        private void OnSymbolNewQuote(Symbol symbol, Quote quote)
        {
            ErrorHandling.SafeExecute(() =>
            {
                _syncLock.WithLock(() =>
                {
                    _values.Add(quote.Ask);
                    CalculateIndicator();
                });
            }, "processing quote update");
        }
        
        private void CalculateIndicator()
        {
            // 計算コード...
        }
        
        public void Dispose()
        {
            if (_disposed)
                return;
                
            _disposed = true;
            
            ErrorHandling.SafeExecute(() =>
            {
                // グループベースの購読解除を使用
                var eventManager = TradingPlatform.BusinessLayer.Core.Instance.EnhancedManagers().EventManager;
                eventManager.UnsubscribeGroup(_eventGroup);
            }, "disposing indicator");
        }
    }
}
```

このプランに従うことで、コードベースはQuantower APIの仕様により適合し、スレッド安全性が向上し、より保守しやすいイベント処理ロジックを得ることができます。