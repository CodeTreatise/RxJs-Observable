# Lesson 6: Subjects & Multicasting Internals

## Learning Objectives
By the end of this lesson, you will:
- Master all types of Subjects and their internal workings
- Understand multicasting mechanisms and their performance implications
- Implement advanced Subject patterns for complex scenarios
- Build custom Subject implementations for specialized use cases
- Apply Subjects effectively in Angular applications

## Prerequisites
- Understanding of Hot vs Cold Observables
- Knowledge of Observer pattern
- Familiarity with RxJS operators
- Basic understanding of multicasting concepts

## 1. Subject Fundamentals

### What is a Subject?
A Subject is both an Observable and an Observer. It can emit values to multiple subscribers (multicast) and can also be subscribed to like any Observable.

```typescript
import { Subject } from 'rxjs';

// Subject acts as both Observable and Observer
const subject = new Subject<number>();

// As Observable - can be subscribed to
subject.subscribe(value => console.log('Observer A:', value));
subject.subscribe(value => console.log('Observer B:', value));

// As Observer - can receive values
subject.next(1);
subject.next(2);
subject.complete();

// Output:
// Observer A: 1
// Observer B: 1
// Observer A: 2
// Observer B: 2
```

### Subject Interface Deep Dive
```typescript
interface Subject<T> extends Observable<T>, Observer<T> {
  observers: Observer<T>[];          // List of subscribed observers
  closed: boolean;                   // Whether the subject is closed
  hasError: boolean;                 // Whether an error occurred
  thrownError: any;                  // The error that was thrown
  
  // Observer methods
  next(value: T): void;
  error(err: any): void;
  complete(): void;
  
  // Observable methods  
  subscribe(observer?: PartialObserver<T>): Subscription;
  
  // Subject-specific methods
  asObservable(): Observable<T>;     // Return as Observable (hide Subject methods)
  unsubscribe(): void;               // Unsubscribe all observers
}
```

## 2. Basic Subject Implementation

### Understanding Internal Mechanics
```typescript
// Simplified Subject implementation to understand internals
class SimpleSubject<T> {
  private observers: Array<{
    next?: (value: T) => void;
    error?: (error: any) => void;
    complete?: () => void;
  }> = [];
  
  private isStopped = false;
  private hasError = false;
  private thrownError: any = null;
  
  // Observer interface implementation
  next(value: T): void {
    if (this.isStopped) return;
    
    // Deliver to all observers
    this.observers.forEach(observer => {
      try {
        observer.next?.(value);
      } catch (error) {
        console.error('Error in observer:', error);
      }
    });
  }
  
  error(err: any): void {
    if (this.isStopped) return;
    
    this.isStopped = true;
    this.hasError = true;
    this.thrownError = err;
    
    this.observers.forEach(observer => {
      try {
        observer.error?.(err);
      } catch (error) {
        console.error('Error in error handler:', error);
      }
    });
    
    this.observers = [];
  }
  
  complete(): void {
    if (this.isStopped) return;
    
    this.isStopped = true;
    
    this.observers.forEach(observer => {
      try {
        observer.complete?.();
      } catch (error) {
        console.error('Error in complete handler:', error);
      }
    });
    
    this.observers = [];
  }
  
  // Observable interface implementation
  subscribe(observer: {
    next?: (value: T) => void;
    error?: (error: any) => void;
    complete?: () => void;
  }): () => void {
    
    if (this.hasError) {
      observer.error?.(this.thrownError);
      return () => {};
    }
    
    if (this.isStopped) {
      observer.complete?.();
      return () => {};
    }
    
    this.observers.push(observer);
    
    // Return unsubscribe function
    return () => {
      const index = this.observers.indexOf(observer);
      if (index >= 0) {
        this.observers.splice(index, 1);
      }
    };
  }
}
```

## 3. BehaviorSubject Deep Dive

### BehaviorSubject Characteristics
BehaviorSubject stores the current value and emits it to new subscribers immediately.

