# Memory Management & Leaks Prevention

## Learning Objectives
- Understand memory leak patterns in RxJS applications
- Master subscription management and cleanup strategies
- Implement efficient resource disposal patterns
- Debug and detect memory leaks in reactive code
- Optimize memory usage in long-running applications
- Build memory-safe reactive architectures

## Introduction to RxJS Memory Management

Memory leaks in RxJS applications are among the most common and critical issues developers face. Unlike traditional memory leaks, RxJS memory leaks often involve active subscriptions that prevent garbage collection of entire component trees and associated data structures.

### Common Memory Leak Scenarios
- Unsubscribed observables in components
- Event listeners that aren't removed
- Long-running timers and intervals
- Circular references in custom operators
- Retained references in closures
- Hot observables without proper cleanup

### Memory Leak Impact
- Gradual performance degradation
- Increased CPU and memory usage
- Browser crashes in severe cases
- Unpredictable application behavior
- Poor user experience

## Understanding Subscription Lifecycle

### Subscription Fundamentals

```typescript
// subscription-basics.ts
import { Observable, Subscription, Subject, BehaviorSubject } from 'rxjs';
import { takeUntil, finalize, share, shareReplay } from 'rxjs/operators';

/**
 * Basic subscription management patterns
 */
export class SubscriptionManager {
  private subscriptions = new Subscription();
  private destroy$ = new Subject<void>();

  // Pattern 1: Manual subscription management
  addSubscription(subscription: Subscription): void {
    this.subscriptions.add(subscription);
  }

  // Pattern 2: TakeUntil pattern
  takeUntilDestroy<T>() {
    return takeUntil<T>(this.destroy$);
  }

  // Pattern 3: Subscription array
  private subscriptionArray: Subscription[] = [];

  addToArray(subscription: Subscription): void {
    this.subscriptionArray.push(subscription);
  }

  // Cleanup methods
  cleanup(): void {
    // Method 1: Subscription container
    this.subscriptions.unsubscribe();

    // Method 2: Destroy subject
    this.destroy$.next();
    this.destroy$.complete();

    // Method 3: Array cleanup
    this.subscriptionArray.forEach(sub => sub.unsubscribe());
    this.subscriptionArray.length = 0;
  }
}

/**
 * Component with proper subscription management
 */
@Component({
  selector: 'app-memory-safe',
  template: `
    <div class="memory-safe-component">
      <div *ngIf="data$ | async as data">
        {{ data.message }}
      </div>
      <button (click)="loadData()">Load Data</button>
      <button (click)="startPolling()">Start Polling</button>
      <button (click)="stopPolling()">Stop Polling</button>
    </div>
  `
})
export class MemorySafeComponent implements OnInit, OnDestroy {
  private readonly subscriptionManager = new SubscriptionManager();
  private readonly destroy$ = new Subject<void>();
  private pollingSubscription?: Subscription;

  // Using async pipe (automatically unsubscribes)
  readonly data$ = this.dataService.getData().pipe(
    this.subscriptionManager.takeUntilDestroy(),
    shareReplay(1)
  );

  constructor(private dataService: DataService) {}

  ngOnInit(): void {
    // Pattern 1: Using subscription manager
    const timerSub = interval(1000).pipe(
      this.subscriptionManager.takeUntilDestroy()
    ).subscribe(tick => {
      console.log('Timer tick:', tick);
    });

    // Pattern 2: Manual subscription tracking
    const dataSub = this.dataService.getLiveData().subscribe(data => {
      this.processData(data);
    });
    this.subscriptionManager.addSubscription(dataSub);

    // Pattern 3: Using takeUntil directly
    this.dataService.getNotifications().pipe(
      takeUntil(this.destroy$)
    ).subscribe(notification => {
      this.showNotification(notification);
    });
  }

  loadData(): void {
    // Short-lived subscription for one-time operations
    this.dataService.fetchData().pipe(
      take(1) // Automatically completes after first emission
    ).subscribe(data => {
      this.processData(data);
    });
  }

  startPolling(): void {
    this.stopPolling(); // Stop any existing polling

    this.pollingSubscription = interval(5000).pipe(
      switchMap(() => this.dataService.pollData()),
      takeUntil(this.destroy$),
      catchError(error => {
        console.error('Polling error:', error);
        return EMPTY; // Continue polling despite errors
      })
    ).subscribe(data => {
      this.processPolledData(data);
    });
  }

  stopPolling(): void {
    if (this.pollingSubscription) {
      this.pollingSubscription.unsubscribe();
      this.pollingSubscription = undefined;
    }
  }

  ngOnDestroy(): void {
    // Cleanup all subscriptions
    this.subscriptionManager.cleanup();
    this.destroy$.next();
    this.destroy$.complete();
    this.stopPolling();
  }

  private processData(data: any): void {
    // Process data
  }

  private processPolledData(data: any): void {
    // Process polled data
  }

  private showNotification(notification: any): void {
    // Show notification
  }
}
```

