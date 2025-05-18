# Quantower API - Core クラスコードインデックス

## 1. 概要

`TradingPlatform.BusinessLayer.Core`クラスはQuantower APIのメインエントリーポイントです。このクラスを通して、取引プラットフォームのすべての主要ビジネスロジックエンティティ（接続、アカウント、シンボル、ポジション、注文など）にアクセスできます。

```csharp
// Core インスタンスの取得
Core core = Core.Instance;
```

## 2. 主要プロパティ

### 2.1 シングルトンアクセス

| プロパティ | 説明 |
|----------|------|
| `Instance` | Coreのシングルトンインスタンスへのアクセスを提供（API全体のエントリーポイント） |

### 2.2 マネージャークラス

| プロパティ | 説明 |
|----------|------|
| `Connections` | 接続の作成と管理 |
| `Loggers` | システムログ管理 |
| `Indicators` | テクニカル指標の作成と管理 |
| `Strategies` | トレード戦略の管理 |
| `OrderPlacingStrategies` | 注文発注戦略の管理 |
| `VolumeAnalysis` | ボリューム分析計算へのアクセス |
| `LocalOrders` | ローカル注文の管理 |
| `TimeUtils` | 時間操作ユーティリティ |
| `SymbolsMapping` | シンボルマッピング管理 |
| `CustomSessions` | カスタムセッション管理 |
| `MailUtils` | メール送信ユーティリティ |
| `TradingProtection` | トレード保護機能 |

### 2.3 ビジネスオブジェクトコレクション

| プロパティ | 説明 |
|----------|------|
| `Symbols` | 利用可能なすべてのシンボル配列 |
| `SymbolTypes` | 利用可能なすべてのシンボルタイプ配列 |
| `Accounts` | 利用可能なすべてのアカウント配列 |
| `Assets` | 利用可能なすべての資産配列 |
| `Exchanges` | 利用可能なすべての取引所配列 |
| `Orders` | 利用可能なすべての注文配列 |
| `OrderTypes` | 利用可能なすべての注文タイプ配列 |
| `Positions` | 利用可能なすべてのポジション配列 |
| `ClosedPositions` | 利用可能なすべての決済済みポジション配列 |
| `CorporateActions` | 利用可能なすべてのコーポレートアクション配列 |
| `ReportTypes` | 利用可能なすべてのレポートタイプ配列 |
| `TradingSignals` | 利用可能なすべてのトレードシグナル配列 |
| `DeliveredAssets` | 利用可能なすべての配信資産配列 |

### 2.4 その他のプロパティ

| プロパティ | 説明 |
|----------|------|
| `TradingStatus` | 現在の取引ステータス |
| `CurrentVersion` | 現在のAPIバージョン |
| `AdvancedTradingOperations` | 高度な取引操作 |

## 3. 主要メソッド

### 3.1 注文関連メソッド

```csharp
// 基本的な成行注文の発注
TradingOperationResult result = Core.Instance.PlaceOrder(symbol, account, Side.Buy);

// 詳細なパラメータ指定による注文発注
var request = new PlaceOrderRequestParameters
{
    Symbol = symbol,                        // 必須
    Account = account,                      // 必須
    Side = Side.Buy,                        // 必須（Buy/Sell）
    OrderTypeId = OrderType.Market,         // 必須
    Quantity = 0.5,                         // 必須
    Price = 1.52,                           // オプション
    TriggerPrice = 1.52,                    // オプション
    TimeInForce = TimeInForce.Day,          // オプション
};
result = Core.Instance.PlaceOrder(request);

// 注文の修正
result = Core.Instance.ModifyOrder(order, TimeInForce.Day, 1.0, price: 1.5);

// 注文のキャンセル
result = Core.Instance.CancelOrder(order, sendingSource: null);

// ポジションのクローズ
result = Core.Instance.ClosePosition(position, closeQuantity);
```

### 3.2 データアクセスメソッド

```csharp
// シンボル検索
IList<Symbol> symbols = Core.Instance.SearchSymbols(new SearchSymbolsRequestParameters {
    FilterName = "EURUSD"
});

// シンボル取得
Symbol symbol = Core.Instance.GetSymbol(new GetSymbolRequestParameters {
    SymbolId = "EURUSD"
});

// 注文タイプ取得
OrderType orderType = Core.Instance.GetOrderType("Market");

// 注文ID検索
Order order = Core.Instance.GetOrderById(orderId);

// ポジションID検索
Position position = Core.Instance.GetPositionById(positionId);

// 損益計算
PnL pnl = Core.Instance.CalculatePnL(new PnLRequestParameters {
    Symbol = symbol,
    Account = account,
    OpenPrice = 1.1,
    ClosePrice = 1.2,
    Side = Side.Buy,
    Quantity = 1.0
});

// 約定履歴取得
IList<Trade> trades = Core.Instance.GetTrades(new TradesHistoryRequestParameters {
    FromTime = DateTime.UtcNow.AddDays(-7)
});

// 注文履歴取得
IList<OrderHistory> orderHistory = Core.Instance.GetOrdersHistory(new OrdersHistoryRequestParameters {
    FromTime = DateTime.UtcNow.AddDays(-7)
});

// レポート取得
Report report = Core.Instance.GetReport(new ReportRequestParameters {
    ReportType = reportType
});
```

