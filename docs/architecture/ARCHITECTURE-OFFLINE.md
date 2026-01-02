# TanStack DB Offline Transactions Architecture

This document details the offline transactions system in `@tanstack/offline-transactions`, which provides persistent transaction queuing, multi-tab coordination, retry policies, and graceful degradation for offline-first applications.

## System Overview

The offline transactions package enables applications to:
- **Queue transactions** when offline and execute them when online
- **Coordinate across tabs** using leader election to prevent duplicate processing
- **Retry failed transactions** with configurable backoff strategies
- **Persist transactions** to IndexedDB or localStorage with automatic fallback

```mermaid
flowchart TD
    subgraph OfflineSystem["Offline Transactions System"]
        Executor["OfflineExecutor"]:::main
        
        subgraph Storage["Persistence"]
            Outbox["OutboxManager"]:::storage
            IDB["IndexedDBAdapter"]:::adapter
            LS["LocalStorageAdapter"]:::adapter
        end

        subgraph Coordination["Multi-Tab Coordination"]
            Leader["LeaderElection"]:::coord
            WebLocks["WebLocksLeader"]:::coord
            Broadcast["BroadcastChannelLeader"]:::coord
        end

        subgraph Execution["Transaction Execution"]
            TxExecutor["TransactionExecutor"]:::exec
            KeyScheduler["KeyScheduler"]:::exec
            Retry["RetryPolicy"]:::exec
        end

        subgraph Connectivity["Network Detection"]
            Online["OnlineDetector"]:::network
        end
    end

    Executor --> Outbox
    Executor --> Leader
    Executor --> TxExecutor
    Executor --> Online

    Outbox --> IDB
    Outbox --> LS

    Leader --> WebLocks
    Leader --> Broadcast

    TxExecutor --> KeyScheduler
    TxExecutor --> Retry

    %% Click events
    click Executor "https://github.com/TanStack/db/blob/main/packages/offline-transactions/src/OfflineExecutor.ts"
    click Outbox "https://github.com/TanStack/db/blob/main/packages/offline-transactions/src/outbox/OutboxManager.ts"
    click TxExecutor "https://github.com/TanStack/db/blob/main/packages/offline-transactions/src/executor/TransactionExecutor.ts"
    click Leader "https://github.com/TanStack/db/blob/main/packages/offline-transactions/src/coordination/LeaderElection.ts"
    click Retry "https://github.com/TanStack/db/blob/main/packages/offline-transactions/src/retry/RetryPolicy.ts"

    %% Styles
    classDef main fill:#4f46e5,stroke:#3730a3,color:#fff
    classDef storage fill:#059669,stroke:#047857,color:#fff
    classDef adapter fill:#7c3aed,stroke:#6d28d9,color:#fff
    classDef coord fill:#0891b2,stroke:#0e7490,color:#fff
    classDef exec fill:#dc2626,stroke:#b91c1c,color:#fff
    classDef network fill:#d97706,stroke:#b45309,color:#fff
```

## OfflineExecutor: Main Coordinator

```mermaid
flowchart TD
    subgraph OfflineExecutor["OfflineExecutor"]
        Config["OfflineConfig"]:::config
        Mode["mode: OfflineMode"]:::prop
        Diagnostic["storageDiagnostic"]:::prop
        
        subgraph Methods["Public Methods"]
            CreateTx["createTransaction()"]:::method
            CreateAction["createOfflineAction()"]:::method
            Execute["executeNow()"]:::method
            GetPending["getPendingTransactions()"]:::method
            Cleanup["cleanup()"]:::method
        end
    end

    subgraph Dependencies["Internal Components"]
        Storage["StorageAdapter"]:::dep
        Outbox["OutboxManager"]:::dep
        Scheduler["KeyScheduler"]:::dep
        Executor["TransactionExecutor"]:::dep
        Leader["LeaderElection"]:::dep
        Online["OnlineDetector"]:::dep
    end

    Config --> OfflineExecutor
    OfflineExecutor --> Storage
    OfflineExecutor --> Outbox
    OfflineExecutor --> Scheduler
    OfflineExecutor --> Executor
    OfflineExecutor --> Leader
    OfflineExecutor --> Online

    %% Styles
    classDef config fill:#4f46e5,stroke:#3730a3,color:#fff
    classDef prop fill:#7c3aed,stroke:#6d28d9,color:#fff
    classDef method fill:#059669,stroke:#047857,color:#fff
    classDef dep fill:#0891b2,stroke:#0e7490,color:#fff
```