### Advanced Subscription Patterns

```typescript
// advanced-subscription-patterns.ts

/**
 * Subscription pool for managing many dynamic subscriptions
 */
export class SubscriptionPool {
  private subscriptions = new Map<string, Subscription>();
  private destroyed = false;

  add(key: string, subscription: Subscription): void {
    if (this.destroyed) {
      subscription.unsubscribe();
      return;
    }

    // Remove existing subscription with same key
    this.remove(key);
    this.subscriptions.set(key, subscription);
  }

  remove(key: string): boolean {
    const subscription = this.subscriptions.get(key);
    if (subscription) {
      subscription.unsubscribe();
      return this.subscriptions.delete(key);
    }
    return false;
  }

  clear(): void {
    this.subscriptions.forEach(sub => sub.unsubscribe());
    this.subscriptions.clear();
  }

  has(key: string): boolean {
    return this.subscriptions.has(key);
  }

  size(): number {
    return this.subscriptions.size;
  }

  destroy(): void {
    this.clear();
    this.destroyed = true;
  }
}

/**
 * Conditional subscription management
 */
export class ConditionalSubscriptionManager {
  private subscriptions = new Map<string, {
    subscription: Subscription;
    condition: () => boolean;
    cleanup?: () => void;
  }>();

  private checkInterval?: number;

  constructor(private checkIntervalMs: number = 5000) {
    this.startConditionalCheck();
  }

  addConditional(
    key: string, 
    subscription: Subscription, 
    condition: () => boolean,
    cleanup?: () => void
  ): void {
    this.subscriptions.set(key, {
      subscription,
      condition,
      cleanup
    });
  }

  private startConditionalCheck(): void {
    this.checkInterval = window.setInterval(() => {
      this.checkConditions();
    }, this.checkIntervalMs);
  }

  private checkConditions(): void {
    for (const [key, { subscription, condition, cleanup }] of this.subscriptions) {
      if (!condition()) {
        subscription.unsubscribe();
        cleanup?.();
        this.subscriptions.delete(key);
      }
    }
  }

  destroy(): void {
    if (this.checkInterval) {
      clearInterval(this.checkInterval);
      this.checkInterval = undefined;
    }

    this.subscriptions.forEach(({ subscription, cleanup }) => {
      subscription.unsubscribe();
      cleanup?.();
    });
    this.subscriptions.clear();
  }
}

/**
 * Auto-disposing observables
 */
export const createAutoDisposingObservable = <T>(
  factory: () => Observable<T>,
  lifetime: number
): Observable<T> => {
  return new Observable<T>(subscriber => {
    const source = factory();
    const subscription = source.subscribe(subscriber);

    // Auto-dispose after lifetime
    const timeout = setTimeout(() => {
      subscription.unsubscribe();
      subscriber.complete();
    }, lifetime);

    return () => {
      clearTimeout(timeout);
      subscription.unsubscribe();
    };
  });
};

// Usage examples
const dynamicComponent = {
  subscriptionPool: new SubscriptionPool(),
  conditionalManager: new ConditionalSubscriptionManager(),

  onUserSelected(userId: string): void {
    // Dynamic subscription based on user selection
    const userDataSub = this.userService.getUserData(userId).subscribe(data => {
      this.displayUserData(data);
    });

    this.subscriptionPool.add(`user-${userId}`, userDataSub);
  },

  onFeatureEnabled(feature: string): void {
    // Conditional subscription that auto-removes when feature is disabled
    const featureSub = this.featureService.getFeatureData(feature).subscribe(data => {
      this.processFeatureData(data);
    });

    this.conditionalManager.addConditional(
      `feature-${feature}`,
      featureSub,
      () => this.featureService.isEnabled(feature),
      () => console.log(`Feature ${feature} disabled, subscription removed`)
    );
  },

  destroy(): void {
    this.subscriptionPool.destroy();
    this.conditionalManager.destroy();
  }
};
```

