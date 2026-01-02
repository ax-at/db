# TanStack DB Mutations Architecture

This document details the mutation and transaction system in TanStack DB, including the transaction lifecycle, optimistic updates, and paced mutation strategies.

## Mutation System Overview

TanStack DB uses a transaction-based mutation system that provides:
- **Optimistic updates**: Instant UI feedback before server confirmation
- **Automatic rollback**: Reverts changes if the server rejects them
- **Mutation merging**: Combines multiple mutations within a transaction
- **Paced mutations**: Controlled timing via debounce/throttle/queue strategies

```mermaid
flowchart TD
    subgraph MutationSystem["Mutation System"]
        Transaction["Transaction"]:::core
        Mutations["mutations: Array<PendingMutation>"]:::data
        MutationFn["mutationFn"]:::handler
        Handlers["onInsert / onUpdate / onDelete"]:::handler
    end

    subgraph States["Transaction States"]
        Pending["pending"]:::state
        Persisting["persisting"]:::state
        Completed["completed"]:::state
        Failed["failed"]:::state
        Cancelled["cancelled"]:::state
    end

    subgraph Pacing["Paced Mutations"]
        Debounce["debounceStrategy"]:::strategy
        Throttle["throttleStrategy"]:::strategy
        Queue["queueStrategy"]:::strategy
    end

    Transaction --> Mutations
    Transaction --> MutationFn
    MutationFn --> Handlers

    Pending --> Persisting
    Persisting --> Completed
    Persisting --> Failed
    Pending --> Cancelled

    Debounce --> Transaction
    Throttle --> Transaction
    Queue --> Transaction

    %% Click events
    click Transaction "https://github.com/TanStack/db/blob/main/packages/db/src/transactions.ts"
    click Debounce "https://github.com/TanStack/db/blob/main/packages/db/src/strategies/debounceStrategy.ts"
    click Throttle "https://github.com/TanStack/db/blob/main/packages/db/src/strategies/throttleStrategy.ts"
    click Queue "https://github.com/TanStack/db/blob/main/packages/db/src/strategies/queueStrategy.ts"

    %% Styles
    classDef core fill:#4f46e5,stroke:#3730a3,color:#fff
    classDef data fill:#7c3aed,stroke:#6d28d9,color:#fff
    classDef handler fill:#059669,stroke:#047857,color:#fff
    classDef state fill:#0891b2,stroke:#0e7490,color:#fff
    classDef strategy fill:#d97706,stroke:#b45309,color:#fff
```

## Transaction Lifecycle

### State Machine

```mermaid
stateDiagram-v2
    [*] --> pending: createTransaction()
    
    pending --> pending: mutate()
    pending --> persisting: commit()
    pending --> cancelled: rollback() / abandon
    
    persisting --> completed: mutationFn success
    persisting --> failed: mutationFn error
    
    completed --> [*]
    failed --> [*]
    cancelled --> [*]

    note right of pending
        Optimistic changes applied
        Can add more mutations
    end note
    
    note right of persisting
        Executing mutationFn
        Cannot add mutations
    end note
    
    note right of completed
        Server confirmed
        Synced state updated
    end note
    
    note right of failed
        Optimistic changes
        rolled back
    end note
```

### Transaction Lifecycle Sequence

