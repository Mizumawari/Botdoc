# クイックスタートガイド

このドキュメントでは、DynamicExitStrategyImplementationプロジェクトを使って開発を始めるための手順を説明します。

## 目次

1. [環境セットアップ](#環境セットアップ)
2. [プロジェクト構造](#プロジェクト構造)
3. [基本的な使用方法](#基本的な使用方法)
4. [構成とカスタマイズ](#構成とカスタマイズ)
5. [デバッグとログ記録](#デバッグとログ記録)
6. [サンプルコード](#サンプルコード)

## 環境セットアップ

### 前提条件

- Visual Studio 2019以上
- .NET Framework 4.7.2以上
- TradingPlatform.BusinessLayer参照

### プロジェクトのインストール

1. リポジトリをクローン：
   ```bash
   git clone https://your-repository-url/DynamicExitStrategyImplementation.git
   ```

2. Visual Studioでソリューションを開く：
   ```
   DynamicExitStrategyImplementation.sln
   ```

3. 依存関係をインストール：
   ```bash
   nuget restore DynamicExitStrategyImplementation.sln
   ```

4. ビルドを実行：
   ```bash
   msbuild DynamicExitStrategyImplementation.sln
   ```

## プロジェクト構造

プロジェクトは以下の主要なディレクトリで構成されています：

- `Core/` - 基盤となるクラスとユーティリティ
  - `BaseComponents/` - 基本コンポーネントクラス
  - `Events/` - イベント処理フレームワーク
  - `Managers/` - リソースと機能の管理クラス
- `Data/` - データ構造と操作
  - `CircularBuffer.cs` - 時系列データ用の循環バッファ
- `EntryLogic/` - エントリー戦略の実装
  - `MomentumEntrySignal.cs` - モメンタムベースのエントリーシグナル
- `ExitLogic/` - 出口戦略の実装
  - `DynamicExitStrategy.cs` - 動的出口戦略
  - `SimpleMomentumExitStrategy.cs` - シンプルなモメンタム出口戦略
- `Indicators/` - テクニカル指標
  - `DeltaMomentum.cs` - デルタモメンタム指標
  - `VolatilityRegime.cs` - ボラティリティレジーム分析
- `Managers/` - トレーディング管理
  - `OrderManager.cs` - 注文管理
  - `PositionManager.cs` - ポジション管理
- `Fibonacci/` - フィボナッチツール
- `Extensions/` - 拡張メソッド

## 基本的な使用方法

### コンポーネントの初期化

すべてのコンポーネントは基本的に以下のパターンで初期化されます：

```csharp
// コンポーネントのインスタンス化
var entrySignal = new MomentumEntrySignal();
var exitStrategy = new DynamicExitStrategy();
var positionManager = new PositionManager();
var orderManager = new OrderManager();

// シンボルと設定で初期化
Symbol symbol = Symbol.Create("BTCUSD");
entrySignal.Initialize(symbol);
exitStrategy.Initialize(symbol);
positionManager.Initialize(symbol);
orderManager.Initialize(symbol);

// コンポーネント間の接続
entrySignal.EntrySignalGenerated += exitStrategy.OnEntrySignalGenerated;
exitStrategy.ExitSignalGenerated += positionManager.OnExitSignalGenerated;
positionManager.PositionChanged += orderManager.OnPositionChanged;
```

### データのフィード

マーケットデータは各コンポーネントに以下のように提供します：

```csharp
// マーケットデータの受信と処理
public void OnQuoteUpdate(Quote quote)
{
    // 各コンポーネントにデータを提供
    entrySignal.OnQuoteUpdate(quote);
    exitStrategy.OnQuoteUpdate(quote);
    positionManager.OnQuoteUpdate(quote);
    orderManager.OnQuoteUpdate(quote);
}

public void OnTradeUpdate(Trade trade)
{
    // 各コンポーネントにデータを提供
    entrySignal.OnTradeUpdate(trade);
    exitStrategy.OnTradeUpdate(trade);
    positionManager.OnTradeUpdate(trade);
    orderManager.OnTradeUpdate(trade);
}
```

### イベント処理

イベントの購読と処理は以下のように行います：

```csharp
// イベントを購読
entrySignal.EntrySignalGenerated += OnEntrySignalGenerated;
exitStrategy.ExitSignalGenerated += OnExitSignalGenerated;
positionManager.PositionChanged += OnPositionChanged;

// イベントハンドラの実装
private void OnEntrySignalGenerated(object sender, EntrySignalEventArgs e)
{
    Console.WriteLine($"エントリーシグナル: {e.SignalType}, 価格: {e.Price}");
    // シグナルに基づく処理
}

private void OnExitSignalGenerated(object sender, ExitSignalEventArgs e)
{
    Console.WriteLine($"出口シグナル: {e.SignalType}, 価格: {e.Price}");
    // シグナルに基づく処理
}

private void OnPositionChanged(object sender, PositionChangedEventArgs e)
{
    Console.WriteLine($"ポジション変更: {e.NewPosition}, PNL: {e.Pnl}");
    // ポジション変更に基づく処理
}
```

### リソース管理

リソースの適切な解放を確保するには：

```csharp
// リソースマネージャを使用
var resourceManager = new ResourceManager();
resourceManager.RegisterResource(entrySignal);
resourceManager.RegisterResource(exitStrategy);
resourceManager.RegisterResource(positionManager);
resourceManager.RegisterResource(orderManager);

// 最終的に解放
public void Cleanup()
{
    resourceManager.DisposeAll();
}

// または各コンポーネントを個別に解放
public void Cleanup()
{
    entrySignal.Dispose();
    exitStrategy.Dispose();
    positionManager.Dispose();
    orderManager.Dispose();
}
```

## 構成とカスタマイズ

### パラメータの設定

各コンポーネントはパラメータを設定することでカスタマイズできます：

```csharp
// モメンタムエントリーシグナルの設定
entrySignal.MomentumPeriod = 14;
entrySignal.SignalThreshold = 0.5;
entrySignal.EnableAdaptiveThreshold = true;

// 動的出口戦略の設定
exitStrategy.EnableTrailingStop = true;
exitStrategy.TrailingStopDistance = 0.1;
exitStrategy.EnableTimeBased = true;
exitStrategy.MaxHoldingTimeTicks = 100;
```

### カスタム戦略の作成

カスタム戦略を実装するには、適切な基本クラスを継承します：

```csharp
// カスタムエントリー戦略
public class CustomEntryStrategy : SignalComponentBase, IEntrySignal
{
    // エントリーシグナルイベント
    public event EventHandler<EntrySignalEventArgs> EntrySignalGenerated;
    
    // 実装ロジック
    protected override void OnQuoteUpdate(Quote quote)
    {
        base.OnQuoteUpdate(quote);
        
        // カスタムシグナルロジック
        if (YourCustomLogic())
        {
            EntrySignalGenerated?.Invoke(this, new EntrySignalEventArgs
            {
                Symbol = Symbol,
                Price = quote.Bid,
                SignalType = SignalType.Long,
                Timestamp = DateTime.UtcNow
            });
        }
    }
    
    private bool YourCustomLogic()
    {
        // カスタムロジックを実装
        return true; // シグナル生成条件
    }
}

// カスタム出口戦略
public class CustomExitStrategy : ExitStrategyBase
{
    protected override void EvaluateExitConditions(Quote quote)
    {
        // カスタム出口条件を評価
        if (YourExitCondition())
        {
            GenerateExitSignal(ExitType.TakeProfit, quote.Bid);
        }
    }
    
    private bool YourExitCondition()
    {
        // カスタム条件を実装
        return false; // 出口条件
    }
}
```

## デバッグとログ記録

### ログ記録

ログ記録は標準化されたメソッドを使用して行います：

```csharp
// 様々なレベルでのログ記録
LogMessage("コンポーネントを初期化しました", LoggingLevel.System);
LogWarning("これは注意が必要な状態です");
LogError("エラーが発生しました: " + exception.Message);
LogDebug("デバッグ情報: 現在の値 = " + currentValue);
```

### デバッグサポート

開発およびデバッグ時には、詳細ログを有効にすることをお勧めします：

```csharp
// デバッグ設定
#if DEBUG
    // デバッグ環境での設定
    enableDetailedLogging = true;
    enablePerformanceTracking = true;
#else
    // リリース環境での設定
    enableDetailedLogging = false;
    enablePerformanceTracking = false;
#endif

// 診断情報の表示
public void ShowDiagnostics()
{
    if (enableDetailedLogging)
    {
        Console.WriteLine("診断情報:");
        Console.WriteLine($"バッファサイズ: {_buffer.Count}/{_buffer.Capacity}");
        Console.WriteLine($"イベント購読数: {_eventManager.SubscriberCount}");
        Console.WriteLine($"メモリ使用量: {Process.GetCurrentProcess().WorkingSet64 / 1024 / 1024} MB");
    }
}
```

## サンプルコード

### シンプルなトレーディングシステムの実装

```csharp
public class SimpleTradingSystem : IDisposable
{
    private MomentumEntrySignal _entrySignal;
    private DynamicExitStrategy _exitStrategy;
    private PositionManager _positionManager;
    private OrderManager _orderManager;
    private ResourceManager _resourceManager;
    
    public SimpleTradingSystem(Symbol symbol)
    {
        // コンポーネントの初期化
        _entrySignal = new MomentumEntrySignal();
        _exitStrategy = new DynamicExitStrategy();
        _positionManager = new PositionManager();
        _orderManager = new OrderManager();
        _resourceManager = new ResourceManager();
        
        // リソース登録
        _resourceManager.RegisterResource(_entrySignal);
        _resourceManager.RegisterResource(_exitStrategy);
        _resourceManager.RegisterResource(_positionManager);
        _resourceManager.RegisterResource(_orderManager);
        
        // 初期化
        _entrySignal.Initialize(symbol);
        _exitStrategy.Initialize(symbol);
        _positionManager.Initialize(symbol);
        _orderManager.Initialize(symbol);
        
        // イベント接続
        _entrySignal.EntrySignalGenerated += _exitStrategy.OnEntrySignalGenerated;
        _exitStrategy.ExitSignalGenerated += _positionManager.OnExitSignalGenerated;
        _positionManager.PositionChanged += _orderManager.OnPositionChanged;
        
        // システムイベント処理
        _entrySignal.EntrySignalGenerated += OnEntrySignalGenerated;
        _exitStrategy.ExitSignalGenerated += OnExitSignalGenerated;
        _positionManager.PositionChanged += OnPositionChanged;
        _orderManager.OrderExecuted += OnOrderExecuted;
    }
    
    // マーケットデータ処理
    public void ProcessQuote(Quote quote)
    {
        _entrySignal.OnQuoteUpdate(quote);
        _exitStrategy.OnQuoteUpdate(quote);
        _positionManager.OnQuoteUpdate(quote);
        _orderManager.OnQuoteUpdate(quote);
    }
    
    // イベントハンドラ
    private void OnEntrySignalGenerated(object sender, EntrySignalEventArgs e)
    {
        Console.WriteLine($"Entry signal: {e.SignalType} at {e.Price}");
    }
    
    private void OnExitSignalGenerated(object sender, ExitSignalEventArgs e)
    {
        Console.WriteLine($"Exit signal: {e.ExitType} at {e.Price}");
    }
    
    private void OnPositionChanged(object sender, PositionChangedEventArgs e)
    {
        Console.WriteLine($"Position: {e.NewPosition}, PNL: {e.Pnl}");
    }
    
    private void OnOrderExecuted(object sender, OrderExecutedEventArgs e)
    {
        Console.WriteLine($"Order executed: {e.OrderId}, Price: {e.ExecutionPrice}");
    }
    
    // リソース解放
    public void Dispose()
    {
        _resourceManager.DisposeAll();
    }
}

// 使用例
public void RunExample()
{
    Symbol symbol = Symbol.Create("BTCUSD");
    using (var tradingSystem = new SimpleTradingSystem(symbol))
    {
        // マーケットデータのシミュレーション
        for (int i = 0; i < 100; i++)
        {
            var quote = new Quote
            {
                Symbol = symbol,
                Bid = 10000 + Math.Sin(i * 0.1) * 100,
                Ask = 10000 + Math.Sin(i * 0.1) * 100 + 1
            };
            tradingSystem.ProcessQuote(quote);
            Thread.Sleep(100); // シミュレーション用ディレイ
        }
    }
}
```

### カスタム指標の統合

```csharp
public class CustomIndicatorSystem
{
    private DeltaMomentum _deltaMomentum;
    private VolatilityRegime _volatilityRegime;
    private CircularBuffer<double> _customBuffer;
    
    public CustomIndicatorSystem(Symbol symbol, int period)
    {
        // 指標の初期化
        _deltaMomentum = new DeltaMomentum(period);
        _volatilityRegime = new VolatilityRegime();
        _customBuffer = new CircularBuffer<double>(1000);
        
        // シンボル設定
        _deltaMomentum.Initialize(symbol);
        _volatilityRegime.Initialize(symbol);
        
        // イベント接続
        _deltaMomentum.ValueUpdated += OnDeltaMomentumUpdated;
        _volatilityRegime.RegimeChanged += OnVolatilityRegimeChanged;
    }
    
    // 指標更新ハンドラ
    private void OnDeltaMomentumUpdated(object sender, ValueEventArgs<double> e)
    {
        Console.WriteLine($"Delta Momentum: {e.Value}");
        _customBuffer.Add(e.Value);
        
        // バッファから統計を計算
        if (_customBuffer.Count > 10)
        {
            double average = _customBuffer.TakeLast(10).Average();
            double stdev = CalculateStandardDeviation(_customBuffer.TakeLast(10));
            Console.WriteLine($"Avg: {average}, StDev: {stdev}");
        }
    }
    
    private void OnVolatilityRegimeChanged(object sender, RegimeEventArgs e)
    {
        Console.WriteLine($"Volatility Regime: {e.RegimeType}");
        
        // レジームに基づいたパラメータ調整
        switch (e.RegimeType)
        {
            case RegimeType.HighVolatility:
                // 高ボラティリティ環境の調整
                break;
            case RegimeType.LowVolatility:
                // 低ボラティリティ環境の調整
                break;
        }
    }
    
    // ヘルパーメソッド
    private double CalculateStandardDeviation(IEnumerable<double> values)
    {
        double avg = values.Average();
        return Math.Sqrt(values.Average(v => Math.Pow(v - avg, 2)));
    }
    
    // データ更新
    public void Update(double price)
    {
        _deltaMomentum.Update(price);
        _volatilityRegime.Update(price);
    }
}
```

これらのサンプルを参考に、DynamicExitStrategyImplementationを活用した開発を開始してください。より詳細な情報は、以下のドキュメントを参照してください：

- [実装ガイド](../implementation-guide.md)
- [アーキテクチャ概要](../architecture-overview.md)
- [トラブルシューティングガイド](./troubleshooting-guide.md)
- [テストガイド](./testing-guide.md)