## Detecting Memory Leaks

### Memory Leak Detection Tools

```typescript
// memory-leak-detection.ts

/**
 * Subscription tracker for debugging
 */
export class SubscriptionTracker {
  private static instance: SubscriptionTracker;
  private subscriptions = new Map<string, {
    subscription: Subscription;
    stack: string;
    timestamp: number;
    component?: string;
  }>();

  private static readonly MAX_TRACKED_SUBSCRIPTIONS = 10000;

  static getInstance(): SubscriptionTracker {
    if (!SubscriptionTracker.instance) {
      SubscriptionTracker.instance = new SubscriptionTracker();
    }
    return SubscriptionTracker.instance;
  }

  track(subscription: Subscription, component?: string): string {
    const id = this.generateId();
    const stack = this.captureStack();

    if (this.subscriptions.size >= SubscriptionTracker.MAX_TRACKED_SUBSCRIPTIONS) {
      console.warn('Subscription tracker limit reached. Possible memory leak!');
      this.cleanup();
    }

    this.subscriptions.set(id, {
      subscription,
      stack,
      timestamp: Date.now(),
      component
    });

    // Monkey patch unsubscribe to track disposal
    const originalUnsubscribe = subscription.unsubscribe.bind(subscription);
    subscription.unsubscribe = () => {
      this.untrack(id);
      originalUnsubscribe();
    };

    return id;
  }

  untrack(id: string): void {
    this.subscriptions.delete(id);
  }

  getActiveSubscriptions(): Array<{
    id: string;
    component?: string;
    age: number;
    stack: string;
  }> {
    const now = Date.now();
    return Array.from(this.subscriptions.entries()).map(([id, data]) => ({
      id,
      component: data.component,
      age: now - data.timestamp,
      stack: data.stack
    }));
  }

  findSuspiciousSubscriptions(maxAgeMs: number = 300000): Array<any> {
    return this.getActiveSubscriptions().filter(sub => sub.age > maxAgeMs);
  }

  generateReport(): string {
    const active = this.getActiveSubscriptions();
    const suspicious = this.findSuspiciousSubscriptions();
    
    return `
Subscription Report:
- Total active: ${active.length}
- Suspicious (>5min): ${suspicious.length}
- By component: ${this.groupByComponent(active)}
- Oldest: ${Math.max(...active.map(s => s.age))}ms
    `.trim();
  }

  private generateId(): string {
    return `sub_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }

  private captureStack(): string {
    const stack = new Error().stack || '';
    return stack.split('\n').slice(2, 6).join('\n'); // Get relevant stack frames
  }

  private groupByComponent(subscriptions: any[]): string {
    const groups = subscriptions.reduce((acc, sub) => {
      const key = sub.component || 'unknown';
      acc[key] = (acc[key] || 0) + 1;
      return acc;
    }, {} as Record<string, number>);

    return Object.entries(groups)
      .map(([component, count]) => `${component}: ${count}`)
      .join(', ');
  }

  private cleanup(): void {
    // Remove old tracked subscriptions that are likely already disposed
    const cutoff = Date.now() - 600000; // 10 minutes
    for (const [id, data] of this.subscriptions) {
      if (data.timestamp < cutoff) {
        this.subscriptions.delete(id);
      }
    }
  }
}

/**
 * Memory monitoring utility
 */
export class MemoryMonitor {
  private measurements: Array<{
    timestamp: number;
    usedJSHeapSize: number;
    totalJSHeapSize: number;
    jsHeapSizeLimit: number;
  }> = [];

  private monitoringInterval?: number;

  startMonitoring(intervalMs: number = 5000): void {
    if (this.monitoringInterval) {
      this.stopMonitoring();
    }

    this.monitoringInterval = window.setInterval(() => {
      this.takeMeasurement();
    }, intervalMs);
  }

  stopMonitoring(): void {
    if (this.monitoringInterval) {
      clearInterval(this.monitoringInterval);
      this.monitoringInterval = undefined;
    }
  }

  private takeMeasurement(): void {
    if ('memory' in performance) {
      const memory = (performance as any).memory;
      this.measurements.push({
        timestamp: Date.now(),
        usedJSHeapSize: memory.usedJSHeapSize,
        totalJSHeapSize: memory.totalJSHeapSize,
        jsHeapSizeLimit: memory.jsHeapSizeLimit
      });

      // Keep only last 100 measurements
      if (this.measurements.length > 100) {
        this.measurements.shift();
      }

      this.checkForLeaks();
    }
  }

