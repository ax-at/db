# TanStack DB Collections Architecture

This document details the collection types and sync patterns in TanStack DB, covering the various collection implementations and how they integrate with different data sources.

## Collection Type Hierarchy

```mermaid
flowchart TD
    subgraph CoreAPI["Core Collection API"]
        createCollection["createCollection()"]:::core
        CollectionConfig["CollectionConfig"]:::config
    end

    subgraph BuiltIn["Built-in Collection Types"]
        LocalOnly["localOnlyCollectionOptions"]:::builtin
        LocalStorage["localStorageCollectionOptions"]:::builtin
    end

    subgraph Providers["Collection Providers"]
        QueryColl["queryCollectionOptions"]:::provider
        ElectricColl["electricCollectionOptions"]:::provider
        PowerSyncColl["powerSyncCollectionOptions"]:::provider
        RxDBColl["rxdbCollectionOptions"]:::provider
        TrailBaseColl["trailbaseCollectionOptions"]:::provider
    end

    subgraph Custom["Custom Implementation"]
        CustomSync["Custom sync config"]:::custom
    end

    CollectionConfig --> createCollection
    
    LocalOnly --> CollectionConfig
    LocalStorage --> CollectionConfig
    
    QueryColl --> CollectionConfig
    ElectricColl --> CollectionConfig
    PowerSyncColl --> CollectionConfig
    RxDBColl --> CollectionConfig
    TrailBaseColl --> CollectionConfig
    
    CustomSync --> CollectionConfig

    %% Click events
    click createCollection "https://github.com/TanStack/db/blob/main/packages/db/src/collection/index.ts"
    click LocalOnly "https://github.com/TanStack/db/blob/main/packages/db/src/local-only.ts"
    click LocalStorage "https://github.com/TanStack/db/blob/main/packages/db/src/local-storage.ts"
    click QueryColl "https://github.com/TanStack/db/blob/main/packages/query-db-collection/src/query.ts"
    click ElectricColl "https://github.com/TanStack/db/blob/main/packages/electric-db-collection/src/index.ts"
    click PowerSyncColl "https://github.com/TanStack/db/blob/main/packages/powersync-db-collection/src/index.ts"
    click RxDBColl "https://github.com/TanStack/db/blob/main/packages/rxdb-db-collection/src/index.ts"

    %% Styles
    classDef core fill:#4f46e5,stroke:#3730a3,color:#fff
    classDef config fill:#7c3aed,stroke:#6d28d9,color:#fff
    classDef builtin fill:#059669,stroke:#047857,color:#fff
    classDef provider fill:#0891b2,stroke:#0e7490,color:#fff
    classDef custom fill:#d97706,stroke:#b45309,color:#fff
```

## Collection Types Reference

| Type | Package | Persistence | Sync Mode | Use Case |
|------|---------|-------------|-----------|----------|
| Local-Only | `@tanstack/db` | In-memory | Loopback | Temporary data, form state |
| LocalStorage | `@tanstack/db` | Browser storage | Cross-tab sync | Settings, preferences |
| Query | `@tanstack/query-db-collection` | None | Fetch-based | REST APIs, GraphQL |
| Electric | `@tanstack/electric-db-collection` | SQLite | Real-time sync | Offline-first apps |
| PowerSync | `@tanstack/powersync-db-collection` | SQLite | Bidirectional | Mobile apps |
| RxDB | `@tanstack/rxdb-db-collection` | Various | RxJS streams | Reactive data stores |
| TrailBase | `@tanstack/trailbase-db-collection` | TrailBase | REST + subscriptions | TrailBase backends |

## Sync Modes

TanStack DB supports three sync modes that determine when data is loaded:

```mermaid
flowchart TD
    subgraph SyncModes["Sync Modes"]
        Eager["Eager Mode"]:::eager
        OnDemand["On-Demand Mode"]:::ondemand
        Progressive["Progressive Mode"]:::progressive
    end

    subgraph EagerDesc["Eager Mode"]
        E1["All data loaded upfront"]:::desc
        E2["collection.preload()"]:::desc
        E3["Best for small datasets"]:::desc
    end

    subgraph OnDemandDesc["On-Demand Mode"]
        O1["Data loaded per query"]:::desc
        O2["loadSubset() handler required"]:::desc
        O3["Best for large datasets"]:::desc
    end

    subgraph ProgressiveDesc["Progressive Mode"]
        P1["Start with partial data"]:::desc
        P2["Load more on demand"]:::desc
        P3["Combines eager + on-demand"]:::desc
    end

    Eager --> E1
    Eager --> E2
    Eager --> E3

    OnDemand --> O1
    OnDemand --> O2
    OnDemand --> O3

    Progressive --> P1
    Progressive --> P2
    Progressive --> P3

    %% Styles
    classDef eager fill:#059669,stroke:#047857,color:#fff
    classDef ondemand fill:#0891b2,stroke:#0e7490,color:#fff
    classDef progressive fill:#d97706,stroke:#b45309,color:#fff
    classDef desc fill:#6b7280,stroke:#4b5563,color:#fff
```

### Sync Mode Decision Tree

```mermaid
flowchart TD
    Start["Dataset Size?"]:::question
    
    Small["< 10k rows"]:::size
    Medium["10k - 100k rows"]:::size
    Large["> 100k rows"]:::size

    Start --> Small
    Start --> Medium
    Start --> Large

    Small --> UseEager["Use Eager Mode"]:::decision
    Medium --> NeedAll{"Need all data\nat once?"}:::question
    Large --> UseOnDemand["Use On-Demand Mode"]:::decision

    NeedAll -->|"Yes"| UseEager
    NeedAll -->|"No"| UseProgressive["Use Progressive Mode"]:::decision

    UseEager --> EagerConfig["syncMode: 'eager'"]:::config
    UseOnDemand --> OnDemandConfig["syncMode: 'on-demand'\n+ loadSubset handler"]:::config
    UseProgressive --> ProgressiveConfig["syncMode: 'progressive'\n+ initial data + loadSubset"]:::config

    %% Styles
    classDef question fill:#4f46e5,stroke:#3730a3,color:#fff
    classDef size fill:#7c3aed,stroke:#6d28d9,color:#fff
    classDef decision fill:#059669,stroke:#047857,color:#fff
    classDef config fill:#0891b2,stroke:#0e7490,color:#fff
```

## Built-in Collection Types

### Local-Only Collection

```mermaid
flowchart LR
    subgraph LocalOnly["Local-Only Collection"]
        Create["createCollection(localOnlyCollectionOptions(...))"]:::create
        Memory["In-Memory State"]:::storage
        Loopback["Loopback Sync"]:::sync
    end

    subgraph Features["Features"]
        F1["No persistence"]:::feature
        F2["Immediate confirmation"]:::feature
        F3["initialData support"]:::feature
    end

    Create --> Memory
    Memory <-->|"optimistic ↔ synced"| Loopback

    %% Styles
    classDef create fill:#4f46e5,stroke:#3730a3,color:#fff
    classDef storage fill:#059669,stroke:#047857,color:#fff
    classDef sync fill:#0891b2,stroke:#0e7490,color:#fff
    classDef feature fill:#6b7280,stroke:#4b5563,color:#fff
```

### LocalStorage Collection

```mermaid
flowchart LR
    subgraph LocalStorage["LocalStorage Collection"]
        Create["createCollection(localStorageCollectionOptions(...))"]:::create
        Browser["Browser Storage"]:::storage
        CrossTab["Storage Event Listener"]:::sync
    end

    subgraph Tabs["Browser Tabs"]
        Tab1["Tab 1"]:::tab
        Tab2["Tab 2"]:::tab
        Tab3["Tab 3"]:::tab
    end

    Create --> Browser
    Browser <-->|"storage events"| CrossTab
    CrossTab <-->|"sync"| Tab1
    CrossTab <-->|"sync"| Tab2
    CrossTab <-->|"sync"| Tab3

    %% Styles
    classDef create fill:#4f46e5,stroke:#3730a3,color:#fff
    classDef storage fill:#059669,stroke:#047857,color:#fff
    classDef sync fill:#0891b2,stroke:#0e7490,color:#fff
    classDef tab fill:#d97706,stroke:#b45309,color:#fff
```

