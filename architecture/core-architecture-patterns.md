# Core Architecture Patterns and Dependencies

## Core Module Architectural Patterns

The Core module forms the foundational layer of the DynamicExitStrategyImplementation project, providing essential services and patterns used throughout the codebase. This document outlines the key architectural patterns identified.

### 1. Resource Management Pattern

The project implements a centralized resource tracking and disposal mechanism through `ResourceManagement.cs`. This pattern ensures proper cleanup of resources like timers, event subscriptions, and cancellation tokens.

**Key components:**
- `RegisterResource`: Tracks IDisposable resources with component ownership
- `DisposeResource`: Safely disposes individual resources
- `DisposeComponentResources`: Disposes all resources for a specific component
- `SafeDispose`: Thread-safe disposal utility

**Benefits:**
- Prevents resource leaks by tracking all disposable objects
- Provides component-level resource isolation
- Enables safe disposal even during exceptions
- Supports debugging with resource status logging

### 2. Thread Safety Patterns

The codebase implements multiple thread synchronization patterns through `LockHelper.cs` to ensure thread safety across components.

**Key mechanisms:**
- `WithReadLock`, `WithWriteLock`, `WithUpgradeableReadLock`: Extension methods for ReaderWriterLockSlim
- `WithLock`: Extension method for standard lock(object) pattern
- Thread-safe flag checking with Interlocked.Exchange

**Applications:**
- CircularBuffer implementation uses synchronization to protect data access
- ResourceManagement uses reader-writer locks for efficient concurrent access
- Event subscription system uses upgradeable read locks for subscriptions

### 3. Event Management Pattern

The `EnhancedEventManager` implements a sophisticated event subscription tracking system with automatic cleanup.

**Key features:**
- Subscription grouping for logical organization
- Automatic event handler cleanup on disposal
- Health monitoring for event activity
- Subscription status reporting

**Benefits:**
- Prevents event handler memory leaks
- Enables component-level event isolation
- Provides diagnostic information for debugging

### 4. Error Handling Pattern

The `ErrorHandling.cs` class provides standardized error handling patterns throughout the application.

**Key patterns:**
- `SafeExecute`: Error-safe execution wrapper for synchronous operations
- `SafeExecuteAsync`: Error-safe execution wrapper for asynchronous operations
- `RetryAsync`: Built-in retry mechanism with exponential backoff

**Benefits:**
- Consistent error handling across the codebase
- Centralized logging of errors with standardized format
- Improved resilience through retry mechanisms

### 5. Manager Provider Pattern

The `ManagerProvider` class implements a singleton pattern to provide centralized access to core services.

**Key services:**
- EventManager: Handles event subscription tracking
- ResourceManager: Provides resource lifecycle management
- BufferManager: Manages historical data buffers

**Benefits:**
- Single point of access for all core services
- Controlled initialization and disposal sequence
- Centralized status reporting for diagnostics

## Core Dependencies

The Core module has minimal external dependencies, making it highly reusable:

1. **External Dependencies:**
   - TradingPlatform.BusinessLayer: For logging and core trading platform interfaces

2. **Internal Dependencies:**
   - Core.DisposableBase → Core.ResourceManagement
   - Core.Events.EnhancedEventManager → Core.Threading.LockHelper
   - Core.Events.EnhancedEventManager → Core.ResourceManagement (via DisposableBase)
   - Core.Managers.ManagerProvider → Core.Managers.EventManager, ResourceManager, BufferManager
   
3. **Design Pattern Dependencies:**
   - CircularBuffer → LockHelper (for thread safety)
   - All components → ErrorHandling (for standardized error management)
   - All disposable components → ResourceManagement (for resource cleanup)

## Initialization Flow

The typical initialization sequence is:

1. ManagerProvider singleton is created
2. ResourceManager is initialized
3. EventManager is initialized and registered with ResourceManager
4. BufferManager is initialized and registered with ResourceManager
5. Component-specific resources are registered as components are created
6. Event subscriptions are registered with the EventManager

## Disposal Flow

The disposal sequence is:

1. Component's Dispose method is called
2. Component passes control to DisposableBase.Dispose
3. DisposableBase marks the component as disposed using thread-safe flag
4. DisposableBase calls component's OnDispose implementation
5. ResourceManagement disposes all resources registered by the component

## Namespace Migration Note

The codebase appears to be in the midst of a namespace migration from `ScalpingStrategyProject` to `DynamicExitStrategyImplementation`, as evidenced by some classes still using the old namespace.