  private checkForLeaks(): void {
    if (this.measurements.length < 10) return;

    const recent = this.measurements.slice(-10);
    const growthRate = this.calculateGrowthRate(recent);

    if (growthRate > 1024 * 1024) { // 1MB growth per measurement
      console.warn('Potential memory leak detected!', {
        growthRate: `${(growthRate / 1024 / 1024).toFixed(2)}MB per measurement`,
        currentUsage: `${(recent[recent.length - 1].usedJSHeapSize / 1024 / 1024).toFixed(2)}MB`
      });
    }
  }

  private calculateGrowthRate(measurements: typeof this.measurements): number {
    if (measurements.length < 2) return 0;

    const first = measurements[0].usedJSHeapSize;
    const last = measurements[measurements.length - 1].usedJSHeapSize;
    
    return (last - first) / measurements.length;
  }

  getMemoryStats(): any {
    if (this.measurements.length === 0) return null;

    const latest = this.measurements[this.measurements.length - 1];
    const growthRate = this.calculateGrowthRate(this.measurements);

    return {
      current: {
        used: `${(latest.usedJSHeapSize / 1024 / 1024).toFixed(2)}MB`,
        total: `${(latest.totalJSHeapSize / 1024 / 1024).toFixed(2)}MB`,
        limit: `${(latest.jsHeapSizeLimit / 1024 / 1024).toFixed(2)}MB`
      },
      growth: `${(growthRate / 1024 / 1024).toFixed(2)}MB per measurement`,
      utilization: `${((latest.usedJSHeapSize / latest.jsHeapSizeLimit) * 100).toFixed(2)}%`
    };
  }
}

// Debugging utilities
export const debugSubscription = (observable: Observable<any>, label: string = 'Debug') => {
  return observable.pipe(
    tap({
      subscribe: () => console.log(`[${label}] Subscribed`),
      next: (value) => console.log(`[${label}] Next:`, value),
      error: (error) => console.error(`[${label}] Error:`, error),
      complete: () => console.log(`[${label}] Complete`),
      unsubscribe: () => console.log(`[${label}] Unsubscribed`)
    })
  );
};
```

### Component Memory Profiling

```typescript
// component-memory-profiling.ts

/**
 * Decorator for automatic subscription tracking
 */
export function TrackSubscriptions(componentName: string) {
  return function<T extends new(...args: any[]) => any>(constructor: T) {
    const tracker = SubscriptionTracker.getInstance();

    return class extends constructor {
      private _trackedSubscriptions = new Set<string>();

      ngOnInit() {
        console.log(`[${componentName}] Component initialized`);
        if (super.ngOnInit) {
          super.ngOnInit();
        }
      }

      ngOnDestroy() {
        console.log(`[${componentName}] Component destroying, active subscriptions:`, this._trackedSubscriptions.size);
        
        // Check for unsubscribed subscriptions
        if (this._trackedSubscriptions.size > 0) {
          console.warn(`[${componentName}] Potential memory leak: ${this._trackedSubscriptions.size} subscriptions not cleaned up`);
        }

        if (super.ngOnDestroy) {
          super.ngOnDestroy();
        }
      }

      // Override subscribe methods to track subscriptions
      subscribe(observable: Observable<any>): Subscription {
        const subscription = observable.subscribe.apply(observable, Array.from(arguments).slice(1));
        const id = tracker.track(subscription, componentName);
        this._trackedSubscriptions.add(id);

        // Override unsubscribe to remove from tracking
        const originalUnsubscribe = subscription.unsubscribe.bind(subscription);
        subscription.unsubscribe = () => {
          this._trackedSubscriptions.delete(id);
          originalUnsubscribe();
        };

        return subscription;
      }
    };
  };
}

/**
 * Memory-aware component base class
 */
export abstract class MemoryAwareComponent implements OnInit, OnDestroy {
  private readonly _subscriptions = new SubscriptionPool();
  private readonly _destroy$ = new Subject<void>();
  private readonly _memoryMonitor = new MemoryMonitor();

  protected readonly destroy$ = this._destroy$.asObservable();

  ngOnInit(): void {
    this._memoryMonitor.startMonitoring();
    this.onInit();
  }