## Offline Transaction Lifecycle

```mermaid
sequenceDiagram
    participant App as Application
    participant OE as OfflineExecutor
    participant Outbox as OutboxManager
    participant Storage as StorageAdapter
    participant Leader as LeaderElection
    participant Exec as TransactionExecutor
    participant Server as Backend API

    Note over App,Server: Transaction Creation

    App->>OE: createTransaction(config)
    OE->>OE: Create OfflineTransaction
    OE->>Outbox: add(transaction)
    Outbox->>Storage: set(serialized)
    OE-->>App: Transaction handle

    Note over App,Server: Optimistic Update

    App->>App: transaction.mutate()
    App->>App: UI updates instantly

    Note over App,Server: Commit Attempt

    App->>OE: transaction.commit()
    OE->>Leader: isLeader()?
    
    alt Is Leader Tab
        Leader-->>OE: true
        OE->>Exec: execute(transaction)
        
        alt Online
            Exec->>Server: mutationFn()
            
            alt Success
                Server-->>Exec: 200 OK
                Exec->>Outbox: remove(tx.id)
                Exec-->>App: Resolve
            else Failure - Retriable
                Server-->>Exec: Error
                Exec->>Exec: Apply retry policy
                Exec->>Outbox: update(tx, retryInfo)
                Note over Exec: Retry later
            else Failure - Non-retriable
                Server-->>Exec: Non-retriable error
                Exec->>Outbox: remove(tx.id)
                Exec-->>App: Reject with error
            end
            
        else Offline
            OE-->>App: Transaction queued
            Note over OE: Will retry when online
        end
        
    else Not Leader Tab
        Leader-->>OE: false
        OE->>Outbox: Queue transaction
        Note over OE: Leader tab will process
    end
```

## Storage System

```mermaid
flowchart TD
    subgraph StorageProbe["Storage Availability Probe"]
        Start["createStorage()"]:::start
        
        ProbeIDB["Probe IndexedDB"]:::probe
        ProbeLStor["Probe localStorage"]:::probe
        
        IDBAvail{{"IndexedDB\navailable?"}}:::decision
        LSAvail{{"localStorage\navailable?"}}:::decision
        
        UseIDB["Use IndexedDBAdapter"]:::result
        UseLS["Use LocalStorageAdapter"]:::result
        OnlineOnly["Online-only mode"]:::result
    end

    Start --> ProbeIDB
    ProbeIDB --> IDBAvail
    
    IDBAvail -->|"Yes"| UseIDB
    IDBAvail -->|"No"| ProbeLStor
    
    ProbeLStor --> LSAvail
    LSAvail -->|"Yes"| UseLS
    LSAvail -->|"No"| OnlineOnly

    %% Click events
    click UseIDB "https://github.com/TanStack/db/blob/main/packages/offline-transactions/src/storage/IndexedDBAdapter.ts"
    click UseLS "https://github.com/TanStack/db/blob/main/packages/offline-transactions/src/storage/LocalStorageAdapter.ts"

    %% Styles
    classDef start fill:#4f46e5,stroke:#3730a3,color:#fff
    classDef probe fill:#7c3aed,stroke:#6d28d9,color:#fff
    classDef decision fill:#d97706,stroke:#b45309,color:#fff
    classDef result fill:#059669,stroke:#047857,color:#fff
```

### Storage Adapter Interface

```typescript
interface StorageAdapter {
  get(key: string): Promise<string | null>
  set(key: string, value: string): Promise<void>
  delete(key: string): Promise<void>
  keys(): Promise<Array<string>>
}
```

### Storage Diagnostics

| Code | Mode | Description |
|------|------|-------------|
| `STORAGE_AVAILABLE` | offline | IndexedDB or localStorage available |
| `INDEXEDDB_UNAVAILABLE` | offline | Using localStorage fallback |
| `STORAGE_BLOCKED` | online-only | Private browsing / security restrictions |
| `QUOTA_EXCEEDED` | online-only | Storage quota exceeded |
| `UNKNOWN_ERROR` | online-only | Unknown storage error |

## Leader Election

Multi-tab coordination ensures only one tab processes offline transactions to prevent duplicates:

