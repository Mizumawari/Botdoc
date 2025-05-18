# Quantower API サンプルコード集

このドキュメントはQuantower APIを使用する際のサンプルコードをまとめたリファレンスです。各セクションには一般的なユースケースのコード例が含まれています。

## 1. 基本構造と初期化

```csharp
// APIのメインエントリーポイント
Symbol symbol = Core.Instance.Symbols.FirstOrDefault();
symbol = Core.Instance.Symbols.FirstOrDefault(s => s.Name == "EUR/USD");
Account account = Core.Instance.Accounts.FirstOrDefault(a => a.Name == "<account-name>");

// インジケーターの基本構造
public class MyIndicator : Indicator
{
    public MyIndicator() : base()
    {
        // 基本プロパティの設定
        Name = "MyIndicator";
        Description = "My indicator's annotation";
        
        // ラインの追加
        AddLineSeries("line1", Color.CadetBlue, 1, LineStyle.Solid);
        
        // チャートメインウィンドウか別ウィンドウか
        SeparateWindow = false; // trueなら別ウィンドウ
    }
    
    protected override void OnInit() { /* 初期化処理 */ }
    protected override void OnUpdate(UpdateArgs args) { /* 更新処理 */ }
    protected override void OnClear() { /* クリーンアップ処理 */ }
}

// 戦略クラスの基本構造
public class MyStrategy : Strategy
{
    [InputParameter("Symbol")] 
    public Symbol CurrentSymbol { get; set; }
    
    [InputParameter("Account")] 
    public Account CurrentAccount { get; set; }
    
    protected override void OnRun() { /* 戦略開始時の処理 */ }
    protected override void OnStop() { /* 戦略停止時の処理 */ }
    protected override void OnRemove() { /* 戦略削除時の処理 */ }
    
    // ログ出力
    protected void LogMessage(string message) { this.Log(message); }
    protected void LogError(string message) { this.Log(message, StrategyLoggingLevel.Error); }
}
```

## 2. シンボルと市場データ

```csharp
// シンボル検索
var symbolsSearchResult = Core.Instance.SearchSymbols(new SearchSymbolsRequestParameters()
{
    FilterName = symbolName,
    SymbolTypes = new[] { SymbolType.Futures },
    ConnectionId = connection.Id,
    ExchangeIds = connection.BusinessObjects.Exchanges.Select(x => x.Id).ToArray()
});

// シンボル詳細の取得
Symbol symbol = Core.Instance.GetSymbol(new GetSymbolRequestParameters()
{
    SymbolId = result.Id
});

// マーケットデータ購読
symbol.NewQuote += Symbol_NewQuote;    // Bid/Askの更新
symbol.NewLevel2 += Symbol_NewLevel2;  // 板情報（L2）の更新
symbol.NewLast += Symbol_NewLast;      // 約定の更新
symbol.NewDayBar += Symbol_NewDayBar;  // 日足の更新

// イベントハンドラの例
private void Symbol_NewQuote(Symbol symbol, Quote quote)
{
    double bid = quote.Bid;
    double bidSize = quote.BidSize;
    double ask = quote.Ask;
    double askSize = quote.AskSize;
}

// 板情報（DOM）の取得
var depthOfMarket = symbol.DepthOfMarket.GetDepthOfMarketAggregatedCollections();
```

## 3. ヒストリカルデータとバーアクセス

```csharp
// 取得方法
// 1分足の履歴データを取得（過去1日分）
HistoricalData historicalData = symbol.GetHistory(Period.MIN1, DateTime.Now.AddDays(-1));

// ティック（BidとAsk）の履歴データを取得（過去1時間分）
historicalData = symbol.GetHistory(Period.TICK1, HistoryType.BidAsk, DateTime.Now.AddHours(-1));

// カスタム集計を使用した履歴データの取得（レンコチャート）
historicalData = symbol.GetHistory(new HistoryAggregationRenko(Period.MIN1, 10, RenkoStyle.AdvancedClassic, 100, 100, true, true), 
                                  HistoryType.Last, DateTime.Now.AddDays(-1));

// 履歴データ内での移動
// 現在のバー
IHistoryItem currentBar = historicalData[0];
// 1つ前のバー
IHistoryItem previousBar = historicalData[1];
// 指定インデックスのバー
IHistoryItem bar = historicalData[10];

// バーデータへのアクセス（バーの場合）
double open = historyItem[PriceType.Open];
double high = historyItem[PriceType.High];
double low = historyItem[PriceType.Low];
double close = historyItem[PriceType.Close];
double volume = historyItem[PriceType.Volume];

// 型キャストでのアクセス（バーの場合）
HistoryItemBar barItem = historyItem as HistoryItemBar;
if (barItem != null)
{
    DateTime time = barItem.TimeLeft;
    open = barItem.Open;
    high = barItem.High;
    low = barItem.Low;
    close = barItem.Close;
}

// イベント購読
historicalData.HistoryItemUpdated += HistoricalDataOnHistoryItemUpdated;
historicalData.NewHistoryItem += HistoricalDataOnNewHistoryItem;
```