```typescript
import { BehaviorSubject } from 'rxjs';

class CustomBehaviorSubject<T> extends BehaviorSubject<T> {
  private currentValue: T;
  
  constructor(initialValue: T) {
    super(initialValue);
    this.currentValue = initialValue;
  }
  
  // Override next to track current value
  next(value: T): void {
    this.currentValue = value;
    super.next(value);
  }
  
  // Get current value synchronously
  getValue(): T {
    if (this.hasError) {
      throw this.thrownError;
    }
    if (this.closed) {
      throw new Error('BehaviorSubject has been closed');
    }
    return this.currentValue;
  }
  
  // Custom method to update value conditionally
  updateIf(predicate: (current: T) => boolean, newValue: T): boolean {
    if (predicate(this.currentValue)) {
      this.next(newValue);
      return true;
    }
    return false;
  }
}

// Usage example
const userState = new CustomBehaviorSubject({ name: 'John', age: 30 });

// New subscribers immediately get current value
userState.subscribe(user => console.log('Subscriber 1:', user));
// Output: Subscriber 1: { name: 'John', age: 30 }

// Update value
userState.next({ name: 'Jane', age: 25 });

// New subscriber gets the latest value
userState.subscribe(user => console.log('Subscriber 2:', user));
// Output: Subscriber 2: { name: 'Jane', age: 25 }

// Conditional update
const updated = userState.updateIf(
  user => user.age < 30,
  { name: 'Jane', age: 26 }
);
console.log('Updated:', updated); // true
```

### Advanced BehaviorSubject Patterns
```typescript
// State management with BehaviorSubject
class StateManager<T> {
  private state$: BehaviorSubject<T>;
  
  constructor(initialState: T) {
    this.state$ = new BehaviorSubject(initialState);
  }
  
  // Get current state snapshot
  getSnapshot(): T {
    return this.state$.getValue();
  }
  
  // Subscribe to state changes
  select<K extends keyof T>(key: K): Observable<T[K]> {
    return this.state$.pipe(
      map(state => state[key]),
      distinctUntilChanged()
    );
  }
  
  // Update partial state
  patch(partialState: Partial<T>): void {
    const currentState = this.state$.getValue();
    const newState = { ...currentState, ...partialState };
    this.state$.next(newState);
  }
  
  // Update with function
  update(updateFn: (state: T) => T): void {
    const currentState = this.state$.getValue();
    const newState = updateFn(currentState);
    this.state$.next(newState);
  }
  
  // Reset to initial state
  reset(initialState: T): void {
    this.state$.next(initialState);
  }
  
  // Get observable stream
  asObservable(): Observable<T> {
    return this.state$.asObservable();
  }
}

// Usage in Angular service
@Injectable({ providedIn: 'root' })
export class UserService {
  private stateManager = new StateManager({
    user: null as User | null,
    loading: false,
    error: null as string | null
  });
  
  // Expose specific state slices
  user$ = this.stateManager.select('user');
  loading$ = this.stateManager.select('loading');
  error$ = this.stateManager.select('error');
  
  loadUser(id: string): void {
    this.stateManager.patch({ loading: true, error: null });
    
    this.http.get<User>(`/api/users/${id}`).subscribe({
      next: user => this.stateManager.patch({ user, loading: false }),
      error: error => this.stateManager.patch({ 
        error: error.message, 
        loading: false 
      })
    });
  }
}
```

## 4. ReplaySubject Deep Dive

### ReplaySubject Implementation Details
ReplaySubject stores the last N values and replays them to new subscribers.

```typescript
// Simplified ReplaySubject implementation
class CustomReplaySubject<T> {
  private buffer: T[] = [];
  private bufferSize: number;
  private windowTime: number;
  private timestampProvider: () => number;
  private timestamps: number[] = [];
  
  constructor(
    bufferSize: number = Infinity,
    windowTime: number = Infinity,
    timestampProvider: () => number = () => Date.now()
  ) {
    this.bufferSize = bufferSize;
    this.windowTime = windowTime;
    this.timestampProvider = timestampProvider;
  }
  
  next(value: T): void {
    const now = this.timestampProvider();
    
    // Add to buffer
    this.buffer.push(value);
    this.timestamps.push(now);
    
    // Trim buffer by size
    if (this.buffer.length > this.bufferSize) {
      this.buffer.shift();
      this.timestamps.shift();
    }
    
    // Trim buffer by time
    this.trimByTime(now);
    
    // Emit to current subscribers
    this.emitToSubscribers(value);
  }
  
  private trimByTime(now: number): void {
    if (this.windowTime < Infinity) {
      const cutoff = now - this.windowTime;
      
      while (this.timestamps.length > 0 && this.timestamps[0] <= cutoff) {
        this.buffer.shift();
        this.timestamps.shift();
      }
    }
  }
  
  subscribe(observer: any): () => void {
    // Replay buffered values to new subscriber
    const now = this.timestampProvider();
    this.trimByTime(now);
    
    this.buffer.forEach(value => {
      observer.next?.(value);
    });
    
    // Add to subscribers list
    return this.addSubscriber(observer);
  }
}

// Usage examples
const replaySubject = new ReplaySubject<number>(3); // Buffer last 3 values

replaySubject.next(1);
replaySubject.next(2);
replaySubject.next(3);
replaySubject.next(4);

// New subscriber gets last 3 values: 2, 3, 4
replaySubject.subscribe(value => console.log('New subscriber:', value));

// Time-based replay
const timeBasedReplay = new ReplaySubject<string>(Infinity, 5000); // 5 second window

timeBasedReplay.next('Message 1');
setTimeout(() => timeBasedReplay.next('Message 2'), 3000);
setTimeout(() => timeBasedReplay.next('Message 3'), 6000);

// Subscriber at 7 seconds will only get messages from last 5 seconds
setTimeout(() => {
  timeBasedReplay.subscribe(msg => console.log('Time-based:', msg));
}, 7000);
```

