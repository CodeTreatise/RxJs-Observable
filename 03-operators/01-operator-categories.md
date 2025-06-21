# RxJS Operator Categories & Implementation 🟢

## 🎯 Learning Objectives
By the end of this lesson, you will understand:
- The complete taxonomy of RxJS operators
- How operators are categorized and why
- When to use each category of operators
- The implementation patterns behind different operator types
- How to choose the right operator for any scenario

## 📚 Complete RxJS Operator Taxonomy

### Overview of All Operator Categories

```
┌─────────────────────────────────────────────────────────────┐
│                    RxJS Operators                           │
├─────────────────────────────────────────────────────────────┤
│ 1. Creation Operators (Functions)                          │
│    └── Create new Observables from scratch                 │
├─────────────────────────────────────────────────────────────┤
│ 2. Transformation Operators (Pipeable)                     │
│    └── Transform emitted values                            │
├─────────────────────────────────────────────────────────────┤
│ 3. Filtering Operators (Pipeable)                          │
│    └── Filter which values get emitted                     │
├─────────────────────────────────────────────────────────────┤
│ 4. Combination Operators (Functions + Pipeable)            │
│    └── Combine multiple Observables                        │
├─────────────────────────────────────────────────────────────┤
│ 5. Error Handling Operators (Pipeable)                     │
│    └── Handle and recover from errors                      │
├─────────────────────────────────────────────────────────────┤
│ 6. Utility Operators (Pipeable)                            │
│    └── Side effects, debugging, and utilities              │
├─────────────────────────────────────────────────────────────┤
│ 7. Conditional Operators (Functions + Pipeable)            │
│    └── Conditional logic and branching                     │
├─────────────────────────────────────────────────────────────┤
│ 8. Mathematical Operators (Pipeable)                       │
│    └── Mathematical and aggregate operations               │
├─────────────────────────────────────────────────────────────┤
│ 9. Multicasting Operators (Pipeable)                       │
│    └── Share executions among multiple subscribers         │
└─────────────────────────────────────────────────────────────┘
```

## 🏗️ 1. Creation Operators

**Purpose**: Create new Observables from various sources

### Core Creation Operators

```typescript
import { 
  of, from, interval, timer, range,
  fromEvent, ajax, defer, empty, 
  never, throwError, generate 
} from 'rxjs';

// Static values
const static$ = of(1, 2, 3);
// ──1──2──3──|

// From arrays/iterables
const array$ = from([1, 2, 3]);
// ──1──2──3──|

// Time-based
const timer$ = interval(1000);
// ──0──1──2──3──...

// Events
const clicks$ = fromEvent(document, 'click');
// ──click──click──click──...

// HTTP requests
const http$ = ajax('/api/data');
// ──────response──|

// Lazy creation
const lazy$ = defer(() => of(Math.random()));
// ──random──| (new value each subscription)
```

### Marble Diagram Examples

```
of(1, 2, 3):
──1──2──3──|

interval(1000):
──0──1──2──3──4──...
  1s 1s 1s 1s 1s

fromEvent(button, 'click'):
──────c────c──c─────c──...
     click events

timer(2000, 1000):
────────0──1──2──3──...
  2s    1s 1s 1s 1s
```

## 🔄 2. Transformation Operators

**Purpose**: Transform values emitted by the source Observable

### Core Transformation Operators

```typescript
import { 
  map, switchMap, mergeMap, concatMap, exhaustMap,
  scan, reduce, buffer, bufferTime, 
  window, windowTime, groupBy 
} from 'rxjs/operators';

// Simple transformation
source$.pipe(map(x => x * 2))
// source: ──1───2───3───|
// result: ──2───4───6───|

// Higher-order mapping
source$.pipe(switchMap(x => timer(1000).pipe(map(() => x))))
// source: ──a─────b─c───|
//            \     \ \
//             \     X \
//              1s     1s
// result: ──────a─────c|

// Accumulation
source$.pipe(scan((acc, x) => acc + x, 0))
// source: ──1───2───3───|
// result: ──1───3───6───|
```