## Provider Collection Types

### Query Collection (TanStack Query)

```mermaid
flowchart TB
    subgraph QueryCollection["Query Collection"]
        Config["queryCollectionOptions(...)"]:::config
        Observer["QueryObserver"]:::observer
        Cache["TanStack Query Cache"]:::cache
    end

    subgraph TanStackQuery["TanStack Query"]
        QueryClient["QueryClient"]:::query
        QueryFn["queryFn"]:::query
        QueryKey["queryKey"]:::query
    end

    subgraph Server["Server"]
        API["REST API / GraphQL"]:::server
    end

    subgraph Collection["Collection"]
        SyncState["Synced State"]:::state
        OptState["Optimistic State"]:::state
    end

    Config --> Observer
    Observer --> QueryClient
    QueryClient --> QueryFn
    QueryFn --> API
    API --> QueryFn
    QueryFn --> Cache
    Cache --> SyncState

    QueryKey --> Observer

    %% Click events
    click Config "https://github.com/TanStack/db/blob/main/packages/query-db-collection/src/query.ts"

    %% Styles
    classDef config fill:#4f46e5,stroke:#3730a3,color:#fff
    classDef observer fill:#059669,stroke:#047857,color:#fff
    classDef cache fill:#7c3aed,stroke:#6d28d9,color:#fff
    classDef query fill:#0891b2,stroke:#0e7490,color:#fff
    classDef server fill:#6b7280,stroke:#4b5563,color:#fff
    classDef state fill:#d97706,stroke:#b45309,color:#fff
```

### Electric Collection (Real-time Sync)

```mermaid
flowchart TB
    subgraph ElectricCollection["Electric Collection"]
        Config["electricCollectionOptions(...)"]:::config
        Shape["ShapeStream"]:::stream
    end

    subgraph Electric["ElectricSQL"]
        Server["Electric Server"]:::electric
        Postgres["PostgreSQL"]:::electric
        ShapeLog["Shape Log"]:::electric
    end

    subgraph Collection["Collection"]
        SyncState["Synced State"]:::state
        OptState["Optimistic State"]:::state
    end

    subgraph Client["Client"]
        UI["UI Component"]:::ui
    end

    Config --> Shape
    Shape <-->|"real-time sync"| Server
    Server <--> Postgres
    Server --> ShapeLog
    ShapeLog --> Shape
    Shape --> SyncState
    SyncState --> UI

    %% Click events
    click Config "https://github.com/TanStack/db/blob/main/packages/electric-db-collection/src/index.ts"

    %% Styles
    classDef config fill:#4f46e5,stroke:#3730a3,color:#fff
    classDef stream fill:#059669,stroke:#047857,color:#fff
    classDef electric fill:#0891b2,stroke:#0e7490,color:#fff
    classDef state fill:#d97706,stroke:#b45309,color:#fff
    classDef ui fill:#7c3aed,stroke:#6d28d9,color:#fff
```

## Sync Configuration

Every collection requires a `sync` configuration that defines how data flows in and out:

```mermaid
flowchart TD
    subgraph SyncConfig["SyncConfig Interface"]
        Sync["sync: (params) => cleanup"]:::method
        GetMeta["getSyncMetadata: () => metadata"]:::method
    end

    subgraph SyncParams["Sync Parameters"]
        Collection["collection"]:::param
        Begin["begin()"]:::param
        Write["write(change)"]:::param
        Commit["commit()"]:::param
        MarkReady["markReady()"]:::param
        Truncate["truncate()"]:::param
    end

    subgraph SyncRes["Optional Return"]
        Cleanup["cleanup: () => void"]:::return
        LoadSubset["loadSubset: (opts) => Promise"]:::return
        UnloadSubset["unloadSubset: (opts) => void"]:::return
    end

    SyncConfig --> Sync
    SyncConfig --> GetMeta
    
    Sync --> SyncParams
    Sync --> SyncRes

    Begin --> Write
    Write --> Commit
    Commit --> MarkReady

    %% Styles
    classDef method fill:#4f46e5,stroke:#3730a3,color:#fff
    classDef param fill:#059669,stroke:#047857,color:#fff
    classDef return fill:#0891b2,stroke:#0e7490,color:#fff
```