```mermaid
flowchart TD
    subgraph LeaderElection["Leader Election Strategies"]
        Check["createLeaderElection()"]:::check
        
        WebLocks{{"Web Locks API\navailable?"}}:::decision
        Broadcast{{"BroadcastChannel\navailable?"}}:::decision
        
        UseWebLocks["WebLocksLeader"]:::strategy
        UseBroadcast["BroadcastChannelLeader"]:::strategy
        UseFallback["Always leader (single tab)"]:::strategy
    end

    Check --> WebLocks
    WebLocks -->|"Yes"| UseWebLocks
    WebLocks -->|"No"| Broadcast
    
    Broadcast -->|"Yes"| UseBroadcast
    Broadcast -->|"No"| UseFallback

    %% Click events
    click UseWebLocks "https://github.com/TanStack/db/blob/main/packages/offline-transactions/src/coordination/WebLocksLeader.ts"
    click UseBroadcast "https://github.com/TanStack/db/blob/main/packages/offline-transactions/src/coordination/BroadcastChannelLeader.ts"

    %% Styles
    classDef check fill:#4f46e5,stroke:#3730a3,color:#fff
    classDef decision fill:#d97706,stroke:#b45309,color:#fff
    classDef strategy fill:#059669,stroke:#047857,color:#fff
```

### Leader Election Sequence

```mermaid
sequenceDiagram
    participant Tab1 as Tab 1
    participant Tab2 as Tab 2
    participant Tab3 as Tab 3
    participant Lock as Web Locks API

    Note over Tab1,Lock: Leader Election

    Tab1->>Lock: requestLeadership()
    Lock-->>Tab1: Granted (leader)
    
    Tab2->>Lock: requestLeadership()
    Lock-->>Tab2: Queued (follower)
    
    Tab3->>Lock: requestLeadership()
    Lock-->>Tab3: Queued (follower)

    Note over Tab1,Lock: Tab1 closes

    Tab1->>Lock: releaseLeadership()
    Lock-->>Tab2: Granted (new leader)
    Tab2->>Tab2: onLeadershipChange(true)
    Tab2->>Tab2: Start processing queue
```

## Retry Policy

```mermaid
flowchart TD
    subgraph RetryPolicy["Retry Policy"]
        Config["RetryPolicyConfig"]:::config
        Calculator["BackoffCalculator"]:::calc
        NonRetriable["NonRetriableError"]:::error
    end

    subgraph Settings["Configuration"]
        MaxAttempts["maxAttempts: 3"]:::setting
        BaseDelay["baseDelay: 1000ms"]:::setting
        MaxDelay["maxDelay: 30000ms"]:::setting
        JitterFactor["jitterFactor: 0.1"]:::setting
    end

    subgraph Backoff["Exponential Backoff"]
        Attempt1["Attempt 1: ~1000ms"]:::attempt
        Attempt2["Attempt 2: ~2000ms"]:::attempt
        Attempt3["Attempt 3: ~4000ms"]:::attempt
        GiveUp["Max attempts exceeded"]:::giveup
    end

    Config --> Calculator
    Calculator --> Attempt1
    Attempt1 -->|"fail"| Attempt2
    Attempt2 -->|"fail"| Attempt3
    Attempt3 -->|"fail"| GiveUp

    NonRetriable --> GiveUp

    %% Click events
    click Calculator "https://github.com/TanStack/db/blob/main/packages/offline-transactions/src/retry/BackoffCalculator.ts"
    click NonRetriable "https://github.com/TanStack/db/blob/main/packages/offline-transactions/src/retry/NonRetriableError.ts"

    %% Styles
    classDef config fill:#4f46e5,stroke:#3730a3,color:#fff
    classDef calc fill:#059669,stroke:#047857,color:#fff
    classDef error fill:#dc2626,stroke:#b91c1c,color:#fff
    classDef setting fill:#7c3aed,stroke:#6d28d9,color:#fff
    classDef attempt fill:#0891b2,stroke:#0e7490,color:#fff
    classDef giveup fill:#6b7280,stroke:#4b5563,color:#fff
```

### Backoff Formula

```
delay = min(baseDelay * 2^attempt, maxDelay) * (1 + random * jitterFactor)
```

## Key Scheduler

The KeyScheduler ensures transactions affecting the same keys are processed in order:

