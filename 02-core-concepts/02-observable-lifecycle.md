# Observable Lifecycle & Internal Workings 🟡

## 🎯 Learning Objectives
By the end of this lesson, you will understand:
- The complete Observable lifecycle from creation to disposal
- Internal execution flow and timing
- How subscriptions work internally
- Observable state management
- Memory management and garbage collection
- Performance implications of lifecycle management

## 🔄 Observable Lifecycle Overview

### The Complete Lifecycle

```
┌─────────────────────────────────────────────────────────┐
│                Observable Lifecycle                     │
├─────────────────────────────────────────────────────────┤
│ 1. Creation     → Observable function defined           │
│ 2. Subscription → Observer subscribes, execution starts │
│ 3. Execution    → Values emitted, operators applied     │
│ 4. Termination  → Complete, error, or unsubscribe      │
│ 5. Cleanup      → Resources freed, teardown executed   │
└─────────────────────────────────────────────────────────┘
```

### Lifecycle States

```typescript
enum ObservableState {
  CREATED = 'created',      // Observable defined but not subscribed
  EXECUTING = 'executing',  // Active subscription, emitting values
  COMPLETED = 'completed',  // Successfully completed
  ERROR = 'error',         // Terminated with error
  UNSUBSCRIBED = 'unsubscribed' // Manually unsubscribed
}
```

## 🏗️ Phase 1: Creation

When you create an Observable, you're defining a **recipe** for data production, not executing it.

### Creation Examples

```typescript
import { Observable, of, interval } from 'rxjs';

// 1. Observable is just a function definition
console.log('--- Creation Phase ---');

const observable$ = new Observable(observer => {
  console.log('❌ This does NOT run during creation');
  observer.next('value');
});

console.log('Observable created, but nothing executed');

// 2. Creation operators also don't execute
const numbers$ = of(1, 2, 3);
console.log('of() Observable created, values not emitted yet');

const timer$ = interval(1000);
console.log('interval() Observable created, timer not started');
```

### Internal Creation Structure

```typescript
// Simplified Observable implementation
class Observable<T> {
  constructor(
    private _subscribe: (observer: Observer<T>) => TeardownLogic
  ) {
    // Creation phase - just storing the subscription function
    console.log('Observable created with subscription function');
  }

  subscribe(observer: Observer<T>): Subscription {
    console.log('Subscription starting...');
    
    // This is where execution actually begins
    const teardown = this._subscribe(observer);
    
    return new Subscription(() => {
      console.log('Cleanup executing...');
      if (typeof teardown === 'function') {
        teardown();
      }
    });
  }
}
```

## 🎬 Phase 2: Subscription

Subscription is when the Observable **springs to life** and begins execution.

### Subscription Process

```typescript
import { Observable } from 'rxjs';

const lifecycle$ = new Observable(observer => {
  console.log('🚀 EXECUTION STARTED');
  
  // Track subscription state
  let isActive = true;
  
  console.log('📡 Setting up data source');
  
  // Simulate async data source
  const intervalId = setInterval(() => {
    if (isActive) {
      const value = Math.random();
      console.log('📤 Emitting value:', value);
      observer.next(value);
      
      // Complete after 5 emissions
      if (value > 0.8) {
        console.log('✅ Completing Observable');
        observer.complete();
        isActive = false;
      }
    }
  }, 1000);
  
  // Cleanup function (teardown logic)
  return () => {
    console.log('🧹 CLEANUP STARTED');
    isActive = false;
    clearInterval(intervalId);
    console.log('🧹 CLEANUP COMPLETED');
  };
});

console.log('=== Before Subscription ===');

const subscription = lifecycle$.subscribe({
  next: value => console.log('👀 Observer received:', value),
  error: err => console.log('❌ Observer error:', err),
  complete: () => console.log('🏁 Observer complete')
});

console.log('=== After Subscription ===');

// Unsubscribe after 3 seconds
setTimeout(() => {
  console.log('=== Unsubscribing ===');
  subscription.unsubscribe();
}, 3000);
```