### 3.3 オプション・先物関連メソッド

```csharp
// 先物契約取得
IList<Symbol> futures = Core.Instance.GetFutureContracts(underlierSymbol);

// オプションシリーズ取得
IList<OptionSerie> optionSeries = Core.Instance.GetOptionSeries(underlierSymbol);

// 権利行使価格取得
IList<Symbol> strikes = Core.Instance.GetStrikes(optionSerie);
```

### 3.4 その他のメソッド

```csharp
// カスタムリクエスト送信
Core.Instance.SendCustomRequest(connectionId, customRequestParameters);

// カスタムメッセージの購読
Core.Instance.SubscribeToCustomMessages(handler, messageTypeId);

// アラート表示
Core.Instance.Alert("アラートメッセージ", symbol.Name, connection.Name);

// 時間同期強制
Core.Instance.ForceTimeSync();
```

## 4. 主要イベント

```csharp
// 取引ステータス変更時
Core.Instance.OnTradingStatusChanged += (status) => {
    Console.WriteLine($"取引ステータス変更: {status}");
};

// アカウント追加時
Core.Instance.AccountAdded += (account) => {
    Console.WriteLine($"アカウント追加: {account.Name}");
};

// シンボル追加時
Core.Instance.SymbolAdded += (symbol) => {
    Console.WriteLine($"シンボル追加: {symbol.Name}");
};

// 注文関連イベント
Core.Instance.OrderAdded += (order) => {
    Console.WriteLine($"注文追加: {order.Id}");
};
Core.Instance.OrderRemoved += (order) => {
    Console.WriteLine($"注文削除: {order.Id}");
};

// ポジション関連イベント
Core.Instance.PositionAdded += (position) => {
    Console.WriteLine($"ポジション追加: {position.Id}");
};
Core.Instance.PositionRemoved += (position) => {
    Console.WriteLine($"ポジション削除: {position.Id}");
};

// 決済済みポジション関連イベント
Core.Instance.ClosedPositionAdded += (closedPosition) => {
    Console.WriteLine($"決済済みポジション追加: {closedPosition.Id}");
};

// 約定イベント
Core.Instance.TradeAdded += (trade) => {
    Console.WriteLine($"約定追加: {trade.Id}");
};

// その他のイベント
Core.Instance.CorporateActionAdded += (corporateAction) => { /* ... */ };
Core.Instance.OrdersHistoryAdded += (orderHistory) => { /* ... */ };
Core.Instance.DeliveredAssetAdded += (deliveredAsset) => { /* ... */ };
Core.Instance.DealTicketReceived += (dealTicket) => { /* ... */ };
Core.Instance.TradingSignalUpdate += (sender, e) => { /* ... */ };
Core.Instance.OnAlert += (alert) => { /* ... */ };
```

## 5. 実装パターン

### 5.1 シングルトンパターン

```csharp
// Core はシングルトンパターンを使用しています
Core core = Core.Instance;
```

### 5.2 ビジネスオブジェクトの直接参照

```csharp
// リアルタイム情報へのアクセス - 推奨アプローチ
double bid = symbol.Bid;
double ask = symbol.Ask;
double last = symbol.Last;

// ポジション情報への直接アクセス
double positionQuantity = position.Quantity;
double openPrice = position.OpenPrice;
double currentPnL = position.GrossPnL.Value; // null チェック推奨

// 注文状態への直接アクセス
string orderStatus = order.Status.ToString();
double orderPrice = order.Price;
```

### 5.3 イベント駆動型設計

```csharp
// マーケットデータのイベント購読
symbol.NewQuote += (symbol, quote) => {
    // 最新の気配値に対応
};

symbol.NewLast += (symbol, last) => {
    // 最新の約定に対応
};

// 注文・ポジションイベント
Core.Instance.OrderAdded += (order) => {
    // 新規注文に対応
};

Core.Instance.PositionAdded += (position) => {
    // 新規ポジションに対応
};
```

## 6. 重要な注意点

1. **リソース管理**:
   - イベントハンドラはリソースリークを避けるために適切にアンサブスクライブする必要があります。

2. **リアルタイムデータ**:
   - 価格やポジション情報などのリアルタイムデータは、ラッパークラスを介さず公式APIから直接参照することが推奨されています。

3. **エラーハンドリング**:
   - すべての操作結果(`TradingOperationResult`)はステータスをチェックして適切にエラーを処理する必要があります。

4. **スレッドセーフティ**:
   - 複数スレッドからの同時アクセスを考慮して設計する必要があります。

5. **パフォーマンス**:
   - 頻繁なコレクション列挙は避け、必要に応じてキャッシュの使用を検討しましょう。

---

この索引は、`TradingPlatform.BusinessLayer.Core`クラスの主要コンポーネントをカバーしています。APIの詳細や拡張機能については、公式ドキュメントを参照してください。