### Advanced ReplaySubject Patterns
```typescript
// Caching service with ReplaySubject
@Injectable({ providedIn: 'root' })
export class CacheService {
  private caches = new Map<string, ReplaySubject<any>>();
  
  getCachedData<T>(
    key: string,
    dataProvider: () => Observable<T>,
    cacheSize: number = 1,
    windowTime: number = 300000 // 5 minutes
  ): Observable<T> {
    
    if (!this.caches.has(key)) {
      const subject = new ReplaySubject<T>(cacheSize, windowTime);
      this.caches.set(key, subject);
      
      // Load data and cache it
      dataProvider().subscribe({
        next: data => subject.next(data),
        error: error => {
          subject.error(error);
          this.caches.delete(key); // Remove failed cache
        }
      });
    }
    
    return this.caches.get(key)!.asObservable();
  }
  
  invalidateCache(key: string): void {
    const cache = this.caches.get(key);
    if (cache) {
      cache.complete();
      this.caches.delete(key);
    }
  }
  
  clearAllCaches(): void {
    this.caches.forEach(cache => cache.complete());
    this.caches.clear();
  }
}
```

## 5. AsyncSubject Deep Dive

### AsyncSubject Characteristics
AsyncSubject only emits the last value when the Subject completes.

```typescript
import { AsyncSubject } from 'rxjs';

// Custom AsyncSubject with additional features
class CustomAsyncSubject<T> extends AsyncSubject<T> {
  private hasValue = false;
  private latestValue: T | undefined;
  
  next(value: T): void {
    this.hasValue = true;
    this.latestValue = value;
    super.next(value);
  }
  
  complete(): void {
    if (this.hasValue) {
      console.log('AsyncSubject completing with value:', this.latestValue);
    } else {
      console.log('AsyncSubject completing without value');
    }
    super.complete();
  }
  
  // Get current value if available (but won't emit until complete)
  peekValue(): T | undefined {
    return this.latestValue;
  }
  
  // Check if subject has received any value
  hasReceivedValue(): boolean {
    return this.hasValue;
  }
}

// Usage example
const asyncSubject = new CustomAsyncSubject<string>();

asyncSubject.subscribe(value => console.log('Subscriber 1:', value));

asyncSubject.next('First');
asyncSubject.next('Second');
asyncSubject.next('Last'); // Only this will be emitted

// No output yet - waiting for complete()

asyncSubject.subscribe(value => console.log('Subscriber 2:', value));
asyncSubject.complete();

// Output:
// AsyncSubject completing with value: Last
// Subscriber 1: Last
// Subscriber 2: Last
```

### AsyncSubject Use Cases
```typescript
// Promise-like behavior with AsyncSubject
class PromiseLikeSubject<T> extends AsyncSubject<T> {
  private resolved = false;
  private rejected = false;
  
  resolve(value: T): void {
    if (this.resolved || this.rejected) return;
    
    this.resolved = true;
    this.next(value);
    this.complete();
  }
  
  reject(error: any): void {
    if (this.resolved || this.rejected) return;
    
    this.rejected = true;
    this.error(error);
  }
  
  // Promise-like then method
  then<TResult1 = T, TResult2 = never>(
    onfulfilled?: (value: T) => TResult1 | PromiseLike<TResult1>,
    onrejected?: (reason: any) => TResult2 | PromiseLike<TResult2>
  ): Promise<TResult1 | TResult2> {
    return this.pipe(take(1)).toPromise().then(onfulfilled, onrejected);
  }
}

// Usage
const promiseLikeSubject = new PromiseLikeSubject<number>();

promiseLikeSubject.then(
  value => console.log('Resolved:', value),
  error => console.log('Rejected:', error)
);

// Resolve after some async operation
setTimeout(() => {
  promiseLikeSubject.resolve(42);
}, 1000);
```