### Flattening Strategy Comparison

```typescript
// switchMap - Cancel previous inner Observable
source$.pipe(switchMap(x => inner$(x)))
// source: ──a───b───c───|
//           \    \    \
//            i1   X    i3
// result: ────i1──────i3|

// mergeMap - Run all inner Observables concurrently  
source$.pipe(mergeMap(x => inner$(x)))
// source: ──a───b───c───|
//           \    \    \
//            i1   i2   i3
// result: ────i1─i2─i3──|

// concatMap - Run inner Observables sequentially
source$.pipe(concatMap(x => inner$(x))) 
// source: ──a───b───c───|
//           \         \
//            i1──wait──i2──i3
// result: ────i1──────i2──i3|

// exhaustMap - Ignore new values while inner Observable is active
source$.pipe(exhaustMap(x => inner$(x)))
// source: ──a───b───c───|
//           \    X    X
//            i1  ignored
// result: ────i1─────────|
```

## 🔍 3. Filtering Operators

**Purpose**: Filter which values pass through the Observable

### Core Filtering Operators

```typescript
import { 
  filter, take, skip, first, last,
  takeUntil, takeWhile, skipUntil, skipWhile,
  distinct, distinctUntilChanged, debounceTime,
  throttleTime, sample, audit 
} from 'rxjs/operators';

// Value-based filtering
source$.pipe(filter(x => x > 5))
// source: ──1───8───3───9───2───|
// result: ──────8───────9───────|

// Count-based filtering
source$.pipe(take(3))
// source: ──a───b───c───d───e───|
// result: ──a───b───c───|

// Time-based filtering
source$.pipe(debounceTime(300))
// source: ──a─b─c─────d───e─f───|
// result: ─────────c─────d─────f|

// Condition-based filtering
source$.pipe(takeWhile(x => x < 5))
// source: ──1───2───6───3───4───|
// result: ──1───2───|
```

### Time-based Filtering Comparison

```typescript
// debounceTime - Emit after silence period
source$.pipe(debounceTime(300))
// source: ──a─b─c─────d───e─f───|
// result: ─────────c─────d─────f|

// throttleTime - Emit first, then ignore for period
source$.pipe(throttleTime(300))
// source: ──a─b─c─────d───e─f───|  
// result: ──a─────────d───e─────|

// sample - Emit latest value at intervals
source$.pipe(sample(interval(300)))
// source: ──a─b─c─d─e─f─g─h─i───|
// timer:  ─────────T─────────T───
// result: ─────────e─────────i───|

// audit - Like throttleTime but emits last value
source$.pipe(audit(() => timer(300)))
// source: ──a─b─c─────d───e─f───|
// result: ─────────c─────d─────f|
```

## 🤝 4. Combination Operators

**Purpose**: Combine multiple Observables into one

### Core Combination Operators

```typescript
import { 
  merge, concat, combineLatest, zip,
  forkJoin, race, startWith, withLatestFrom 
} from 'rxjs';
import { mergeWith, concatWith } from 'rxjs/operators';

// Merge - Combine concurrently
merge(obs1$, obs2$)
// obs1: ──a───c───e───|
// obs2: ────b───d───f─|  
// result: ──a─b─c─d─e─f|

// Concat - Combine sequentially
concat(obs1$, obs2$)
// obs1: ──a───b───|
// obs2: ──────────c───d───|
// result: ──a───b───c───d───|

// CombineLatest - Emit when any source emits
combineLatest([obs1$, obs2$])
// obs1: ──a─────c─────e─|
// obs2: ────b─────d─────f|
// result: ────[a,b]─[c,b]─[c,d]─[e,d]─[e,f]|

// Zip - Wait for all sources to emit
zip(obs1$, obs2$)
// obs1: ──a─────c─────e─|
// obs2: ────b─────d─────f|
// result: ────[a,b]───[c,d]───[e,f]|
```

### Advanced Combination Patterns