### Multiple Subscriptions

```typescript
import { Observable } from 'rxjs';

let subscriptionCount = 0;

const multiSub$ = new Observable(observer => {
  const id = ++subscriptionCount;
  console.log(`🔥 Execution ${id} started`);
  
  observer.next(`Value from execution ${id}`);
  
  return () => {
    console.log(`🧹 Cleanup ${id} executed`);
  };
});

console.log('--- Multiple Subscriptions ---');

const sub1 = multiSub$.subscribe(value => console.log('Sub1:', value));
const sub2 = multiSub$.subscribe(value => console.log('Sub2:', value));
const sub3 = multiSub$.subscribe(value => console.log('Sub3:', value));

// Each subscription creates a separate execution!
// Output:
// 🔥 Execution 1 started
// Sub1: Value from execution 1
// 🔥 Execution 2 started  
// Sub2: Value from execution 2
// 🔥 Execution 3 started
// Sub3: Value from execution 3
```

## ⚡ Phase 3: Execution

During execution, the Observable emits values and operators process them.

### Execution Flow

```typescript
import { Observable } from 'rxjs';
import { map, filter, tap } from 'rxjs/operators';

const source$ = new Observable<number>(observer => {
  console.log('📡 Source execution started');
  
  const values = [1, 2, 3, 4, 5];
  
  values.forEach((value, index) => {
    setTimeout(() => {
      console.log(`📤 Source emitting: ${value}`);
      observer.next(value);
      
      if (index === values.length - 1) {
        console.log('✅ Source completing');
        observer.complete();
      }
    }, index * 500);
  });
  
  return () => console.log('🧹 Source cleanup');
});

const processed$ = source$.pipe(
  tap(value => console.log(`👁️  tap: saw ${value}`)),
  filter(value => {
    const keep = value % 2 === 0;
    console.log(`🔍 filter: ${value} ${keep ? 'kept' : 'filtered out'}`);
    return keep;
  }),
  map(value => {
    const mapped = value * 10;
    console.log(`🔄 map: ${value} → ${mapped}`);
    return mapped;
  }),
  tap(value => console.log(`👁️  final tap: ${value}`))
);

console.log('--- Starting execution ---');

processed$.subscribe({
  next: value => console.log(`🎯 Final result: ${value}`),
  complete: () => console.log('🏁 Pipeline completed')
});
```

### Execution Timing

```typescript
import { Observable, of } from 'rxjs';
import { delay, tap } from 'rxjs/operators';

// Synchronous execution
console.log('--- Synchronous Execution ---');
console.log('Before sync subscription');

of(1, 2, 3).pipe(
  tap(value => console.log(`Sync tap: ${value}`))
).subscribe(value => console.log(`Sync result: ${value}`));

console.log('After sync subscription');

// Asynchronous execution
console.log('\n--- Asynchronous Execution ---');
console.log('Before async subscription');

of(1, 2, 3).pipe(
  delay(100),
  tap(value => console.log(`Async tap: ${value}`))
).subscribe(value => console.log(`Async result: ${value}`));

console.log('After async subscription');

// Output demonstrates execution timing:
// Before sync subscription
// Sync tap: 1
// Sync result: 1
// Sync tap: 2
// Sync result: 2
// Sync tap: 3
// Sync result: 3
// After sync subscription
// Before async subscription
// After async subscription
// (100ms later...)
// Async tap: 1
// Async result: 1
// Async tap: 2
// Async result: 2
// Async tap: 3
// Async result: 3
```

## 🛑 Phase 4: Termination

Observables can terminate in three ways: completion, error, or unsubscription.

### Completion

