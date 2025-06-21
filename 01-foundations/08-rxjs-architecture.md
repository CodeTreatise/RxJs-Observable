# RxJS Library Architecture & Design 🟡

## 🎯 Learning Objectives
By the end of this lesson, you will understand:
- The overall architecture of the RxJS library
- How different RxJS modules are organized
- The design patterns used throughout RxJS
- Internal implementation strategies
- How RxJS achieves performance and extensibility

## 🏗️ RxJS Library Structure

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    RxJS Library                         │
├─────────────────────────────────────────────────────────┤
│  Core                                                   │
│  ├── Observable                                         │
│  ├── Observer                                          │
│  ├── Subscription                                      │
│  └── Scheduler                                         │
├─────────────────────────────────────────────────────────┤
│  Creation Operators                                     │
│  ├── of, from, interval, timer                        │
│  └── fromEvent, fromPromise, etc.                     │
├─────────────────────────────────────────────────────────┤
│  Pipeable Operators                                     │
│  ├── Transformation (map, switchMap, etc.)            │
│  ├── Filtering (filter, take, etc.)                   │
│  ├── Combination (merge, combineLatest, etc.)         │
│  └── Utility (tap, delay, etc.)                       │
├─────────────────────────────────────────────────────────┤
│  Subjects                                               │
│  ├── Subject, BehaviorSubject                         │
│  ├── ReplaySubject, AsyncSubject                      │
│  └── WebSocketSubject                                 │
├─────────────────────────────────────────────────────────┤
│  Schedulers                                             │
│  ├── asap, async, queue                               │
│  └── animationFrame, VirtualTime                      │
├─────────────────────────────────────────────────────────┤
│  Testing                                                │
│  ├── TestScheduler                                     │
│  ├── Marble Testing                                    │
│  └── Cold/Hot Observables                             │
└─────────────────────────────────────────────────────────┘
```

### Module Organization

```typescript
// Core exports from 'rxjs'
import { 
  Observable, 
  Observer, 
  Subscription, 
  Subject,
  BehaviorSubject,
  ReplaySubject,
  AsyncSubject
} from 'rxjs';

// Creation operators
import { 
  of, 
  from, 
  interval, 
  timer, 
  fromEvent,
  combineLatest,
  merge,
  zip
} from 'rxjs';

// Pipeable operators from 'rxjs/operators'
import { 
  map, 
  filter, 
  switchMap, 
  tap,
  catchError,
  retry
} from 'rxjs/operators';

// Testing utilities from 'rxjs/testing'
import { TestScheduler } from 'rxjs/testing';

// AJAX utilities from 'rxjs/ajax'
import { ajax } from 'rxjs/ajax';

// WebSocket utilities from 'rxjs/webSocket'
import { webSocket } from 'rxjs/webSocket';
```

## 🎯 Core Design Patterns

### 1. **Factory Pattern** - Observable Creation

```typescript
// Factory functions for creating Observables
function of<T>(...values: T[]): Observable<T> {
  return new Observable(observer => {
    values.forEach(value => observer.next(value));
    observer.complete();
  });
}

function interval(period: number): Observable<number> {
  return new Observable(observer => {
    let count = 0;
    const id = setInterval(() => {
      observer.next(count++);
    }, period);
    
    return () => clearInterval(id);
  });
}

// Usage - Factory pattern hides complexity
const numbers$ = of(1, 2, 3);
const timer$ = interval(1000);
```

### 2. **Builder Pattern** - Operator Chaining

```typescript
// Fluent interface for building complex streams
const result$ = source$.pipe(
  filter(x => x > 0),        // Builder step 1
  map(x => x * 2),           // Builder step 2
  debounceTime(300),         // Builder step 3
  distinctUntilChanged(),    // Builder step 4
  take(10)                   // Final step
);

// Internal implementation uses function composition
function pipe<T, R>(...operators: OperatorFunction<any, any>[]): OperatorFunction<T, R> {
  return (source: Observable<T>) => {
    return operators.reduce((acc, operator) => operator(acc), source);
  };
}
```

### 3. **Strategy Pattern** - Schedulers

```typescript
// Different scheduling strategies
abstract class Scheduler {
  abstract schedule<T>(
    work: (this: SchedulerAction<T>, state?: T) => void,
    delay?: number,
    state?: T
  ): Subscription;
}

// Concrete strategies
class AsyncScheduler extends Scheduler {
  schedule<T>(work: Function, delay = 0): Subscription {
    return new Subscription(() => {
      const id = setTimeout(work, delay);
      return () => clearTimeout(id);
    });
  }
}

class QueueScheduler extends Scheduler {
  schedule<T>(work: Function): Subscription {
    // Synchronous execution
    work();
    return new Subscription();
  }
}

