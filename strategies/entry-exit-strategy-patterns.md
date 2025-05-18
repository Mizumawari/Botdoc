# エントリー・出口戦略設計パターン

このドキュメントでは、DynamicExitStrategyImplementationで実装されているエントリー戦略と出口戦略の設計パターン、責任分離、およびベストプラクティスについて説明します。

## 目次

1. [全体アーキテクチャ](#全体アーキテクチャ)
2. [エントリー戦略の設計](#エントリー戦略の設計)
3. [出口戦略の設計](#出口戦略の設計)
4. [複数ポジション対応](#複数ポジション対応)
5. [イベント駆動モデル](#イベント駆動モデル)
6. [拡張と最適化](#拡張と最適化)

## 全体アーキテクチャ

DynamicExitStrategyImplementationプロジェクトでは、明確な責任分離に基づいたモジュール構造を採用しています：

```
[SymbolPositionSubscriptionManager] ← イベント購読集中管理
        ↓
[PositionManager] ← 損益/数量の変化を検知、外部に通知
        ↓
[DynamicExitStrategy] ← Exit条件の評価とExit処理（監視・実行）
        ↑
      Smash戦略本体 ← 各マネージャの統括と監視起動の司令塔
```

### 責任分離

| コンポーネント | 主な責任と禁止事項 |
|--------------|------------------|
| SymbolPositionSubscriptionManager | Symbol/PositionのPUSHイベントを購読するのみ（ロジックは持たない） |
| PositionManager | 損益変化 / 数量変化 を検知してイベント通知。状態保持責任あり |
| DynamicExitStrategy | Exit判定・トレーリング実行・Exit完了通知を行う。ポジション状態は外から受け取るだけ |
| Smash本体 | 上記コンポーネントの初期化と接続を担う。状態は持たず、購読と中継のみ行う |

### 処理フロー

1. **ポジションが追加される**
   - SymbolPositionSubscriptionManager によって価格・ポジション購読
   - PositionManager に登録され、変化監視開始

2. **エントリーが成功したとき**
   - Smash から DynamicExitStrategy.StartMonitoring(positionId, price, qty, side, atr) を呼ぶ

3. **価格や損益に変化**
   - PositionManager が PositionPnLChanged / PositionQuantityChanged を発火
   - DynamicExitStrategy が評価し、Exit条件を満たしたら監視終了・Exit処理を発動

## エントリー戦略の設計

エントリー戦略は「フェーズ制」を採用し、相場状況に応じて自動的に動作モードを切り替える仕組みを実装しています。

### フェーズ設計：3段階モード

| フェーズ名 | 行動方針 | 主要トリガー例 |
|-----------|--------|--------------|
| ScalpMode | スキャル的に反応重視 | 小モメンタム + VolumeSpike |
| TrendMode | トレンドフォロー的に乗る | 方向一致 + DOM傾き + モメ/出来高持続 |
| ReversalReady | 逆方向への構えに移行 | 急反転兆候 + 板圧力 + 高ATR |

### 切替条件（スコア型）

```
スコアA（加速感）：モメンタム + Volume持続 + DOM整合性  
スコアB（反転兆候）：出来高偏り + 方向反転 + 板勢い変化

if Aスコア > 0.8:
   → TrendMode
elif Bスコア > 0.7:
   → ReversalReady
else:
   → ScalpMode（デフォルト）
```

### EntryContextの設計

各エントリーには、その背景・条件・信頼度などのコンテキスト情報が付与されます：

```csharp
// EntryContext（ENTRYメタ情報）
public class EntryContext
{
    public string PositionId { get; set; }     // 紐づくポジションのID（唯一）
    public Side Side { get; set; }             // Long / Short
    public double EntryPrice { get; set; }     // 約定価格
    public double EntryATR { get; set; }       // ENTRY時のATR
    public double SignalStrength { get; set; } // ENTRY判断時の信頼スコア（0〜1）
    public string EntryType { get; set; }      // "scalp" / "trend" / "reversal" など
    public Dictionary<string, object> IndicatorsUsed { get; set; } // ENTRYに使われたインジケータ情報
    public DateTime Timestamp { get; set; }    // ENTRY発生時刻
}
```

## 出口戦略の設計

出口戦略は複数ポジション対応を前提とし、ポジションIDごとのExit判断ユニットで構成されています。

### コア構成

```
[PositionManager]                ← 複数ポジションの登録・購読・状態同期
    └─ [PositionTracker]         ← 個別ポジションの状態管理と変化検出
         └─ [ExitContext]        ← トレーリング・TP/SL等のExit条件の設定・記録
         └─ [ExitEvaluator]      ← Exit条件のロジック評価（ShouldExit系）
[DynamicExitStrategy]            ← Exit評価・実行のインターフェース層
[Smash]                          ← 全体制御、初期化、マーケット全体Exit制御
```

### ExitStrategyUnitの設計

```csharp
// ExitStrategyUnit（個別EXIT判断ユニット）
public class ExitStrategyUnit
{
    public EntryContext EntryContext { get; private set; } // ENTRYメタ情報
    public double CurrentPnL { get; private set; }        // EXIT判断時の損益
    public TimeSpan ElapsedTime { get; private set; }     // 経過時間
    public double BestPrice { get; private set; }         // トレール判定や利幅評価に使用
    public Dictionary<string, double> DynamicThresholds { get; private set; } // 可変EXIT条件
    
    public bool EvaluateExit() { /* ... */ }              // EXIT判断
    public ExitOrder GenerateExitOrder() { /* ... */ }    // EXIT命令生成
}
```

### Exit判断ロジック

エントリーコンテキストに基づいたExit判断の例：

```csharp
// ExitEvaluator (EntryContextに基づくEXIT切替)
if (entry.Type == "trend" && entry.SignalStrength > 0.75)
{
    // 利大追求型
    if (pnl >= entry.EntryATR * 2.5)
        return TrailingStop();
}
else if (entry.Type == "scalp")
{
    // 利小＋高速撤退型
    if (ElapsedTime > TimeSpan.FromSeconds(30) || pnl > atr * 0.5)
        return MarketClose();
}
else if (entry.Type == "reversal")
{
    if (価格が失速 || 再反転兆候)
        return ExitImmediately();
}
```

## 複数ポジション対応

### PositionManager設計

複数ポジション管理のためのデータ構造：

```csharp
private readonly Dictionary<string, PositionInfo> _positions = new Dictionary<string, PositionInfo>();
private readonly Dictionary<string, Symbol> _subscribedSymbols = new Dictionary<string, Symbol>();
private readonly ReaderWriterLockSlim _positionsLock = new ReaderWriterLockSlim();

// マネージドポジション情報をカプセル化
private class PositionInfo
{
    public string Id { get; set; }
    public Side Side { get; set; }
    public double EntryPrice { get; set; }
    public double Quantity { get; set; }
    public DateTime EntryTime { get; set; }
    public double? LastPnL { get; set; }
    public bool HasPriceSubscription { get; set; }
}
```

### DynamicExitStrategy設計

複数ポジション監視のための構造：

```csharp
// 複数ポジション追跡のためのコレクション
private readonly Dictionary<string, ExitContext> _monitoredPositions = new Dictionary<string, ExitContext>();
private readonly object _positionsLock = new object();

// 監視コンテキスト情報
private class ExitContext
{
    public string PositionId { get; set; }
    public Side PositionSide { get; set; }
    public double EntryPrice { get; set; }
    public double OriginalQuantity { get; set; }
    public double RemainingQuantity { get; set; }
    public DateTime EntryTime { get; set; }
    public double EntryAtr { get; set; }
    public double BestPrice { get; set; }
    public double LastTrailingUpdatePrice { get; set; }
    public DateTime LastTrailingUpdate { get; set; }
    public string TrailingStopOrderId { get; set; }
    public bool PartialTakeProfit1Executed { get; set; }
    public bool PartialTakeProfit2Executed { get; set; }
    public bool BreakEvenActivated { get; set; }
    public Dictionary<ExitReason, bool> ExitReasons { get; set; } = new Dictionary<ExitReason, bool>();
}
```

## イベント駆動モデル

このプロジェクトでは徹底したイベント駆動モデルを採用し、各コンポーネント間の連携を実現しています。

### イベントフロー

```
[Symbol.NewQuote] → PositionManager → [PositionPnLChanged] → DynamicExitStrategy
[Position.Updated] → PositionManager → [PositionQuantityChanged] → DynamicExitStrategy
```

### イベント購読の一元管理

```csharp
// シンボル価格更新の購読を確実にする
private void EnsureSymbolSubscription(Symbol symbol)
{
    if (symbol == null) return;
    
    string symbolId = symbol.Id;
    
    try
    {
        _positionsLock.EnterUpgradeableReadLock();
        
        if (!_subscribedSymbols.ContainsKey(symbolId))
        {
            try
            {
                _positionsLock.EnterWriteLock();
                
                // 重複追加を防止して再登録
                symbol.NewQuote -= Symbol_NewQuote;
                symbol.NewQuote += Symbol_NewQuote;
                symbol.NewLast -= Symbol_NewLast;
                symbol.NewLast += Symbol_NewLast;
                
                _subscribedSymbols[symbolId] = symbol;
            }
            finally
            {
                _positionsLock.ExitWriteLock();
            }
        }
    }
    finally
    {
        _positionsLock.ExitUpgradeableReadLock();
    }
}
```

## 拡張と最適化

### 将来拡張：「板読み（Order Book Bias）」

| 要素 | 内容 | LEVEL1で再現するなら |
|-----|-----|------------------|
| Iceberg兆候 | 小ロットの連打頻度 | TickSize単位の連続注文 |
| スプーフィング兆候 | 高い価格に一時的注文集中 →消える | 瞬間ボリューム + 反発の非一致 |
| バイアス方向 | 買い/売り注文の片寄り | Bid > Ask or その逆 |
| 買板厚み偏差 | レベル差を統計的に分析 | LEVEL1では難 →「気配厚みの連続度」で代替可 |

### 設計指針まとめ

| 項目 | 戦略 |
|-----|-----|
| モード制御 | 3モード制（Scalp / Trend / Reversal）で自然遷移 |
| 閾値調整 | 各モードごとにスコア or 状況による柔軟調整 |
| Entry | モードに応じた閾値＋反応感度で制御 |
| EXIT | Entryの雑さを補う「本丸」 |
| 板拡張 | LEVEL1ベースで疑似バイアス検知 |
| 将来拡張 | 環境相場パターンを判別（Ranging, Breakoutなど） |

### メリット

| 項目 | 内容 |
|-----|-----|
| 🔄 整合性 | ENTRYの判断基準に合わせたEXIT判断ができる（全体の一貫性） |
| 🧠 柔軟性 | EXIT条件をENTRYのタイプや強さで最適化できる |
| 📊 多ポジ対応 | 複数のENTRY/EXITが衝突せず並列に処理可能 |
| 🔍 解析性 | 「なぜこのEXITだったか？」のロジック検証がしやすい |
| 🔧 拡張性 | EXIT戦略単体でテスト・交換が可能（戦略プラグイン化も可能） |

## 最終的な原則

### 単一責任の原則
- 各クラスとメソッドは1つの責任のみを持つ
- PositionManager: ポジション状態の管理と同期
- DynamicExitStrategy: エグジット条件の評価と実行

### オープン/クローズドの原則
- 拡張に対してオープンであり、修正に対してクローズド
- 特定のポジションIDをパラメータとして取る設計に変更

### 依存関係逆転の原則
- インターフェースに依存し、具体的な実装には依存しない
- IPositionManager インターフェースを使用した再設計

### 効率性の原則
- パフォーマンスを考慮したデータ構造とアルゴリズム
- ReaderWriterLockSlim によるスレッドセーフなアクセス制御
- ロック範囲の最小化と部分的なデータキャプチャ