  ngOnDestroy(): void {
    console.log('Memory stats before destroy:', this._memoryMonitor.getMemoryStats());
    
    this._subscriptions.destroy();
    this._destroy$.next();
    this._destroy$.complete();
    this._memoryMonitor.stopMonitoring();
    
    this.onDestroy();
  }

  protected abstract onInit(): void;
  protected abstract onDestroy(): void;

  protected addSubscription(key: string, subscription: Subscription): void {
    this._subscriptions.add(key, subscription);
  }

  protected removeSubscription(key: string): boolean {
    return this._subscriptions.remove(key);
  }

  protected takeUntilDestroy<T>() {
    return takeUntil<T>(this._destroy$);
  }

  protected getMemoryStats(): any {
    return this._memoryMonitor.getMemoryStats();
  }
}

// Usage example
@TrackSubscriptions('UserProfileComponent')
export class UserProfileComponent extends MemoryAwareComponent {
  userProfile$ = this.userService.currentUser$.pipe(
    this.takeUntilDestroy(),
    shareReplay(1)
  );

  protected onInit(): void {
    // Component-specific initialization
    const settingsSub = this.settingsService.settings$.subscribe(settings => {
      this.applySettings(settings);
    });
    this.addSubscription('settings', settingsSub);

    // Auto-tracked via decorator
    this.userService.notifications$.pipe(
      this.takeUntilDestroy()
    ).subscribe(notification => {
      this.showNotification(notification);
    });
  }

  protected onDestroy(): void {
    console.log('UserProfileComponent destroyed, memory stats:', this.getMemoryStats());
  }

  private applySettings(settings: any): void {
    // Apply settings
  }

  private showNotification(notification: any): void {
    // Show notification
  }
}
```

## Optimizing Share Operators

### Efficient Sharing Strategies

```typescript
// efficient-sharing.ts

/**
 * Smart sharing based on subscriber count and data characteristics
 */
export const smartShare = <T>(
  options: {
    refCount?: boolean;
    bufferSize?: number;
    windowTime?: number;
    maxSubscribers?: number;
  } = {}
) => {
  const {
    refCount = true,
    bufferSize = 1,
    windowTime,
    maxSubscribers = Infinity
  } = options;

  return (source: Observable<T>): Observable<T> => {
    let subscriberCount = 0;
    let sharedObservable: Observable<T> | null = null;

    return new Observable<T>(subscriber => {
      subscriberCount++;

      if (subscriberCount > maxSubscribers) {
        console.warn(`Maximum subscribers (${maxSubscribers}) exceeded for shared observable`);
        subscriber.error(new Error('Too many subscribers'));
        return;
      }

      // Create shared observable on first subscription
      if (!sharedObservable) {
        sharedObservable = source.pipe(
          shareReplay({
            bufferSize,
            windowTime,
            refCount
          })
        );
      }

      const subscription = sharedObservable.subscribe(subscriber);

      return () => {
        subscriberCount--;
        subscription.unsubscribe();

        // Clean up shared observable when no subscribers
        if (subscriberCount === 0 && refCount) {
          sharedObservable = null;
        }
      };
    });
  };
};

/**
 * Conditional sharing based on data size
 */
export const conditionalShare = <T>(
  condition: (value: T) => boolean,
  shareOptions: any = {}
) => {
  return (source: Observable<T>): Observable<T> => {
    return new Observable<T>(subscriber => {
      let shouldShare = false;
      let sharedSource: Observable<T> | null = null;

      const subscription = source.subscribe({
        next: value => {
          if (condition(value) && !shouldShare) {
            shouldShare = true;
            sharedSource = source.pipe(shareReplay(shareOptions));
            // Re-subscribe to shared source
            subscription.unsubscribe();
            sharedSource.subscribe(subscriber);
          } else if (!shouldShare) {
            subscriber.next(value);
          }
        },
        error: err => subscriber.error(err),
        complete: () => subscriber.complete()
      });

      return () => subscription.unsubscribe();
    });
  };
};

/**
 * Memory-conscious caching
 */
export class MemoryConsciousCache<K, V> {
  private cache = new Map<K, { value: V; timestamp: number; accessCount: number }>();
  private readonly maxSize: number;
  private readonly ttl: number;

  constructor(maxSize: number = 100, ttlMs: number = 300000) {
    this.maxSize = maxSize;
    this.ttl = ttlMs;
  }