```mermaid
sequenceDiagram
    participant UI as UI Component
    participant Tx as Transaction
    participant Coll as Collection
    participant State as CollectionState
    participant Handler as Mutation Handler
    participant Server as Backend Server

    Note over UI,Server: Transaction Creation and Mutations

    UI->>Tx: createTransaction({ mutationFn })
    Tx-->>UI: transaction (state: pending)
    
    UI->>Tx: tx.mutate(() => ...)
    Tx->>Tx: registerTransaction()
    
    activate Tx
    UI->>Coll: collection.insert(data)
    Coll->>Tx: Add PendingMutation
    Coll->>State: Add to optimisticUpserts
    State-->>UI: Instant UI update
    deactivate Tx
    
    Tx->>Tx: unregisterTransaction()
    
    Note over UI,Server: Commit and Persistence

    UI->>Tx: tx.commit()
    Tx->>Tx: state = persisting
    Tx->>Handler: onInsert({ transaction })
    Handler->>Server: API request
    
    alt Success
        Server-->>Handler: 200 OK
        Handler-->>Tx: Resolve
        Tx->>Tx: state = completed
        Tx->>State: Clear optimistic state
        Note over State: Server sync brings confirmed data
        Tx-->>UI: isPersisted.promise resolves
    else Failure
        Server-->>Handler: Error
        Handler-->>Tx: Reject
        Tx->>Tx: state = failed
        Tx->>State: Rollback optimistic changes
        Tx-->>UI: isPersisted.promise rejects
    end
```

## PendingMutation Structure

```mermaid
flowchart TD
    subgraph PendingMutation["PendingMutation<T>"]
        Type["type: 'insert' | 'update' | 'delete'"]:::field
        Key["key: TKey"]:::field
        GlobalKey["globalKey: string"]:::field
        Original["original: T"]:::field
        Modified["modified: T"]:::field
        Changes["changes: Partial<T>"]:::field
        Optimistic["optimistic: boolean"]:::field
        Collection["collection: Collection"]:::field
        Metadata["metadata?: unknown"]:::field
        SyncMetadata["syncMetadata?: Record"]:::field
        MutationId["mutationId: string"]:::field
        CreatedAt["createdAt: Date"]:::field
        UpdatedAt["updatedAt: Date"]:::field
    end

    %% Styles
    classDef field fill:#059669,stroke:#047857,color:#fff
```

## Mutation Merging

When multiple mutations target the same key within a transaction, they are merged:

```mermaid
flowchart TD
    subgraph MergingRules["Mutation Merging Truth Table"]
        R1["insert + update → insert (merged)"]:::rule
        R2["insert + delete → null (cancel both)"]:::rule
        R3["update + delete → delete"]:::rule
        R4["update + update → update (merged changes)"]:::rule
        R5["delete + delete → delete (latest)"]:::rule
        R6["insert + insert → insert (latest)"]:::rule
    end

    subgraph Example["Example: insert then update"]
        M1["insert({ id: 1, name: 'Alice' })"]:::insert
        M2["update(1, d => d.name = 'Bob')"]:::update
        Result["Result: insert({ id: 1, name: 'Bob' })"]:::result
    end

    M1 --> M2
    M2 --> Result

    %% Styles
    classDef rule fill:#4f46e5,stroke:#3730a3,color:#fff
    classDef insert fill:#059669,stroke:#047857,color:#fff
    classDef update fill:#0891b2,stroke:#0e7490,color:#fff
    classDef result fill:#d97706,stroke:#b45309,color:#fff
```

## Paced Mutations

Paced mutations provide controlled timing for mutation persistence:

```mermaid
flowchart TD
    subgraph PacedMutations["createPacedMutations()"]
        Config["PacedMutationsConfig"]:::config
        OnMutate["onMutate(variables)"]:::handler
        MutationFn["mutationFn({ transaction })"]:::handler
        Strategy["strategy: Strategy"]:::strategy
    end

    subgraph Strategies["Timing Strategies"]
        Debounce["debounceStrategy({ wait: 500 })"]:::debounce
        Throttle["throttleStrategy({ wait: 200 })"]:::throttle
        Queue["queueStrategy({ wait: 100 })"]:::queue
    end

    subgraph Behavior["Behavior"]
        DebounceBehavior["Waits for pause in input"]:::behavior
        ThrottleBehavior["Max 1 call per interval"]:::behavior
        QueueBehavior["Sequential processing"]:::behavior
    end

    Config --> OnMutate
    Config --> MutationFn
    Config --> Strategy

    Strategy --> Debounce
    Strategy --> Throttle
    Strategy --> Queue

    Debounce --> DebounceBehavior
    Throttle --> ThrottleBehavior
    Queue --> QueueBehavior

    %% Click events
    click PacedMutations "https://github.com/TanStack/db/blob/main/packages/db/src/paced-mutations.ts"
    click Debounce "https://github.com/TanStack/db/blob/main/packages/db/src/strategies/debounceStrategy.ts"
    click Throttle "https://github.com/TanStack/db/blob/main/packages/db/src/strategies/throttleStrategy.ts"
    click Queue "https://github.com/TanStack/db/blob/main/packages/db/src/strategies/queueStrategy.ts"

    %% Styles
    classDef config fill:#4f46e5,stroke:#3730a3,color:#fff
    classDef handler fill:#059669,stroke:#047857,color:#fff
    classDef strategy fill:#7c3aed,stroke:#6d28d9,color:#fff
    classDef debounce fill:#0891b2,stroke:#0e7490,color:#fff
    classDef throttle fill:#dc2626,stroke:#b91c1c,color:#fff
    classDef queue fill:#d97706,stroke:#b45309,color:#fff
    classDef behavior fill:#6b7280,stroke:#4b5563,color:#fff
```