## 4. インジケーターとテクニカル分析

```csharp
// インジケーターのパラメーター定義
[InputParameter("Period of MA", 0, 1, 999, 1, 0)]
public int Period = 10;

[InputParameter("Sources prices for MA", 1, variants: new object[]{
    "Close", PriceType.Close,
    "Open", PriceType.Open,
    "High", PriceType.High,
    "Low", PriceType.Low,
    "Typical", PriceType.Typical,
    "Median", PriceType.Median,
    "Weighted", PriceType.Weighted
})]
public PriceType SourcePrice = PriceType.Close;

// 組み込みインジケーターの使用
// SMAの作成
Indicator sma = Core.Instance.Indicators.BuiltIn.SMA(this.Period, PriceType.Close);
// インジケーターの追加
this.AddIndicator(this.sma);

// 別のインジケーターをインスタンス化
IndicatorInfo emaIndicatorInfo = Core.Instance.Indicators.All.FirstOrDefault(info => info.Name == "Exponential Moving Average");
Indicator emaInd = Core.Instance.Indicators.CreateIndicator(emaIndicatorInfo);

// インジケーターのパラメーター設定
emaInd.Settings = new List<SettingItem>
{
    new SettingItemInteger("MaPeriod", 10)
};

// インジケーターの値を取得
double smaValue = this.sma.GetValue();
// オフセットを指定して値を取得（例：1つ前の値）
double prevSmaValue = this.sma.GetValue(1);

// インジケーター内での価格取得ヘルパー
double currentClose = GetPrice(SourcePrice);      // 現在の価格
double previousClose = GetPrice(SourcePrice, 1);  // 1つ前の価格

// 値の設定（インジケーターの出力を設定）
SetValue(calculatedValue);           // 最初のラインに値を設定
SetValue(anotherValue, 1);           // 2番目のラインに値を設定
```

## 5. 注文執行と取引

```csharp
// 注文リクエストの作成（詳細版）
var request = new PlaceOrderRequestParameters
{
    Symbol = symbol,                        // 必須
    Account = account,                      // 必須
    Side = Side.Buy,                        // 必須（Buy/Sell）
    OrderTypeId = OrderType.Market,         // 必須（Market, Limit, Stop, StopLimit, TrailingStop）
    Quantity = 0.5,                         // 必須
    
    Price = 1.52,                           // オプション（指値注文に必要）
    TriggerPrice = 1.52,                    // オプション（逆指値注文に必要）
    TrailOffset = 20,                       // オプション（トレール注文に必要）
    
    TimeInForce = TimeInForce.Day,          // オプション（Day, GTC, GTD, GTT, FOK, IOC）
    ExpirationTime = Core.Instance.TimeUtils.DateTimeUtcNow.AddDays(1), // オプション
    StopLoss = SlTpHolder.CreateSL(1.4),    // オプション（損切り）
    TakeProfit = SlTpHolder.CreateTP(2.2)   // オプション（利確）
};

// 注文送信
var result = Core.Instance.PlaceOrder(request);

// 簡易版（成行注文）
result = Core.Instance.PlaceOrder(symbol, account, Side.Buy);
// 簡易版（指値注文）
result = Core.Instance.PlaceOrder(symbol, account, Side.Sell, price: 1.5);
// 簡易版（逆指値注文）
result = Core.Instance.PlaceOrder(symbol, account, Side.Buy, triggerPrice: 2.1);

// 注文結果の確認
if (result.Status == TradingOperationResultStatus.Failure)
    this.LogError(result.Message);

// ポジションに対する損切り・利確注文の設定
private void PlaceCloseOrder(Position position, CloseOrderType closeOrderType, double priceOffset)
{
    var request = new PlaceOrderRequestParameters
    {
        Symbol = this.CurrentSymbol,
        Account = this.CurrentAccount,
        Side = position.Side == Side.Buy ? Side.Sell : Side.Buy,
        Quantity = position.Quantity,
        PositionId = position.Id,
        AdditionalParameters = new List<SettingItem>
        {
            new SettingItemBoolean(OrderType.REDUCE_ONLY, true)
        }
    };
    
    // 損切り注文の場合
    if (closeOrderType == CloseOrderType.StopLoss)
    {
        // 逆指値注文タイプを取得
        var orderType = this.CurrentSymbol.GetAlowedOrderTypes(OrderTypeUsage.CloseOrder)
            .FirstOrDefault(ot => ot.Behavior == OrderTypeBehavior.Stop);
        
        request.OrderTypeId = orderType.Id;
        request.TriggerPrice = position.Side == Side.Buy ? 
            position.OpenPrice - priceOffset : position.OpenPrice + priceOffset;
    }
    // 利確注文の場合
    else
    {
        // 指値注文タイプを取得
        var orderType = this.CurrentSymbol.GetAlowedOrderTypes(OrderTypeUsage.CloseOrder)
            .FirstOrDefault(ot => ot.Behavior == OrderTypeBehavior.Limit);
        
        request.OrderTypeId = orderType.Id;
        request.Price = position.Side == Side.Buy ? 
            position.OpenPrice + priceOffset : position.OpenPrice - priceOffset;
    }
    
    var result = Core.PlaceOrder(request);
}
```

