# Architecture Diagram and Dependencies

## Module Layer Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Trading Platform                          │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Core Module                               │
│                                                                 │
│  ┌───────────────┐  ┌────────────────┐  ┌────────────────────┐  │
│  │ DisposableBase│  │ ErrorHandling  │  │ ResourceManagement │  │
│  └───────────────┘  └────────────────┘  └────────────────────┘  │
│                                                                 │
│  ┌───────────────┐  ┌────────────────┐  ┌────────────────────┐  │
│  │ Threading     │  │ Events         │  │ Managers           │  │
│  └───────────────┘  └────────────────┘  └────────────────────┘  │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────────┐
│                      Data Module                                │
│                                                                │
│  ┌────────────────┐  ┌────────────────┐  ┌─────────────────┐   │
│  │ CircularBuffer │  │ BufferManager  │  │ BufferEntry     │   │
│  └────────────────┘  └────────────────┘  └─────────────────┘   │
└────────────────────────────┬───────────────────────────────────┘
                             │
                  ┌──────────┴───────────┐
                  │                      │
                  ▼                      ▼
┌──────────────────────────┐    ┌──────────────────────────┐
│    Indicators Module      │    │     Fibonacci Module     │
│                          │    │                          │
│  ┌──────────────────┐    │    │  ┌──────────────────┐    │
│  │ DeltaMomentum    │    │    │  │ FibonacciTools   │    │
│  └──────────────────┘    │    │  └──────────────────┘    │
│                          │    │                          │
│  ┌──────────────────┐    │    │  ┌──────────────────┐    │
│  │ VolatilityRegime │    │    │  │ FibonacciLevels  │    │
│  └──────────────────┘    │    │  └──────────────────┘    │
│                          │    │                          │
│  ┌──────────────────┐    │    │  ┌──────────────────┐    │
│  │ HigherTimeframe  │    │    │  │ FibonacciSequence│    │
│  └──────────────────┘    │    │  └──────────────────┘    │
└──────────────┬───────────┘    └─────────────┬────────────┘
               │                              │
               └──────────────┬───────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Trading Strategy Modules                      │
│                                                                 │
│  ┌───────────────────────┐         ┌─────────────────────────┐  │
│  │    EntryLogic Module  │         │   ExitLogic Module      │  │
│  │                       │         │                         │  │
│  │ ┌───────────────────┐ │         │ ┌─────────────────────┐ │  │
│  │ │ MomentumEntrySignal│ │         │ │ BaseExitStrategy   │ │  │
│  │ └───────────────────┘ │         │ └─────────────────────┘ │  │
│  │                       │         │                         │  │
│  │ ┌───────────────────┐ │         │ ┌─────────────────────┐ │  │
│  │ │ IEntrySignal      │ │         │ │ UnifiedExitStrategy │ │  │
│  │ └───────────────────┘ │         │ └─────────────────────┘ │  │
│  └───────────────────────┘         └─────────────────────────┘  │
│                                                                 │
└────────────────────────────────┬────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Position Management                          │
│                                                                 │
│  ┌───────────────────────┐         ┌─────────────────────────┐  │
│  │   OrderManager        │         │   PositionManager       │  │
│  └───────────────────────┘         └─────────────────────────┘  │
│                                                                 │
│  ┌───────────────────────┐         ┌─────────────────────────┐  │
│  │ PositionSubscription  │         │ HierarchicalPositionSizer│ │
│  └───────────────────────┘         └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## Core Dependencies

### Core Module

The Core module is the foundation layer that provides essential services to all other modules:

```
┌─────────────────────────┐
│      DisposableBase     │
└──┬─────────────────────┬┘
   │                     │
   ▼                     ▼
┌─────────────┐   ┌────────────────┐
│ErrorHandling│   │ResourceManagement│
└─────────────┘   └────────────────┘
        │               │
        └───────┬───────┘
                ▼
┌──────────────────────────┐
│ManagerProvider           │
├──────────────────────────┤
│ - EventManager           │
│ - ResourceManager        │
│ - BufferManager          │
└──────────────────────────┘
```

### Data Module

The Data module provides data structures and buffer management capabilities:

```
┌─────────────────────┐
│   CircularBuffer    │
└─────────┬───────────┘
          │
┌─────────▼───────────┐       ┌───────────────────┐
│   BufferManager     │───────▶│  BufferEntry     │
└─────────────────────┘       └───────────────────┘
          │
┌─────────▼───────────┐
│EnhancedBufferManager│
└─────────────────────┘
```

### Trading Strategy Dependencies

The trading strategies build on the core indicators and utilities:

```
┌─────────────────────┐     ┌─────────────────────┐
│   DeltaMomentum     │     │   FibonacciTools    │
└─────────┬───────────┘     └─────────┬───────────┘
          │                           │
          │     ┌─────────────────────┐
          │     │   VolatilityRegime  │
          │     └─────────┬───────────┘
          │               │
┌─────────▼───────────────▼───────────┐
│        EntryLogic                   │
│  ┌─────────────────────────────┐    │
│  │    MomentumEntrySignal      │    │
└──┴─────────────────────────────┴────┘
          │
          ▼
┌─────────────────────────────────────┐
│          ExitLogic                  │
│  ┌─────────────────────────────┐    │
│  │      BaseExitStrategy       │    │
│  └─────────────────────────────┘    │
│               │                     │
│  ┌────────────▼────────────────┐    │
│  │     UnifiedExitStrategy     │    │
└──┴─────────────────────────────┴────┘
```