### Sync Protocol Sequence

```mermaid
sequenceDiagram
    participant Source as Data Source
    participant Sync as Sync Function
    participant State as CollectionState
    participant UI as UI Component

    Note over Source,UI: Initial Sync

    Source->>Sync: Data available
    Sync->>Sync: begin()
    
    loop For each item
        Sync->>Sync: write({ type: 'insert', value })
    end
    
    Sync->>Sync: commit()
    Sync->>State: Apply synced transaction
    Sync->>Sync: markReady()
    State-->>UI: Collection ready

    Note over Source,UI: Incremental Updates

    Source->>Sync: Data changed
    Sync->>Sync: begin()
    Sync->>Sync: write({ type: 'update', value })
    Sync->>Sync: commit()
    Sync->>State: Apply synced transaction
    State-->>UI: Update propagated

    Note over Source,UI: Full Refresh

    Source->>Sync: Refresh needed
    Sync->>Sync: begin()
    Sync->>Sync: truncate()
    
    loop For each item
        Sync->>Sync: write({ type: 'insert', value })
    end
    
    Sync->>Sync: commit()
    Sync->>State: Apply with truncate
    State-->>UI: Full refresh
```

## Collection Utilities

Each collection type provides specific utility functions:

```mermaid
flowchart TD
    subgraph BaseUtils["Base Utilities (all collections)"]
        preload["collection.preload()"]:::base
        cleanup["collection.cleanup()"]:::base
        events["collection.on(event, handler)"]:::base
    end

    subgraph LocalOnlyUtils["Local-Only Utilities"]
        acceptMut1["utils.acceptMutations(tx)"]:::local
    end

    subgraph LocalStorageUtils["LocalStorage Utilities"]
        clearStorage["utils.clearStorage()"]:::storage
        getSize["utils.getStorageSize()"]:::storage
        acceptMut2["utils.acceptMutations(tx)"]:::storage
    end

    subgraph QueryUtils["Query Collection Utilities"]
        refetch["utils.refetch()"]:::query
        getStatus["utils.getFetchStatus()"]:::query
        directWrite["utils.directInsert/Update/Delete()"]:::query
    end

    subgraph ElectricUtils["Electric Collection Utilities"]
        getShape["utils.getShape()"]:::electric
        isLive["utils.isLive()"]:::electric
    end

    %% Click events
    click preload "https://github.com/TanStack/db/blob/main/packages/db/src/collection/sync.ts"
    click acceptMut1 "https://github.com/TanStack/db/blob/main/packages/db/src/local-only.ts"
    click clearStorage "https://github.com/TanStack/db/blob/main/packages/db/src/local-storage.ts"
    click refetch "https://github.com/TanStack/db/blob/main/packages/query-db-collection/src/query.ts"

    %% Styles
    classDef base fill:#4f46e5,stroke:#3730a3,color:#fff
    classDef local fill:#059669,stroke:#047857,color:#fff
    classDef storage fill:#7c3aed,stroke:#6d28d9,color:#fff
    classDef query fill:#0891b2,stroke:#0e7490,color:#fff
    classDef electric fill:#d97706,stroke:#b45309,color:#fff
```

## Effect-TS Porting Guide

### Collection Types → Effect Layer

```typescript
// Effect-TS collection layer pattern
interface Collection<T, TKey> extends Effect.Tag<Collection<T, TKey>, {
  readonly get: (key: TKey) => Effect.Effect<Option<T>>
  readonly insert: (data: T) => Effect.Effect<void, ValidationError>
  readonly subscribeChanges: Stream.Stream<Array<Change<T>>>
}> {}

const CollectionLive = <T, TKey>(config: CollectionConfig<T, TKey>) =>
  Layer.effect(
    Collection<T, TKey>,
    Effect.gen(function* () {
      const stateRef = yield* Ref.make(initialState)
      // ... implementation
    })
  )
```

### LocalOnly → Effect Ref