```typescript
// Race - First to emit wins
race(slow$, fast$)
// slow: ────────a───b───|
// fast: ──x───y───z───|
// result: ──x───y───z───| (fast wins)

// ForkJoin - Wait for all to complete, emit last values
forkJoin([obs1$, obs2$, obs3$])
// obs1: ──a───b───c───|
// obs2: ────x───y─────|  
// obs3: ──────m───n───|
// result: ─────────────[c,y,n]|

// WithLatestFrom - Combine with latest from other source
main$.pipe(withLatestFrom(other$))
// main:  ──a─────c─────e─|
// other: ────b─────d─────|
// result: ────────[c,b]─[e,d]|
```

## 🚨 5. Error Handling Operators

**Purpose**: Handle errors and provide recovery mechanisms

### Core Error Handling Operators

```typescript
import { 
  catchError, retry, retryWhen, 
  throwIfEmpty, timeout, onErrorResumeNext 
} from 'rxjs/operators';

// Catch and replace with fallback
source$.pipe(catchError(err => of('fallback')))
// source: ──a───b───X
// result: ──a───b───fallback|

// Retry on error
source$.pipe(retry(3))
// source: ──a───X
// retry1: ──────a───X
// retry2: ──────────a───X  
// retry3: ──────────────a───X

// Conditional retry
source$.pipe(
  retryWhen(errors => 
    errors.pipe(
      delay(1000),
      take(3)
    )
  )
)
// Retry with delay, max 3 times

// Timeout handling
source$.pipe(
  timeout(5000),
  catchError(err => of('timeout'))
)
// Emit 'timeout' if source doesn't emit within 5 seconds
```

### Error Recovery Patterns

```typescript
// Graceful degradation
const robustData$ = primarySource$.pipe(
  timeout(3000),
  catchError(() => secondarySource$),
  catchError(() => of(defaultData))
);

// Exponential backoff retry
const withBackoff$ = source$.pipe(
  retryWhen(errors =>
    errors.pipe(
      scan((retryCount, err) => {
        if (retryCount >= 3) throw err;
        return retryCount + 1;
      }, 0),
      delay(retryCount => 1000 * Math.pow(2, retryCount))
    )
  )
);
```

## 🛠️ 6. Utility Operators

**Purpose**: Side effects, debugging, and utility functions

### Core Utility Operators

```typescript
import { 
  tap, delay, delayWhen, finalize,
  repeat, repeatWhen, share, shareReplay,
  toArray, materialize, dematerialize 
} from 'rxjs/operators';

// Side effects (debugging, logging)
source$.pipe(
  tap(value => console.log('Debug:', value)),
  tap({
    next: value => console.log('Next:', value),
    error: err => console.log('Error:', err),
    complete: () => console.log('Complete')
  })
)

// Timing control
source$.pipe(
  delay(1000), // Delay all emissions by 1s
  delayWhen(value => timer(value * 100)) // Variable delay
)

// Cleanup and finalization
source$.pipe(
  finalize(() => console.log('Stream finalized'))
)

// Sharing and caching
const shared$ = expensive$.pipe(
  shareReplay({ bufferSize: 1, refCount: true })
);
```

### Debugging and Development Utilities

```typescript
// Advanced debugging
const debugOperator = <T>(label: string) => 
  tap<T>({
    next: value => console.log(`[${label}] Next:`, value),
    error: err => console.log(`[${label}] Error:`, err),
    complete: () => console.log(`[${label}] Complete`),
    subscribe: () => console.log(`[${label}] Subscribe`),
    unsubscribe: () => console.log(`[${label}] Unsubscribe`)
  });

// Performance monitoring
const withTiming = <T>(label: string) => (source: Observable<T>) =>
  defer(() => {
    const start = performance.now();
    return source.pipe(
      finalize(() => {
        const duration = performance.now() - start;
        console.log(`[${label}] Duration: ${duration}ms`);
      })
    );
  });
```

## 🔀 7. Conditional Operators

**Purpose**: Conditional logic and branching

### Core Conditional Operators