### Debounce Strategy

```mermaid
sequenceDiagram
    participant User as User Input
    participant Pace as PacedMutation
    participant Timer as Debounce Timer
    participant Tx as Transaction
    participant Server as Server

    Note over User,Server: Debounce: wait=500ms

    User->>Pace: mutate("A")
    Pace->>Tx: Apply optimistic
    Pace->>Timer: Start 500ms timer
    
    User->>Pace: mutate("AB")
    Pace->>Tx: Apply optimistic
    Pace->>Timer: Reset timer
    
    User->>Pace: mutate("ABC")
    Pace->>Tx: Apply optimistic
    Pace->>Timer: Reset timer
    
    Note over Timer: 500ms passes with no input
    
    Timer->>Tx: commit()
    Tx->>Server: Send all merged mutations
    Server-->>Tx: Success
```

### Throttle Strategy

```mermaid
sequenceDiagram
    participant User as User Input
    participant Pace as PacedMutation
    participant Tx as Transaction
    participant Server as Server

    Note over User,Server: Throttle: wait=200ms

    User->>Pace: mutate("A") @ t=0
    Pace->>Tx: Apply optimistic
    Pace->>Tx: commit() immediately
    Tx->>Server: Send
    
    User->>Pace: mutate("B") @ t=50ms
    Pace->>Tx: Apply optimistic
    Note over Pace: Queued until t=200ms
    
    User->>Pace: mutate("C") @ t=100ms
    Pace->>Tx: Apply optimistic (merged with B)
    
    Note over Pace: t=200ms reached
    Pace->>Tx: commit()
    Tx->>Server: Send merged B+C
```

### Queue Strategy

```mermaid
sequenceDiagram
    participant User as User Input
    participant Pace as PacedMutation
    participant Queue as Queue
    participant Server as Server

    Note over User,Server: Queue: sequential processing

    User->>Pace: mutate(item1)
    Pace->>Queue: Enqueue tx1
    Queue->>Server: Process tx1
    
    User->>Pace: mutate(item2)
    Pace->>Queue: Enqueue tx2
    Note over Queue: tx2 waits for tx1
    
    User->>Pace: mutate(item3)
    Pace->>Queue: Enqueue tx3
    
    Server-->>Queue: tx1 complete
    Queue->>Server: Process tx2
    
    Server-->>Queue: tx2 complete
    Queue->>Server: Process tx3
```

## Optimistic Update Flow