```mermaid
flowchart TD
    subgraph KeyScheduler["KeyScheduler"]
        Queue["Transaction Queue"]:::queue
        KeyMap["Key → Transaction Map"]:::map
        
        Schedule["schedule(keys, tx)"]:::method
        GetNext["getNextExecutable()"]:::method
        Complete["markComplete(tx)"]:::method
    end

    subgraph Ordering["Key-based Ordering"]
        TX1["TX1: keys=[A, B]"]:::tx
        TX2["TX2: keys=[B, C]"]:::tx
        TX3["TX3: keys=[D]"]:::tx
        
        Parallel["TX1, TX3 can run in parallel"]:::info
        Serial["TX2 waits for TX1"]:::info
    end

    TX1 --> Parallel
    TX3 --> Parallel
    TX2 --> Serial

    %% Click events
    click KeyScheduler "https://github.com/TanStack/db/blob/main/packages/offline-transactions/src/executor/KeyScheduler.ts"

    %% Styles
    classDef queue fill:#4f46e5,stroke:#3730a3,color:#fff
    classDef map fill:#7c3aed,stroke:#6d28d9,color:#fff
    classDef method fill:#059669,stroke:#047857,color:#fff
    classDef tx fill:#0891b2,stroke:#0e7490,color:#fff
    classDef info fill:#6b7280,stroke:#4b5563,color:#fff
```

## API: OfflineTransaction and OfflineAction

```mermaid
flowchart LR
    subgraph OfflineTransaction["OfflineTransaction"]
        TxMutate["mutate(() => ...)"]:::method
        TxCommit["commit()"]:::method
        TxRollback["rollback()"]:::method
    end

    subgraph OfflineAction["OfflineAction"]
        ActionExecute["execute(input)"]:::method
        ActionPending["pendingTransactions"]:::prop
    end

    subgraph Difference["Key Differences"]
        TxDiff["Manual transaction control"]:::diff
        ActionDiff["Automatic batching"]:::diff
    end

    OfflineTransaction --> TxDiff
    OfflineAction --> ActionDiff

    %% Click events
    click OfflineTransaction "https://github.com/TanStack/db/blob/main/packages/offline-transactions/src/api/OfflineTransaction.ts"
    click OfflineAction "https://github.com/TanStack/db/blob/main/packages/offline-transactions/src/api/OfflineAction.ts"

    %% Styles
    classDef method fill:#059669,stroke:#047857,color:#fff
    classDef prop fill:#7c3aed,stroke:#6d28d9,color:#fff
    classDef diff fill:#0891b2,stroke:#0e7490,color:#fff
```

## Full Offline Flow

```mermaid
sequenceDiagram
    participant User as User
    participant App as Application
    participant OE as OfflineExecutor
    participant Storage as Storage
    participant Network as Network

    Note over User,Network: User goes offline

    Network-->>OE: offline event
    OE->>OE: mode = offline

    User->>App: Create transaction
    App->>OE: createTransaction()
    OE->>Storage: Persist to outbox
    App->>App: Apply optimistic update
    App-->>User: Instant UI feedback

    User->>App: Create more transactions
    App->>OE: createTransaction()
    OE->>Storage: Add to queue

    Note over User,Network: User comes back online

    Network-->>OE: online event
    OE->>OE: mode = offline (still processing)
    
    OE->>OE: isLeader()?
    
    alt Is Leader
        loop For each pending transaction
            OE->>OE: Execute transaction
            OE->>Storage: Remove on success
        end
        OE->>OE: mode = offline (queue empty)
    end

    Note over User,Network: All synced
    App-->>User: Transactions confirmed
```

## File Structure

```
packages/offline-transactions/src/
├── index.ts                    # Public exports
├── OfflineExecutor.ts          # Main coordinator class
├── types.ts                    # Type definitions
│
├── api/
│   ├── OfflineTransaction.ts   # Transaction API
│   └── OfflineAction.ts        # Action API (auto-batching)
│
├── storage/
│   ├── StorageAdapter.ts       # Storage interface
│   ├── IndexedDBAdapter.ts     # IndexedDB implementation
│   └── LocalStorageAdapter.ts  # localStorage fallback
│
├── outbox/
│   ├── OutboxManager.ts        # Transaction queue management
│   └── TransactionSerializer.ts # Serialization/deserialization
│
├── executor/
│   ├── TransactionExecutor.ts  # Transaction execution engine
│   └── KeyScheduler.ts         # Key-based ordering
│
├── coordination/
│   ├── LeaderElection.ts       # Base class
│   ├── WebLocksLeader.ts       # Web Locks implementation
│   └── BroadcastChannelLeader.ts # BroadcastChannel fallback
│
├── connectivity/
│   └── OnlineDetector.ts       # Network status detection
│
├── retry/
│   ├── RetryPolicy.ts          # Retry configuration
│   ├── BackoffCalculator.ts    # Exponential backoff
│   └── NonRetriableError.ts    # Non-retriable error marker
│
└── telemetry/
    └── tracer.ts               # OpenTelemetry integration
```