## 6. イベント処理

```csharp
// シンボル関連イベント
symbol.NewQuote += Symbol_NewQuote;    // Bid/Ask更新時
symbol.NewLevel2 += Symbol_NewLevel2;  // 板情報更新時
symbol.NewLast += Symbol_NewLast;      // 約定発生時
symbol.NewDayBar += Symbol_NewDayBar;  // 日足更新時

// 歴史データ関連イベント
historicalData.HistoryItemUpdated += HistoricalDataOnHistoryItemUpdated; // 既存バー更新時
historicalData.NewHistoryItem += HistoricalDataOnNewHistoryItem;         // 新しいバー作成時

// 注文・ポジション関連イベント
Core.PositionAdded += CoreOnPositionAdded;       // 新しいポジション作成時
Core.PositionChanged += CoreOnPositionChanged;   // ポジション変更時
Core.PositionRemoved += CoreOnPositionRemoved;   // ポジション決済時

// チャートマウスイベント
this.CurrentChart.MouseDown += CurrentChart_MouseDown;  // マウスボタンが押された時
this.CurrentChart.MouseUp += CurrentChart_MouseUp;      // マウスボタンが離された時

// イベントハンドラの例
private void CoreOnPositionAdded(Position position)
{
    // ポジションが追加された時の処理
}

private void HistoricalDataOnNewHistoryItem(object sender, HistoryEventArgs e)
{
    // 新しいバーが作成された時の処理
    IHistoryItem historyItem = e.HistoryItem;
}

// イベントアンサブスクライブ（重要：リソースリーク防止）
protected override void OnStop()
{
    if (this.Symbol != null)
        this.Symbol.NewLast -= Symbol_NewLast;
    
    Core.PositionAdded -= CoreOnPositionAdded;
}
```

## 7. 描画とUI

```csharp
// チャート描画のオーバーライド
public override void OnPaintChart(PaintChartEventArgs args)
{
    base.OnPaintChart(args);
    
    Graphics gr = args.Graphics;
    Font font = new Font("Arial", 8, FontStyle.Regular);
    
    // テキスト描画
    gr.DrawString("Hello World", font, Brushes.LightBlue, 30, 50);
    
    // ライン描画
    Pen pen = new Pen(Brushes.Orange, 2);
    gr.DrawLine(pen, startPoint, endPoint);
    
    // 座標変換（価格→Y座標）
    int yCoord = (int)this.CurrentChart.MainWindow.CoordinatesConverter.GetChartY(price);
    // 座標変換（時間→X座標）
    int xCoord = (int)this.CurrentChart.MainWindow.CoordinatesConverter.GetChartX(time);
    
    // 可視バーの取得
    DateTime leftTime = mainWindow.CoordinatesConverter.GetTime(mainWindow.ClientRectangle.Left);
    DateTime rightTime = mainWindow.CoordinatesConverter.GetTime(mainWindow.ClientRectangle.Right);
    int leftIndex = (int)mainWindow.CoordinatesConverter.GetBarIndex(leftTime);
    int rightIndex = (int)Math.Ceiling(mainWindow.CoordinatesConverter.GetBarIndex(rightTime));
    
    // 各バーに対する描画
    for (int i = leftIndex; i <= rightIndex; i++)
    {
        if (i > 0 && i < this.HistoricalData.Count && this.HistoricalData[i, SeekOriginHistory.Begin] is HistoryItemBar bar)
        {
            // バーの処理
        }
    }
}
```