```typescript
import { Observable } from 'rxjs';

const completing$ = new Observable(observer => {
  console.log('🚀 Starting execution');
  
  observer.next('First');
  observer.next('Second');
  
  setTimeout(() => {
    observer.next('Third');
    console.log('✅ Calling complete()');
    observer.complete();
    
    // These will be ignored after completion
    observer.next('This will not emit');
    observer.error(new Error('This will not error'));
  }, 1000);
  
  return () => console.log('🧹 Cleanup called');
});

completing$.subscribe({
  next: value => console.log('📨 Received:', value),
  error: err => console.log('❌ Error:', err),
  complete: () => console.log('🏁 Completed!')
});
```

### Error Termination

```typescript
import { Observable } from 'rxjs';

const erroring$ = new Observable(observer => {
  console.log('🚀 Starting execution');
  
  observer.next('First');
  observer.next('Second');
  
  setTimeout(() => {
    console.log('❌ Calling error()');
    observer.error(new Error('Something went wrong!'));
    
    // These will be ignored after error
    observer.next('This will not emit');
    observer.complete(); // This will not execute
  }, 1000);
  
  return () => console.log('🧹 Cleanup called');
});

erroring$.subscribe({
  next: value => console.log('📨 Received:', value),
  error: err => console.log('❌ Error caught:', err.message),
  complete: () => console.log('🏁 Completed!')
});
```

### Unsubscription

```typescript
import { Observable } from 'rxjs';

const unsubscribing$ = new Observable(observer => {
  console.log('🚀 Starting execution');
  
  let count = 0;
  const intervalId = setInterval(() => {
    count++;
    console.log(`📤 Emitting: ${count}`);
    observer.next(count);
  }, 500);
  
  return () => {
    console.log('🧹 Cleanup: clearing interval');
    clearInterval(intervalId);
  };
});

const subscription = unsubscribing$.subscribe({
  next: value => console.log('📨 Received:', value),
  error: err => console.log('❌ Error:', err),
  complete: () => console.log('🏁 Completed!')
});

// Unsubscribe after 2 seconds
setTimeout(() => {
  console.log('🛑 Unsubscribing...');
  subscription.unsubscribe();
  console.log('🛑 Unsubscribed');
}, 2000);
```

## 🧹 Phase 5: Cleanup

Cleanup ensures resources are properly freed and prevents memory leaks.

### Teardown Logic Patterns

```typescript
import { Observable, Subscription } from 'rxjs';

// 1. Function teardown
const functionTeardown$ = new Observable(observer => {
  const resource = acquireResource();
  
  return () => {
    console.log('Function teardown');
    releaseResource(resource);
  };
});

// 2. Subscription teardown
const subscriptionTeardown$ = new Observable(observer => {
  const childSub = anotherObservable$.subscribe();
  
  return childSub; // Subscription as teardown
});

// 3. Multiple teardown functions
const multipleTeardown$ = new Observable(observer => {
  const resource1 = acquireResource1();
  const resource2 = acquireResource2();
  const childSub = anotherObservable$.subscribe();
  
  return () => {
    releaseResource1(resource1);
    releaseResource2(resource2);
    childSub.unsubscribe();
  };
});

// 4. Subscription composition
const composedTeardown$ = new Observable(observer => {
  const subscription = new Subscription();
  
  subscription.add(() => console.log('Teardown 1'));
  subscription.add(() => console.log('Teardown 2'));
  subscription.add(childObservable$.subscribe());
  
  return subscription;
});

function acquireResource() { return { id: Math.random() }; }
function releaseResource(resource: any) { console.log('Released:', resource.id); }
function acquireResource1() { return 'resource1'; }
function releaseResource1(resource: string) { console.log('Released:', resource); }
function acquireResource2() { return 'resource2'; }
function releaseResource2(resource: string) { console.log('Released:', resource); }
const anotherObservable$ = new Observable(() => {});
```

## 🔍 Internal State Management

### Observable State Tracking

