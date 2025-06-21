# How Operators Work Internally 🟡

## 🎯 Learning Objectives
By the end of this lesson, you will understand:
- The internal architecture of RxJS operators
- How pipeable operators are implemented
- The lifting mechanism and operator composition
- How to create custom operators
- Performance implications of operator design
- Advanced operator patterns and techniques

## 🔧 Operator Fundamentals

### What is an Operator?

An operator is a **function that takes an Observable and returns a new Observable**. Operators enable functional composition and transformation of streams.

```typescript
type OperatorFunction<T, R> = (source: Observable<T>) => Observable<R>;

// Basic operator signature
function map<T, R>(project: (value: T) => R): OperatorFunction<T, R> {
  return (source: Observable<T>) => {
    return new Observable<R>(observer => {
      // Implementation here
    });
  };
}
```

### Operator Categories

```
┌─────────────────────────────────────────┐
│              RxJS Operators             │
├─────────────────────────────────────────┤
│ Creation Operators                      │
│ ├── of, from, interval, timer           │
│ └── fromEvent, ajax, webSocket          │
├─────────────────────────────────────────┤
│ Pipeable Operators                      │
│ ├── Transformation (map, switchMap)     │
│ ├── Filtering (filter, take, skip)     │
│ ├── Combination (merge, combineLatest)  │
│ ├── Error Handling (catchError, retry) │
│ ├── Utility (tap, delay, timeout)      │
│ └── Conditional (iif, defaultIfEmpty)  │
└─────────────────────────────────────────┘
```

## 🏗️ Operator Implementation Architecture

### 1. **Basic Operator Structure**

```typescript
function customMap<T, R>(
  transform: (value: T, index: number) => R
): OperatorFunction<T, R> {
  
  return function mapOperation(source: Observable<T>): Observable<R> {
    return new Observable<R>(observer => {
      let index = 0;
      
      // Subscribe to source Observable
      const subscription = source.subscribe({
        next: (value: T) => {
          try {
            const result = transform(value, index++);
            observer.next(result);
          } catch (error) {
            observer.error(error);
          }
        },
        error: (err) => observer.error(err),
        complete: () => observer.complete()
      });
      
      // Return teardown logic
      return () => subscription.unsubscribe();
    });
  };
}

// Usage
const source$ = of(1, 2, 3);
const doubled$ = source$.pipe(
  customMap(x => x * 2)
);
```

### 2. **The Lifting Mechanism**

RxJS uses a "lifting" mechanism to optimize operator composition:

```typescript
// Simplified lifting implementation
class Observable<T> {
  source?: Observable<any>;
  operator?: Operator<any, T>;

  constructor(subscribe?: SubscribeFunction<T>) {
    if (subscribe) {
      this._subscribe = subscribe;
    }
  }

  // Lifting creates a new Observable with operator reference
  lift<R>(operator: Operator<T, R>): Observable<R> {
    const observable = new Observable<R>();
    observable.source = this;
    observable.operator = operator;
    return observable;
  }

  // Subscribe executes the operator chain
  subscribe(observer: Observer<T>): Subscription {
    if (this.operator) {
      // Execute operator on source
      return this.operator.call(observer, this.source);
    } else {
      // Direct subscription
      return this._subscribe(observer);
    }
  }
}
```

### 3. **Operator Interface**

```typescript
interface Operator<T, R> {
  call(observer: Observer<R>, source: Observable<T>): TeardownLogic;
}

// Map operator implementation using Operator interface
class MapOperator<T, R> implements Operator<T, R> {
  constructor(private project: (value: T, index: number) => R) {}

  call(observer: Observer<R>, source: Observable<T>): TeardownLogic {
    return source.subscribe(new MapSubscriber(observer, this.project));
  }
}

class MapSubscriber<T, R> extends Subscriber<T> {
  private index = 0;

  constructor(
    destination: Observer<R>,
    private project: (value: T, index: number) => R
  ) {
    super(destination);
  }

  protected _next(value: T): void {
    try {
      const result = this.project(value, this.index++);
      this.destination.next(result);
    } catch (error) {
      this.destination.error(error);
    }
  }
}
```

## 🔄 Core Operator Patterns

### 1. **Synchronous Transformation**

```typescript
// Simple value transformation
function multiply(factor: number): OperatorFunction<number, number> {
  return (source: Observable<number>) => {
    return new Observable<number>(observer => {
      return source.subscribe({
        next: value => observer.next(value * factor),
        error: err => observer.error(err),
        complete: () => observer.complete()
      });
    });
  };
}

// Marble diagram: multiply(2)
// source: ──1───2───3───|
// result: ──2───4───6───|
```

