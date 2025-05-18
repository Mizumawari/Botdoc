# トラブルシューティングガイド

このドキュメントは、DynamicExitStrategyImplementationプロジェクトの一般的な問題と解決策を提供します。

## 目次

1. [コンパイルエラー](#コンパイルエラー)
2. [実行時エラー](#実行時エラー)
3. [メモリリーク](#メモリリーク)
4. [パフォーマンス問題](#パフォーマンス問題)
5. [スレッド関連の問題](#スレッド関連の問題)
6. [リソース管理の問題](#リソース管理の問題)
7. [イベント処理の問題](#イベント処理の問題)
8. [データバッファの問題](#データバッファの問題)

## コンパイルエラー

### CS0246: 型または名前空間名が見つかりません

**問題**: `'ScalpingStrategyProject'という名前空間名が見つかりません`

**解決策**: 
- 名前空間の移行が進行中のため、参照を`DynamicExitStrategyImplementation`に更新します。
- 以下のように名前空間を変更します:

```csharp
// 変更前
using ScalpingStrategyProject;

// 変更後
using DynamicExitStrategyImplementation;
```

### CS0246: ファイルの参照エラー

**問題**: `'XXX.cs'が参照できません`

**解決策**:
- プロジェクトファイル（.csproj）に該当ファイルが含まれているか確認します。
- ファイルパスが正しいディレクトリ構造と一致しているか確認します。
- リファクタリングによりファイル名が変更されている可能性があります。最新のファイル名を確認してください。

### CS0122: アクセスレベルエラー

**問題**: `'XXX'は'YYY'内にあるため、アクセスできません`

**解決策**:
- アクセス修飾子（public, private, protected, internal）が適切に設定されているか確認します。
- 継承関係を確認し、protected memberが適切に参照されているか確認します。
- 内部クラスのアクセス修飾子を確認します。

## 実行時エラー

### NullReferenceException

**問題**: `オブジェクト参照がオブジェクト インスタンスに設定されていません`

**解決策**:
- 初期化順序を確認します。特に`Initialize()`メソッドが正しく呼び出されているか確認します。
- ライフサイクルイベントで`null`チェックを追加します。
- `SafeExecute`パターンを使用してヌルチェックを組み込みます：

```csharp
ErrorHandling.SafeExecute(() => {
    if (component != null) {
        component.Process();
    }
}, "コンポーネント処理中にエラーが発生しました");
```

### InvalidOperationException

**問題**: `操作が無効な状態で実行されました`

**解決策**:
- コンポーネントの状態を確認します。特に`IsInitialized`フラグが設定されているか確認します。
- 操作の前提条件を確認します（例：バッファに十分なデータがあるか）。
- 設定パラメータが有効な範囲内にあるか確認します。

```csharp
if (!IsInitialized)
{
    LogMessage("コンポーネントが初期化されていないため、操作を実行できません", LoggingLevel.Error);
    return;
}
```

### ObjectDisposedException

**問題**: `破棄されたオブジェクトに対してメソッドが呼び出されました`

**解決策**:
- `IsDisposed`フラグを確認し、破棄されたオブジェクトへのアクセスを防止します。
- イベントハンドラを正しくアンサブスクライブします。
- DisposableBaseクラスを継承し、標準化された破棄パターンを使用します。

```csharp
public void SomeMethod()
{
    if (IsDisposed)
    {
        throw new ObjectDisposedException(nameof(YourClassName));
    }
    
    // 通常の処理を続行
}
```

## メモリリーク

### イベントハンドラのリーク

**問題**: オブジェクトが破棄された後もイベントハンドラが残り、メモリリークが発生する

**解決策**:
- `Dispose()`メソッド内ですべてのイベントハンドラをアンサブスクライブします。
- `EnhancedEventManager`を使用して、イベント参照を追跡・管理します。
- 弱参照（WeakReference）を使用したイベント購読を検討します。

```csharp
protected override void Dispose(bool disposing)
{
    if (disposing)
    {
        // イベントハンドラをアンサブスクライブ
        if (_eventManager != null)
        {
            _eventManager.UnsubscribeAll();
            _eventManager = null;
        }
        
        // その他のリソースをクリーンアップ
    }
    
    base.Dispose(disposing);
}
```

### バッファリーク

**問題**: CircularBufferが大量のメモリを保持したままになる

**解決策**:
- バッファサイズを適切に設定します（無限に大きくしない）。
- 不要になったバッファを明示的にクリアします。
- ResourceManagerに登録して、破棄を確実にします。

```csharp
// バッファサイズを適切に制限
var buffer = new CircularBuffer<double>(maxSize: 1000);

// 使用後にクリーンアップ
buffer.Clear();
```

## パフォーマンス問題

### 高CPU使用率

**問題**: 処理中にCPU使用率が異常に高くなる

**解決策**:
- ループ内でのオブジェクト生成を最小限に抑えます。
- 繰り返し計算にはキャッシュを使用します。
- データの前処理と計算の最適化を検討します。

```csharp
// 不要なオブジェクト生成を避ける
private readonly List<double> _reuseList = new List<double>(100);

public void Process()
{
    _reuseList.Clear(); // 再利用して新しいインスタンスを作らない
    
    // _reuseListにデータを追加して処理
}
```

### スレッド競合

**問題**: 複数スレッドからのアクセスによりパフォーマンスが低下する

**解決策**:
- 読み取り操作と書き込み操作を分離し、ReaderWriterLockSlimを使用します。
- 短期的なロックを使用し、ロック内での重い処理を避けます。
- ロックの粒度を最適化し、必要な部分だけをロックします。

```csharp
private readonly ReaderWriterLockSlim _lock = new ReaderWriterLockSlim();

public void ReadOperation()
{
    _lock.EnterReadLock();
    try
    {
        // 読み取り操作（複数スレッドが同時に実行可能）
    }
    finally
    {
        _lock.ExitReadLock();
    }
}

public void WriteOperation()
{
    _lock.EnterWriteLock();
    try
    {
        // 書き込み操作（排他的ロックを取得）
    }
    finally
    {
        _lock.ExitWriteLock();
    }
}
```

## スレッド関連の問題

### デッドロック

**問題**: 複数のロックが相互に待機状態になり、デッドロックが発生する

**解決策**:
- ロックの取得順序を一貫させます（常に同じ順序でロックを取得）。
- `LockHelper`を使用してタイムアウトを設定し、デッドロックを検出します。
- ネストしたロックを最小限に抑えます。

```csharp
// ロックの取得順序を一貫させる
private readonly object _lockA = new object();
private readonly object _lockB = new object();

public void SafeLocking()
{
    // 常に_lockAを先に取得、次に_lockB
    lock (_lockA)
    {
        // 処理
        lock (_lockB)
        {
            // ネストした処理
        }
    }
}

// タイムアウト付きロック
public bool TryOperation()
{
    return LockHelper.TryExecuteWithLock(_lock, TimeSpan.FromSeconds(5), () =>
    {
        // 処理（5秒以内に完了しないとタイムアウト）
    });
}
```

### 競合状態

**問題**: 複数スレッドが同時にデータを変更し、予期しない結果が発生する

**解決策**:
- `Interlocked`クラスを使用して原子的操作を実行します。
- 複雑な操作には適切なロックを使用します。
- イミュータブルデータ構造を検討します。

```csharp
// 原子的なカウンター操作
private long _counter;

public void IncrementCounter()
{
    Interlocked.Increment(ref _counter);
}

public long GetCounter()
{
    return Interlocked.Read(ref _counter);
}
```

## リソース管理の問題

### リソースリーク

**問題**: `IDisposable`リソースが正しく解放されない

**解決策**:
- `using`ステートメントを使用して、リソースの自動破棄を確保します。
- `ResourceManager`にリソースを登録して追跡します。
- `DisposableBase`を継承して標準的な破棄パターンを実装します。

```csharp
// usingによる自動破棄
public void ProcessWithFile(string path)
{
    using (var file = new FileStream(path, FileMode.Open))
    {
        // ファイル処理
    } // ここでfileは自動的に破棄される
}

// リソース登録
public void RegisterComponent(IDisposable component)
{
    _resourceManager.RegisterResource(component);
}
```

### 循環参照

**問題**: 相互参照によりガベージコレクションが妨げられる

**解決策**:
- イベントハンドラの登録解除を確実に行います。
- 循環参照を避けるために弱参照（WeakReference）を使用します。
- 依存関係を見直し、単方向の依存関係を優先します。

```csharp
// 弱参照の使用例
private WeakReference<TargetClass> _targetRef;

public void SetTarget(TargetClass target)
{
    _targetRef = new WeakReference<TargetClass>(target);
}

public void UseTarget()
{
    if (_targetRef.TryGetTarget(out TargetClass target))
    {
        target.DoSomething();
    }
}
```

## イベント処理の問題

### イベントハンドラエラー

**問題**: イベントハンドラ内の例外が呼び出し元に伝播する

**解決策**:
- `EnhancedEventManager`を使用して、例外を捕捉・ログ記録します。
- 各ハンドラを個別に実行し、一つの障害が他に影響しないようにします。
- イベント発行時に`SafeExecute`パターンを使用します。

```csharp
public void RaiseEvent(object sender, EventArgs e)
{
    if (_handlers != null)
    {
        foreach (var handler in _handlers.ToArray())
        {
            ErrorHandling.SafeExecute(() => 
            {
                handler(sender, e);
            }, $"イベントハンドラの実行中にエラーが発生しました: {handler.Method.Name}");
        }
    }
}
```

### イベント購読漏れ

**問題**: イベントが発生しても適切に処理されない

**解決策**:
- 初期化時に必要なすべてのイベント購読を確認します。
- 診断ツールを使用してイベント購読状態を監視します。
- イベント購読状態をログに記録します。

```csharp
// 初期化時のイベント購読
public void Initialize()
{
    _manager.DataUpdated += OnDataUpdated;
    _manager.PositionChanged += OnPositionChanged;
    _manager.ErrorOccurred += OnErrorOccurred;
    
    LogMessage($"イベント購読が完了しました: {GetType().Name}", LoggingLevel.Debug);
}
```

## データバッファの問題

### バッファオーバーフロー

**問題**: 固定サイズのバッファに過剰なデータが書き込まれる

**解決策**:
- `CircularBuffer`を使用して自動的に古いデータを上書きします。
- バッファサイズを適切に設定し、必要に応じて動的に調整します。
- データフロー制御メカニズムを実装します。

```csharp
// 循環バッファの使用
private CircularBuffer<MarketData> _dataBuffer = new CircularBuffer<MarketData>(1000);

public void AddData(MarketData data)
{
    _dataBuffer.Add(data); // 自動的に古いデータを上書き
}
```

### バッファ同期問題

**問題**: 複数スレッドがバッファに同時アクセスし、データの整合性が損なわれる

**解決策**:
- バッファ操作にスレッドセーフな実装を使用します。
- `EnhancedBufferManager`を使用して同期アクセスを管理します。
- 読み取り専用操作には同時アクセスを許可し、書き込み時のみロックします。

```csharp
// スレッドセーフなバッファアクセス
private readonly CircularBuffer<double> _buffer = new CircularBuffer<double>(100);
private readonly ReaderWriterLockSlim _bufferLock = new ReaderWriterLockSlim();

public void AddToBuffer(double value)
{
    _bufferLock.EnterWriteLock();
    try
    {
        _buffer.Add(value);
    }
    finally
    {
        _bufferLock.ExitWriteLock();
    }
}

public double[] GetBufferData()
{
    _bufferLock.EnterReadLock();
    try
    {
        return _buffer.ToArray();
    }
    finally
    {
        _bufferLock.ExitReadLock();
    }
}
```

---

問題が解決しない場合は、以下の追加リソースを参照してください：

- [エラーハンドリングパターン](../architecture/error-handling-patterns.md)
- [スレッド安全パターン](../architecture/thread-safety-patterns.md)
- [リソース管理](../architecture/resource-management.md)
- [イベント処理ガイド](../guides/event-handling-guide.md)

また、問題の報告や質問は、プロジェクトの管理者に連絡してください。