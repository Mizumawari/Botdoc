# TradingPlatform 統合ガイド

このガイドでは、統合インディケーターアーキテクチャを TradingPlatform.BusinessLayer と統合する方法について説明します。

## 概要

`TradingPlatformIndicatorIntegration` クラスは、統合インディケーターアーキテクチャを TradingPlatform.BusinessLayer と接続し、インディケーターがトレーディングプラットフォームからマーケットデータを受信できるようにします。

## クイックスタート例

```csharp
// 統合マネージャーの作成
var integration = new TradingPlatformIndicatorIntegration();

// インディケーターの作成
var rsi = new RSIIndicator(14);
var macd = new MACDIndicator(12, 26, 9);

// インディケーターの初期化
rsi.Initialize();
macd.Initialize();

// シンボルに接続
string symbol = "BTCUSDT";
integration.ConnectIndicatorToSymbol(rsi, symbol);
integration.ConnectIndicatorToSymbol(macd, symbol);

// 履歴データの読み込み（オプション）
integration.InitializeIndicatorWithHistory(rsi, symbol);
integration.InitializeIndicatorWithHistory(macd, symbol);

// シグナルの購読
rsi.SignalGenerated += (sender, e) => {
    Console.WriteLine($"RSI Signal: {e.SignalType}, Strength: {e.Strength}");
};

macd.SignalGenerated += (sender, e) => {
    Console.WriteLine($"MACD Signal: {e.SignalType}, Strength: {e.Strength}");
};

// リアルタイム更新の開始
integration.StartRealtimeUpdates(symbol);

// 終了時
integration.StopRealtimeUpdates(symbol);
rsi.Dispose();
macd.Dispose();
```

## 主要コンポーネント

### TradingPlatformIndicatorIntegration

統合のメインクラスで以下を処理します：
- インディケーターをシンボルマーケットデータに接続
- 初期化のための履歴データの読み込み
- リアルタイムデータフローの管理

### SymbolMarketDataAdapter

以下の処理を行うアダプター：
- TradingPlatformのマーケットデータイベントを購読
- プラットフォームイベントを統合MarketDataEventArgsに変換
- データを接続されたインディケーターに転送

## インテグレーションパターン

### 戦略実装例

```csharp
public class SimpleStrategy : IDisposable
{
    private TradingPlatformIndicatorIntegration _integration;
    private RSIIndicator _rsi;
    private MACDIndicator _macd;
    private string _symbol;
    private bool _isLong = false;
    
    public SimpleStrategy(string symbol)
    {
        _symbol = symbol;
        _integration = new TradingPlatformIndicatorIntegration();
        _rsi = new RSIIndicator(14);
        _macd = new MACDIndicator(12, 26, 9);
        
        // 初期化
        _rsi.Initialize();
        _macd.Initialize();
        
        // 接続
        _integration.ConnectIndicatorToSymbol(_rsi, symbol);
        _integration.ConnectIndicatorToSymbol(_macd, symbol);
        
        // 購読
        _rsi.ValueUpdated += OnIndicatorsUpdated;
        _macd.ValueUpdated += OnIndicatorsUpdated;
    }
    
    public void Start()
    {
        // 履歴の読み込み
        _integration.InitializeIndicatorWithHistory(_rsi, _symbol);
        _integration.InitializeIndicatorWithHistory(_macd, _symbol);
        
        // 更新の開始
        _integration.StartRealtimeUpdates(_symbol);
    }
    
    private void OnIndicatorsUpdated(object sender, IndicatorUpdateEventArgs e)
    {
        // トレード条件のチェック
        if (_rsi.CurrentValue < 30 && _macd.IsPositiveCrossover() && !_isLong)
        {
            // 買いシグナル
            Console.WriteLine("BUY signal generated");
            ExecuteBuy();
            _isLong = true;
        }
        else if (_rsi.CurrentValue > 70 && _macd.IsNegativeCrossover() && _isLong)
        {
            // 売りシグナル
            Console.WriteLine("SELL signal generated");
            ExecuteSell();
            _isLong = false;
        }
    }
    
    private void ExecuteBuy()
    {
        // 買い執行ロジックの実装
    }
    
    private void ExecuteSell()
    {
        // 売り執行ロジックの実装
    }
    
    public void Dispose()
    {
        _rsi.ValueUpdated -= OnIndicatorsUpdated;
        _macd.ValueUpdated -= OnIndicatorsUpdated;
        
        _integration.StopRealtimeUpdates(_symbol);
        
        _rsi.Dispose();
        _macd.Dispose();
    }
}
```

### 複数時間枠分析

```csharp
// 時間枠インディケーターの作成
var timeframeIndicator = new TimeframeIndicator(
    TimeSpan.FromMinutes(1), 
    new[] { TimeSpan.FromMinutes(5), TimeSpan.FromMinutes(15) }
);

// 初期化と接続
timeframeIndicator.Initialize();
integration.ConnectIndicatorToSymbol(timeframeIndicator, symbol);

// トレンド変更の購読
timeframeIndicator.TrendChanged += (sender, e) => {
    if (e.Timeframe == TimeSpan.FromMinutes(15))
    {
        Console.WriteLine($"15分足トレンドが{e.TrendDirection}に変更");
        
        // トレードシグナルのフィルターとして使用
        _tradingEnabled = e.TrendDirection == TrendDirection.Up;
    }
};
```

### 手動データフィード

テストやカスタムデータソース用：