### 2. **Filtering Pattern**

```typescript
function customFilter<T>(
  predicate: (value: T, index: number) => boolean
): OperatorFunction<T, T> {
  
  return (source: Observable<T>) => {
    return new Observable<T>(observer => {
      let index = 0;
      
      return source.subscribe({
        next: (value: T) => {
          try {
            if (predicate(value, index++)) {
              observer.next(value);
            }
            // If predicate is false, we simply don't emit
          } catch (error) {
            observer.error(error);
          }
        },
        error: err => observer.error(err),
        complete: () => observer.complete()
      });
    });
  };
}

// Marble diagram: filter(x => x % 2 === 0)
// source: ──1───2───3───4───5───|
// result: ──────2───────4───────|
```

### 3. **Higher-Order Observable Pattern**

```typescript
function customSwitchMap<T, R>(
  project: (value: T) => Observable<R>
): OperatorFunction<T, R> {
  
  return (source: Observable<T>) => {
    return new Observable<R>(observer => {
      let innerSubscription: Subscription | null = null;
      
      const outerSubscription = source.subscribe({
        next: (value: T) => {
          // Cancel previous inner subscription
          if (innerSubscription) {
            innerSubscription.unsubscribe();
          }
          
          try {
            const innerObservable = project(value);
            innerSubscription = innerObservable.subscribe({
              next: innerValue => observer.next(innerValue),
              error: err => observer.error(err),
              complete: () => {
                innerSubscription = null;
                // Don't complete outer unless source completes
              }
            });
          } catch (error) {
            observer.error(error);
          }
        },
        error: err => observer.error(err),
        complete: () => {
          if (!innerSubscription) {
            observer.complete();
          } else {
            // Complete when inner completes
            const originalComplete = innerSubscription.add(() => {
              observer.complete();
            });
          }
        }
      });
      
      return () => {
        outerSubscription.unsubscribe();
        if (innerSubscription) {
          innerSubscription.unsubscribe();
        }
      };
    });
  };
}
```

### 4. **Stateful Operator Pattern**

```typescript
function customScan<T, R>(
  accumulator: (acc: R, value: T, index: number) => R,
  seed: R
): OperatorFunction<T, R> {
  
  return (source: Observable<T>) => {
    return new Observable<R>(observer => {
      let acc = seed;
      let index = 0;
      
      return source.subscribe({
        next: (value: T) => {
          try {
            acc = accumulator(acc, value, index++);
            observer.next(acc);
          } catch (error) {
            observer.error(error);
          }
        },
        error: err => observer.error(err),
        complete: () => observer.complete()
      });
    });
  };
}

// Marble diagram: scan((acc, x) => acc + x, 0)
// source: ──1───2───3───4───|
// result: ──1───3───6───10──|
```

## 🚀 Advanced Operator Implementations

### 1. **debounceTime Implementation**

```typescript
function customDebounceTime<T>(
  dueTime: number,
  scheduler: SchedulerLike = asyncScheduler
): OperatorFunction<T, T> {
  
  return (source: Observable<T>) => {
    return new Observable<T>(observer => {
      let lastValue: T;
      let hasValue = false;
      let timeout: Subscription | null = null;
      
      const subscription = source.subscribe({
        next: (value: T) => {
          lastValue = value;
          hasValue = true;
          
          // Cancel previous timeout
          if (timeout) {
            timeout.unsubscribe();
          }
          
          // Schedule new emission
          timeout = scheduler.schedule(() => {
            if (hasValue) {
              observer.next(lastValue);
              hasValue = false;
            }
          }, dueTime);
        },
        error: err => observer.error(err),
        complete: () => {
          // Emit final value if pending
          if (timeout) {
            timeout.unsubscribe();
          }
          if (hasValue) {
            observer.next(lastValue);
          }
          observer.complete();
        }
      });
      
      return () => {
        subscription.unsubscribe();
        if (timeout) {
          timeout.unsubscribe();
        }
      };
    });
  };
}

// Marble diagram: debounceTime(3)
// source: ──a─b─c─────d───e─f───|
// result: ─────────c─────d─────f|
```

### 2. **combineLatest Implementation**