```typescript
import { 
  iif, defaultIfEmpty, every, find,
  findIndex, isEmpty 
} from 'rxjs';

// Conditional Observable creation
const conditional$ = iif(
  () => Math.random() > 0.5,
  of('heads'),
  of('tails')
);

// Default values for empty streams
source$.pipe(
  filter(x => x > 100), // Might emit nothing
  defaultIfEmpty('no matches')
)
// If no values pass filter, emit 'no matches'

// Stream validation
source$.pipe(
  every(x => x > 0) // true if all values > 0
)
// source: ──1───2───3───|
// result: ──────────────true|

// Finding values
source$.pipe(
  find(x => x > 5) // First value > 5
)
// source: ──1───8───3───9───|
// result: ──────8|
```

## 🧮 8. Mathematical Operators

**Purpose**: Mathematical operations and aggregations

### Core Mathematical Operators

```typescript
import { 
  count, max, min, reduce, scan,
  sum, average 
} from 'rxjs/operators';

// Aggregation operators
source$.pipe(count())
// source: ──a───b───c───|
// result: ──────────────3|

source$.pipe(reduce((acc, x) => acc + x, 0))
// source: ──1───2───3───|  
// result: ──────────────6|

source$.pipe(scan((acc, x) => acc + x, 0))
// source: ──1───2───3───|
// result: ──1───3───6───|

// Statistical operators
numbers$.pipe(max())
// numbers: ──3───1───4───2───|
// result:  ─────────────────4|

numbers$.pipe(min())  
// numbers: ──3───1───4───2───|
// result:  ─────────────────1|
```

### Custom Mathematical Operators

```typescript
// Custom average operator
const average = () => (source: Observable<number>) =>
  source.pipe(
    reduce(
      (acc, value, index) => ({
        sum: acc.sum + value,
        count: index + 1
      }),
      { sum: 0, count: 0 }
    ),
    map(acc => acc.sum / acc.count)
  );

// Moving average operator
const movingAverage = (windowSize: number) => 
  (source: Observable<number>) =>
    source.pipe(
      scan((acc, value) => {
        const newWindow = [...acc, value];
        if (newWindow.length > windowSize) {
          newWindow.shift();
        }
        return newWindow;
      }, [] as number[]),
      map(window => window.reduce((sum, val) => sum + val, 0) / window.length)
    );
```

## 📡 9. Multicasting Operators

**Purpose**: Share Observable executions among multiple subscribers

### Core Multicasting Operators

```typescript
import { 
  share, shareReplay, publish, multicast,
  refCount, connect 
} from 'rxjs/operators';

// Basic sharing
const shared$ = expensive$.pipe(share());

// Replay for late subscribers  
const cached$ = api$.pipe(
  shareReplay({ bufferSize: 1, refCount: true })
);

// Manual multicasting control
const multicasted$ = source$.pipe(
  multicast(new Subject()),
  refCount()
);

// Publishing with manual connection
const published$ = source$.pipe(publish());
const connection = published$.connect();
// Later: connection.unsubscribe();
```

## 🎯 Operator Selection Guide

### Decision Tree for Operator Selection

```
What do you want to do?
├─ Create new Observable?
│  └─ Use Creation Operators (of, from, interval, etc.)
├─ Transform values?
│  ├─ Simple transformation? → map
│  ├─ Async transformation? → switchMap, mergeMap, concatMap
│  └─ Accumulate values? → scan, reduce
├─ Filter values?
│  ├─ By condition? → filter
│  ├─ By count? → take, skip
│  ├─ By time? → debounceTime, throttleTime
│  └─ By uniqueness? → distinct, distinctUntilChanged
├─ Combine Observables?
│  ├─ Merge emissions? → merge
│  ├─ Latest from each? → combineLatest
│  ├─ Pair emissions? → zip
│  └─ Sequential? → concat
├─ Handle errors?
│  ├─ Provide fallback? → catchError
│  ├─ Retry? → retry, retryWhen
│  └─ Timeout? → timeout
├─ Side effects?
│  └─ Use tap, finalize
└─ Share execution?
   └─ Use share, shareReplay
```