## 6. Multicasting Operators

### multicast() Operator Deep Dive
```typescript
import { Subject, multicast, interval, ConnectableObservable } from 'rxjs';

// Understanding multicast internals
function customMulticast<T>(
  source: Observable<T>,
  subjectFactory: () => Subject<T>
): ConnectableObservable<T> {
  
  let subject: Subject<T> | null = null;
  let subscription: Subscription | null = null;
  let refCount = 0;
  
  const connectableObservable = new Observable<T>(subscriber => {
    if (!subject) {
      subject = subjectFactory();
    }
    
    const subjectSubscription = subject.subscribe(subscriber);
    refCount++;
    
    return () => {
      refCount--;
      subjectSubscription.unsubscribe();
      
      if (refCount === 0 && subscription) {
        subscription.unsubscribe();
        subscription = null;
        subject = null;
      }
    };
  }) as ConnectableObservable<T>;
  
  connectableObservable.connect = () => {
    if (!subscription && subject) {
      subscription = source.subscribe(subject);
    }
    return subscription!;
  };
  
  return connectableObservable;
}

// Usage
const source = interval(1000);
const multicasted = customMulticast(source, () => new Subject<number>());

// Multiple subscribers
multicasted.subscribe(x => console.log('Sub 1:', x));
multicasted.subscribe(x => console.log('Sub 2:', x));

// Start execution
multicasted.connect();
```

### publish() Variants
```typescript
import { 
  publish, 
  publishBehavior, 
  publishReplay, 
  publishLast,
  share,
  shareReplay
} from 'rxjs/operators';

const source = interval(1000).pipe(take(5));

// Different multicasting strategies
const published = source.pipe(publish());                    // Basic Subject
const publishedBehavior = source.pipe(publishBehavior(0));   // BehaviorSubject
const publishedReplay = source.pipe(publishReplay(2));       // ReplaySubject
const publishedLast = source.pipe(publishLast());            // AsyncSubject

// Automatic connection with share operators
const shared = source.pipe(share());                         // Auto-connecting
const sharedReplay = source.pipe(shareReplay(1));           // With replay

// Custom share implementation
function customShare<T>(): OperatorFunction<T, T> {
  return (source: Observable<T>) => {
    return source.pipe(
      multicast(() => new Subject<T>()),
      refCount()
    );
  };
}
```

## 7. Advanced Subject Patterns

### Subject Composition
```typescript
// Combining multiple subjects for complex state management
class ComplexStateManager {
  private dataSubject = new BehaviorSubject<any[]>([]);
  private loadingSubject = new BehaviorSubject<boolean>(false);
  private errorSubject = new BehaviorSubject<string | null>(null);
  
  // Computed states
  readonly state$ = combineLatest([
    this.dataSubject,
    this.loadingSubject,
    this.errorSubject
  ]).pipe(
    map(([data, loading, error]) => ({
      data,
      loading,
      error,
      hasData: data.length > 0,
      isIdle: !loading && !error,
      isError: !!error
    }))
  );
  
  // Individual state streams
  readonly data$ = this.dataSubject.asObservable();
  readonly loading$ = this.loadingSubject.asObservable();
  readonly error$ = this.errorSubject.asObservable();
  
  // Actions
  setLoading(loading: boolean): void {
    this.loadingSubject.next(loading);
  }
  
  setData(data: any[]): void {
    this.dataSubject.next(data);
    this.errorSubject.next(null);
  }
  
  setError(error: string): void {
    this.errorSubject.next(error);
    this.loadingSubject.next(false);
  }
  
  addItem(item: any): void {
    const currentData = this.dataSubject.value;
    this.dataSubject.next([...currentData, item]);
  }
  
  removeItem(id: string): void {
    const currentData = this.dataSubject.value;
    this.dataSubject.next(currentData.filter(item => item.id !== id));
  }
  
  reset(): void {
    this.dataSubject.next([]);
    this.loadingSubject.next(false);
    this.errorSubject.next(null);
  }
}
```