  get(key: K): V | undefined {
    const entry = this.cache.get(key);
    if (!entry) return undefined;

    const now = Date.now();
    if (now - entry.timestamp > this.ttl) {
      this.cache.delete(key);
      return undefined;
    }

    entry.accessCount++;
    return entry.value;
  }

  set(key: K, value: V): void {
    const now = Date.now();

    // Evict if at capacity
    if (this.cache.size >= this.maxSize) {
      this.evictLeastUsed();
    }

    this.cache.set(key, {
      value,
      timestamp: now,
      accessCount: 1
    });
  }

  private evictLeastUsed(): void {
    let leastUsedKey: K | null = null;
    let leastAccessCount = Infinity;

    for (const [key, entry] of this.cache) {
      if (entry.accessCount < leastAccessCount) {
        leastAccessCount = entry.accessCount;
        leastUsedKey = key;
      }
    }

    if (leastUsedKey !== null) {
      this.cache.delete(leastUsedKey);
    }
  }

  clear(): void {
    this.cache.clear();
  }

  size(): number {
    return this.cache.size;
  }
}

// Cached observable creator
export const createCachedObservable = <T>(
  factory: () => Observable<T>,
  cacheKey: string,
  cache: MemoryConsciousCache<string, Observable<T>>
): Observable<T> => {
  const cached = cache.get(cacheKey);
  if (cached) {
    return cached;
  }

  const observable = factory().pipe(
    shareReplay({ bufferSize: 1, refCount: true }),
    finalize(() => {
      // Remove from cache when observable completes
      cache.clear();
    })
  );

  cache.set(cacheKey, observable);
  return observable;
};
```

## Testing Memory Management

### Memory Leak Testing

```typescript
// memory-leak-testing.spec.ts

describe('Memory Management', () => {
  let memoryMonitor: MemoryMonitor;
  let subscriptionTracker: SubscriptionTracker;

  beforeEach(() => {
    memoryMonitor = new MemoryMonitor();
    subscriptionTracker = SubscriptionTracker.getInstance();
  });

  afterEach(() => {
    memoryMonitor.stopMonitoring();
  });

  it('should not leak memory on component destruction', fakeAsync(() => {
    const fixture = TestBed.createComponent(TestComponent);
    const component = fixture.componentInstance;

    const initialSubscriptions = subscriptionTracker.getActiveSubscriptions().length;

    fixture.detectChanges();
    tick(1000);

    // Component should have active subscriptions
    const activeSubscriptions = subscriptionTracker.getActiveSubscriptions().length;
    expect(activeSubscriptions).toBeGreaterThan(initialSubscriptions);

    // Destroy component
    fixture.destroy();
    tick(100);

    // Subscriptions should be cleaned up
    const finalSubscriptions = subscriptionTracker.getActiveSubscriptions().length;
    expect(finalSubscriptions).toBe(initialSubscriptions);
  }));

  it('should handle high-frequency subscriptions without memory growth', fakeAsync(() => {
    memoryMonitor.startMonitoring(100);
    
    const source$ = interval(1);
    const subscriptions: Subscription[] = [];

    // Create many short-lived subscriptions
    for (let i = 0; i < 1000; i++) {
      const sub = source$.pipe(take(1)).subscribe();
      subscriptions.push(sub);
    }

    tick(2000);

    // Check memory growth
    const stats = memoryMonitor.getMemoryStats();
    expect(stats).toBeDefined();
    
    // Cleanup
    subscriptions.forEach(sub => sub.unsubscribe());
    tick(1000);
  }));

  it('should detect subscription leaks in components', () => {
    const tracker = SubscriptionTracker.getInstance();
    
    class LeakyComponent {
      ngOnInit() {
        // Simulate subscription without cleanup
        const sub = interval(1000).subscribe();
        tracker.track(sub, 'LeakyComponent');
        // No unsubscribe!
      }
    }

    const component = new LeakyComponent();
    component.ngOnInit();

    const suspicious = tracker.findSuspiciousSubscriptions(0);
    expect(suspicious.length).toBeGreaterThan(0);
    expect(suspicious[0].component).toBe('LeakyComponent');
  });

  it('should properly manage subscription pools', () => {
    const pool = new SubscriptionPool();
    
    // Add subscriptions
    pool.add('test1', interval(1000).subscribe());
    pool.add('test2', interval(1000).subscribe());
    
    expect(pool.size()).toBe(2);
    
    // Remove specific subscription
    expect(pool.remove('test1')).toBe(true);
    expect(pool.size()).toBe(1);
    
    // Clear all
    pool.clear();
    expect(pool.size()).toBe(0);
  });

  it('should handle conditional subscription cleanup', fakeAsync(() => {
    const manager = new ConditionalSubscriptionManager(100);
    let shouldKeep = true;

    manager.addConditional(
      'test',
      interval(100).subscribe(),
      () => shouldKeep
    );

    tick(50);
    expect(manager['subscriptions'].size).toBe(1);

    // Condition becomes false
    shouldKeep = false;
    tick(150); // Wait for condition check

    expect(manager['subscriptions'].size).toBe(0);
    
    manager.destroy();
  }));
});