```typescript
class StatefulObservable<T> extends Observable<T> {
  private _state: ObservableState = ObservableState.CREATED;
  private _subscriptionCount = 0;
  private _lastError: any = null;
  
  get state(): ObservableState {
    return this._state;
  }
  
  get subscriptionCount(): number {
    return this._subscriptionCount;
  }
  
  get lastError(): any {
    return this._lastError;
  }
  
  subscribe(observer: Observer<T>): Subscription {
    console.log(`State before subscription: ${this._state}`);
    this._subscriptionCount++;
    this._state = ObservableState.EXECUTING;
    
    const originalObserver = {
      next: (value: T) => {
        console.log(`State during emission: ${this._state}`);
        observer.next?.(value);
      },
      error: (err: any) => {
        this._state = ObservableState.ERROR;
        this._lastError = err;
        console.log(`State after error: ${this._state}`);
        observer.error?.(err);
      },
      complete: () => {
        this._state = ObservableState.COMPLETED;
        console.log(`State after completion: ${this._state}`);
        observer.complete?.();
      }
    };
    
    const subscription = super.subscribe(originalObserver);
    
    // Wrap unsubscribe to track state
    const originalUnsubscribe = subscription.unsubscribe.bind(subscription);
    subscription.unsubscribe = () => {
      this._state = ObservableState.UNSUBSCRIBED;
      console.log(`State after unsubscribe: ${this._state}`);
      originalUnsubscribe();
    };
    
    return subscription;
  }
}

// Usage
const stateful$ = new StatefulObservable<number>(observer => {
  observer.next(1);
  observer.next(2);
  observer.complete();
});

console.log('Initial state:', stateful$.state);
const sub = stateful$.subscribe(value => console.log('Value:', value));
console.log('Final state:', stateful$.state);
```

## 🧠 Memory Management Deep Dive

### Memory Leak Prevention

```typescript
import { Observable, fromEvent, interval } from 'rxjs';
import { takeUntil } from 'rxjs/operators';

// ❌ Memory leak example
class LeakyComponent {
  ngOnInit() {
    // No cleanup - memory leak!
    interval(1000).subscribe(value => {
      console.log('Leaky interval:', value);
    });
    
    // No cleanup - memory leak!
    fromEvent(window, 'resize').subscribe(event => {
      console.log('Leaky resize:', event);
    });
  }
}

// ✅ Proper memory management
class SafeComponent {
  private destroy$ = new Subject<void>();
  
  ngOnInit() {
    // Automatic cleanup with takeUntil
    interval(1000).pipe(
      takeUntil(this.destroy$)
    ).subscribe(value => {
      console.log('Safe interval:', value);
    });
    
    fromEvent(window, 'resize').pipe(
      takeUntil(this.destroy$)
    ).subscribe(event => {
      console.log('Safe resize:', event);
    });
  }
  
  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### Garbage Collection Optimization

```typescript
import { Observable } from 'rxjs';

// Efficient Observable that helps garbage collection
const gcFriendly$ = new Observable(observer => {
  const data = new Array(1000000).fill(0); // Large data
  
  // Process data
  const processedData = data.map(x => x + 1);
  
  observer.next(processedData);
  observer.complete();
  
  return () => {
    // Help garbage collection by nullifying references
    data.length = 0;
    processedData.length = 0;
    console.log('Large data cleared for GC');
  };
});

// Usage with immediate unsubscription for testing
const subscription = gcFriendly$.subscribe();
subscription.unsubscribe(); // Triggers cleanup
```

## 📊 Performance Monitoring

### Execution Time Tracking

```typescript
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

function withPerformanceTracking<T>(source: Observable<T>, name: string): Observable<T> {
  return new Observable<T>(observer => {
    const startTime = performance.now();
    let emissionCount = 0;
    
    console.log(`🔥 ${name}: Execution started`);
    
    const subscription = source.pipe(
      tap(() => {
        emissionCount++;
        const elapsed = performance.now() - startTime;
        console.log(`📊 ${name}: Emission ${emissionCount} at ${elapsed.toFixed(2)}ms`);
      })
    ).subscribe({
      next: value => observer.next(value),
      error: err => {
        const elapsed = performance.now() - startTime;
        console.log(`❌ ${name}: Error after ${elapsed.toFixed(2)}ms`);
        observer.error(err);
      },
      complete: () => {
        const elapsed = performance.now() - startTime;
        console.log(`✅ ${name}: Completed after ${elapsed.toFixed(2)}ms with ${emissionCount} emissions`);
        observer.complete();
      }
    });
    
    return () => {
      const elapsed = performance.now() - startTime;
      console.log(`🧹 ${name}: Cleanup after ${elapsed.toFixed(2)}ms`);
      subscription.unsubscribe();
    };
  });
}

