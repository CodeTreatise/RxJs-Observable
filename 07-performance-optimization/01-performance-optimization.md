# RxJS Performance Optimization

## Table of Contents
1. [Introduction to RxJS Performance](#introduction)
2. [Performance Fundamentals](#fundamentals)
3. [Operator Optimization Strategies](#operator-optimization)
4. [Subscription Management](#subscription-management)
5. [Stream Composition Optimization](#stream-composition)
6. [Angular-Specific Optimizations](#angular-optimizations)
7. [Performance Monitoring & Measurement](#monitoring)
8. [Real-World Optimization Examples](#examples)
9. [Best Practices](#best-practices)
10. [Exercises](#exercises)

## Introduction to RxJS Performance {#introduction}

Performance optimization in RxJS applications is crucial for delivering smooth user experiences, especially in complex Angular applications with multiple data streams and reactive patterns.

### Performance Impact Areas

```typescript
interface RxJSPerformanceMetrics {
  // Memory Performance
  subscriptionCount: number;
  memoryLeakSources: string[];
  heapUsage: number;
  
  // Execution Performance
  operatorChainLength: number;
  emissionFrequency: number;
  processingTime: number;
  
  // User Experience
  firstContentfulPaint: number;
  interactionDelay: number;
  changeDetectionCycles: number;
}

// Performance bottleneck indicators
const PERFORMANCE_THRESHOLDS = {
  maxSubscriptions: 100,
  maxOperatorChain: 15,
  maxEmissionDelay: 16, // 60fps threshold
  maxMemoryGrowth: 50, // MB per minute
  maxChangeDetectionTime: 5 // ms
};
```

### Common Performance Anti-patterns

```typescript
// ❌ ANTI-PATTERN: Excessive subscriptions
class BadPerformanceComponent implements OnInit {
  ngOnInit() {
    // Creates new subscription on every call
    this.getUserData().subscribe(user => {
      this.getOrderHistory(user.id).subscribe(orders => {
        this.getOrderDetails(orders).subscribe(details => {
          // Nested subscriptions create memory leaks
        });
      });
    });
  }

  // Heavy computation in map operator
  processData() {
    return this.dataStream$.pipe(
      map(data => {
        // ❌ Synchronous heavy computation blocks the thread
        return this.heavyComputationSync(data);
      })
    );
  }

  // Unoptimized filtering
  filterData() {
    return this.items$.pipe(
      // ❌ Multiple operators doing similar work
      filter(items => items.length > 0),
      map(items => items.filter(item => item.active)),
      map(items => items.filter(item => item.visible)),
      map(items => items.sort((a, b) => a.name.localeCompare(b.name)))
    );
  }
}
```

## Performance Fundamentals {#fundamentals}

### Observable Creation Optimization

```typescript
class OptimizedObservableCreation {
  // ✅ OPTIMIZED: Use factory functions for expensive observables
  createOptimizedDataStream(): Observable<Data[]> {
    return defer(() => {
      // Lazy creation - only when subscribed
      const startTime = performance.now();
      
      return this.http.get<Data[]>('/api/data').pipe(
        map(data => this.optimizeDataStructure(data)),
        shareReplay(1), // Cache result for multiple subscribers
        tap(() => {
          const endTime = performance.now();
          console.log(`Data stream created in ${endTime - startTime}ms`);
        })
      );
    });
  }

  // ✅ Optimized data structure transformation
  private optimizeDataStructure(data: any[]): Data[] {
    // Use for loop instead of array methods for better performance
    const result: Data[] = new Array(data.length);
    for (let i = 0; i < data.length; i++) {
      result[i] = {
        id: data[i].id,
        name: data[i].name,
        // Pre-compute expensive calculations
        displayName: this.computeDisplayName(data[i]),
        sortKey: data[i].name.toLowerCase()
      };
    }
    return result;
  }

  // ✅ Efficient stream combination
  combineStreamsEfficiently(): Observable<CombinedData> {
    return combineLatest([
      this.stream1$.pipe(distinctUntilChanged()),
      this.stream2$.pipe(distinctUntilChanged()),
      this.stream3$.pipe(distinctUntilChanged())
    ]).pipe(
      // Debounce to prevent excessive recalculations
      debounceTime(0),
      map(([data1, data2, data3]) => ({
        ...data1,
        ...data2,
        computed: this.computeCombinedValue(data1, data2, data3)
      })),
      shareReplay(1)
    );
  }
}
```

### Memory-Efficient Stream Patterns

```typescript
class MemoryEfficientStreams {
  // ✅ Efficient infinite scroll implementation
  createInfiniteScroll<T>(
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
          next: (newItems) => {
            // Memory optimization: limit total items in memory
            const maxItems = pageSize * 10; // Keep only 10 pages
            if (allItems.length > maxItems) {
              // Remove old items from beginning
              allItems = allItems.slice(pageSize);
            }
            
            allItems = [...allItems, ...newItems];
            observer.next(allItems);
            currentPage++;
            isLoading = false;
          },
          error: (error) => observer.error(error)
        });
      };

      // Expose load function
      (observer as any).loadMore = loadNextPage;
      loadNextPage(); // Load first page

      return () => {
        allItems = []; // Clean up memory
      };
    });
  }

  // ✅ Efficient data caching with TTL
  createCachedStream<T>(
    source$: Observable<T>,
    ttlMs: number = 300000 // 5 minutes
  ): Observable<T> {
    let cache: { data: T; timestamp: number } | null = null;
    
    return defer(() => {
      const now = Date.now();
      
      if (cache && (now - cache.timestamp) < ttlMs) {
        return of(cache.data);
      }
      
      return source$.pipe(
        tap(data => {
          cache = { data, timestamp: now };
        }),
        shareReplay(1)
      );
    });
  }

  // ✅ Backpressure handling for high-frequency streams
  handleBackpressure<T>(
    source$: Observable<T>,
    strategy: 'drop' | 'buffer' | 'sample' = 'sample'
  ): Observable<T> {
    switch (strategy) {
      case 'drop':
        // Drop emissions if consumer is slow
        return source$.pipe(
          concatMap(value => 
            of(value).pipe(
              delay(0), // Yield to event loop
              take(1)
            )
          )
        );
        
      case 'buffer':
        // Buffer emissions and process in batches
        return source$.pipe(
          bufferTime(100),
          filter(buffer => buffer.length > 0),
          mergeMap(buffer => from(buffer))
        );
        
      case 'sample':
      default:
        // Sample latest value periodically
        return source$.pipe(
          sampleTime(16) // 60fps sampling
        );
    }
  }
}
```

## Operator Optimization Strategies {#operator-optimization}

### High-Performance Operator Patterns

```typescript
class OperatorOptimization {
  // ✅ Optimized filtering and transformation
  optimizeFilterAndTransform<T, R>(
    source$: Observable<T[]>,
    predicate: (item: T) => boolean,
    transform: (item: T) => R
  ): Observable<R[]> {
    return source$.pipe(
      map(items => {
        // Single pass through array instead of multiple operators
        const result: R[] = [];
        for (let i = 0; i < items.length; i++) {
          if (predicate(items[i])) {
            result.push(transform(items[i]));
          }
        }
        return result;
      }),
      // Only emit if result actually changed
      distinctUntilChanged((prev, curr) => 
        prev.length === curr.length && 
        prev.every((item, index) => item === curr[index])
      )
    );
  }

  // ✅ Efficient search implementation
  createOptimizedSearch<T>(
    source$: Observable<T[]>,
    searchTerm$: Observable<string>,
    searchFn: (items: T[], term: string) => T[]
  ): Observable<T[]> {
    return combineLatest([
      source$.pipe(shareReplay(1)),
      searchTerm$.pipe(
        debounceTime(300),
        distinctUntilChanged(),
        startWith('')
      )
    ]).pipe(
      map(([items, term]) => {
        if (!term.trim()) return items;
        
        // Use binary search or indexed search for large datasets
        return this.performOptimizedSearch(items, term, searchFn);
      }),
      shareReplay(1)
    );
  }

  private performOptimizedSearch<T>(
    items: T[],
    term: string,
    searchFn: (items: T[], term: string) => T[]
  ): T[] {
    // For large datasets, consider using web workers
    if (items.length > 10000) {
      return this.searchInWebWorker(items, term, searchFn);
    }
    
    return searchFn(items, term);
  }

  // ✅ Web Worker integration for heavy computations
  private searchInWebWorker<T>(
    items: T[],
    term: string,
    searchFn: (items: T[], term: string) => T[]
  ): T[] {
    // Fallback to synchronous search if Web Workers not available
    if (typeof Worker === 'undefined') {
      return searchFn(items, term);
    }

    // This would typically return an Observable for async processing
    // Simplified for demonstration
    return searchFn(items, term);
  }

  // ✅ Optimized error handling
  createResilientStream<T>(
    source$: Observable<T>,
    retryConfig: {
      maxRetries: number;
      delay: number;
      backoffMultiplier: number;
    }
  ): Observable<T> {
    return source$.pipe(
      retryWhen(errors =>
        errors.pipe(
          scan((acc, error) => ({ ...acc, count: acc.count + 1 }), 
               { count: 0 }),
          mergeMap(({ count }) => {
            if (count >= retryConfig.maxRetries) {
              return throwError(() => new Error('Max retries exceeded'));
            }
            
            const delay = retryConfig.delay * 
                         Math.pow(retryConfig.backoffMultiplier, count);
            
            return timer(delay);
          })
        )
      ),
      catchError(error => {
        console.error('Stream failed permanently:', error);
        return EMPTY; // or return fallback observable
      })
    );
  }
}
```

### Custom High-Performance Operators

```typescript
// ✅ Custom operator for efficient debouncing with leading edge
function debounceTimeWithLeading<T>(
  dueTime: number,
  leading: boolean = true
): MonoTypeOperatorFunction<T> {
  return (source: Observable<T>) => {
    return new Observable<T>(observer => {
      let lastEmissionTime = 0;
      let timeoutId: any = null;
      let hasEmitted = false;

      return source.subscribe({
        next(value) {
          const now = Date.now();
          
          if (leading && (!hasEmitted || (now - lastEmissionTime) >= dueTime)) {
            observer.next(value);
            lastEmissionTime = now;
            hasEmitted = true;
          }
          
          if (timeoutId) {
            clearTimeout(timeoutId);
          }
          
          timeoutId = setTimeout(() => {
            if (!leading || (Date.now() - lastEmissionTime) >= dueTime) {
              observer.next(value);
              lastEmissionTime = Date.now();
            }
          }, dueTime);
        },
        error(err) {
          if (timeoutId) clearTimeout(timeoutId);
          observer.error(err);
        },
        complete() {
          if (timeoutId) clearTimeout(timeoutId);
          observer.complete();
        }
      });
    });
  };
}

// ✅ Custom operator for memory-efficient window operations
function slidingWindowEfficient<T>(
  windowSize: number
): OperatorFunction<T, T[]> {
  return (source: Observable<T>) => {
    return new Observable<T[]>(observer => {
      const window: T[] = [];
      
      return source.subscribe({
        next(value) {
          window.push(value);
          
          // Maintain window size efficiently
          if (window.length > windowSize) {
            window.shift(); // Remove oldest item
          }
          
          // Only emit when window is full
          if (window.length === windowSize) {
            observer.next([...window]); // Create shallow copy
          }
        },
        error(err) {
          observer.error(err);
        },
        complete() {
          // Emit remaining items if any
          if (window.length > 0) {
            observer.next([...window]);
          }
          observer.complete();
        }
      });
    });
  };
}

// ✅ Custom operator for efficient value comparison
function distinctUntilChangedDeep<T>(): MonoTypeOperatorFunction<T> {
  return distinctUntilChanged((prev, curr) => {
    return JSON.stringify(prev) === JSON.stringify(curr);
  });
}

// Usage examples
const optimizedStream$ = source$.pipe(
  debounceTimeWithLeading(300, true),
  slidingWindowEfficient(5),
  distinctUntilChangedDeep()
);
```

## Subscription Management {#subscription-management}

### Advanced Subscription Strategies

```typescript
@Injectable({
  providedIn: 'root'
})
export class OptimizedSubscriptionManager {
  private subscriptionPool = new Map<string, Subscription>();
  private subscriptionGroups = new Map<string, Subscription[]>();

  // ✅ Subscription pooling for reusable streams
  getOrCreateSubscription<T>(
    key: string,
    observableFactory: () => Observable<T>,
    callback: (value: T) => void
  ): Subscription {
    if (this.subscriptionPool.has(key)) {
      return this.subscriptionPool.get(key)!;
    }

    const subscription = observableFactory().subscribe(callback);
    this.subscriptionPool.set(key, subscription);
    return subscription;
  }

  // ✅ Grouped subscription management
  addToGroup(groupName: string, subscription: Subscription): void {
    if (!this.subscriptionGroups.has(groupName)) {
      this.subscriptionGroups.set(groupName, []);
    }
    this.subscriptionGroups.get(groupName)!.push(subscription);
  }

  unsubscribeGroup(groupName: string): void {
    const subscriptions = this.subscriptionGroups.get(groupName);
    if (subscriptions) {
      subscriptions.forEach(sub => sub.unsubscribe());
      this.subscriptionGroups.delete(groupName);
    }
  }

  // ✅ Automatic cleanup with component lifecycle
  createAutoCleanupSubscription<T>(
    source$: Observable<T>,
    component: any
  ): Observable<T> {
    const destroy$ = new Subject<void>();
    
    // Hook into component's ngOnDestroy
    const originalDestroy = component.ngOnDestroy?.bind(component);
    component.ngOnDestroy = () => {
      destroy$.next();
      destroy$.complete();
      if (originalDestroy) originalDestroy();
    };

    return source$.pipe(takeUntil(destroy$));
  }

  // ✅ Smart subscription sharing
  createSharedSubscription<T>(
    source$: Observable<T>,
    key: string,
    ttl: number = 300000 // 5 minutes
  ): Observable<T> {
    const now = Date.now();
    const cached = this.subscriptionPool.get(key);
    
    if (cached && (cached as any).timestamp && 
        (now - (cached as any).timestamp) < ttl) {
      return (cached as any).observable;
    }

    const shared$ = source$.pipe(
      shareReplay(1),
      finalize(() => {
        setTimeout(() => {
          this.subscriptionPool.delete(key);
        }, ttl);
      })
    );

    (shared$ as any).timestamp = now;
    this.subscriptionPool.set(key, shared$ as any);
    
    return shared$;
  }

  // Cleanup all subscriptions
  cleanup(): void {
    this.subscriptionPool.forEach(sub => sub.unsubscribe());
    this.subscriptionPool.clear();
    
    this.subscriptionGroups.forEach(subs => 
      subs.forEach(sub => sub.unsubscribe())
    );
    this.subscriptionGroups.clear();
  }
}
```

### Optimized Component Subscription Patterns

```typescript
// ✅ Base class for optimized subscription management
export abstract class OptimizedComponent implements OnInit, OnDestroy {
  protected destroy$ = new Subject<void>();
  private subscriptionManager = inject(OptimizedSubscriptionManager);

  ngOnInit(): void {
    this.initializeSubscriptions();
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }

  protected abstract initializeSubscriptions(): void;

  // Helper method for automatic unsubscription
  protected subscribe<T>(
    source$: Observable<T>,
    callback: (value: T) => void,
    errorCallback?: (error: any) => void
  ): void {
    source$.pipe(
      takeUntil(this.destroy$)
    ).subscribe({
      next: callback,
      error: errorCallback || (error => console.error(error))
    });
  }

  // Helper for conditional subscriptions
  protected subscribeWhen<T>(
    condition$: Observable<boolean>,
    source$: Observable<T>,
    callback: (value: T) => void
  ): void {
    condition$.pipe(
      switchMap(shouldSubscribe => 
        shouldSubscribe ? source$ : EMPTY
      ),
      takeUntil(this.destroy$)
    ).subscribe(callback);
  }
}

// Usage example
@Component({
  selector: 'app-optimized-data',
  template: `
    <div *ngFor="let item of items; trackBy: trackByFn">
      {{ item.name }}
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class OptimizedDataComponent extends OptimizedComponent {
  items: Item[] = [];

  constructor(
    private dataService: DataService,
    private cdr: ChangeDetectorRef
  ) {
    super();
  }

  protected initializeSubscriptions(): void {
    // Optimized data loading with error handling
    this.subscribe(
      this.dataService.getItems().pipe(
        catchError(error => {
          console.error('Failed to load items:', error);
          return of([]); // Fallback to empty array
        })
      ),
      items => {
        this.items = items;
        this.cdr.markForCheck();
      }
    );

    // Conditional subscription based on user permissions
    this.subscribeWhen(
      this.dataService.hasAdminPermissions$,
      this.dataService.getAdminData(),
      adminData => {
        console.log('Admin data loaded:', adminData);
      }
    );
  }

  trackByFn(index: number, item: Item): any {
    return item.id;
  }
}
```

## Stream Composition Optimization {#stream-composition}

### Efficient Stream Merging Strategies

```typescript
class StreamCompositionOptimizer {
  // ✅ Optimized merge for multiple similar streams
  mergeHomogeneousStreams<T>(
    streams: Observable<T>[],
    options: {
      concurrent?: number;
      bufferSize?: number;
    } = {}
  ): Observable<T> {
    const { concurrent = 3, bufferSize = 100 } = options;
    
    if (streams.length === 0) return EMPTY;
    if (streams.length === 1) return streams[0];
    
    // Use mergeAll for better performance with many streams
    return from(streams).pipe(
      mergeAll(concurrent),
      // Buffer to reduce subscription overhead
      bufferCount(bufferSize),
      mergeMap(buffered => from(buffered))
    );
  }

  // ✅ Smart combineLatest with selective updates
  combineLatestSelective<T extends Record<string, any>>(
    sources: { [K in keyof T]: Observable<T[K]> },
    keys?: (keyof T)[]
  ): Observable<Partial<T>> {
    const sourceKeys = keys || Object.keys(sources) as (keyof T)[];
    const sourceArray = sourceKeys.map(key => 
      sources[key].pipe(map(value => ({ key, value })))
    );

    return merge(...sourceArray).pipe(
      scan((acc, { key, value }) => ({ ...acc, [key]: value }), {} as Partial<T>),
      // Only emit when all required keys have values
      filter(state => sourceKeys.every(key => key in state)),
      distinctUntilChanged((prev, curr) => {
        return sourceKeys.every(key => prev[key] === curr[key]);
      })
    );
  }

  // ✅ Efficient stream switching with cancellation
  createSmartSwitch<T>(
    trigger$: Observable<string>,
    streamMap: Map<string, Observable<T>>,
    defaultKey?: string
  ): Observable<T> {
    return trigger$.pipe(
      startWith(defaultKey || ''),
      distinctUntilChanged(),
      switchMap(key => {
        const stream = streamMap.get(key);
        if (!stream) {
          console.warn(`No stream found for key: ${key}`);
          return EMPTY;
        }
        return stream.pipe(
          catchError(error => {
            console.error(`Stream error for key ${key}:`, error);
            return EMPTY;
          })
        );
      })
    );
  }

  // ✅ Priority-based stream merging
  mergePriorityStreams<T>(
    streams: { stream: Observable<T>; priority: number }[]
  ): Observable<T> {
    // Sort by priority (higher number = higher priority)
    const sortedStreams = streams
      .sort((a, b) => b.priority - a.priority)
      .map(({ stream, priority }) => 
        stream.pipe(map(value => ({ value, priority })))
      );

    return merge(...sortedStreams).pipe(
      scan((acc, current) => {
        // Only emit if current priority is >= last emitted priority
        if (!acc || current.priority >= acc.priority) {
          return current;
        }
        return acc;
      }, null as { value: T; priority: number } | null),
      filter(Boolean),
      map(item => item.value),
      distinctUntilChanged()
    );
  }
}
```

### Advanced Flattening Optimizations

```typescript
class FlatteningOptimizer {
  // ✅ Memory-efficient concatMap with queue limit
  concatMapWithLimit<T, R>(
    project: (value: T, index: number) => Observable<R>,
    queueLimit: number = 100
  ): OperatorFunction<T, R> {
    return (source: Observable<T>) => {
      return new Observable<R>(observer => {
        const queue: { value: T; index: number }[] = [];
        let currentIndex = 0;
        let isProcessing = false;
        let completed = false;

        const processNext = () => {
          if (isProcessing || queue.length === 0) return;
          
          isProcessing = true;
          const { value, index } = queue.shift()!;
          
          project(value, index).subscribe({
            next: result => observer.next(result),
            error: err => observer.error(err),
            complete: () => {
              isProcessing = false;
              if (queue.length > 0) {
                processNext();
              } else if (completed) {
                observer.complete();
              }
            }
          });
        };

        const subscription = source.subscribe({
          next: value => {
            if (queue.length >= queueLimit) {
              // Drop oldest items when queue is full
              queue.shift();
            }
            queue.push({ value, index: currentIndex++ });
            processNext();
          },
          error: err => observer.error(err),
          complete: () => {
            completed = true;
            if (!isProcessing && queue.length === 0) {
              observer.complete();
            }
          }
        });

        return () => subscription.unsubscribe();
      });
    };
  }

  // ✅ Adaptive mergeMap based on system load
  adaptiveMergeMap<T, R>(
    project: (value: T, index: number) => Observable<R>,
    options: {
      minConcurrency?: number;
      maxConcurrency?: number;
      loadThreshold?: number;
    } = {}
  ): OperatorFunction<T, R> {
    const { 
      minConcurrency = 1, 
      maxConcurrency = 10, 
      loadThreshold = 0.8 
    } = options;

    return (source: Observable<T>) => {
      return new Observable<R>(observer => {
        let currentConcurrency = Math.floor((minConcurrency + maxConcurrency) / 2);
        let activeSubscriptions = 0;
        let loadMeasurements: number[] = [];

        const measureLoad = () => {
          const load = activeSubscriptions / currentConcurrency;
          loadMeasurements.push(load);
          
          // Keep only recent measurements
          if (loadMeasurements.length > 10) {
            loadMeasurements.shift();
          }

          // Adjust concurrency based on load
          const avgLoad = loadMeasurements.reduce((a, b) => a + b, 0) / loadMeasurements.length;
          
          if (avgLoad > loadThreshold && currentConcurrency > minConcurrency) {
            currentConcurrency = Math.max(minConcurrency, currentConcurrency - 1);
          } else if (avgLoad < 0.5 && currentConcurrency < maxConcurrency) {
            currentConcurrency = Math.min(maxConcurrency, currentConcurrency + 1);
          }
        };

        return source.pipe(
          mergeMap((value, index) => {
            activeSubscriptions++;
            return project(value, index).pipe(
              finalize(() => {
                activeSubscriptions--;
                measureLoad();
              })
            );
          }, currentConcurrency)
        ).subscribe(observer);
      });
    };
  }
}
```

## Angular-Specific Optimizations {#angular-optimizations}

### OnPush Strategy Optimization

```typescript
@Component({
  selector: 'app-optimized-list',
  template: `
    <div class="list-container">
      <div class="list-header">
        <input #searchInput 
               placeholder="Search..." 
               (input)="search$.next($event.target.value)">
        <span class="item-count">{{ (filteredItems$ | async)?.length }} items</span>
      </div>
      
      <cdk-virtual-scroll-viewport itemSize="50" class="list-viewport">
        <div *cdkVirtualFor="let item of filteredItems$ | async; 
                            trackBy: trackByFn" 
             class="list-item">
          <app-item-component [item]="item" 
                              (itemChange)="onItemChange($event)">
          </app-item-component>
        </div>
      </cdk-virtual-scroll-viewport>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class OptimizedListComponent extends OptimizedComponent {
  search$ = new Subject<string>();
  itemChange$ = new Subject<ItemChangeEvent>();
  
  items$: Observable<Item[]>;
  filteredItems$: Observable<Item[]>;

  constructor(
    private itemService: ItemService,
    private cdr: ChangeDetectorRef
  ) {
    super();
  }

  protected initializeSubscriptions(): void {
    // ✅ Optimized data loading with caching
    this.items$ = this.itemService.getItems().pipe(
      shareReplay(1),
      catchError(error => {
        console.error('Failed to load items:', error);
        return of([]);
      })
    );

    // ✅ Optimized search with debouncing
    this.filteredItems$ = combineLatest([
      this.items$,
      this.search$.pipe(
        debounceTime(300),
        distinctUntilChanged(),
        startWith('')
      )
    ]).pipe(
      map(([items, searchTerm]) => 
        this.filterItems(items, searchTerm)
      ),
      shareReplay(1)
    );

    // ✅ Handle item changes efficiently
    this.subscribe(
      this.itemChange$.pipe(
        debounceTime(100),
        switchMap(event => 
          this.itemService.updateItem(event.item).pipe(
            catchError(error => {
              console.error('Failed to update item:', error);
              return of(null);
            })
          )
        )
      ),
      updatedItem => {
        if (updatedItem) {
          // Trigger minimal change detection
          this.cdr.markForCheck();
        }
      }
    );
  }

  private filterItems(items: Item[], searchTerm: string): Item[] {
    if (!searchTerm.trim()) return items;
    
    const lowerSearchTerm = searchTerm.toLowerCase();
    return items.filter(item => 
      item.name.toLowerCase().includes(lowerSearchTerm) ||
      item.description.toLowerCase().includes(lowerSearchTerm)
    );
  }

  trackByFn(index: number, item: Item): any {
    return item.id;
  }

  onItemChange(event: ItemChangeEvent): void {
    this.itemChange$.next(event);
  }
}
```

### Async Pipe Optimization

```typescript
@Component({
  selector: 'app-async-optimized',
  template: `
    <!-- ✅ Single subscription with async pipe -->
    <ng-container *ngIf="viewModel$ | async as vm">
      <div class="header">
        <h1>{{ vm.title }}</h1>
        <span class="status" [class]="vm.status">{{ vm.status }}</span>
      </div>
      
      <div class="content">
        <div class="loading" *ngIf="vm.isLoading">Loading...</div>
        <div class="error" *ngIf="vm.error">{{ vm.error }}</div>
        
        <div class="data" *ngIf="vm.data && !vm.isLoading">
          <div *ngFor="let item of vm.data; trackBy: trackByFn">
            {{ item.name }}
          </div>
        </div>
      </div>
    </ng-container>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class AsyncOptimizedComponent extends OptimizedComponent {
  // ✅ Single view model stream instead of multiple async pipes
  viewModel$: Observable<ViewModel>;

  constructor(private dataService: DataService) {
    super();
  }

  protected initializeSubscriptions(): void {
    const data$ = this.dataService.getData().pipe(
      startWith(null),
      catchError(error => of({ error: error.message }))
    );

    const loading$ = merge(
      of(true),
      data$.pipe(map(() => false))
    );

    this.viewModel$ = combineLatest([
      data$,
      loading$,
      of('Data Dashboard'), // title
      of('active') // status
    ]).pipe(
      map(([data, isLoading, title, status]) => ({
        title,
        status,
        isLoading,
        data: Array.isArray(data) ? data : null,
        error: (data as any)?.error || null
      })),
      shareReplay(1)
    );
  }

  trackByFn(index: number, item: any): any {
    return item.id || index;
  }
}

interface ViewModel {
  title: string;
  status: string;
  isLoading: boolean;
  data: any[] | null;
  error: string | null;
}
```

## Performance Monitoring & Measurement {#monitoring}

### Performance Metrics Collection

```typescript
@Injectable({
  providedIn: 'root'
})
export class RxJSPerformanceMonitor {
  private metrics = new Map<string, PerformanceMetric[]>();
  private observers = new Map<string, PerformanceObserver>();

  // ✅ Monitor observable performance
  monitorObservable<T>(
    source$: Observable<T>,
    label: string
  ): Observable<T> {
    const startTime = performance.now();
    let emissionCount = 0;
    let errorCount = 0;

    return new Observable<T>(observer => {
      const subscription = source$.subscribe({
        next: (value) => {
          emissionCount++;
          this.recordMetric(label, {
            type: 'emission',
            timestamp: performance.now(),
            value: performance.now() - startTime,
            count: emissionCount
          });
          observer.next(value);
        },
        error: (error) => {
          errorCount++;
          this.recordMetric(label, {
            type: 'error',
            timestamp: performance.now(),
            value: performance.now() - startTime,
            count: errorCount,
            error: error.message
          });
          observer.error(error);
        },
        complete: () => {
          const totalTime = performance.now() - startTime;
          this.recordMetric(label, {
            type: 'complete',
            timestamp: performance.now(),
            value: totalTime,
            count: emissionCount,
            metadata: {
              averageEmissionTime: emissionCount > 0 ? totalTime / emissionCount : 0,
              errorRate: errorCount / Math.max(emissionCount, 1)
            }
          });
          observer.complete();
        }
      });

      return () => {
        subscription.unsubscribe();
        this.recordMetric(label, {
          type: 'unsubscribe',
          timestamp: performance.now(),
          value: performance.now() - startTime
        });
      };
    });
  }

  private recordMetric(label: string, metric: PerformanceMetric): void {
    if (!this.metrics.has(label)) {
      this.metrics.set(label, []);
    }
    
    const metrics = this.metrics.get(label)!;
    metrics.push(metric);
    
    // Keep only recent metrics to prevent memory leaks
    if (metrics.length > 1000) {
      metrics.splice(0, 500); // Remove oldest half
    }
  }

  // ✅ Get performance report
  getPerformanceReport(label?: string): PerformanceReport {
    if (label) {
      return this.generateReport(label, this.metrics.get(label) || []);
    }

    const reports: { [key: string]: PerformanceReport } = {};
    for (const [key, metrics] of this.metrics.entries()) {
      reports[key] = this.generateReport(key, metrics);
    }

    return reports as any;
  }

  private generateReport(label: string, metrics: PerformanceMetric[]): PerformanceReport {
    const emissions = metrics.filter(m => m.type === 'emission');
    const errors = metrics.filter(m => m.type === 'error');
    const completions = metrics.filter(m => m.type === 'complete');

    return {
      label,
      totalEmissions: emissions.length,
      totalErrors: errors.length,
      totalCompletions: completions.length,
      averageEmissionTime: emissions.length > 0 
        ? emissions.reduce((acc, m) => acc + m.value, 0) / emissions.length 
        : 0,
      errorRate: emissions.length > 0 ? errors.length / emissions.length : 0,
      memoryUsage: this.getCurrentMemoryUsage(),
      recommendations: this.generateRecommendations(metrics)
    };
  }

  private getCurrentMemoryUsage(): number {
    const memory = (performance as any).memory;
    return memory ? memory.usedJSHeapSize / 1024 / 1024 : 0; // MB
  }

  private generateRecommendations(metrics: PerformanceMetric[]): string[] {
    const recommendations: string[] = [];
    
    const emissions = metrics.filter(m => m.type === 'emission');
    const avgEmissionTime = emissions.length > 0 
      ? emissions.reduce((acc, m) => acc + m.value, 0) / emissions.length 
      : 0;

    if (avgEmissionTime > 100) {
      recommendations.push('Consider debouncing or throttling high-frequency emissions');
    }

    if (metrics.filter(m => m.type === 'error').length > 0) {
      recommendations.push('Implement error handling and retry logic');
    }

    if (emissions.length > 1000) {
      recommendations.push('Consider using pagination or virtual scrolling for large datasets');
    }

    return recommendations;
  }

  // Cleanup
  clearMetrics(label?: string): void {
    if (label) {
      this.metrics.delete(label);
    } else {
      this.metrics.clear();
    }
  }
}

interface PerformanceMetric {
  type: 'emission' | 'error' | 'complete' | 'unsubscribe';
  timestamp: number;
  value: number;
  count?: number;
  error?: string;
  metadata?: any;
}

interface PerformanceReport {
  label: string;
  totalEmissions: number;
  totalErrors: number;
  totalCompletions: number;
  averageEmissionTime: number;
  errorRate: number;
  memoryUsage: number;
  recommendations: string[];
}
```

## Real-World Optimization Examples {#examples}

### E-commerce Product Search Optimization

```typescript
@Component({
  selector: 'app-product-search',
  template: `
    <div class="search-container">
      <input #searchInput 
             placeholder="Search products..."
             (input)="searchTerms$.next($event.target.value)">
      
      <div class="filters">
        <select (change)="categoryFilter$.next($event.target.value)">
          <option value="">All Categories</option>
          <option *ngFor="let cat of categories" [value]="cat">{{ cat }}</option>
        </select>
        
        <input type="range" 
               min="0" max="1000" 
               (input)="priceFilter$.next(+$event.target.value)">
      </div>

      <div class="results" *ngIf="searchResults$ | async as results">
        <div class="results-info">
          {{ results.total }} products found in {{ results.searchTime }}ms
        </div>
        
        <cdk-virtual-scroll-viewport itemSize="120" class="products-list">
          <div *cdkVirtualFor="let product of results.products; trackBy: trackByProductId"
               class="product-card">
            <img [src]="product.imageUrl" [alt]="product.name" loading="lazy">
            <h3>{{ product.name }}</h3>
            <p class="price">{{ product.price | currency }}</p>
          </div>
        </cdk-virtual-scroll-viewport>
      </div>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ProductSearchComponent extends OptimizedComponent {
  searchTerms$ = new BehaviorSubject<string>('');
  categoryFilter$ = new BehaviorSubject<string>('');
  priceFilter$ = new BehaviorSubject<number>(1000);

  categories = ['Electronics', 'Clothing', 'Books', 'Home'];
  searchResults$: Observable<SearchResults>;

  constructor(
    private productService: ProductService,
    private performanceMonitor: RxJSPerformanceMonitor
  ) {
    super();
  }

  protected initializeSubscriptions(): void {
    // ✅ Optimized search with multiple filters
    const searchFilters$ = combineLatest([
      this.searchTerms$.pipe(
        debounceTime(300),
        distinctUntilChanged(),
        map(term => term.trim())
      ),
      this.categoryFilter$.pipe(distinctUntilChanged()),
      this.priceFilter$.pipe(
        debounceTime(100),
        distinctUntilChanged()
      )
    ]).pipe(
      // Prevent unnecessary API calls when search term is too short
      filter(([searchTerm]) => searchTerm.length === 0 || searchTerm.length >= 2),
      shareReplay(1)
    );

    this.searchResults$ = searchFilters$.pipe(
      tap(() => console.time('product-search')),
      switchMap(([searchTerm, category, maxPrice]) =>
        this.performanceMonitor.monitorObservable(
          this.productService.searchProducts({
            query: searchTerm,
            category: category || undefined,
            maxPrice
          }).pipe(
            map(products => ({
              products,
              total: products.length,
              searchTime: 0, // Will be updated below
              filters: { searchTerm, category, maxPrice }
            })),
            catchError(error => {
              console.error('Search failed:', error);
              return of({
                products: [],
                total: 0,
                searchTime: 0,
                error: error.message
              });
            })
          ),
          'product-search'
        )
      ),
      tap(results => {
        console.timeEnd('product-search');
        // Update search time in results
        const performanceEntry = performance.getEntriesByName('product-search').pop();
        if (performanceEntry) {
          (results as any).searchTime = Math.round(performanceEntry.duration);
        }
      }),
      shareReplay(1)
    );
  }

  trackByProductId(index: number, product: Product): string {
    return product.id;
  }
}

interface SearchResults {
  products: Product[];
  total: number;
  searchTime: number;
  error?: string;
}
```

### Real-time Dashboard Optimization

```typescript
@Component({
  selector: 'app-realtime-dashboard',
  template: `
    <div class="dashboard">
      <div class="metrics-grid">
        <div *ngFor="let metric of metrics$ | async; trackBy: trackByMetricId"
             class="metric-card"
             [class.critical]="metric.value > metric.threshold">
          <h3>{{ metric.name }}</h3>
          <div class="metric-value">{{ metric.value | number:'1.2-2' }}</div>
          <div class="metric-trend" [class]="metric.trend">
            {{ metric.trend === 'up' ? '↗' : metric.trend === 'down' ? '↘' : '→' }}
          </div>
        </div>
      </div>

      <div class="charts-section">
        <app-time-series-chart [data]="chartData$ | async"></app-time-series-chart>
      </div>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class RealtimeDashboardComponent extends OptimizedComponent {
  metrics$: Observable<DashboardMetric[]>;
  chartData$: Observable<ChartDataPoint[]>;

  private metricsBuffer: DashboardMetric[] = [];
  private chartBuffer: ChartDataPoint[] = [];

  constructor(
    private metricsService: MetricsService,
    private performanceMonitor: RxJSPerformanceMonitor,
    private cdr: ChangeDetectorRef
  ) {
    super();
  }

  protected initializeSubscriptions(): void {
    // ✅ Optimized real-time metrics with buffering
    const rawMetrics$ = this.metricsService.getRealtimeMetrics().pipe(
      // Sample to prevent overwhelming the UI
      sampleTime(1000),
      catchError(error => {
        console.error('Metrics stream error:', error);
        return of([]);
      })
    );

    this.metrics$ = this.performanceMonitor.monitorObservable(
      rawMetrics$,
      'dashboard-metrics'
    ).pipe(
      scan((acc, newMetrics) => {
        // Update existing metrics or add new ones
        const updated = [...acc];
        
        newMetrics.forEach(newMetric => {
          const existingIndex = updated.findIndex(m => m.id === newMetric.id);
          if (existingIndex >= 0) {
            // Calculate trend
            const oldValue = updated[existingIndex].value;
            const trend = newMetric.value > oldValue ? 'up' : 
                         newMetric.value < oldValue ? 'down' : 'stable';
            
            updated[existingIndex] = { ...newMetric, trend };
          } else {
            updated.push({ ...newMetric, trend: 'stable' });
          }
        });

        return updated;
      }, [] as DashboardMetric[]),
      shareReplay(1)
    );

    // ✅ Optimized chart data with sliding window
    this.chartData$ = rawMetrics$.pipe(
      map(metrics => metrics.map(m => ({
        timestamp: Date.now(),
        value: m.value,
        metricId: m.id
      }))),
      scan((acc, newPoints) => {
        const updated = [...acc, ...newPoints];
        
        // Keep only last 100 points to prevent memory growth
        if (updated.length > 100) {
          return updated.slice(-100);
        }
        
        return updated;
      }, [] as ChartDataPoint[]),
      // Throttle chart updates to improve performance
      throttleTime(500),
      shareReplay(1)
    );

    // ✅ Performance monitoring and alerts
    this.subscribe(
      interval(30000).pipe( // Check every 30 seconds
        switchMap(() => 
          of(this.performanceMonitor.getPerformanceReport('dashboard-metrics'))
        )
      ),
      report => {
        if (report.averageEmissionTime > 100) {
          console.warn('Dashboard performance degraded:', report);
        }
      }
    );
  }

  trackByMetricId(index: number, metric: DashboardMetric): string {
    return metric.id;
  }
}

interface DashboardMetric {
  id: string;
  name: string;
  value: number;
  threshold: number;
  trend: 'up' | 'down' | 'stable';
}

interface ChartDataPoint {
  timestamp: number;
  value: number;
  metricId: string;
}
```

## Best Practices {#best-practices}

### Performance Optimization Checklist

```typescript
const RXJS_PERFORMANCE_BEST_PRACTICES = {
  observable_creation: [
    '✅ Use defer() for expensive observable creation',
    '✅ Implement proper caching with shareReplay()',
    '✅ Use factory functions for reusable observables',
    '✅ Avoid creating observables in templates'
  ],
  
  operator_optimization: [
    '✅ Combine multiple operators into single map operations',
    '✅ Use distinctUntilChanged() to prevent duplicate emissions',
    '✅ Implement debouncing for user input streams',
    '✅ Use takeUntil() for proper subscription cleanup'
  ],
  
  subscription_management: [
    '✅ Use async pipe instead of manual subscriptions',
    '✅ Implement subscription pooling for similar streams',
    '✅ Use OnPush change detection with observables',
    '✅ Create base classes for subscription management'
  ],
  
  memory_optimization: [
    '✅ Implement proper unsubscription strategies',
    '✅ Use virtual scrolling for large lists',
    '✅ Limit buffer sizes in operators',
    '✅ Monitor memory usage with browser dev tools'
  ],
  
  angular_integration: [
    '✅ Use trackBy functions in *ngFor',
    '✅ Implement OnPush change detection strategy',
    '✅ Create view model streams for complex components',
    '✅ Use CDK Virtual Scrolling for performance'
  ]
};
```

### Performance Budget Guidelines

```typescript
interface RxJSPerformanceBudget {
  maxSubscriptions: number;
  maxOperatorChainLength: number;
  maxEmissionFrequency: number; // per second
  maxMemoryGrowth: number; // MB per hour
  maxProcessingTime: number; // ms per emission
}

const PERFORMANCE_BUDGETS: Record<string, RxJSPerformanceBudget> = {
  mobile: {
    maxSubscriptions: 50,
    maxOperatorChainLength: 10,
    maxEmissionFrequency: 30,
    maxMemoryGrowth: 10,
    maxProcessingTime: 16 // 60fps
  },
  
  desktop: {
    maxSubscriptions: 200,
    maxOperatorChainLength: 15,
    maxEmissionFrequency: 100,
    maxMemoryGrowth: 50,
    maxProcessingTime: 8
  },
  
  server: {
    maxSubscriptions: 1000,
    maxOperatorChainLength: 20,
    maxEmissionFrequency: 1000,
    maxMemoryGrowth: 200,
    maxProcessingTime: 1
  }
};
```

## Exercises {#exercises}

### Exercise 1: Performance Audit Tool

Create a comprehensive performance audit tool that:
- Analyzes existing RxJS code for performance issues
- Identifies memory leaks and subscription problems
- Provides actionable optimization recommendations
- Generates performance reports with metrics

### Exercise 2: Optimized Data Table

Build a high-performance data table component with:
- Virtual scrolling for large datasets
- Optimized filtering and sorting
- Real-time updates with minimal re-renders
- Memory-efficient pagination

### Exercise 3: Real-time Performance Monitor

Implement a real-time performance monitoring system that:
- Tracks RxJS stream performance in production
- Provides alerting for performance degradation
- Visualizes performance metrics over time
- Suggests automatic optimizations

### Exercise 4: Bundle Size Optimization

Create a bundle analysis tool that:
- Identifies unused RxJS operators
- Suggests tree-shaking optimizations
- Provides alternative lightweight operators
- Measures bundle size impact of changes

---

**Next Steps:**
- Explore common RxJS pitfalls and solutions
- Learn about memory leak prevention strategies
- Master bundle size optimization techniques
- Implement browser compatibility solutions