## 8. カスタム設定とパラメーター

```csharp
// 基本パラメーター
[InputParameter("Period", 0, 1, 999, 1, 0)]
public int Period = 13;

[InputParameter("Type of Moving Average", 1, variants: new object[] {
    "Simple", MaMode.SMA,
    "Exponential", MaMode.EMA,
    "Modified", MaMode.SMMA,
    "Linear Weighted", MaMode.LWMA}
)]
public MaMode MAType = MaMode.SMA;

// ドロップダウンの作成
public override IList<SettingItem> Settings
{
    get
    {
        var settings = base.Settings;

        var firstOption = new SelectItem("First option", 1);
        var secondOption = new SelectItem("Second option", 2);

        settings.Add(new SettingItemSelectorLocalized("someSelectorField", 
            new SelectItem("", this.someSelectorField), 
            new List<SelectItem> { firstOption, secondOption })
        {
            Text = "Selector",
            SortIndex = 10
        });

        // 条件付き表示（第一オプション選択時のみ表示）
        settings.Add(new SettingItemDouble("firstDependentField", this.firstDependentField)
        {
            Text = "First field",
            SortIndex = 20,
            Relation = new SettingItemRelationVisibility("someSelectorField", firstOption)
        });

        return settings;
    }
    set
    {
        // 値の取得方法
        if (value.TryGetValue("someSelectorField", out int selectorValue))
            this.someSelectorField = selectorValue;
    }
}

// 他のインジケーターの設定を取り込む
if (superTrendInd != null)
{
    // Super Trendインジケーターから表示用の入力パラメーターを取得
    var superTrendIndSettings = superTrendInd.Settings
        .Where(s => s.SeparatorGroup != null && s.SeparatorGroup.Text == "Input parameters" && s.VisibilityMode == VisibilityMode.Visible)
        .ToList();

    // 別グループに設定
    SettingItemSeparatorGroup superTrendIndSeparator = new SettingItemSeparatorGroup("Super Trend");
    foreach (var s in superTrendIndSettings)
        s.SeparatorGroup = superTrendIndSeparator;

    // 設定に追加
    result.Add(new SettingItemGroup("SuperTrend", superTrendIndSettings));
}
```

## 9. ユーティリティ

```csharp
// 日時関連
DateTime utcNow = Core.Instance.TimeUtils.DateTimeUtcNow;
DateTime localTime = Core.Instance.TimeUtils.DateTimeUtcNow.AddDays(-1);

// ティック（価格）フォーマット
string formattedPrice = this.Symbol.FormatPrice(price);

// 数量フォーマット
string formattedQuantity = this.Symbol.FormatQuantity(size);

// セッション情報へのアクセス
var sessionsContainer = symbol.FindSessionsContainer();
if (sessionsContainer != null)
{
    foreach (var session in sessionsContainer.ActiveSessions)
    {
        // セッション名、開始/終了時刻
        string name = session.Name;
        TimeSpan openTime = session.OpenTime;
        TimeSpan closeTime = session.CloseTime;
        
        // 現在アクティブかどうか
        bool isActive = session.ContainsDate(Core.Instance.TimeUtils.DateTimeUtcNow);
    }
}

// ロギング
Core.Instance.Loggers.Log("Message", LoggingLevel.System);
```

## 10. 非同期処理