```typescript
// Effect-TS local-only collection
const LocalOnlyLive = <T, TKey>(config: LocalOnlyConfig<T, TKey>) =>
  Layer.effect(
    Collection<T, TKey>,
    Effect.gen(function* () {
      const dataRef = yield* Ref.make<HashMap<TKey, T>>(
        HashMap.fromIterable(config.initialData ?? [])
      )
      
      const insert = (data: T) => Effect.gen(function* () {
        const key = config.getKey(data)
        yield* Ref.update(dataRef, HashMap.set(key, data))
      })
      
      return { insert, /* ... */ }
    })
  )
```

### LocalStorage → Effect KeyValueStore

```typescript
// Effect-TS localStorage collection
import { KeyValueStore } from "@effect/platform"

const LocalStorageLive = <T, TKey>(config: LocalStorageConfig<T, TKey>) =>
  Layer.effect(
    Collection<T, TKey>,
    Effect.gen(function* () {
      const store = yield* KeyValueStore.KeyValueStore
      const dataRef = yield* Ref.make<HashMap<TKey, T>>(HashMap.empty())
      
      // Initial load from storage
      const stored = yield* store.get(config.storageKey)
      if (Option.isSome(stored)) {
        const parsed = yield* Effect.try(() => JSON.parse(stored.value))
        yield* Ref.set(dataRef, HashMap.fromIterable(parsed))
      }
      
      return { /* ... */ }
    })
  ).pipe(Layer.provide(KeyValueStore.layerLocalStorage))
```

### Query Collection → Effect.cached + retry

```typescript
// Effect-TS query collection
const QueryCollectionLive = <T, TKey>(config: QueryConfig<T, TKey>) =>
  Layer.effect(
    Collection<T, TKey>,
    Effect.gen(function* () {
      const fetchData = yield* Effect.cached(
        config.queryFn.pipe(
          Effect.retry(Schedule.exponential("100 millis")),
          Effect.timeout("30 seconds")
        )
      )
      
      const refetch = Effect.gen(function* () {
        yield* Effect.sync(() => { /* invalidate cache */ })
        return yield* fetchData
      })
      
      return { refetch, /* ... */ }
    })
  )
```

### Electric Collection → Stream + Hub

```typescript
// Effect-TS electric collection with real-time sync
const ElectricCollectionLive = <T, TKey>(config: ElectricConfig<T, TKey>) =>
  Layer.effect(
    Collection<T, TKey>,
    Effect.gen(function* () {
      const changesHub = yield* Hub.unbounded<Array<Change<T>>>()
      const dataRef = yield* Ref.make<HashMap<TKey, T>>(HashMap.empty())
      
      // Subscribe to Electric shape stream
      const syncFiber = yield* Stream.fromAsyncIterable(
        config.shapeStream,
        e => new Error(String(e))
      ).pipe(
        Stream.tap(changes => Effect.gen(function* () {
          yield* applyChanges(dataRef, changes)
          yield* Hub.publish(changesHub, changes)
        })),
        Stream.runDrain,
        Effect.fork
      )
      
      return {
        subscribeChanges: Stream.fromHub(changesHub),
        cleanup: Fiber.interrupt(syncFiber),
        /* ... */
      }
    })
  )
```

## File Structure

```
packages/
├── db/src/
│   ├── local-only.ts           # Local-only collection options
│   └── local-storage.ts        # LocalStorage collection options
│
├── query-db-collection/src/
│   ├── query.ts                # TanStack Query integration
│   └── manual-sync.ts          # Direct write utilities
│
├── electric-db-collection/src/
│   └── index.ts                # ElectricSQL integration
│
├── powersync-db-collection/src/
│   └── index.ts                # PowerSync integration
│
├── rxdb-db-collection/src/
│   └── index.ts                # RxDB integration
│
└── trailbase-db-collection/src/
    └── index.ts                # TrailBase integration
```

## Related Documentation

- [ARCHITECTURE-OVERVIEW.md](./ARCHITECTURE-OVERVIEW.md) - High-level overview
- [ARCHITECTURE-CORE.md](./ARCHITECTURE-CORE.md) - Core collection internals
- [ARCHITECTURE-MUTATIONS.md](./ARCHITECTURE-MUTATIONS.md) - Mutation handling