// Usage
const tracked$ = withPerformanceTracking(
  interval(100).pipe(take(5)),
  'Timer Test'
);

tracked$.subscribe();
```

## 🧪 Testing Lifecycle

### Lifecycle Testing with TestScheduler

```typescript
import { TestScheduler } from 'rxjs/testing';
import { map, delay } from 'rxjs/operators';

describe('Observable Lifecycle', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should complete lifecycle correctly', () => {
    testScheduler.run(({ cold, expectObservable, expectSubscriptions }) => {
      const source$ = cold('--a--b--c--|');
      const subs =         '^----------!';
      const expected =     '--a--b--c--|';
      
      const result$ = source$.pipe(
        map(x => x.toUpperCase())
      );
      
      expectObservable(result$).toBe(expected, { a: 'A', b: 'B', c: 'C' });
      expectSubscriptions(source$.subscriptions).toBe(subs);
    });
  });

  it('should handle unsubscription', () => {
    testScheduler.run(({ cold, expectObservable, expectSubscriptions }) => {
      const source$ = cold('--a--b--c--d--e--|');
      const unsub =        '-------!';
      const subs =         '^------!';
      const expected =     '--a--b-';
      
      expectObservable(source$, unsub).toBe(expected);
      expectSubscriptions(source$.subscriptions).toBe(subs);
    });
  });
});
```

## 🎯 Best Practices

### ✅ **Lifecycle Best Practices**

1. **Always provide cleanup** in custom Observables
2. **Use `takeUntil()` pattern** for component cleanup
3. **Monitor subscription count** in development
4. **Handle all termination scenarios** (complete, error, unsubscribe)
5. **Release resources explicitly** in teardown functions
6. **Test lifecycle behaviors** with marble testing
7. **Use `finalize()` operator** for guaranteed cleanup

### ❌ **Common Lifecycle Mistakes**

1. **Missing teardown logic** in custom Observables
2. **Not unsubscribing** in components
3. **Continuing work after termination**
4. **Memory leaks** from unreleased resources
5. **Synchronous blocking** in teardown functions

## 🎯 Quick Assessment

**Lifecycle Questions:**

1. When does Observable execution actually start?
2. What happens if you call `observer.next()` after `observer.complete()`?
3. How many times does teardown logic execute for multiple subscribers?
4. What's the difference between completion and unsubscription?

**Answers:**

1. **When `subscribe()` is called** (lazy execution)
2. **Nothing** - emissions after completion are ignored
3. **Once per subscription** - each subscription has its own teardown
4. **Completion** is natural end, **unsubscription** is manual cancellation

## 🌟 Key Takeaways

- **Observables are lazy** - execution starts only on subscription
- **Each subscription** creates a separate execution context
- **Proper cleanup** is essential for memory management
- **Termination can happen** via completion, error, or unsubscription
- **State management** helps track Observable lifecycle
- **Performance monitoring** reveals execution characteristics
- **Testing lifecycle** ensures robust Observable behavior

## 🚀 Next Steps

Now that you understand the Observable lifecycle, you're ready to practice **Reading & Drawing Marble Diagrams** to visualize these lifecycle concepts and operator behaviors.

**Next Lesson**: [Reading & Drawing Marble Diagrams](./03-marble-diagrams-practice.md) 🟢

---

🎉 **Excellent!** You now understand the complete Observable lifecycle from creation to cleanup. This knowledge is crucial for building efficient, leak-free reactive applications!