```typescript
function customCombineLatest<T extends readonly unknown[]>(
  ...observables: [...ObservableInputTuple<T>]
): Observable<T> {
  
  return new Observable<T>(observer => {
    const values: (T[number] | typeof EMPTY_VALUE)[] = 
      new Array(observables.length).fill(EMPTY_VALUE);
    let completedCount = 0;
    let hasEmittedOnce = false;
    
    const subscriptions = observables.map((obs, index) => {
      return obs.subscribe({
        next: (value: T[number]) => {
          values[index] = value;
          
          // Emit only if all observables have emitted at least once
          if (values.every(v => v !== EMPTY_VALUE)) {
            if (!hasEmittedOnce) {
              hasEmittedOnce = true;
            }
            observer.next([...values] as T);
          }
        },
        error: err => observer.error(err),
        complete: () => {
          completedCount++;
          if (completedCount === observables.length) {
            observer.complete();
          }
        }
      });
    });
    
    return () => {
      subscriptions.forEach(sub => sub.unsubscribe());
    };
  });
}
```

### 3. **Custom Error Handling Operator**

```typescript
function customRetry<T>(
  count: number = -1,
  delay: number = 0
): OperatorFunction<T, T> {
  
  return (source: Observable<T>) => {
    return new Observable<T>(observer => {
      let retryCount = 0;
      let subscription: Subscription;
      
      const subscribe = () => {
        subscription = source.subscribe({
          next: value => observer.next(value),
          error: err => {
            if (count < 0 || retryCount < count) {
              retryCount++;
              
              if (delay > 0) {
                // Retry after delay
                setTimeout(subscribe, delay);
              } else {
                // Retry immediately
                subscribe();
              }
            } else {
              // Max retries reached
              observer.error(err);
            }
          },
          complete: () => observer.complete()
        });
      };
      
      subscribe();
      
      return () => {
        if (subscription) {
          subscription.unsubscribe();
        }
      };
    });
  };
}
```

## 🎨 Creating Custom Operators

### 1. **Simple Custom Operator**

```typescript
// Custom operator that logs values
function log<T>(label: string = 'LOG'): OperatorFunction<T, T> {
  return (source: Observable<T>) => {
    return new Observable<T>(observer => {
      console.log(`[${label}] Subscribed`);
      
      const subscription = source.subscribe({
        next: value => {
          console.log(`[${label}] Next:`, value);
          observer.next(value);
        },
        error: err => {
          console.log(`[${label}] Error:`, err);
          observer.error(err);
        },
        complete: () => {
          console.log(`[${label}] Complete`);
          observer.complete();
        }
      });
      
      return () => {
        console.log(`[${label}] Unsubscribed`);
        subscription.unsubscribe();
      };
    });
  };
}

// Usage
source$.pipe(
  log('Debug'),
  map(x => x * 2),
  log('After map')
).subscribe();
```

### 2. **Conditional Operator**

```typescript
function skipUntil<T>(
  condition: (value: T) => boolean
): OperatorFunction<T, T> {
  
  return (source: Observable<T>) => {
    return new Observable<T>(observer => {
      let shouldEmit = false;
      
      return source.subscribe({
        next: (value: T) => {
          if (!shouldEmit && condition(value)) {
            shouldEmit = true;
          }
          
          if (shouldEmit) {
            observer.next(value);
          }
        },
        error: err => observer.error(err),
        complete: () => observer.complete()
      });
    });
  };
}

// Usage
numbers$.pipe(
  skipUntil(x => x > 5)
).subscribe(console.log);
```

### 3. **Buffering Operator**

```typescript
function bufferCount<T>(
  bufferSize: number,
  startBufferEvery: number = bufferSize
): OperatorFunction<T, T[]> {
  
  return (source: Observable<T>) => {
    return new Observable<T[]>(observer => {
      const buffers: T[][] = [];
      let count = 0;
      
      return source.subscribe({
        next: (value: T) => {
          // Create new buffer if needed
          if (count % startBufferEvery === 0) {
            buffers.push([]);
          }
          
          // Add value to all active buffers
          buffers.forEach(buffer => {
            if (buffer.length < bufferSize) {
              buffer.push(value);
            }
          });
          
          // Emit and remove completed buffers
          for (let i = buffers.length - 1; i >= 0; i--) {
            if (buffers[i].length === bufferSize) {
              observer.next(buffers[i]);
              buffers.splice(i, 1);
            }
          }
          
          count++;
        },
        error: err => observer.error(err),
        complete: () => {
          // Emit remaining buffers
          buffers.forEach(buffer => {
            if (buffer.length > 0) {
              observer.next(buffer);
            }
          });
          observer.complete();
        }
      });
    });
  };
}
```

## ⚡ Performance Considerations

### 1. **Operator Fusion**

