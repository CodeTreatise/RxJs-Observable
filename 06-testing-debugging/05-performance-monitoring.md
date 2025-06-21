# Performance Monitoring & Profiling RxJS Observables

## Table of Contents
1. [Introduction to RxJS Performance](#introduction)
2. [Performance Monitoring Tools](#monitoring-tools)
3. [Profiling Observable Streams](#profiling-streams)
4. [Memory Leak Detection](#memory-leaks)
5. [Performance Optimization Strategies](#optimization)
6. [Angular-Specific Performance](#angular-performance)
7. [Real-World Examples](#examples)
8. [Best Practices](#best-practices)
9. [Exercises](#exercises)

## Introduction to RxJS Performance {#introduction}

Performance monitoring is crucial for RxJS applications to ensure optimal user experience and resource utilization.

### Key Performance Metrics

```typescript
interface RxJSPerformanceMetrics {
  subscriptionCount: number;
  memoryUsage: number;
  executionTime: number;
  operatorChainLength: number;
  hotObservableCount: number;
  coldObservableCount: number;
}
```

### Common Performance Issues

```typescript
// ❌ Performance Anti-patterns
class BadPerformanceExample {
  // Memory leak - no unsubscription
  ngOnInit() {
    this.service.getData().subscribe(data => {
      // Process data
    });
  }

  // Excessive subscriptions
  processData() {
    for (let i = 0; i < 1000; i++) {
      this.http.get(`/api/data/${i}`).subscribe();
    }
  }

  // Heavy computation in operator
  transformData() {
    return this.source$.pipe(
      map(data => {
        // Heavy synchronous computation
        return this.heavyComputation(data);
      })
    );
  }
}
```

## Performance Monitoring Tools {#monitoring-tools}

### Angular DevTools Integration

```typescript
import { Injectable } from '@angular/core';
import { Observable, tap, finalize } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class RxJSMonitoringService {
  private activeSubscriptions = new Map<string, number>();
  private performanceMetrics = new Map<string, any>();

  monitorObservable<T>(
    source$: Observable<T>,
    label: string
  ): Observable<T> {
    const startTime = performance.now();
    let subscriptionCount = 0;

    return source$.pipe(
      tap(() => {
        subscriptionCount++;
        this.updateMetrics(label, {
          subscriptionCount,
          lastEmission: new Date()
        });
      }),
      finalize(() => {
        const endTime = performance.now();
        this.logPerformanceMetrics(label, {
          duration: endTime - startTime,
          totalEmissions: subscriptionCount
        });
      })
    );
  }

  private updateMetrics(label: string, metrics: any): void {
    this.performanceMetrics.set(label, {
      ...this.performanceMetrics.get(label),
      ...metrics
    });
  }

  private logPerformanceMetrics(label: string, metrics: any): void {
    console.group(`🔍 RxJS Performance: ${label}`);
    console.log('Metrics:', metrics);
    console.log('Memory Usage:', this.getMemoryUsage());
    console.groupEnd();
  }

  private getMemoryUsage(): any {
    return (performance as any).memory || {
      usedJSHeapSize: 0,
      totalJSHeapSize: 0
    };
  }

  getActiveSubscriptions(): Map<string, number> {
    return this.activeSubscriptions;
  }

  getPerformanceReport(): any {
    return Object.fromEntries(this.performanceMetrics);
  }
}
```

### Custom Performance Decorator

```typescript
function MonitorPerformance(label?: string) {
  return function (
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    const originalMethod = descriptor.value;
    const methodLabel = label || `${target.constructor.name}.${propertyKey}`;

    descriptor.value = function (...args: any[]) {
      const startTime = performance.now();
      const result = originalMethod.apply(this, args);

      if (result && typeof result.subscribe === 'function') {
        return result.pipe(
          tap(() => {
            const endTime = performance.now();
            console.log(`⏱️ ${methodLabel}: ${endTime - startTime}ms`);
          })
        );
      }

      return result;
    };

    return descriptor;
  };
}

// Usage
class DataService {
  @MonitorPerformance('API Call')
  getData(): Observable<any> {
    return this.http.get('/api/data').pipe(
      map(data => this.processData(data))
    );
  }
}
```

## Profiling Observable Streams {#profiling-streams}

### Stream Profiler Utility

```typescript
interface StreamProfile {
  streamId: string;
  operatorChain: string[];
  emissionCount: number;
  averageEmissionTime: number;
  memorySnapshot: any;
  subscriptionDuration: number;
}

class ObservableProfiler {
  private profiles = new Map<string, StreamProfile>();
  private streamCounter = 0;

  profile<T>(source$: Observable<T>, options?: {
    label?: string;
    sampleInterval?: number;
  }): Observable<T> {
    const streamId = options?.label || `stream-${++this.streamCounter}`;
    const profile: StreamProfile = {
      streamId,
      operatorChain: [],
      emissionCount: 0,
      averageEmissionTime: 0,
      memorySnapshot: {},
      subscriptionDuration: 0
    };

    const startTime = performance.now();
    let emissionTimes: number[] = [];

    return new Observable<T>(observer => {
      const subscription = source$.subscribe({
        next: (value) => {
          const emissionTime = performance.now();
          emissionTimes.push(emissionTime);
          profile.emissionCount++;
          profile.averageEmissionTime = 
            emissionTimes.reduce((a, b) => a + b, 0) / emissionTimes.length;

          // Sample memory usage periodically
          if (profile.emissionCount % (options?.sampleInterval || 10) === 0) {
            profile.memorySnapshot = this.captureMemorySnapshot();
          }

          observer.next(value);
        },
        error: (error) => {
          this.finalizeProfile(streamId, profile, startTime);
          observer.error(error);
        },
        complete: () => {
          this.finalizeProfile(streamId, profile, startTime);
          observer.complete();
        }
      });

      return () => {
        subscription.unsubscribe();
        this.finalizeProfile(streamId, profile, startTime);
      };
    });
  }

  private finalizeProfile(
    streamId: string,
    profile: StreamProfile,
    startTime: number
  ): void {
    profile.subscriptionDuration = performance.now() - startTime;
    this.profiles.set(streamId, profile);
    this.logProfile(profile);
  }

  private captureMemorySnapshot(): any {
    const memory = (performance as any).memory;
    return memory ? {
      used: memory.usedJSHeapSize,
      total: memory.totalJSHeapSize,
      limit: memory.jsHeapSizeLimit
    } : null;
  }

  private logProfile(profile: StreamProfile): void {
    console.group(`📊 Stream Profile: ${profile.streamId}`);
    console.log('Emissions:', profile.emissionCount);
    console.log('Average Emission Time:', profile.averageEmissionTime.toFixed(2), 'ms');
    console.log('Total Duration:', profile.subscriptionDuration.toFixed(2), 'ms');
    console.log('Memory Snapshot:', profile.memorySnapshot);
    console.groupEnd();
  }

  getProfile(streamId: string): StreamProfile | undefined {
    return this.profiles.get(streamId);
  }

  getAllProfiles(): StreamProfile[] {
    return Array.from(this.profiles.values());
  }

  generateReport(): string {
    const profiles = this.getAllProfiles();
    let report = '📈 RxJS Performance Report\n';
    report += '='.repeat(30) + '\n\n';

    profiles.forEach(profile => {
      report += `Stream: ${profile.streamId}\n`;
      report += `  Emissions: ${profile.emissionCount}\n`;
      report += `  Duration: ${profile.subscriptionDuration.toFixed(2)}ms\n`;
      report += `  Avg Emission Time: ${profile.averageEmissionTime.toFixed(2)}ms\n\n`;
    });

    return report;
  }
}
```

### Usage Example

```typescript
@Component({
  selector: 'app-performance-demo',
  template: `
    <div>
      <button (click)="startProfiling()">Start Profiling</button>
      <button (click)="showReport()">Show Report</button>
      <div *ngFor="let item of items$ | async">{{ item }}</div>
    </div>
  `
})
export class PerformanceDemoComponent implements OnInit, OnDestroy {
  private profiler = new ObservableProfiler();
  private destroy$ = new Subject<void>();
  items$!: Observable<any[]>;

  constructor(private dataService: DataService) {}

  ngOnInit() {
    this.items$ = this.profiler.profile(
      this.dataService.getItems(),
      { label: 'data-stream', sampleInterval: 5 }
    ).pipe(
      takeUntil(this.destroy$)
    );
  }

  startProfiling() {
    const testStream$ = interval(100).pipe(
      take(50),
      map(x => x * 2),
      filter(x => x % 4 === 0)
    );

    this.profiler.profile(testStream$, {
      label: 'test-stream'
    }).subscribe();
  }

  showReport() {
    console.log(this.profiler.generateReport());
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## Memory Leak Detection {#memory-leaks}

### Memory Leak Monitor

```typescript
interface MemoryLeakDetector {
  subscriptionRegistry: Map<string, {
    subscription: Subscription;
    stackTrace: string;
    timestamp: number;
    component: string;
  }>;
}

@Injectable({
  providedIn: 'root'
})
export class MemoryLeakDetectionService implements MemoryLeakDetector {
  subscriptionRegistry = new Map();
  private leakThreshold = 100; // Max subscriptions per component

  registerSubscription(
    subscription: Subscription,
    component: string,
    context?: string
  ): void {
    const id = this.generateId();
    const stackTrace = new Error().stack || 'No stack trace available';

    this.subscriptionRegistry.set(id, {
      subscription,
      stackTrace,
      timestamp: Date.now(),
      component: `${component}${context ? `::${context}` : ''}`
    });

    this.checkForLeaks(component);
  }

  unregisterSubscription(subscription: Subscription): void {
    for (const [id, entry] of this.subscriptionRegistry.entries()) {
      if (entry.subscription === subscription) {
        this.subscriptionRegistry.delete(id);
        break;
      }
    }
  }

  private checkForLeaks(component: string): void {
    const componentSubscriptions = Array.from(this.subscriptionRegistry.values())
      .filter(entry => entry.component.startsWith(component));

    if (componentSubscriptions.length > this.leakThreshold) {
      console.warn(`🚨 Potential memory leak detected in ${component}`);
      console.warn(`Active subscriptions: ${componentSubscriptions.length}`);
      this.logSuspiciousSubscriptions(componentSubscriptions);
    }
  }

  private logSuspiciousSubscriptions(subscriptions: any[]): void {
    const oldSubscriptions = subscriptions.filter(
      sub => Date.now() - sub.timestamp > 60000 // 1 minute
    );

    if (oldSubscriptions.length > 0) {
      console.group('🔍 Long-running subscriptions:');
      oldSubscriptions.forEach((sub, index) => {
        console.log(`${index + 1}. Component: ${sub.component}`);
        console.log(`   Age: ${(Date.now() - sub.timestamp) / 1000}s`);
        console.log(`   Stack trace:`, sub.stackTrace);
      });
      console.groupEnd();
    }
  }

  private generateId(): string {
    return Math.random().toString(36).substr(2, 9);
  }

  getActiveSubscriptionsCount(): number {
    return this.subscriptionRegistry.size;
  }

  getLeakReport(): any {
    const subscriptions = Array.from(this.subscriptionRegistry.values());
    const componentGroups = subscriptions.reduce((acc, sub) => {
      const component = sub.component.split('::')[0];
      acc[component] = (acc[component] || 0) + 1;
      return acc;
    }, {} as Record<string, number>);

    return {
      totalSubscriptions: subscriptions.length,
      componentBreakdown: componentGroups,
      oldestSubscription: subscriptions.length > 0 ? 
        Math.min(...subscriptions.map(s => s.timestamp)) : null
    };
  }
}
```

### Auto-Tracking Subscription Decorator

```typescript
function TrackSubscriptions(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor
) {
  const originalMethod = descriptor.value;

  descriptor.value = function (...args: any[]) {
    const result = originalMethod.apply(this, args);
    
    if (result && typeof result.subscribe === 'function') {
      const component = this.constructor.name;
      const leakDetector = inject(MemoryLeakDetectionService);
      
      return new Observable(observer => {
        const subscription = result.subscribe(observer);
        leakDetector.registerSubscription(subscription, component, propertyKey);
        
        return () => {
          leakDetector.unregisterSubscription(subscription);
          subscription.unsubscribe();
        };
      });
    }
    
    return result;
  };

  return descriptor;
}

// Usage
class DataComponent {
  @TrackSubscriptions
  loadData(): Observable<any> {
    return this.http.get('/api/data');
  }
}
```

## Performance Optimization Strategies {#optimization}

### Operator Performance Comparison

```typescript
class PerformanceOptimizer {
  // ✅ Optimized filtering
  optimizedFiltering<T>(
    source$: Observable<T[]>,
    predicate: (item: T) => boolean
  ): Observable<T[]> {
    return source$.pipe(
      // Use map instead of multiple filter operations
      map(items => items.filter(predicate)),
      // Debounce to reduce unnecessary computations
      debounceTime(100),
      // Share to prevent multiple executions
      shareReplay(1)
    );
  }

  // ✅ Batch HTTP requests
  batchRequests<T>(
    requests: Observable<T>[],
    batchSize: number = 5
  ): Observable<T[]> {
    return from(requests).pipe(
      bufferCount(batchSize),
      concatMap(batch => forkJoin(batch)),
      scan((acc, batch) => [...acc, ...batch], [] as T[])
    );
  }

  // ✅ Efficient data transformation
  efficientTransformation<T, R>(
    source$: Observable<T[]>,
    transform: (item: T) => R
  ): Observable<R[]> {
    return source$.pipe(
      map(items => {
        // Use for loop instead of array methods for better performance
        const result: R[] = new Array(items.length);
        for (let i = 0; i < items.length; i++) {
          result[i] = transform(items[i]);
        }
        return result;
      }),
      shareReplay(1)
    );
  }

  // ✅ Memory-efficient infinite scroll
  infiniteScroll<T>(
    loadPage: (page: number) => Observable<T[]>,
    pageSize: number = 20
  ): Observable<T[]> {
    return new Observable<T[]>(observer => {
      let currentPage = 0;
      let allItems: T[] = [];
      let isLoading = false;

      const loadNextPage = () => {
        if (isLoading) return;
        
        isLoading = true;
        loadPage(currentPage).subscribe({
          next: (items) => {
            // Only keep last N pages in memory
            const maxItems = pageSize * 10;
            if (allItems.length > maxItems) {
              allItems = allItems.slice(-maxItems);
            }
            
            allItems = [...allItems, ...items];
            observer.next(allItems);
            currentPage++;
            isLoading = false;
          },
          error: (error) => observer.error(error)
        });
      };

      loadNextPage();
      
      return () => {
        // Cleanup
        allItems = [];
      };
    });
  }
}
```

## Angular-Specific Performance {#angular-performance}

### OnPush Change Detection Optimization

```typescript
@Component({
  selector: 'app-optimized',
  template: `
    <div *ngFor="let item of items$ | async; trackBy: trackByFn">
      {{ item.name }}
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class OptimizedComponent implements OnInit {
  items$: Observable<Item[]>;

  constructor(
    private dataService: DataService,
    private cdr: ChangeDetectorRef
  ) {}

  ngOnInit() {
    this.items$ = this.dataService.getItems().pipe(
      // Optimize for OnPush
      distinctUntilChanged((prev, curr) => 
        JSON.stringify(prev) === JSON.stringify(curr)
      ),
      shareReplay(1)
    );
  }

  trackByFn(index: number, item: Item): any {
    return item.id; // Use unique identifier
  }

  // Manual change detection when needed
  forceUpdate() {
    this.cdr.markForCheck();
  }
}
```

### Subscription Management Performance

```typescript
@Injectable()
export class PerformantSubscriptionManager {
  private subscriptions$ = new Subject<void>();

  // Single takeUntil for all subscriptions
  takeUntilDestroy<T>(): MonoTypeOperatorFunction<T> {
    return takeUntil(this.subscriptions$);
  }

  // Batch subscription cleanup
  destroy(): void {
    this.subscriptions$.next();
    this.subscriptions$.complete();
  }

  // Smart subscription pooling
  createManagedObservable<T>(
    factory: () => Observable<T>,
    cacheKey?: string
  ): Observable<T> {
    if (cacheKey && this.cache.has(cacheKey)) {
      return this.cache.get(cacheKey);
    }

    const obs$ = factory().pipe(
      this.takeUntilDestroy(),
      shareReplay(1)
    );

    if (cacheKey) {
      this.cache.set(cacheKey, obs$);
    }

    return obs$;
  }

  private cache = new Map<string, Observable<any>>();
}
```

## Real-World Examples {#examples}

### E-commerce Search Performance

```typescript
@Component({
  selector: 'app-product-search',
  template: `
    <input 
      #searchInput 
      placeholder="Search products..."
      (input)="search$.next($event.target.value)">
    
    <div class="performance-metrics">
      Search Time: {{ searchTime }}ms |
      Results: {{ (results$ | async)?.length }} |
      Memory: {{ memoryUsage }}MB
    </div>

    <div *ngFor="let product of results$ | async" class="product">
      {{ product.name }} - ${{ product.price }}
    </div>
  `
})
export class ProductSearchComponent implements OnInit, OnDestroy {
  search$ = new Subject<string>();
  results$: Observable<Product[]>;
  searchTime = 0;
  memoryUsage = 0;
  
  private profiler = new ObservableProfiler();
  private destroy$ = new Subject<void>();

  constructor(private productService: ProductService) {}

  ngOnInit() {
    this.results$ = this.search$.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      tap(() => console.time('search')),
      switchMap(query => 
        this.profiler.profile(
          this.productService.searchProducts(query),
          { label: 'product-search' }
        )
      ),
      tap(() => {
        console.timeEnd('search');
        this.updatePerformanceMetrics();
      }),
      shareReplay(1),
      takeUntil(this.destroy$)
    );
  }

  private updatePerformanceMetrics(): void {
    const profile = this.profiler.getProfile('product-search');
    if (profile) {
      this.searchTime = profile.averageEmissionTime;
    }

    const memory = (performance as any).memory;
    if (memory) {
      this.memoryUsage = Math.round(memory.usedJSHeapSize / 1024 / 1024);
    }
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### Real-time Dashboard Performance

```typescript
@Component({
  selector: 'app-dashboard',
  template: `
    <div class="dashboard">
      <div class="performance-panel">
        <h3>Performance Metrics</h3>
        <p>Active Streams: {{ activeStreams }}</p>
        <p>Total Emissions: {{ totalEmissions }}</p>
        <p>Memory Usage: {{ memoryUsage }}MB</p>
        <button (click)="optimizePerformance()">Optimize</button>
      </div>

      <div class="data-panels">
        <div *ngFor="let metric of metrics$ | async" class="metric-panel">
          {{ metric.name }}: {{ metric.value }}
        </div>
      </div>
    </div>
  `
})
export class DashboardComponent implements OnInit, OnDestroy {
  metrics$: Observable<Metric[]>;
  activeStreams = 0;
  totalEmissions = 0;
  memoryUsage = 0;

  private metricsSubject$ = new BehaviorSubject<Metric[]>([]);
  private performanceTimer?: number;
  private destroy$ = new Subject<void>();

  constructor(
    private metricsService: MetricsService,
    private performanceService: RxJSMonitoringService
  ) {}

  ngOnInit() {
    this.setupMetricsStream();
    this.startPerformanceMonitoring();
  }

  private setupMetricsStream(): void {
    // Optimized metrics collection
    this.metrics$ = interval(1000).pipe(
      startWith(0),
      switchMap(() => this.metricsService.getMetrics()),
      distinctUntilChanged((prev, curr) => 
        JSON.stringify(prev) === JSON.stringify(curr)
      ),
      tap(metrics => {
        this.totalEmissions += metrics.length;
        this.metricsSubject$.next(metrics);
      }),
      shareReplay(1),
      takeUntil(this.destroy$)
    );
  }

  private startPerformanceMonitoring(): void {
    this.performanceTimer = window.setInterval(() => {
      const report = this.performanceService.getPerformanceReport();
      this.activeStreams = Object.keys(report).length;
      
      const memory = (performance as any).memory;
      if (memory) {
        this.memoryUsage = Math.round(memory.usedJSHeapSize / 1024 / 1024);
      }
    }, 2000);
  }

  optimizePerformance(): void {
    // Force garbage collection (if available in dev tools)
    if ((window as any).gc) {
      (window as any).gc();
    }

    // Reset performance counters
    this.totalEmissions = 0;
    console.log('🚀 Performance optimization triggered');
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
    
    if (this.performanceTimer) {
      clearInterval(this.performanceTimer);
    }
  }
}
```

## Best Practices {#best-practices}

### Performance Monitoring Checklist

```typescript
const PERFORMANCE_BEST_PRACTICES = {
  monitoring: [
    '✅ Use performance profiling tools regularly',
    '✅ Monitor memory usage and subscription counts',
    '✅ Set up automated performance alerts',
    '✅ Profile in production-like environments',
    '✅ Track performance metrics over time'
  ],
  
  optimization: [
    '✅ Use shareReplay() for expensive operations',
    '✅ Implement proper unsubscription strategies',
    '✅ Debounce user input streams',
    '✅ Use OnPush change detection with observables',
    '✅ Batch HTTP requests when possible',
    '✅ Implement virtual scrolling for large lists'
  ],
  
  debugging: [
    '✅ Use meaningful labels for streams',
    '✅ Implement custom operators for complex logic',
    '✅ Add performance tracking decorators',
    '✅ Monitor for memory leaks regularly',
    '✅ Use marble diagrams for visualization'
  ]
};
```

### Performance Budget Guidelines

```typescript
interface PerformanceBudget {
  maxSubscriptions: number;
  maxMemoryUsage: number; // MB
  maxEmissionDelay: number; // ms
  maxOperatorChainLength: number;
}

const PERFORMANCE_BUDGETS: Record<string, PerformanceBudget> = {
  mobile: {
    maxSubscriptions: 50,
    maxMemoryUsage: 100,
    maxEmissionDelay: 100,
    maxOperatorChainLength: 10
  },
  desktop: {
    maxSubscriptions: 200,
    maxMemoryUsage: 500,
    maxEmissionDelay: 50,
    maxOperatorChainLength: 15
  },
  server: {
    maxSubscriptions: 1000,
    maxMemoryUsage: 2000,
    maxEmissionDelay: 10,
    maxOperatorChainLength: 20
  }
};
```

## Exercises {#exercises}

### Exercise 1: Performance Profiler

Create a comprehensive performance profiler that tracks:
- Subscription lifecycle
- Memory usage patterns
- Emission frequency analysis
- Operator performance comparison

### Exercise 2: Memory Leak Detector

Implement an advanced memory leak detection system that:
- Automatically tracks all subscriptions
- Identifies potential leak patterns
- Provides actionable remediation suggestions
- Integrates with Angular component lifecycle

### Exercise 3: Real-time Performance Dashboard

Build a performance monitoring dashboard that displays:
- Live performance metrics
- Memory usage graphs
- Subscription count trends
- Performance optimization recommendations

### Exercise 4: Performance Budget Enforcer

Create a system that:
- Enforces performance budgets
- Alerts when thresholds are exceeded
- Provides automated optimization suggestions
- Integrates with CI/CD pipelines

---

**Next Steps:**
- Explore marble diagram debugging techniques
- Learn about production performance monitoring
- Practice with real-world performance scenarios
- Master advanced debugging strategies