```csharp
// タスク実行
Task.Run(() =>
{
    // バックグラウンド処理
});

// 遅延実行
Task.Delay(TimeSpan.FromSeconds(this.DelaySeconds)).Wait();

// ウェブフックの例
private async Task StartListening(CancellationToken token)
{
    string url = "http://localhost:5000/";
    HttpListener webhookListener = new HttpListener();
    
    try
    {
        webhookListener.Prefixes.Add(url);
        webhookListener.Start();
        
        while (!token.IsCancellationRequested)
        {
            try
            {
                var contextTask = webhookListener.GetContextAsync();
                var completedTask = await Task.WhenAny(contextTask, Task.Delay(Timeout.Infinite, token));
                
                if (completedTask == contextTask)
                {
                    HttpListenerContext context = await contextTask;
                    HttpListenerRequest request = context.Request;
                    
                    using (StreamReader reader = new StreamReader(request.InputStream, Encoding.UTF8))
                    {
                        string message = reader.ReadToEnd();
                        // メッセージ処理
                    }
                }
                else
                    break;
            }
            catch (Exception ex)
            {
                Log($"Webhook error: {ex.Message}", StrategyLoggingLevel.Error);
            }
        }
    }
    finally
    {
        if(webhookListener.IsListening)
        {
            webhookListener.Stop();
            webhookListener.Close();
        }
    }
}
```

## 11. パフォーマンス最適化

```csharp
// 配列再利用によるGC低減
private double[] _priceBuffer = new double[100];
private double[] _calculatedBuffer = new double[100];

// 結果を計算する際に新しい配列を作成せず、既存の配列を再利用
public double[] CalculateValues(double[] prices)
{
    // 入力配列のサイズに合わせてバッファサイズを調整
    if (_priceBuffer.Length < prices.Length)
    {
        Array.Resize(ref _priceBuffer, prices.Length);
        Array.Resize(ref _calculatedBuffer, prices.Length);
    }
    
    // データをコピー
    Array.Copy(prices, _priceBuffer, prices.Length);
    
    // 計算を実行（既存配列に結果を格納）
    for (int i = 0; i < prices.Length; i++)
    {
        _calculatedBuffer[i] = Math.Sqrt(_priceBuffer[i]);
    }
    
    return _calculatedBuffer;
}

// イベントハンドラのキャッシュ
private EventHandler<QuoteEventArgs> _quoteHandler;

// コンストラクタでハンドラを作成
public MyIndicator()
{
    _quoteHandler = new EventHandler<QuoteEventArgs>(Symbol_NewQuote);
}

// イベント登録
private void SubscribeEvents()
{
    if (_symbol != null)
        _symbol.NewQuote += _quoteHandler;
}

// イベント解除
private void UnsubscribeEvents()
{
    if (_symbol != null)
        _symbol.NewQuote -= _quoteHandler;
}

// オブジェクトプール実装例
public class ObjectPool<T> where T : class, new()
{
    private readonly ConcurrentBag<T> _objects;
    private readonly Func<T> _objectGenerator;

    public ObjectPool(Func<T> objectGenerator)
    {
        _objects = new ConcurrentBag<T>();
        _objectGenerator = objectGenerator ?? (() => new T());
    }

    public T Get() => _objects.TryTake(out T item) ? item : _objectGenerator();
    public void Return(T item) => _objects.Add(item);
}

// 使用例
private ObjectPool<List<double>> _listPool = new ObjectPool<List<double>>(() => new List<double>(100));

private void ProcessData()
{
    var list = _listPool.Get();
    try
    {
        // リストを使用
        list.Clear(); // 再利用前にクリア
        
        // データ処理...
    }
    finally
    {
        _listPool.Return(list); // 処理後にプールに返却
    }
}
```

## 12. メモリ管理とリソース解放