```mermaid
flowchart TD
    subgraph OptimisticFlow["Optimistic Update Flow"]
        Action["User Action"]:::action
        Draft["Create Draft Proxy"]:::draft
        Validate["Schema Validation"]:::validate
        Apply["Apply to optimisticUpserts"]:::apply
        Emit["Emit change events"]:::emit
        Query["Live queries update"]:::query
        UI["UI re-renders"]:::ui
    end

    subgraph Persistence["Persistence Flow"]
        Handler["Mutation handler called"]:::handler
        API["API request"]:::api
        Success["Success: clear optimistic"]:::success
        Fail["Failure: rollback"]:::fail
    end

    Action --> Draft
    Draft --> Validate
    Validate -->|"valid"| Apply
    Validate -->|"invalid"| Error["Validation Error"]:::error
    Apply --> Emit
    Emit --> Query
    Query --> UI

    UI --> Handler
    Handler --> API
    API --> Success
    API --> Fail
    Fail --> Rollback["Rollback optimistic"]:::rollback
    Rollback --> Emit

    %% Styles
    classDef action fill:#4f46e5,stroke:#3730a3,color:#fff
    classDef draft fill:#7c3aed,stroke:#6d28d9,color:#fff
    classDef validate fill:#059669,stroke:#047857,color:#fff
    classDef apply fill:#0891b2,stroke:#0e7490,color:#fff
    classDef emit fill:#d97706,stroke:#b45309,color:#fff
    classDef query fill:#84cc16,stroke:#65a30d,color:#fff
    classDef ui fill:#ec4899,stroke:#db2777,color:#fff
    classDef handler fill:#059669,stroke:#047857,color:#fff
    classDef api fill:#6b7280,stroke:#4b5563,color:#fff
    classDef success fill:#10b981,stroke:#059669,color:#fff
    classDef fail fill:#dc2626,stroke:#b91c1c,color:#fff
    classDef error fill:#dc2626,stroke:#b91c1c,color:#fff
    classDef rollback fill:#f59e0b,stroke:#d97706,color:#fff
```

## Ambient Transaction Context

TanStack DB uses an ambient transaction pattern where collection operations automatically join the active transaction:

```mermaid
flowchart TD
    subgraph AmbientTransaction["Ambient Transaction Context"]
        Stack["transactionStack: Array<Transaction>"]:::stack
        Register["registerTransaction()"]:::method
        Unregister["unregisterTransaction()"]:::method
        GetActive["getActiveTransaction()"]:::method
    end

    subgraph Usage["Usage Pattern"]
        TxMutate["tx.mutate(() => { ... })"]:::mutate
        CollOp["collection.insert(data)"]:::operation
        AutoJoin["Automatically joins active tx"]:::auto
    end

    TxMutate --> Register
    Register --> Stack
    
    CollOp --> GetActive
    GetActive --> Stack
    GetActive --> AutoJoin
    
    TxMutate --> Unregister
    Unregister --> Stack

    %% Click events
    click GetActive "https://github.com/TanStack/db/blob/main/packages/db/src/transactions.ts"

    %% Styles
    classDef stack fill:#4f46e5,stroke:#3730a3,color:#fff
    classDef method fill:#059669,stroke:#047857,color:#fff
    classDef mutate fill:#0891b2,stroke:#0e7490,color:#fff
    classDef operation fill:#7c3aed,stroke:#6d28d9,color:#fff
    classDef auto fill:#d97706,stroke:#b45309,color:#fff
```

## File Structure

```
packages/db/src/
├── transactions.ts       # Transaction class and createTransaction
├── paced-mutations.ts    # createPacedMutations
├── proxy.ts              # Draft proxy for mutations
├── scheduler.ts          # Transaction-scoped scheduler
├── deferred.ts           # Deferred promise utility
│
├── strategies/
│   ├── index.ts          # Strategy exports
│   ├── types.ts          # Strategy interface
│   ├── debounceStrategy.ts
│   ├── throttleStrategy.ts
│   └── queueStrategy.ts
│
└── collection/
    └── mutations.ts      # CollectionMutationsManager
```

## Effect-TS Porting Guide

### Transaction → Effect.gen with Scope

