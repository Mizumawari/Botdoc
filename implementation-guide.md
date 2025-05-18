# 実装ガイド

このガイドは、DynamicExitStrategyImplementation（DESI）の主要コンポーネントの実装方法と使用パターンを説明します。

## 目次

1. [エラー処理パターン](#1-エラー処理パターン)
2. [リソース管理](#2-リソース管理)
3. [イベント管理](#3-イベント管理)
4. [スレッド安全な実装](#4-スレッド安全な実装)
5. [データバッファの使用](#5-データバッファの使用)
6. [エントリー/エグジット戦略の実装](#6-エントリーエグジット戦略の実装)
7. [名前空間構成](#7-名前空間構成)

## 1. エラー処理パターン

### SafeExecuteパターン

すべての操作で一貫したエラー処理を実装するためにErrorHandlingクラスを使用します：

```csharp
using DynamicExitStrategyImplementation.Core;

// 同期操作の場合
ErrorHandling.SafeExecute(() =>
{
    // 実行したいコード
}, 
"operation name",  // オペレーション名（ログ用）
LoggingLevel.Error,  // エラー時のログレベル
(ex) => {
    // エラー発生時の追加処理
});

// 戻り値のある操作の場合
int result = ErrorHandling.SafeExecute(() =>
{
    // 値を返すコード
    return 42;
}, 
defaultValue: 0,  // エラー時のデフォルト値
"operation with return value");

// 非同期操作の場合
await ErrorHandling.SafeExecuteAsync(async () =>
{
    // 非同期コード
    await Task.Delay(100);
}, 
"async operation");
```

### 再試行パターン

一時的なエラーを処理するための再試行メカニズム：

```csharp
// 再試行を伴う操作
await ErrorHandling.RetryAsync(async (attempt) =>
{
    // リトライされる可能性のある操作
    await SomeOperationThatMightFailAsync();
}, 
maxRetries: 3,  // 最大再試行回数
delayMilliseconds: 500,  // 再試行間の遅延（ミリ秒）
exponentialBackoff: true);  // 指数バックオフを使用
```

## 2. リソース管理

### リソースの登録と破棄

コンポーネントのリソースライフサイクルを管理するためのベストプラクティス：

```csharp
using DynamicExitStrategyImplementation.Core;

// コンポーネント名
string componentName = "MyComponent";

// リソースの登録
Timer timer = new Timer(OnTimerTick, null, 1000, 1000);
ResourceManagement.RegisterResource("myTimer", timer, componentName);

// 個別リソースの破棄
ResourceManagement.DisposeResource("myTimer", componentName);

// コンポーネントのすべてのリソースを破棄
ResourceManagement.DisposeComponentResources(componentName);
```

### DisposableBaseの継承

`IDisposable`を適切に実装するための基底クラスの使用：

```csharp
using DynamicExitStrategyImplementation.Core;

public class MyComponent : DisposableBase
{
    private Timer _timer;
    
    public MyComponent()
    {
        // タイマーの作成と登録
        _timer = new Timer(OnTimerTick, null, 1000, 1000);
        RegisterResource("timer", _timer);
    }
    
    // DisposableBaseはIDisposeを実装し、このメソッドを呼び出します
    protected override void OnDispose()
    {
        // 追加のクリーンアップロジック
        // 登録されたリソースは自動的に破棄されます
    }
}
```

## 3. イベント管理

### EnhancedEventManagerの使用

イベント管理システムを使用したイベント購読と発行：

```csharp
using DynamicExitStrategyImplementation.Core;
using DynamicExitStrategyImplementation.Core.Events;

// イベントマネージャーの取得
var eventManager = Core.Instance.EnhancedManagers().EventManager;

// イベントグループの作成/取得
var group = eventManager.GetOrCreateGroup("MyComponentEvents");

// シンボルイベントの購読
eventManager.SubscribeSymbolNewQuote(
    "MyComponentEvents",  // グループ名
    symbol,  // 購読するシンボル
    OnSymbolQuoteUpdated,  // イベントハンドラー
    "Symbol Quote Subscription"  // 説明
);

// カスタムイベントの購読
eventManager.Subscribe<MyEventArgs>(
    "MyComponentEvents",  // グループ名
    eventSource,  // イベントソース
    "MyEvent",  // イベント名
    OnMyEvent,  // イベントハンドラー
    "My Custom Event"  // 説明
);

// グループの購読解除
eventManager.UnsubscribeGroup("MyComponentEvents");
```

### イベント発行パターン

イベントを発行する標準的な方法：

```csharp
// イベント定義
public event EventHandler<MyEventArgs> MyEvent;

// イベント発行
protected virtual void OnMyEvent(MyEventArgs e)
{
    // スレッドセーフなイベント発行
    EventHandlerExtensions.SafeInvoke(MyEvent, this, e);
    
    // または ErrorHandling クラスを使用
    ErrorHandling.SafeExecute(() => MyEvent?.Invoke(this, e), 
        "raising MyEvent");
}
```

## 4. スレッド安全な実装

### ロックヘルパーの使用

パフォーマンスとデッドロック防止のための効率的なロック管理：

```csharp
using DynamicExitStrategyImplementation.Core.Threading;

// リーダーライターロックの使用
private readonly ReaderWriterLockSlim _lock = new ReaderWriterLockSlim();

// 読み取りロック
string value = _lock.WithReadLock(() => 
{
    return _someValue;
});

// 書き込みロック
_lock.WithWriteLock(() => 
{
    _someValue = "new value";
});

// アップグレード可能な読み取りロック
_lock.WithUpgradeableReadLock(() => 
{
    if (_someCondition)
    {
        _lock.WithWriteLock(() => 
        {
            // 書き込み操作
        });
    }
});

// 標準ロック
private readonly object _syncLock = new object();

_syncLock.WithLock(() => 
{
    // 排他的アクセスが必要な操作
});
```

### スレッドセーフなフラグ

`Interlocked` を使用した安全なフラグ操作：

```csharp
// スレッドセーフなブール値（int として実装）
private int _disposed;
public bool IsDisposed => Interlocked.CompareExchange(ref _disposed, 0, 0) == 1;

// フラグの設定
Interlocked.Exchange(ref _disposed, 1);

// 安全な一回限りの操作
if (Interlocked.CompareExchange(ref _initialized, 1, 0) == 0)
{
    // 初期化を一度だけ実行
}
```

## 5. データバッファの使用

### CircularBufferの使用

時系列データの効率的な保存と処理：

```csharp
using DynamicExitStrategyImplementation.Data;

// 新しいバッファの作成
var priceBuffer = new CircularBuffer<double>(100);  // 容量100

// データの追加
priceBuffer.Add(42.5);

// 最新の値を取得
double latestValue = priceBuffer.Last();

// すべての値を列挙
foreach (var price in priceBuffer)
{
    // 各価格の処理
}

// 統計計算
double avg = priceBuffer.Average();
double stdDev = priceBuffer.StandardDeviation();
```

### EnhancedBufferManagerの使用

複数バッファを管理するための高度なアプローチ：

```csharp
using DynamicExitStrategyImplementation.Core.Buffers;

// バッファマネージャーの取得または作成
var bufferManager = Core.Instance.EnhancedManagers().BufferManager;

// バッファの作成/取得
var priceBuffer = bufferManager.GetOrCreateBuffer<double>(
    "PriceHistory",  // バッファ名
    100,  // 容量
    "MyComponent",  // オーナー
    "Price history buffer"  // 説明
);

// バッファへのデータ追加
bufferManager.AddToBuffer("PriceHistory", 42.5);

// 統計値の計算
double avg = bufferManager.CalculateAverage("PriceHistory");
```

## 6. エントリー/エグジット戦略の実装

### エントリーシグナルの実装

カスタムエントリーシグナルの作成：

```csharp
using DynamicExitStrategyImplementation.EntryLogic;
using DynamicExitStrategyImplementation.Interfaces;

public class MyEntrySignal : IEntrySignal
{
    // IEntrySignal インターフェイスの実装
    public event EventHandler<EntrySignalEventArgs> EntrySignalDetected;
    
    public bool ShouldEnter(Symbol symbol, out Side side)
    {
        // エントリー条件のロジック
        bool shouldEnter = /* 条件判定 */;
        side = /* Side.Buy または Side.Sell */;
        
        if (shouldEnter)
        {
            // シグナルイベントの発火
            OnEntrySignalDetected(new EntrySignalEventArgs(side, symbol.Last));
        }
        
        return shouldEnter;
    }
    
    protected virtual void OnEntrySignalDetected(EntrySignalEventArgs e)
    {
        EntrySignalDetected?.Invoke(this, e);
    }
}
```

### エグジット戦略の実装

エグジット戦略の拡張：

```csharp
using DynamicExitStrategyImplementation.ExitLogic;
using DynamicExitStrategyImplementation.Core.BaseComponents;

public class MyExitStrategy : ExitStrategyBase
{
    public MyExitStrategy(string name, Symbol symbol) 
        : base(name, symbol)
    {
        // 初期化
        FirstPartialProfitRatio = 1.0;
        StopLossRatio = 1.0;
        EnableTrailingStop = true;
    }
    
    // エグジット条件の評価をオーバーライド
    protected override bool EvaluateExitSignal(ExitContext context, double currentPrice, double holdTimeSec)
    {
        // 基本的なエグジット条件（利確、損切り）
        bool baseCondition = base.EvaluateExitSignal(context, currentPrice, holdTimeSec);
        
        // カスタムエグジット条件
        bool customCondition = /* 独自の条件ロジック */;
        
        // どちらかの条件が満たされたらエグジット
        if (baseCondition || customCondition)
        {
            _lastExitReason = customCondition ? "Custom condition triggered" : base.GetExitReason();
            return true;
        }
        
        return false;
    }
}
```

## 7. 名前空間構成

DESIプロジェクトの名前空間は以下の構造に従っています：

```
DynamicExitStrategyImplementation
|-- Core                   // 基盤コンポーネント
|   |-- BaseComponents     // 基底クラス
|   |-- Buffers            // バッファ管理
|   |-- Events             // イベント管理
|   |-- Managers           // コアマネージャー
|   |-- Resources          // リソース管理
|   |-- Threading          // スレッド同期
|
|-- Data                   // データ構造
|   |-- CircularBuffer     // 循環バッファ実装
|
|-- EntryLogic             // エントリー戦略
|
|-- ExitLogic              // エグジット戦略
|
|-- Extensions             // 拡張メソッド
|
|-- Fibonacci              // フィボナッチツール
|
|-- Indicators             // テクニカル指標
|
|-- Interfaces             // コンポーネントインターフェイス
|
|-- Managers               // 主要マネージャー
|
|-- Metrics                // パフォーマンス測定
|
|-- OrderBook              // 板情報関連
|
|-- Positioning            // ポジショニングロジック
|
|-- Utilities              // ユーティリティクラス
```

### 名前空間移行

古い `ScalpingStrategyProject` 名前空間から新しい `DynamicExitStrategyImplementation` 名前空間への移行時は、以下のパターンに従ってください：

```csharp
// 古い名前空間
using ScalpingStrategyProject;
using ScalpingStrategyProject.Core;

// 新しい名前空間
using DynamicExitStrategyImplementation;
using DynamicExitStrategyImplementation.Core;

// 名前空間の宣言も更新
namespace DynamicExitStrategyImplementation
{
    // クラス実装
}
```

## 結論

このガイドは、DynamicExitStrategyImplementationプロジェクトの主要コンポーネントの実装方法と使用パターンの概要を提供しています。コードの一貫性と品質を確保するために、これらの標準パターンに従うことをお勧めします。詳細な情報や特定のコンポーネントについては、対応するドキュメントファイルを参照してください。