```csharp
// リソース解放パターン (IDisposable実装)
public class MyComponent : IDisposable
{
    private bool _disposed = false;
    private Symbol _symbol;
    private readonly List<HistoricalData> _historicalDataList = new List<HistoricalData>();
    
    // コンストラクタでリソースを初期化
    public MyComponent(Symbol symbol)
    {
        _symbol = symbol;
        // イベント購読
        _symbol.NewQuote += Symbol_NewQuote;
    }
    
    // Dispose パターン実装
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
    
    // リソース解放の実装
    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                // マネージドリソースの解放
                if (_symbol != null)
                {
                    _symbol.NewQuote -= Symbol_NewQuote;
                    _symbol = null;
                }
                
                // ヒストリカルデータのイベント解除とクリーンアップ
                foreach (var historicalData in _historicalDataList)
                {
                    historicalData.NewHistoryItem -= HistoricalData_NewHistoryItem;
                    historicalData.HistoryItemUpdated -= HistoricalData_HistoryItemUpdated;
                }
                _historicalDataList.Clear();
            }
            
            // アンマネージドリソースの解放（該当あれば）
            
            _disposed = true;
        }
    }
    
    // デストラクタ（ファイナライザ）
    ~MyComponent()
    {
        Dispose(false);
    }
    
    // リソース追加メソッド
    public void AddHistoricalData(HistoricalData data)
    {
        ThrowIfDisposed();
        
        data.NewHistoryItem += HistoricalData_NewHistoryItem;
        data.HistoryItemUpdated += HistoricalData_HistoryItemUpdated;
        _historicalDataList.Add(data);
    }
    
    // Disposeチェック
    private void ThrowIfDisposed()
    {
        if (_disposed)
        {
            throw new ObjectDisposedException(nameof(MyComponent));
        }
    }
    
    // イベントハンドラ
    private void Symbol_NewQuote(Symbol symbol, Quote quote)
    {
        if (_disposed) return; // 安全チェック
        
        // 処理...
    }
    
    private void HistoricalData_NewHistoryItem(object sender, HistoryEventArgs e)
    {
        if (_disposed) return; // 安全チェック
        
        // 処理...
    }
    
    private void HistoricalData_HistoryItemUpdated(object sender, HistoryEventArgs e)
    {
        if (_disposed) return; // 安全チェック
        
        // 処理...
    }
}
```

## 13. エラーハンドリングとロギング

```csharp
// 安全な操作実行パターン
private void SafeExecute(Action action, string operationName)
{
    try
    {
        action();
    }
    catch (Exception ex)
    {
        string errorMessage = $"Error in {operationName}: {ex.Message}";
        Log(errorMessage, StrategyLoggingLevel.Error);
        
        // 詳細なエラー情報をデバッグログに記録
        Log($"StackTrace: {ex.StackTrace}", StrategyLoggingLevel.Debug);
        
        // 深刻なエラーの場合は通知
        if (IsOperationCritical(operationName))
        {
            SendNotification(errorMessage);
        }
    }
}

// 使用例
private void ProcessMarketData(Quote quote)
{
    SafeExecute(() => 
    {
        // マーケットデータ処理...
        
        // 注文発注
        var result = Core.Instance.PlaceOrder(_symbol, _account, Side.Buy);
        
        // 結果チェック
        if (result.Status == TradingOperationResultStatus.Failure)
        {
            throw new Exception($"Order placement failed: {result.Message}");
        }
    }, "ProcessMarketData");
}

// 操作の重要度判定
private bool IsOperationCritical(string operationName)
{
    string[] criticalOperations = new[] { 
        "PlaceOrder", "ModifyOrder", "CancelOrder", "ClosePosition" 
    };
    
    return criticalOperations.Any(op => operationName.Contains(op));
}

// 通知送信
private void SendNotification(string message)
{
    // メール通知
    try
    {
        Core.Instance.MailUtils.SendMail("Error notification", message, "user@example.com");
    }
    catch (Exception ex)
    {
        Log($"Failed to send email notification: {ex.Message}", StrategyLoggingLevel.Error);
    }
    
    // アラート
    try
    {
        Core.Instance.Alert(message, "", "");
    }
    catch (Exception ex)
    {
        Log($"Failed to send alert: {ex.Message}", StrategyLoggingLevel.Error);
    }
}

// リトライメカニズム
private T ExecuteWithRetry<T>(Func<T> operation, int maxRetryCount = 3)
{
    int retryCount = 0;
    
    while (true)
    {
        try
        {
            return operation();
        }
        catch (Exception ex)
        {
            retryCount++;
            
            if (retryCount > maxRetryCount)
                throw;
            
            Log($"Operation failed, retrying ({retryCount}/{maxRetryCount}): {ex.Message}", 
                StrategyLoggingLevel.Warning);
            
            // 指数バックオフ
            int delayMs = (int)Math.Pow(2, retryCount) * 100;
            Thread.Sleep(delayMs);
        }
    }
}

// 使用例
private void LoadHistoricalData()
{
    ExecuteWithRetry(() => 
    {
        return _symbol.GetHistory(Period.MIN1, DateTime.Now.AddDays(-1));
    });
}
```

この資料は、Quantower APIの一般的な使用パターンをカバーしています。プロジェクト固有の要件がある場合は、Quantower APIリファレンスを参照するか、具体的な実装例を探してください。