```typescript
// RxJS can optimize operator chains
source$.pipe(
  map(x => x + 1),
  map(x => x * 2),
  map(x => x - 1)
);

// Internally optimized to:
source$.pipe(
  map(x => (x + 1) * 2 - 1)
);
```

### 2. **Memory Efficient Operators**

```typescript
function memoryEfficientTake<T>(count: number): OperatorFunction<T, T> {
  return (source: Observable<T>) => {
    return new Observable<T>(observer => {
      let taken = 0;
      
      const subscription = source.subscribe({
        next: value => {
          if (taken < count) {
            observer.next(value);
            taken++;
            
            if (taken === count) {
              observer.complete();
              subscription.unsubscribe(); // Early termination
            }
          }
        },
        error: err => observer.error(err),
        complete: () => observer.complete()
      });
      
      return () => subscription.unsubscribe();
    });
  };
}
```

### 3. **Avoiding Memory Leaks**

```typescript
function safeOperator<T, R>(
  transform: (value: T) => R
): OperatorFunction<T, R> {
  
  return (source: Observable<T>) => {
    return new Observable<R>(observer => {
      let isUnsubscribed = false;
      
      const subscription = source.subscribe({
        next: value => {
          if (!isUnsubscribed) {
            try {
              const result = transform(value);
              observer.next(result);
            } catch (error) {
              observer.error(error);
            }
          }
        },
        error: err => {
          if (!isUnsubscribed) {
            observer.error(err);
          }
        },
        complete: () => {
          if (!isUnsubscribed) {
            observer.complete();
          }
        }
      });
      
      return () => {
        isUnsubscribed = true;
        subscription.unsubscribe();
      };
    });
  };
}
```

## 🧪 Testing Custom Operators

```typescript
import { TestScheduler } from 'rxjs/testing';

describe('Custom Operators', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should implement custom map correctly', () => {
    testScheduler.run(({ cold, expectObservable }) => {
      const source$ = cold('--a--b--c--|', { a: 1, b: 2, c: 3 });
      const expected =     '--x--y--z--|';
      
      const result$ = source$.pipe(
        customMap(x => x * 2)
      );
      
      expectObservable(result$).toBe(expected, {
        x: 2, y: 4, z: 6
      });
    });
  });

  it('should handle errors in custom operators', () => {
    testScheduler.run(({ cold, expectObservable }) => {
      const source$ = cold('--a--b--c--|', { a: 1, b: 2, c: 3 });
      const expected =     '--x--#';
      
      const result$ = source$.pipe(
        customMap(x => {
          if (x === 2) throw new Error('Test error');
          return x * 2;
        })
      );
      
      expectObservable(result$).toBe(expected, 
        { x: 2 }, 
        new Error('Test error')
      );
    });
  });
});
```

## 🎯 Quick Assessment

**Questions:**

1. What does the lifting mechanism do in RxJS?
2. How do you handle state in custom operators?
3. What's the difference between pipeable and creation operators?
4. How do you ensure memory efficiency in custom operators?

**Answers:**

1. **Lifting** optimizes operator composition by deferring execution until subscription
2. **State management** through closure variables and proper cleanup in teardown
3. **Pipeable**: Transform existing Observables. **Creation**: Create new Observables from scratch
4. **Early termination**, **proper cleanup**, and **avoiding unnecessary computations**

## 🌟 Key Takeaways

- **Operators** are functions that transform Observables into new Observables
- **Lifting mechanism** optimizes operator composition and execution
- **Custom operators** follow consistent patterns for transformation, filtering, and combination
- **State management** in operators requires careful handling of closure variables
- **Memory efficiency** comes from proper cleanup and early termination
- **Testing** custom operators ensures reliability and correctness
- **Performance** considerations include operator fusion and memory management

## 🚀 Next Steps

Congratulations! You've completed **Module 2: Core RxJS Concepts & Internals**. You now understand:

- ✅ How to create Observables with various methods
- ✅ Observable lifecycle and internal workings
- ✅ Marble diagram reading and creation
- ✅ Subscription management and teardown logic
- ✅ Hot vs Cold Observable concepts
- ✅ Subjects and multicasting internals
- ✅ Schedulers and execution context
- ✅ How operators work internally

**Next Module**: [RxJS Operators Deep Dive](../03-operators/01-operator-categories.md) 🟢

In Module 3, you'll master all RxJS operators with comprehensive marble diagrams, practical examples, and real-world applications.

---

🎉 **Exceptional work!** You now have deep knowledge of RxJS internals and can create custom operators with confidence. You're ready to master the extensive RxJS operator library!