// Usage - Strategy is pluggable
of(1, 2, 3, asyncScheduler); // Async execution
of(1, 2, 3, queueScheduler); // Sync execution
```

### 4. **Decorator Pattern** - Operators

```typescript
// Operators decorate Observables with new behavior
function withLogging<T>(source: Observable<T>): Observable<T> {
  return new Observable(observer => {
    console.log('Subscription started');
    
    const subscription = source.subscribe({
      next: value => {
        console.log('Value emitted:', value);
        observer.next(value);
      },
      error: err => {
        console.log('Error occurred:', err);
        observer.error(err);
      },
      complete: () => {
        console.log('Stream completed');
        observer.complete();
      }
    });
    
    return () => {
      console.log('Subscription cleaned up');
      subscription.unsubscribe();
    };
  });
}

// Decorating existing Observable
const decorated$ = withLogging(source$);
```

### 5. **Command Pattern** - Schedulers

```typescript
// Commands encapsulate scheduled work
interface SchedulerAction<T> {
  schedule(state?: T, delay?: number): Subscription;
}

class AsyncAction<T> implements SchedulerAction<T> {
  constructor(
    private scheduler: Scheduler,
    private work: (state?: T) => void
  ) {}
  
  schedule(state?: T, delay = 0): Subscription {
    // Command encapsulates execution
    const id = setTimeout(() => this.work(state), delay);
    return new Subscription(() => clearTimeout(id));
  }
}
```

## 🔧 Internal Implementation Strategies

### 1. **Operator Implementation Pattern**

```typescript
// Standard operator implementation structure
function customOperator<T, R>(
  transformFn: (value: T) => R
): OperatorFunction<T, R> {
  
  return (source: Observable<T>) => {
    return new Observable<R>(observer => {
      // Subscribe to source
      const subscription = source.subscribe({
        next: (value: T) => {
          try {
            const result = transformFn(value);
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
```

### 2. **Higher-Order Observable Handling**

```typescript
// Pattern for flattening higher-order Observables
function flattenStrategy<T, R>(
  project: (value: T) => Observable<R>,
  concurrent = Infinity
): OperatorFunction<T, R> {
  
  return (source: Observable<T>) => {
    return new Observable<R>(observer => {
      let activeSubscriptions = 0;
      let completed = false;
      const subscriptions: Subscription[] = [];
      
      const checkCompletion = () => {
        if (completed && activeSubscriptions === 0) {
          observer.complete();
        }
      };
      
      const sourceSubscription = source.subscribe({
        next: (value: T) => {
          if (activeSubscriptions < concurrent) {
            activeSubscriptions++;
            
            const innerObservable = project(value);
            const innerSubscription = innerObservable.subscribe({
              next: (result: R) => observer.next(result),
              error: (err) => observer.error(err),
              complete: () => {
                activeSubscriptions--;
                checkCompletion();
              }
            });
            
            subscriptions.push(innerSubscription);
          }
        },
        error: (err) => observer.error(err),
        complete: () => {
          completed = true;
          checkCompletion();
        }
      });
      
      return () => {
        sourceSubscription.unsubscribe();
        subscriptions.forEach(sub => sub.unsubscribe());
      };
    });
  };
}
```

### 3. **Memory Management Strategy**

```typescript
// Reference counting for shared Observables
class RefCountSubscription {
  private refCount = 0;
  private connection?: Subscription;
  
  constructor(private connectableObservable: ConnectableObservable<any>) {}
  
  subscribe(observer: Observer<any>): Subscription {
    this.refCount++;
    
    // Connect on first subscription
    if (this.refCount === 1) {
      this.connection = this.connectableObservable.connect();
    }
    
    const subscription = this.connectableObservable.subscribe(observer);
    
    return new Subscription(() => {
      subscription.unsubscribe();
      this.refCount--;
      
      // Disconnect when no more subscribers
      if (this.refCount === 0 && this.connection) {
        this.connection.unsubscribe();
        this.connection = undefined;
      }
    });
  }
}
```

## 📦 Module Architecture Deep Dive

### 1. **Core Module (rxjs/internal)**

```typescript
// Core Observable implementation
export class Observable<T> {
  constructor(
    subscribe?: (this: Observable<T>, observer: Observer<T>) => TeardownLogic
  ) {
    if (subscribe) {
      this._subscribe = subscribe;
    }
  }

  subscribe(observer?: Observer<T>): Subscription;
  subscribe(next?: (value: T) => void): Subscription;
  subscribe(
    next?: (value: T) => void,
    error?: (error: any) => void,
    complete?: () => void
  ): Subscription;
  
  subscribe(
    observerOrNext?: Observer<T> | ((value: T) => void),
    error?: (error: any) => void,
    complete?: () => void
  ): Subscription {
    const observer = toObserver(observerOrNext, error, complete);
    return this._subscribe(observer);
  }

  pipe(): Observable<T>;
  pipe<A>(op1: OperatorFunction<T, A>): Observable<A>;
  pipe<A, B>(op1: OperatorFunction<T, A>, op2: OperatorFunction<A, B>): Observable<B>;
  // ... overloads for more operators
  
  pipe(...operations: OperatorFunction<any, any>[]): Observable<any> {
    return operations.reduce((source, operator) => operator(source), this);
  }
}
```

### 2. **Operator Module Organization**

```typescript
// operators/index.ts - Central export point
export { map } from './map';
export { filter } from './filter';
export { switchMap } from './switchMap';
export { mergeMap } from './mergeMap';
// ... other operators

// operators/map.ts - Individual operator implementation
import { OperatorFunction } from '../types';

export function map<T, R>(project: (value: T, index: number) => R): OperatorFunction<T, R> {
  return (source: Observable<T>) => {
    return new Observable<R>(observer => {
      let index = 0;
      return source.subscribe({
        next: (value: T) => {
          try {
            const result = project(value, index++);
            observer.next(result);
          } catch (err) {
            observer.error(err);
          }
        },
        error: (err) => observer.error(err),
        complete: () => observer.complete()
      });
    });
  };
}
```

### 3. **Subject Module Hierarchy**

```typescript
// Subject inheritance hierarchy
abstract class Subject<T> extends Observable<T> implements Observer<T> {
  observers: Observer<T>[] = [];
  closed = false;
  hasError = false;
  thrownError: any = null;

  next(value: T): void {
    if (this.closed) return;
    
    const observers = this.observers.slice(); // Copy to avoid issues during iteration
    observers.forEach(observer => {
      try {
        observer.next(value);
      } catch (err) {
        // Handle individual observer errors
      }
    });
  }

  error(err: any): void {
    if (this.closed) return;
    
    this.hasError = true;
    this.thrownError = err;
    this.closed = true;
    
    const observers = this.observers;
    this.observers = [];
    
    observers.forEach(observer => {
      observer.error(err);
    });
  }

  complete(): void {
    if (this.closed) return;
    
    this.closed = true;
    
    const observers = this.observers;
    this.observers = [];
    
    observers.forEach(observer => {
      observer.complete();
    });
  }
}

// Specialized subject implementations
class BehaviorSubject<T> extends Subject<T> {
  constructor(private _value: T) {
    super();
  }

  get value(): T {
    return this._value;
  }

  next(value: T): void {
    this._value = value;
    super.next(value);
  }

  subscribe(observer?: Observer<T>): Subscription {
    const subscription = super.subscribe(observer);
    
    // Emit current value to new subscribers
    if (!this.closed && !this.hasError) {
      observer?.next(this._value);
    }
    
    return subscription;
  }
}
```

## 🎨 Performance Optimization Strategies

### 1. **Lazy Evaluation**

```typescript
// Operators are not executed until subscription
const lazyStream$ = source$.pipe(
  map(x => {
    console.log('This only runs when subscribed');
    return x * 2;
  }),
  filter(x => x > 10)
);

// No execution yet...
console.log('Stream defined');

// Now operators execute
lazyStream$.subscribe(console.log);
```

### 2. **Structural Sharing**

```typescript
// Operators share structure where possible
class Observable<T> {
  source?: Observable<any>;
  operator?: Operator<any, T>;

  lift<R>(operator: Operator<T, R>): Observable<R> {
    const observable = new Observable<R>();
    observable.source = this;
    observable.operator = operator;
    return observable;
  }
}

// Efficient operator chaining - shared structure
const result$ = source$
  .pipe(map(x => x * 2))    // Creates new Observable with shared source
  .pipe(filter(x => x > 10)); // Creates new Observable with shared chain
```

### 3. **Subscription Pooling**

```typescript
// Share subscriptions for efficiency
class ShareReplayOperator<T> {
  private buffer: T[] = [];
  private subscription?: Subscription;
  private refCount = 0;

  subscribe(observer: Observer<T>): Subscription {
    // Replay buffered values
    this.buffer.forEach(value => observer.next(value));
    
    this.refCount++;
    
    // Create source subscription on first subscriber
    if (this.refCount === 1) {
      this.subscription = this.source.subscribe({
        next: (value: T) => {
          this.buffer.push(value);
          this.observers.forEach(obs => obs.next(value));
        }
        // ... error and complete handlers
      });
    }
    
    return new Subscription(() => {
      this.refCount--;
      if (this.refCount === 0) {
        this.subscription?.unsubscribe();
      }
    });
  }
}
```

## 🧪 Testing Architecture

### TestScheduler Design

```typescript
// Virtual time scheduling for deterministic testing
export class TestScheduler extends VirtualTimeScheduler {
  private hotObservables: HotObservable<any>[] = [];
  private coldObservables: ColdObservable<any>[] = [];

  createHotObservable<T>(marbles: string, values?: any): HotObservable<T> {
    const observable = new HotObservable(marbles, values, this);
    this.hotObservables.push(observable);
    return observable;
  }

  createColdObservable<T>(marbles: string, values?: any): ColdObservable<T> {
    const observable = new ColdObservable(marbles, values, this);
    this.coldObservables.push(observable);
    return observable;
  }

  flush(): void {
    // Execute all scheduled work in virtual time
    super.flush();
    
    // Verify expectations
    this.hotObservables.forEach(obs => obs.verify());
    this.coldObservables.forEach(obs => obs.verify());
  }
}
```

## 🔍 Error Handling Architecture

### Global Error Strategy

```typescript
// Centralized error handling
class GlobalErrorHandler {
  private static instance: GlobalErrorHandler;
  
  static handleError(error: any, source: Observable<any>): void {
    // Log error
    console.error('RxJS Error:', error);
    
    // Notify error tracking service
    if (this.instance?.errorReporter) {
      this.instance.errorReporter.report(error);
    }
    
    // Provide fallback values based on error type
    if (error instanceof NetworkError) {
      return source.pipe(
        catchError(() => of(null))
      );
    }
  }
}

// Operator-level error isolation
function safeOperator<T, R>(
  operator: OperatorFunction<T, R>
): OperatorFunction<T, R> {
  return (source: Observable<T>) => {
    return source.pipe(
      operator,
      catchError(error => {
        GlobalErrorHandler.handleError(error, source);
        return EMPTY; // Or appropriate fallback
      })
    );
  };
}
```

## 📊 Memory Management Architecture

### Subscription Tree Management

```typescript
// Hierarchical subscription management
class SubscriptionManager {
  private subscriptions = new Map<string, Subscription>();
  private parent?: SubscriptionManager;
  private children = new Set<SubscriptionManager>();

  add(key: string, subscription: Subscription): void {
    this.subscriptions.set(key, subscription);
  }

  remove(key: string): void {
    const subscription = this.subscriptions.get(key);
    subscription?.unsubscribe();
    this.subscriptions.delete(key);
  }

  createChild(): SubscriptionManager {
    const child = new SubscriptionManager();
    child.parent = this;
    this.children.add(child);
    return child;
  }

  unsubscribeAll(): void {
    // Unsubscribe all own subscriptions
    this.subscriptions.forEach(sub => sub.unsubscribe());
    this.subscriptions.clear();
    
    // Unsubscribe all children
    this.children.forEach(child => child.unsubscribeAll());
    this.children.clear();
  }
}
```

## 🎯 Quick Assessment

**Architecture Questions:**

1. What design pattern does the `pipe()` method implement?
2. How does RxJS achieve lazy evaluation?
3. What pattern do RxJS Schedulers implement?
4. How does RxJS handle memory management in operators?

**Answers:**

1. **Builder Pattern** - Fluent interface for chaining operations
2. **Lazy Evaluation** - Operators only execute when subscribed
3. **Strategy Pattern** - Different scheduling algorithms are pluggable
4. **Automatic cleanup** through teardown functions and subscription management

## 🌟 Key Takeaways

- **RxJS** uses multiple design patterns for flexibility and performance
- **Modular architecture** enables tree shaking and custom builds
- **Lazy evaluation** ensures operators only run when needed
- **Memory management** is built into the subscription system
- **Testing architecture** provides deterministic async testing
- **Error handling** can be centralized and customized
- **Performance optimizations** are built into the core architecture

## 🚀 Next Steps

Congratulations! You've completed **Module 1: Foundations & Theory**. You now have a solid understanding of:

- ✅ What Observables and RxJS are
- ✅ Core reactive programming concepts
- ✅ How to read and create marble diagrams
- ✅ Observable internal structure and anatomy
- ✅ The Observer pattern and its variations
- ✅ How Observables compare to other async patterns
- ✅ Setting up RxJS in Angular projects
- ✅ RxJS library architecture and design patterns

**Next Module**: [Core RxJS Concepts & Internals](../02-core-concepts/01-creating-observables.md) 🟢

In Module 2, you'll learn how to create Observables, understand their lifecycle, work with hot/cold Observables, master Subjects, and dive deep into how operators work internally.

---

🎉 **Outstanding Achievement!** You've mastered the foundational concepts of RxJS and reactive programming. You're now ready to start creating and working with Observables in real-world scenarios!