### Common Operator Combinations

```typescript
// Search autocomplete pattern
searchInput$.pipe(
  debounceTime(300),           // Wait for pause
  distinctUntilChanged(),      // Skip duplicates
  switchMap(query =>           // Cancel previous search
    searchAPI(query).pipe(
      catchError(() => of([])) // Handle errors gracefully
    )
  )
);

// Retry with exponential backoff
apiCall$.pipe(
  retryWhen(errors =>
    errors.pipe(
      scan((retryCount, err) => {
        if (retryCount >= 3) throw err;
        return retryCount + 1;
      }, 0),
      delay(retryCount => 1000 * Math.pow(2, retryCount))
    )
  ),
  catchError(err => of(defaultValue))
);

// Polling with error handling
interval(5000).pipe(
  switchMap(() => 
    fetchData().pipe(
      retry(2),
      catchError(err => {
        console.error('Poll failed:', err);
        return EMPTY;
      })
    )
  )
);
```

## 📊 Performance Considerations

### Operator Performance Characteristics

| Operator Category | Memory Usage | CPU Usage | Notes |
|------------------|--------------|-----------|-------|
| **Creation** | Low | Low | Minimal overhead |
| **Transformation** | Variable | Medium | Depends on complexity |
| **Filtering** | Low | Low | Can reduce downstream work |
| **Combination** | Medium-High | Medium | Manages multiple subscriptions |
| **Error Handling** | Low | Low | Lightweight wrappers |
| **Utility** | Low | Variable | Depends on side effects |
| **Mathematical** | Medium | Medium | May buffer values |
| **Multicasting** | Medium | Low | Shares execution cost |

### Optimization Tips

```typescript
// ✅ Good: Early filtering reduces work
source$.pipe(
  filter(x => x > 0),    // Filter early
  map(x => expensiveTransform(x)),
  take(10)
);

// ❌ Bad: Late filtering wastes computation
source$.pipe(
  map(x => expensiveTransform(x)),
  filter(x => x > 0),    // Filter after expensive work
  take(10)
);

// ✅ Good: Use shareReplay for expensive operations
const expensive$ = source$.pipe(
  switchMap(x => heavyComputation(x)),
  shareReplay(1)
);

// ✅ Good: Combine operators where possible
source$.pipe(
  map(x => x * 2),
  map(x => x + 1)
);
// Better as:
source$.pipe(
  map(x => x * 2 + 1)
);
```

## 🎯 Quick Assessment

**Questions:**

1. When would you use `switchMap` vs `mergeMap` vs `concatMap`?
2. What's the difference between `combineLatest` and `zip`?
3. How do you choose between `debounceTime` and `throttleTime`?
4. When should you use `shareReplay`?

**Answers:**

1. **switchMap**: Cancel previous, **mergeMap**: Run concurrent, **concatMap**: Sequential
2. **combineLatest**: Emits when any source emits, **zip**: Waits for all sources
3. **debounceTime**: Emit after silence, **throttleTime**: Emit first then ignore
4. **shareReplay**: When expensive operations need to be shared and cached

## 🌟 Key Takeaways

- **Nine main categories** cover all RxJS operator types
- **Creation operators** build Observables from various sources
- **Transformation operators** reshape and project data
- **Filtering operators** control which values pass through
- **Combination operators** merge multiple streams
- **Error handling** provides robust failure recovery
- **Utility operators** add debugging and side effects
- **Mathematical operators** perform aggregations
- **Multicasting operators** optimize shared executions
- **Operator selection** follows predictable patterns
- **Performance optimization** comes from proper operator ordering

## 🚀 Next Steps

Now that you understand how operators are categorized and when to use each type, you're ready to dive deep into **Creation Operators** with comprehensive marble diagrams and real-world examples.

**Next Lesson**: [Creation Operators with Marble Diagrams](./02-creation-operators.md) 🟢

---

🎉 **Excellent foundation!** You now have a complete mental model of the RxJS operator ecosystem. This knowledge will guide you in choosing the perfect operator for any reactive programming challenge!