### Subject Factory Pattern
```typescript
// Factory for creating different types of subjects
class SubjectFactory {
  static createSubject<T>(type: 'basic'): Subject<T>;
  static createSubject<T>(type: 'behavior', initialValue: T): BehaviorSubject<T>;
  static createSubject<T>(type: 'replay', bufferSize?: number): ReplaySubject<T>;
  static createSubject<T>(type: 'async'): AsyncSubject<T>;
  static createSubject<T>(
    type: 'basic' | 'behavior' | 'replay' | 'async',
    config?: any
  ): Subject<T> | BehaviorSubject<T> | ReplaySubject<T> | AsyncSubject<T> {
    
    switch (type) {
      case 'basic':
        return new Subject<T>();
      case 'behavior':
        return new BehaviorSubject<T>(config);
      case 'replay':
        return new ReplaySubject<T>(config || 1);
      case 'async':
        return new AsyncSubject<T>();
      default:
        throw new Error(`Unknown subject type: ${type}`);
    }
  }
  
  // Create subject with automatic cleanup
  static createManagedSubject<T>(
    type: 'basic' | 'behavior' | 'replay' | 'async',
    config?: any,
    cleanupDelay: number = 30000 // 30 seconds
  ): Subject<T> {
    
    const subject = SubjectFactory.createSubject<T>(type, config);
    
    // Auto-cleanup when no subscribers for specified time
    let cleanupTimer: any;
    let subscriberCount = 0;
    
    const originalSubscribe = subject.subscribe.bind(subject);
    
    subject.subscribe = (observer: any) => {
      subscriberCount++;
      
      if (cleanupTimer) {
        clearTimeout(cleanupTimer);
        cleanupTimer = null;
      }
      
      const subscription = originalSubscribe(observer);
      const originalUnsubscribe = subscription.unsubscribe.bind(subscription);
      
      subscription.unsubscribe = () => {
        subscriberCount--;
        originalUnsubscribe();
        
        if (subscriberCount === 0) {
          cleanupTimer = setTimeout(() => {
            subject.complete();
          }, cleanupDelay);
        }
      };
      
      return subscription;
    };
    
    return subject;
  }
}
```

## 8. Performance Optimization

### Subject Performance Considerations
```typescript
// Performance monitoring for subjects
class PerformanceMonitoredSubject<T> extends Subject<T> {
  private emissionCount = 0;
  private subscriptionCount = 0;
  private startTime = Date.now();
  
  next(value: T): void {
    this.emissionCount++;
    
    const emissionTime = performance.now();
    super.next(value);
    const processingTime = performance.now() - emissionTime;
    
    if (processingTime > 10) { // Log slow emissions
      console.warn(`Slow emission detected: ${processingTime}ms`);
    }
  }
  
  subscribe(observer?: any): Subscription {
    this.subscriptionCount++;
    const subscription = super.subscribe(observer);
    
    const originalUnsubscribe = subscription.unsubscribe.bind(subscription);
    subscription.unsubscribe = () => {
      this.subscriptionCount--;
      originalUnsubscribe();
    };
    
    return subscription;
  }
  
  getStats() {
    return {
      emissionCount: this.emissionCount,
      subscriptionCount: this.subscriptionCount,
      runtime: Date.now() - this.startTime,
      emissionsPerSecond: this.emissionCount / ((Date.now() - this.startTime) / 1000)
    };
  }
}

// Memory-efficient subject for large datasets
class BufferedSubject<T> extends Subject<T> {
  private buffer: T[] = [];
  private maxBuffer: number;
  private flushInterval: number;
  private timer: any;
  
  constructor(maxBuffer = 100, flushInterval = 1000) {
    super();
    this.maxBuffer = maxBuffer;
    this.flushInterval = flushInterval;
    this.setupFlushTimer();
  }
  
  next(value: T): void {
    this.buffer.push(value);
    
    if (this.buffer.length >= this.maxBuffer) {
      this.flush();
    }
  }
  
  private flush(): void {
    if (this.buffer.length > 0) {
      const batch = [...this.buffer];
      this.buffer = [];
      
      // Emit batch instead of individual values
      super.next(batch as any);
    }
  }
  
  private setupFlushTimer(): void {
    this.timer = setInterval(() => {
      this.flush();
    }, this.flushInterval);
  }
  
  complete(): void {
    this.flush();
    clearInterval(this.timer);
    super.complete();
  }
}
```