```csharp
// インディケーターの作成
var rsi = new RSIIndicator(14);
rsi.Initialize();

// シグナルの購読
rsi.SignalGenerated += (sender, e) => {
    Console.WriteLine($"RSI Signal: {e.SignalType}");
};

// 手動でデータをフィード
for (int i = 0; i < historicalPrices.Length; i++)
{
    // マーケットデータイベントの作成
    var marketData = new MarketDataEventArgs
    {
        Price = historicalPrices[i],
        Volume = historicalVolumes[i],
        Timestamp = DateTime.Now.AddMinutes(-historicalPrices.Length + i)
    };
    
    // データ処理
    rsi.ProcessMarketData(marketData);
}
```

## エラー処理

```csharp
try
{
    integration.ConnectIndicatorToSymbol(indicator, symbol);
    integration.StartRealtimeUpdates(symbol);
}
catch (SymbolNotFoundException ex)
{
    Console.WriteLine($"シンボルが見つかりません: {ex.Message}");
}
catch (ConnectionException ex)
{
    Console.WriteLine($"接続エラー: {ex.Message}");
    // リトライロジックの実装
    Task.Delay(5000).ContinueWith(_ => {
        integration.ConnectIndicatorToSymbol(indicator, symbol);
        integration.StartRealtimeUpdates(symbol);
    });
}
```

## ベストプラクティス

1. **マーケットデータに接続する前にインディケーターを初期化**する
2. **不要になったリソースを破棄**する
3. **可能であれば履歴データを使用**してインディケーターを初期化する
4. **接続エラーを適切なリトライロジックで処理**する
5. **メモリリークを防ぐためにイベントの購読を解除**する
6. **パフォーマンス向上のために履歴データのバッチ処理を使用**する

## 高度な機能

### カスタムマーケットデータアダプター

```csharp
public class CustomMarketDataAdapter : IMarketDataAdapter
{
    private string _symbol;
    private List<IIndicator> _connectedIndicators = new List<IIndicator>();
    
    public CustomMarketDataAdapter(string symbol)
    {
        _symbol = symbol;
    }
    
    public void Connect(IIndicator indicator)
    {
        _connectedIndicators.Add(indicator);
    }
    
    public void Disconnect(IIndicator indicator)
    {
        _connectedIndicators.Remove(indicator);
    }
    
    public void ProcessExternalData(double price, double volume, DateTime timestamp)
    {
        // マーケットデータイベントの作成
        var marketData = new MarketDataEventArgs
        {
            Symbol = _symbol,
            Price = price,
            Volume = volume,
            Timestamp = timestamp
        };
        
        // 接続されたすべてのインディケーターに転送
        foreach (var indicator in _connectedIndicators)
        {
            indicator.ProcessMarketData(marketData);
        }
    }
}
```

## 統合の詳細

### イベント購読

統合は、プロジェクトの既存のEnhancedEventManagerを活用して以下を行います：
1. 各アダプターに名前付き購読グループを作成
2. Symbol.NewQuoteおよびSymbol.NewLastイベントを購読
3. ヘルスモニタリングのためのイベント発生を追跡
4. インディケーターが切断されたときに適切に購読解除

### マーケットデータ処理

トレーディングプラットフォームからのマーケットデータは以下のように処理されます：
1. クォートとラストイベントはSymbolMarketDataAdapterによってキャプチャ
2. 更新間隔制限に基づいてデータをフィルタリング
3. 価格値の抽出（クォートに対する中間価格、ラストに対する取引価格）
4. インディケーターを適切な値とタイムスタンプで更新
5. 更新頻度を管理するために更新タイムスタンプを追跡

### エラー処理

統合は既存のエラー処理パターンを使用します：
1. 例外処理のためのSafeExecute
2. LockHelperを使用したスレッドセーフな操作
3. 破棄時の適切なリソースクリーンアップ
4. エラー状態のロギング
5. エラー発生時の適切な機能低下

### リソース管理

リソースは以下を通じて管理されます：
1. リソースマネージャーへの明示的な登録
2. 適切なIDisposable実装
3. スレッドセーフな破棄シーケンス
4. イベントハンドラーのクリーンアップ
5. バックグラウンド操作のキャンセル

## 統合の利点

1. **関心事の明確な分離**：
   - インディケーターはデータソースを認識しない
   - プラットフォーム固有のコードはアダプターに分離
   - トレーディングロジックは純粋でテスト可能な状態を維持

2. **パフォーマンス最適化**：
   - 更新頻度の制限
   - インディケーター間のリソース共有
   - マーケットデータのバッチ処理

3. **柔軟な構成**：
   - データタイプへの選択的な購読
   - 設定可能な更新間隔
   - 履歴データ初期化オプション

4. **シンプルなAPI**：
   - 一貫したインディケーター登録
   - 標準的な接続方法
   - イベントベースの通知システム

## 使用例

```csharp
// 統合の作成
var integration = new TradingPlatformIndicatorIntegration();
integration.Initialize();

// インディケーターの作成
var momentumIndicator = new MomentumIndicator("BTC_Momentum");
momentumIndicator.Initialize();

// インディケーターの登録
integration.RegisterIndicator(momentumIndicator);

// シンボルへの接続
integration.ConnectIndicatorToSymbol(
    momentumIndicator,
    symbol,
    MarketDataTypes.Quote | MarketDataTypes.Last,
    250 // 250ms毎に更新
);

// 履歴で初期化
integration.InitializeIndicatorWithHistory(
    momentumIndicator,
    symbol,
    Period.MIN1,
    60 // 60バー
);

// インディケーターイベントの購読
momentumIndicator.SignalGenerated += (s, e) => {
    // シグナルの処理
};
```