## Detailed Integration Flow

### Entry Signal Processing

```
┌─────────────┐     ┌───────────────┐     ┌──────────────┐
│ Symbol.New* │────▶│ EventManager  │────▶│ EntrySignal  │
└─────────────┘     └───────────────┘     └──────┬───────┘
                                                 │
                                                 ▼
┌─────────────┐     ┌───────────────┐     ┌──────────────┐
│ PositionMgr │◀────│ OrderManager  │◀────│ EntrySignal  │
└─────────────┘     └───────────────┘     │ Detection    │
                                          └──────────────┘
```

### Exit Signal Processing

```
┌─────────────┐     ┌───────────────┐     ┌──────────────┐
│ Symbol.New* │────▶│ EventManager  │────▶│ ExitStrategy │
└─────────────┘     └───────────────┘     └──────┬───────┘
      │                                          │
      │                                          ▼
      │                                   ┌──────────────┐
      │                                   │ ExitSignal   │
      │                                   │ Detection    │
      │                                   └──────┬───────┘
      ▼                                          │
┌─────────────┐     ┌───────────────┐           │
│ Position    │────▶│ PositionMgr   │◀──────────┘
└─────────────┘     └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │ OrderManager  │
                    └───────────────┘
```

## Namespaces and Classes

```
DynamicExitStrategyImplementation
│
├── Core
│   ├── BaseComponents
│   │   ├── ComponentBase.cs
│   │   ├── ExitStrategyBase.cs
│   │   ├── SignalComponentBase.cs
│   │   └── TradingComponentBase.cs
│   ├── Buffers
│   ├── DisposableBase.cs
│   ├── EnhancedCoreExtensions.cs
│   ├── ErrorHandling.cs
│   ├── Events
│   │   ├── EnhancedEventManager.cs
│   │   ├── EventHandlerExtensions.cs
│   │   └── EventSubscription.cs
│   ├── Managers
│   │   ├── BufferManager.cs
│   │   ├── EnhancedManagerProvider.cs
│   │   ├── EventManager.cs
│   │   └── ResourceManager.cs
│   ├── ResourceManagement.cs
│   └── Threading
│       └── LockHelper.cs
│
├── Data
│   ├── CircularBuffer.Core.cs
│   ├── CircularBuffer.Enumeration.cs
│   ├── CircularBuffer.Statistics.cs
│   ├── CircularBuffer.cs
│   └── ListCircularBuffer.cs
│
├── EntryLogic
│   ├── MomentumEntrySignal.Analysis.cs
│   ├── MomentumEntrySignal.Core.cs
│   ├── MomentumEntrySignal.Events.cs
│   ├── MomentumEntrySignal.Signal.cs
│   └── MomentumEntrySignal.cs
│
├── ExitLogic
│   ├── BaseExitStrategy.cs
│   ├── DynamicExitStrategy.Core.cs
│   ├── DynamicExitStrategy.Events.cs
│   ├── DynamicExitStrategy.Exits.cs
│   ├── DynamicExitStrategy.Fibonacci.cs
│   ├── DynamicExitStrategy.Monitoring.cs
│   ├── DynamicExitStrategy.cs
│   ├── DynamicExitStrategyAdapter.cs
│   ├── EnhancedUnifiedExitStrategy.cs
│   ├── SimpleMomentumExitStrategy.Core.cs
│   ├── SimpleMomentumExitStrategy.Events.cs
│   ├── SimpleMomentumExitStrategy.Exits.cs
│   ├── SimpleMomentumExitStrategy.Fibonacci.cs
│   ├── SimpleMomentumExitStrategy.Monitoring.cs
│   ├── SimpleMomentumExitStrategy.cs
│   ├── SimpleMomentumExitStrategyAdapter.cs
│   └── UnifiedExitStrategy.cs
│
├── Extensions
│   ├── DoubleExtensions.cs
│   ├── ListExtensions.cs
│   ├── MathExtensions.cs
│   ├── OrderBookImbalanceExtensions.cs
│   ├── OrderManagerExtensions.cs
│   ├── PositionManagerExtensions.cs
│   └── TechnicalAnalysis.cs
│
├── Fibonacci
│   ├── FibonacciLevels.cs
│   ├── FibonacciSequence.cs
│   └── FibonacciTools.cs
│
├── Indicators
│   ├── DeltaMomentum.Core.cs
│   ├── DeltaMomentum.Events.cs
│   ├── DeltaMomentum.Statistics.cs
│   ├── DeltaMomentum.cs
│   ├── HigherTimeframeAnalyzer.cs
│   └── VolatilityRegime.cs
│
├── Interfaces
│   ├── IEntrySignal.cs
│   ├── IExitSignal.cs
│   └── IPositionManager.cs
│
├── Managers
│   ├── OrderManager.Core.cs
│   ├── OrderManager.Events.cs
│   ├── OrderManager.Execution.cs
│   ├── OrderManager.RiskManagement.cs
│   ├── OrderManager.Tracking.cs
│   ├── OrderManager.cs
│   ├── PositionManager.Core.cs
│   ├── PositionManager.Events.cs
│   ├── PositionManager.Tracking.cs
│   └── PositionManager.cs
│
├── OrderBook
│   ├── OrderBookImbalance.cs
│   └── SpreadAwareExecution.cs
│
├── Positioning
│   └── HierarchicalPositionSizer.cs
│
└── Utilities
    ├── ErrorHandling.cs
    ├── SymbolPositionSubscriptionManager.cs
    └── ValidationUtils.cs
```