## Effect-TS Porting Guide

### OfflineExecutor → Effect Layer

```typescript
// Effect-TS offline executor layer
interface OfflineExecutor {
  readonly createTransaction: (config: TransactionConfig) => Effect.Effect<Transaction>
  readonly getPendingTransactions: Effect.Effect<Array<OfflineTransaction>>
  readonly executeNow: Effect.Effect<void>
  readonly cleanup: Effect.Effect<void>
}

const OfflineExecutorLive = Layer.effect(
  OfflineExecutor,
  Effect.gen(function* () {
    const storage = yield* StorageAdapter
    const outbox = yield* OutboxManager
    const leader = yield* LeaderElection
    
    // ... implementation
  })
)
```

### OutboxManager → Queue + KeyValueStore

```typescript
// Effect-TS outbox manager
import { KeyValueStore } from "@effect/platform"

const OutboxManagerLive = Layer.effect(
  OutboxManager,
  Effect.gen(function* () {
    const store = yield* KeyValueStore.KeyValueStore
    
    const add = (tx: OfflineTransaction) => Effect.gen(function* () {
      const key = `tx:${tx.id}`
      const serialized = yield* serialize(tx)
      yield* store.set(key, serialized)
    })
    
    const getAll = Effect.gen(function* () {
      const keys = yield* store.keys
      const txKeys = keys.filter(k => k.startsWith("tx:"))
      return yield* Effect.forEach(txKeys, k => store.get(k).pipe(
        Effect.map(Option.map(deserialize))
      ))
    })
    
    return { add, getAll, /* ... */ }
  })
).pipe(Layer.provide(KeyValueStore.layerIndexedDB))
```

### LeaderElection → Semaphore

```typescript
// Effect-TS leader election
const LeaderElectionLive = Layer.effect(
  LeaderElection,
  Effect.gen(function* () {
    const semaphore = yield* Semaphore.make(1)
    const isLeaderRef = yield* Ref.make(false)
    const listenersRef = yield* Ref.make<Array<(isLeader: boolean) => void>>([])
    
    const requestLeadership = Effect.gen(function* () {
      const permit = yield* Semaphore.withPermits(semaphore, 1)(Effect.succeed(true))
      yield* Ref.set(isLeaderRef, true)
      yield* notifyListeners(true)
      return permit
    })
    
    const releaseLeadership = Effect.gen(function* () {
      yield* Ref.set(isLeaderRef, false)
      yield* notifyListeners(false)
    })
    
    return { requestLeadership, releaseLeadership, isLeader: Ref.get(isLeaderRef) }
  })
)
```

### RetryPolicy → Schedule

```typescript
// Effect-TS retry policy with Schedule
const createRetrySchedule = (config: RetryConfig) =>
  Schedule.exponential(Duration.millis(config.baseDelay), 2).pipe(
    Schedule.jittered,
    Schedule.either(Schedule.recurs(config.maxAttempts)),
    Schedule.whileOutput(([_, n]) => n < config.maxAttempts)
  )

const executeWithRetry = <A, E>(
  effect: Effect.Effect<A, E>,
  policy: RetryConfig
) => effect.pipe(
  Effect.retry(createRetrySchedule(policy)),
  Effect.catchAll(e => 
    isNonRetriable(e) 
      ? Effect.fail(e) 
      : Effect.retry(effect, createRetrySchedule(policy))
  )
)
```

### KeyScheduler → Effect Queue

```typescript
// Effect-TS key scheduler
const KeySchedulerLive = Layer.effect(
  KeyScheduler,
  Effect.gen(function* () {
    const keyLocks = yield* Ref.make<HashMap<string, Deferred<void>>>(HashMap.empty())
    
    const schedule = (keys: Array<string>, tx: Transaction) => Effect.gen(function* () {
      // Acquire locks for all keys
      const locks = yield* Effect.forEach(keys, key => acquireKeyLock(keyLocks, key))
      
      // Execute transaction
      yield* tx.execute
      
      // Release locks
      yield* Effect.forEach(locks, releaseLock)
    })
    
    return { schedule }
  })
)
```

## Related Documentation

- [ARCHITECTURE-OVERVIEW.md](./ARCHITECTURE-OVERVIEW.md) - High-level overview
- [ARCHITECTURE-MUTATIONS.md](./ARCHITECTURE-MUTATIONS.md) - Transaction system
- [ARCHITECTURE-COLLECTIONS.md](./ARCHITECTURE-COLLECTIONS.md) - Collection types