## 9. Testing Subjects

### Unit Testing Patterns
```typescript
describe('Subject Testing', () => {
  let subject: Subject<number>;
  let results: number[];
  
  beforeEach(() => {
    subject = new Subject<number>();
    results = [];
  });
  
  afterEach(() => {
    subject.complete();
  });
  
  it('should emit values to all subscribers', () => {
    const results1: number[] = [];
    const results2: number[] = [];
    
    subject.subscribe(value => results1.push(value));
    subject.subscribe(value => results2.push(value));
    
    subject.next(1);
    subject.next(2);
    
    expect(results1).toEqual([1, 2]);
    expect(results2).toEqual([1, 2]);
  });
  
  it('should handle errors properly', () => {
    let errorReceived: any;
    
    subject.subscribe({
      next: value => results.push(value),
      error: error => errorReceived = error
    });
    
    subject.next(1);
    subject.error('test error');
    subject.next(2); // Should not be received
    
    expect(results).toEqual([1]);
    expect(errorReceived).toBe('test error');
  });
  
  it('should complete properly', () => {
    let completed = false;
    
    subject.subscribe({
      next: value => results.push(value),
      complete: () => completed = true
    });
    
    subject.next(1);
    subject.complete();
    subject.next(2); // Should not be received
    
    expect(results).toEqual([1]);
    expect(completed).toBe(true);
  });
});

// Testing with TestScheduler
import { TestScheduler } from 'rxjs/testing';

describe('Subject with TestScheduler', () => {
  let testScheduler: TestScheduler;
  
  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });
  
  it('should test timing with marble diagrams', () => {
    testScheduler.run(({ cold, hot, expectObservable }) => {
      const subject = new Subject<string>();
      
      // Subscribe at different times
      const sub1 = '--^--!';
      const sub2 = '----^!';
      
      const source = hot('  a-b-c-d-e-f');
      const expected1 = '  --b-c-d';
      const expected2 = '  ----c-d';
      
      // Pipe source through subject
      source.subscribe(subject);
      
      expectObservable(subject, sub1).toBe(expected1);
      expectObservable(subject, sub2).toBe(expected2);
    });
  });
});
```

## 10. Best Practices

### 1. Choose the Right Subject Type
```typescript
// Decision matrix for subject selection
const subjectDecisionMatrix = {
  'need-current-value': 'BehaviorSubject',
  'need-last-n-values': 'ReplaySubject', 
  'need-final-value-only': 'AsyncSubject',
  'simple-multicasting': 'Subject'
};

// Implementation guide
function chooseSubject(requirements: string[]): string {
  if (requirements.includes('current-value')) return 'BehaviorSubject';
  if (requirements.includes('replay-values')) return 'ReplaySubject';
  if (requirements.includes('final-value-only')) return 'AsyncSubject';
  return 'Subject';
}
```

### 2. Proper Subject Cleanup
```typescript
@Component({
  selector: 'app-subject-example',
  template: `<div>{{ data$ | async }}</div>`
})
export class SubjectExampleComponent implements OnDestroy {
  private dataSubject = new BehaviorSubject<any>(null);
  data$ = this.dataSubject.asObservable();
  
  ngOnDestroy() {
    // Always complete subjects to prevent memory leaks
    this.dataSubject.complete();
  }
}
```

### 3. Expose as Observable
```typescript
@Injectable({ providedIn: 'root' })
export class DataService {
  private dataSubject = new BehaviorSubject<Data[]>([]);
  
  // ✅ GOOD: Expose as Observable to prevent external next() calls
  readonly data$ = this.dataSubject.asObservable();
  
  // ❌ BAD: Exposes Subject methods to consumers
  // readonly data$ = this.dataSubject;
  
  updateData(data: Data[]): void {
    this.dataSubject.next(data);
  }
}
```

## Key Takeaways

1. **Subjects are both Observable and Observer** - they can emit and be subscribed to
2. **Choose the right Subject type** based on your specific requirements
3. **BehaviorSubject** for current state management
4. **ReplaySubject** for caching and replay functionality  
5. **AsyncSubject** for promise-like behavior
6. **Always complete Subjects** to prevent memory leaks
7. **Expose as Observable** to hide Subject implementation details
8. **Use multicasting operators** for performance optimization

## Next Steps
In the next lesson, we'll explore RxJS Schedulers and understand how they control the execution context and timing of Observable operations.
