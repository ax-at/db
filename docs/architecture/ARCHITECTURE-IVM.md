# TanStack DB IVM Architecture

This document details the Incremental View Maintenance (IVM) engine in `@tanstack/db-ivm`, which is a TypeScript implementation of differential dataflow for sub-millisecond reactive query updates.

## Overview

The IVM engine is based on [d2ts](https://github.com/electric-sql/d2ts), a differential dataflow implementation. Instead of re-executing entire queries when data changes, it propagates only the differences (deltas) through a computation graph, achieving O(changes) update complexity rather than O(data).

## Core Concepts

### Differential Dataflow

```mermaid
flowchart LR
    subgraph Traditional["Traditional Query Execution"]
        Data1["100k rows"]:::data
        Query1["Full Query"]:::query
        Result1["Result"]:::result
        
        Data1 -->|"re-execute"| Query1
        Query1 -->|"~100ms"| Result1
    end

    subgraph Differential["Differential Dataflow"]
        Data2["100k rows"]:::data
        Delta["Δ: 1 change"]:::delta
        Pipeline["Incremental Pipeline"]:::query
        Result2["Result + Δ"]:::result
        
        Data2 -.->|"initial"| Pipeline
        Delta -->|"propagate"| Pipeline
        Pipeline -->|"~0.7ms"| Result2
    end

    %% Styles
    classDef data fill:#4f46e5,stroke:#3730a3,color:#fff
    classDef query fill:#059669,stroke:#047857,color:#fff
    classDef result fill:#0891b2,stroke:#0e7490,color:#fff
    classDef delta fill:#dc2626,stroke:#b91c1c,color:#fff
```

### MultiSet: The Core Data Structure

A `MultiSet` is a collection of `(value, multiplicity)` pairs that represents differences:

```typescript
// Positive multiplicity = insertion
[{ id: 1, name: "Alice" }, +1]  // Add row

// Negative multiplicity = deletion
[{ id: 1, name: "Alice" }, -1]  // Remove row

// Update = deletion + insertion
[{ id: 1, name: "Alice" }, -1]  // Delete old
[{ id: 1, name: "Bob" },   +1]  // Insert new
```

```mermaid
flowchart TD
    subgraph MultiSetOps["MultiSet Operations"]
        direction TB
        Insert["Insert: (value, +1)"]:::insert
        Delete["Delete: (value, -1)"]:::delete
        Update["Update: (old, -1) + (new, +1)"]:::update
    end

    subgraph Consolidate["Consolidation"]
        Raw["Raw MultiSet"]:::raw
        Consolidated["Consolidated"]:::consolidated
        
        Raw -->|"consolidate()"| Consolidated
    end

    subgraph Example["Example"]
        E1["(A, +1)"]:::insert
        E2["(A, +1)"]:::insert
        E3["(A, -1)"]:::delete
        E4["Result: (A, +1)"]:::result
        
        E1 --> Merge["merge"]
        E2 --> Merge
        E3 --> Merge
        Merge --> E4
    end

    %% Styles
    classDef insert fill:#059669,stroke:#047857,color:#fff
    classDef delete fill:#dc2626,stroke:#b91c1c,color:#fff
    classDef update fill:#d97706,stroke:#b45309,color:#fff
    classDef raw fill:#6b7280,stroke:#4b5563,color:#fff
    classDef consolidated fill:#4f46e5,stroke:#3730a3,color:#fff
    classDef result fill:#0891b2,stroke:#0e7490,color:#fff
```

## Dataflow Graph Architecture

```mermaid
flowchart TD
    subgraph D2Runtime["D2 Runtime"]
        D2["D2 Graph"]:::runtime
        Step["step()"]:::method
        Run["run()"]:::method
        Finalize["finalize()"]:::method
    end

    subgraph Inputs["Input Streams"]
        Input1["RootStreamBuilder A"]:::input
        Input2["RootStreamBuilder B"]:::input
    end

    subgraph Operators["Operator Pipeline"]
        Filter["FilterOperator"]:::operator
        Map["MapOperator"]:::operator
        Join["JoinOperator"]:::operator
        GroupBy["GroupByOperator"]:::operator
        OrderBy["OrderByOperator"]:::operator
    end

    subgraph Output["Output"]
        OutOp["OutputOperator"]:::output
        Callback["callback(changes)"]:::callback
    end

    subgraph DataFlow["Data Propagation"]
        Writer["DifferenceStreamWriter"]:::stream
        Reader["DifferenceStreamReader"]:::stream
        Queue["Queue of MultiSets"]:::queue
    end

    D2 --> Input1
    D2 --> Input2
    
    Input1 --> Filter
    Input2 --> Join
    
    Filter --> Map
    Map --> Join
    Join --> GroupBy
    GroupBy --> OrderBy
    OrderBy --> OutOp
    OutOp --> Callback

    Writer -->|"sendData"| Queue
    Queue -->|"drain"| Reader

    %% Click events
    click D2 "https://github.com/TanStack/db/blob/main/packages/db-ivm/src/d2.ts"
    click Writer "https://github.com/TanStack/db/blob/main/packages/db-ivm/src/graph.ts"
    click Reader "https://github.com/TanStack/db/blob/main/packages/db-ivm/src/graph.ts"
    click Filter "https://github.com/TanStack/db/blob/main/packages/db-ivm/src/operators/filter.ts"
    click Map "https://github.com/TanStack/db/blob/main/packages/db-ivm/src/operators/map.ts"
    click Join "https://github.com/TanStack/db/blob/main/packages/db-ivm/src/operators/join.ts"

    %% Styles
    classDef runtime fill:#4f46e5,stroke:#3730a3,color:#fff
    classDef method fill:#7c3aed,stroke:#6d28d9,color:#fff
    classDef input fill:#059669,stroke:#047857,color:#fff
    classDef operator fill:#0891b2,stroke:#0e7490,color:#fff
    classDef output fill:#dc2626,stroke:#b91c1c,color:#fff
    classDef callback fill:#d97706,stroke:#b45309,color:#fff
    classDef stream fill:#6b7280,stroke:#4b5563,color:#fff
    classDef queue fill:#84cc16,stroke:#65a30d,color:#fff
```

## Operator Classes

```mermaid
flowchart TD
    subgraph BaseClasses["Base Classes"]
        IOperator["IOperator interface"]:::interface
        Operator["Operator abstract"]:::abstract
    end

    subgraph Unary["Unary Operators"]
        UnaryOp["UnaryOperator"]:::abstract
        LinearOp["LinearUnaryOperator"]:::abstract
        
        MapOp["map"]:::concrete
        FilterOp["filter"]:::concrete
        NegateOp["negate"]:::concrete
        DistinctOp["distinct"]:::concrete
        TopKOp["topK"]:::concrete
    end

    subgraph Binary["Binary Operators"]
        BinaryOp["BinaryOperator"]:::abstract
        
        JoinOp["join"]:::concrete
        ConcatOp["concat"]:::concrete
    end

    subgraph Stateful["Stateful Operators"]
        ReduceOp["reduce"]:::stateful
        GroupByOp["groupBy"]:::stateful
        OrderByOp["orderBy"]:::stateful
        ConsolidateOp["consolidate"]:::stateful
    end

    IOperator --> Operator
    Operator --> UnaryOp
    Operator --> BinaryOp
    
    UnaryOp --> LinearOp
    LinearOp --> MapOp
    LinearOp --> FilterOp
    LinearOp --> NegateOp
    
    UnaryOp --> DistinctOp
    UnaryOp --> TopKOp
    UnaryOp --> ReduceOp
    UnaryOp --> GroupByOp
    UnaryOp --> OrderByOp
    UnaryOp --> ConsolidateOp
    
    BinaryOp --> JoinOp
    BinaryOp --> ConcatOp

    %% Styles
    classDef interface fill:#6b7280,stroke:#4b5563,color:#fff
    classDef abstract fill:#7c3aed,stroke:#6d28d9,color:#fff
    classDef concrete fill:#059669,stroke:#047857,color:#fff
    classDef stateful fill:#dc2626,stroke:#b91c1c,color:#fff
```

## Operators Reference

| Operator | Type | SQL Equivalent | State | Effect-TS Equivalent |
|----------|------|----------------|-------|---------------------|
| `map` | Unary/Linear | SELECT transform | Stateless | `Stream.map` |
| `filter` | Unary/Linear | WHERE | Stateless | `Stream.filter` |
| `negate` | Unary/Linear | - | Stateless | `Stream.map(negate)` |
| `distinct` | Unary | DISTINCT | Stateful | `Stream.changes` |
| `topK` | Unary | LIMIT + OFFSET | Stateful | `Stream.take` |
| `orderBy` | Unary | ORDER BY | Stateful | `Stream.mapAccum` |
| `reduce` | Unary | Aggregations | Stateful | `Stream.runFold` |
| `groupBy` | Unary | GROUP BY | Stateful | `Stream.groupByKey` |
| `consolidate` | Unary | - | Stateful | `Stream.debounce` |
| `join` | Binary | JOIN (all types) | Stateful | `Stream.mergeWith` |
| `concat` | Binary | UNION ALL | Stateless | `Stream.concat` |

## Incremental Update Propagation Sequence

```mermaid
sequenceDiagram
    participant Source as Collection
    participant Input as RootStreamBuilder
    participant Graph as D2 Graph
    participant Op1 as FilterOperator
    participant Op2 as MapOperator
    participant Op3 as OutputOperator
    participant Callback as UI Callback

    Note over Source,Callback: Incremental Update Flow

    Source->>Input: Data change (insert/update/delete)
    Input->>Input: Create MultiSet delta
    Input->>Graph: sendData(delta)
    
    Graph->>Graph: step()
    
    loop For each operator with pending work
        Graph->>Op1: run()
        Op1->>Op1: drain() input messages
        Op1->>Op1: Apply filter to each (value, mult)
        Op1->>Op2: sendData(filtered delta)
        
        Graph->>Op2: run()
        Op2->>Op2: drain() input messages
        Op2->>Op2: Apply transform to each (value, mult)
        Op2->>Op3: sendData(mapped delta)
        
        Graph->>Op3: run()
        Op3->>Op3: drain() input messages
        Op3->>Callback: emit(changes)
    end

    Note over Callback: UI re-renders with changes
```

## Join Algorithm

The join operator implements incremental join using delta processing:

```mermaid
flowchart TD
    subgraph Inputs["Input Deltas"]
        DeltaA["ΔA: Changes to left"]:::delta
        DeltaB["ΔB: Changes to right"]:::delta
    end

    subgraph State["Accumulated State"]
        IndexA["Index A (all left rows)"]:::state
        IndexB["Index B (all right rows)"]:::state
    end

    subgraph JoinLogic["Join Computation"]
        Term1["ΔA ⋈ B_old"]:::term
        Term2["A_old ⋈ ΔB"]:::term
        Term3["ΔA ⋈ ΔB"]:::term
        Union["Union all terms"]:::union
    end

    subgraph OuterHandling["Outer Join Handling"]
        LeftOuter["Left unmatched rows"]:::outer
        RightOuter["Right unmatched rows"]:::outer
        Transitions["Presence transitions"]:::outer
    end

    subgraph Output["Output"]
        Results["Join results"]:::output
    end

    DeltaA --> Term1
    IndexB --> Term1
    
    IndexA --> Term2
    DeltaB --> Term2
    
    DeltaA --> Term3
    DeltaB --> Term3
    
    Term1 --> Union
    Term2 --> Union
    Term3 --> Union
    
    Union --> Results
    
    DeltaA --> LeftOuter
    DeltaB --> RightOuter
    DeltaA --> Transitions
    DeltaB --> Transitions
    
    LeftOuter --> Results
    RightOuter --> Results
    Transitions --> Results

    %% Update state after emission
    DeltaA -.->|"append"| IndexA
    DeltaB -.->|"append"| IndexB

    %% Click events
    click JoinLogic "https://github.com/TanStack/db/blob/main/packages/db-ivm/src/operators/join.ts"

    %% Styles
    classDef delta fill:#dc2626,stroke:#b91c1c,color:#fff
    classDef state fill:#4f46e5,stroke:#3730a3,color:#fff
    classDef term fill:#059669,stroke:#047857,color:#fff
    classDef union fill:#d97706,stroke:#b45309,color:#fff
    classDef outer fill:#7c3aed,stroke:#6d28d9,color:#fff
    classDef output fill:#0891b2,stroke:#0e7490,color:#fff
```

### Join Types

| Type | Inner Results | Left Unmatched | Right Unmatched |
|------|---------------|----------------|-----------------|
| `inner` | Yes | No | No |
| `left` | Yes | Yes (with null) | No |
| `right` | Yes | No | Yes (with null) |
| `full` | Yes | Yes (with null) | Yes (with null) |
| `anti` | No | Yes (with null) | No |

## StreamBuilder and Pipe API

```mermaid
flowchart LR
    subgraph Builder["StreamBuilder API"]
        SB["StreamBuilder<T>"]:::builder
        RSB["RootStreamBuilder<T>"]:::builder
    end

    subgraph PipeChain["Pipe Chain"]
        P1["pipe(filter(...))"]:::pipe
        P2["pipe(map(...))"]:::pipe
        P3["pipe(join(...))"]:::pipe
        P4["pipe(topK(...))"]:::pipe
        P5["pipe(output(...))"]:::pipe
    end

    subgraph Types["Type Flow"]
        T1["T"]:::type
        T2["Filtered<T>"]:::type
        T3["Mapped<T>"]:::type
        T4["Joined<T>"]:::type
        T5["Limited<T>"]:::type
    end

    SB --> RSB
    RSB --> P1
    P1 --> P2
    P2 --> P3
    P3 --> P4
    P4 --> P5

    T1 --> T2
    T2 --> T3
    T3 --> T4
    T4 --> T5

    %% Click events
    click SB "https://github.com/TanStack/db/blob/main/packages/db-ivm/src/d2.ts"

    %% Styles
    classDef builder fill:#4f46e5,stroke:#3730a3,color:#fff
    classDef pipe fill:#059669,stroke:#047857,color:#fff
    classDef type fill:#7c3aed,stroke:#6d28d9,color:#fff
```

### Usage Example

```typescript
import { D2 } from '@tanstack/db-ivm'
import { filter, map, output } from '@tanstack/db-ivm'

// Create dataflow graph
const graph = new D2()

// Create input stream
const input = graph.newInput<{ id: number; value: string }>()

// Build pipeline
input.pipe(
  filter((row) => row.id > 0),
  map((row) => ({ ...row, upper: row.value.toUpperCase() })),
  output((changes) => {
    console.log('Changes:', changes)
  })
)

// Finalize and run
graph.finalize()

// Send data (incremental)
input.sendData([[{ id: 1, value: 'hello' }, 1]])
graph.run()

// Update data
input.sendData([
  [{ id: 1, value: 'hello' }, -1],  // Remove old
  [{ id: 1, value: 'world' }, 1],   // Add new
])
graph.run()
```

## File Structure

```
packages/db-ivm/src/
├── d2.ts              # D2 runtime + StreamBuilder
├── graph.ts           # DifferenceStreamWriter/Reader + Operator base classes
├── multiset.ts        # MultiSet data structure
├── indexes.ts         # Index for join state
├── types.ts           # Type definitions
├── utils.ts           # Utilities
│
├── hashing/
│   ├── index.ts       # Hash exports
│   ├── hash.ts        # Hash function
│   └── murmur.ts      # MurmurHash implementation
│
└── operators/
    ├── index.ts       # All operator exports
    ├── pipe.ts        # Pipe utility
    ├── map.ts         # Map operator
    ├── filter.ts      # Filter operator
    ├── filterBy.ts    # FilterBy operator (index-aware)
    ├── negate.ts      # Negate operator
    ├── concat.ts      # Concat operator
    ├── join.ts        # Join operator (all types)
    ├── reduce.ts      # Reduce operator
    ├── count.ts       # Count aggregation
    ├── distinct.ts    # Distinct operator
    ├── groupBy.ts     # GroupBy operator
    ├── orderBy.ts     # OrderBy operator
    ├── orderByBTree.ts # BTree-optimized ordering
    ├── topK.ts        # TopK (limit/offset)
    ├── topKWithFractionalIndex.ts # TopK with cursor support
    ├── consolidate.ts # Consolidate multiplicities
    ├── keying.ts      # Key extraction utilities
    ├── output.ts      # Output operator
    ├── tap.ts         # Tap (side effects)
    └── debug.ts       # Debug operator
```

## Effect-TS Porting Guide

### D2 Graph → Effect Runtime

```typescript
// Effect-TS equivalent
const DataflowRuntime = Effect.gen(function* () {
  const operators = yield* Ref.make<Array<Operator>>(Array.empty())
  const finalized = yield* Ref.make(false)
  
  const newInput = <T>() => Effect.gen(function* () {
    const queue = yield* Queue.unbounded<MultiSet<T>>()
    return {
      sendData: (data: MultiSet<T>) => Queue.offer(queue, data),
      reader: queue
    }
  })
  
  const step = Effect.gen(function* () {
    const ops = yield* Ref.get(operators)
    yield* Effect.forEach(ops, op => op.run())
  })
  
  const run = Effect.gen(function* () {
    yield* Effect.whileLoop({
      while: pendingWork,
      body: step
    })
  })
  
  return { newInput, step, run }
})
```

### StreamBuilder → Stream Pipeline

```typescript
// Effect-TS stream pipeline
const pipeline = Stream.fromQueue(inputQueue).pipe(
  Stream.filter(predicate),
  Stream.map(transform),
  Stream.groupBy(keyFn),
  Stream.flatMap(processGroup),
  Stream.runForEach(emitChanges)
)
```

### MultiSet → HashMap with Multiplicities

```typescript
// Effect-TS MultiSet
type MultiSet<T> = HashMap<T, number>

const consolidate = <T>(ms: MultiSet<T>): MultiSet<T> =>
  HashMap.filter(ms, (_, mult) => mult !== 0)

const union = <T>(a: MultiSet<T>, b: MultiSet<T>): MultiSet<T> =>
  HashMap.union(a, b, (m1, m2) => m1 + m2)
```

### Join Operator → Stream Merge

```typescript
// Effect-TS incremental join
const incrementalJoin = <K, V1, V2>(
  leftStream: Stream<[K, V1, number]>,
  rightStream: Stream<[K, V2, number]>
) => Effect.gen(function* () {
  const leftIndex = yield* Ref.make(HashMap.empty<K, Array<[V1, number]>>())
  const rightIndex = yield* Ref.make(HashMap.empty<K, Array<[V2, number]>>())
  
  // Process deltas and emit join results...
})
```

## Performance Characteristics

| Operation | Time Complexity | Space Complexity |
|-----------|-----------------|------------------|
| Filter | O(Δ) | O(1) |
| Map | O(Δ) | O(1) |
| Join (inner) | O(Δ × matching keys) | O(N) for indexes |
| GroupBy | O(Δ × log groups) | O(groups) |
| OrderBy | O(Δ × log N) | O(N) for sorted state |
| TopK | O(Δ × log K) | O(K) |
| Consolidate | O(Δ × log Δ) | O(distinct values) |

Where:
- Δ = size of change set (delta)
- N = total accumulated data
- K = limit value

## Related Documentation

- [ARCHITECTURE-OVERVIEW.md](./ARCHITECTURE-OVERVIEW.md) - High-level overview
- [ARCHITECTURE-QUERY.md](./ARCHITECTURE-QUERY.md) - Query compilation to IVM
- [ARCHITECTURE-CORE.md](./ARCHITECTURE-CORE.md) - Collection integration