```typescript
// Effect-TS transaction pattern
interface Transaction<T> {
  readonly id: string
  readonly state: TransactionState
  readonly mutations: Array<PendingMutation<T>>
  readonly commit: Effect.Effect<void, TransactionError>
  readonly rollback: Effect.Effect<void>
}

const createTransaction = <T>(config: TransactionConfig<T>) =>
  Effect.gen(function* () {
    const scope = yield* Effect.scope
    const stateRef = yield* Ref.make<TransactionState>("pending")
    const mutationsRef = yield* Ref.make<Array<PendingMutation<T>>>([])
    const completionDeferred = yield* Deferred.make<void, TransactionError>()
    
    const commit = Effect.gen(function* () {
      yield* Ref.set(stateRef, "persisting")
      const mutations = yield* Ref.get(mutationsRef)
      
      yield* Effect.tryPromise(() => config.mutationFn({ mutations })).pipe(
        Effect.tapError(() => Ref.set(stateRef, "failed")),
        Effect.tap(() => Ref.set(stateRef, "completed")),
        Effect.tap(() => Deferred.succeed(completionDeferred, void 0))
      )
    })
    
    return {
      id: yield* Effect.sync(() => crypto.randomUUID()),
      state: Ref.get(stateRef),
      mutations: Ref.get(mutationsRef),
      commit,
      rollback: Ref.set(stateRef, "cancelled")
    }
  })
```

### Mutation Merging → Pattern Matching

```typescript
// Effect-TS mutation merging with Match
import { Match } from "effect"

const mergeMutations = <T>(
  existing: PendingMutation<T>,
  incoming: PendingMutation<T>
): Option<PendingMutation<T>> =>
  Match.value([existing.type, incoming.type] as const).pipe(
    Match.when(["insert", "update"], () => Option.some({
      ...existing,
      modified: incoming.modified,
      changes: { ...existing.changes, ...incoming.changes }
    })),
    Match.when(["insert", "delete"], () => Option.none()),
    Match.when(["update", "delete"], () => Option.some(incoming)),
    Match.when(["update", "update"], () => Option.some({
      ...incoming,
      original: existing.original,
      changes: { ...existing.changes, ...incoming.changes }
    })),
    Match.orElse(() => Option.some(incoming))
  )
```

### Paced Mutations → Schedule

```typescript
// Effect-TS paced mutations with Schedule
const createPacedMutations = <TVariables, T>(
  config: PacedMutationsConfig<TVariables, T>
) => Effect.gen(function* () {
  const pendingRef = yield* Ref.make<Array<TVariables>>([])
  
  // Debounce schedule
  const debounceSchedule = Schedule.debounce("500 millis")
  
  const flush = Effect.gen(function* () {
    const pending = yield* Ref.getAndSet(pendingRef, [])
    if (pending.length === 0) return
    
    const tx = yield* createTransaction({
      mutationFn: config.mutationFn
    })
    
    for (const variables of pending) {
      yield* tx.mutate(() => config.onMutate(variables))
    }
    
    yield* tx.commit
  })
  
  // Create a stream that flushes on schedule
  const flushFiber = yield* Stream.fromEffect(flush).pipe(
    Stream.schedule(debounceSchedule),
    Stream.runDrain,
    Effect.fork
  )
  
  return (variables: TVariables) => Effect.gen(function* () {
    yield* Ref.update(pendingRef, Array.append(variables))
    // Trigger the debounce...
  })
})
```

### Strategies → Schedule Variants

```typescript
// Effect-TS strategy implementations
const debounceStrategy = (wait: Duration) =>
  Schedule.debounce(wait)

const throttleStrategy = (wait: Duration) =>
  Schedule.spaced(wait)

const queueStrategy = (wait: Duration) =>
  Schedule.fixed(wait).pipe(
    Schedule.ensuring(() => /* release next in queue */)
  )
```

## Related Documentation

- [ARCHITECTURE-OVERVIEW.md](./ARCHITECTURE-OVERVIEW.md) - High-level overview
- [ARCHITECTURE-CORE.md](./ARCHITECTURE-CORE.md) - Collection state management
- [ARCHITECTURE-OFFLINE.md](./ARCHITECTURE-OFFLINE.md) - Offline transaction persistence