/**
 * Memory stress testing utility
 */
export class MemoryStressTester {
  private subscriptions: Subscription[] = [];
  private observables: Observable<any>[] = [];

  createManyObservables(count: number): void {
    for (let i = 0; i < count; i++) {
      const obs = interval(Math.random() * 1000).pipe(
        map(x => ({ id: i, value: x, data: new Array(100).fill(Math.random()) })),
        shareReplay(1)
      );
      this.observables.push(obs);
    }
  }

  createManySubscriptions(count: number): void {
    this.observables.forEach(obs => {
      for (let i = 0; i < count; i++) {
        const sub = obs.subscribe();
        this.subscriptions.push(sub);
      }
    });
  }

  cleanup(): void {
    this.subscriptions.forEach(sub => sub.unsubscribe());
    this.subscriptions.length = 0;
    this.observables.length = 0;
  }

  getStats(): any {
    return {
      observables: this.observables.length,
      subscriptions: this.subscriptions.length,
      memory: (performance as any).memory
    };
  }
}
```

## Best Practices

### 1. Subscription Management
- Always unsubscribe from manual subscriptions
- Use takeUntil pattern for automatic cleanup
- Prefer async pipe when possible
- Use subscription pools for dynamic subscriptions

### 2. Observable Sharing
- Use shareReplay with refCount for expensive operations
- Implement memory-conscious caching strategies
- Monitor subscriber counts for shared observables
- Clean up shared resources when no longer needed

### 3. Memory Monitoring
- Implement subscription tracking in development
- Monitor memory usage in long-running applications
- Set up alerts for suspicious memory growth
- Profile components for memory leaks

### 4. Component Design
- Design components with cleanup in mind
- Use base classes for consistent memory management
- Implement proper lifecycle hooks
- Test memory behavior thoroughly

### 5. Error Handling
- Handle errors in subscriptions properly
- Prevent error-induced memory leaks
- Implement circuit breakers for failing streams
- Clean up resources on errors

## Common Pitfalls

### 1. Forgotten Unsubscribes
```typescript
// ❌ Bad: No cleanup
ngOnInit() {
  interval(1000).subscribe(x => console.log(x)); // Memory leak!
}

// ✅ Good: Proper cleanup
ngOnInit() {
  interval(1000).pipe(
    takeUntil(this.destroy$)
  ).subscribe(x => console.log(x));
}
```

### 2. Circular References
```typescript
// ❌ Bad: Circular reference
const subject = new Subject();
subject.subscribe(value => {
  subject.next(value); // Circular reference!
});

// ✅ Good: Avoid circularity
const subject = new Subject();
subject.pipe(
  filter(value => someCondition(value))
).subscribe(value => {
  // Process without circular reference
});
```

### 3. Shared Observable Leaks
```typescript
// ❌ Bad: Share without cleanup
const shared$ = source$.pipe(share());

// ✅ Good: Proper sharing
const shared$ = source$.pipe(
  shareReplay({ bufferSize: 1, refCount: true })
);
```

## Summary

Effective memory management in RxJS applications requires proactive strategies and monitoring. Key takeaways:

- Implement consistent subscription cleanup patterns across your application
- Use monitoring tools to detect and prevent memory leaks
- Design components with memory management as a primary concern
- Test memory behavior thoroughly, especially in long-running applications
- Choose appropriate sharing strategies based on data characteristics
- Monitor subscription lifecycles and clean up resources promptly
- Implement automatic cleanup mechanisms where possible
- Profile memory usage regularly and address issues early

Proper memory management ensures your reactive applications remain performant and stable over time, providing excellent user experiences even in resource-constrained environments.
