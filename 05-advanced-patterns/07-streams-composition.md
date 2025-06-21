# Complex Stream Composition Techniques

## Learning Objectives
By the end of this lesson, you will be able to:
- Master advanced stream composition patterns and techniques
- Build complex data pipelines using multiple observable streams
- Implement sophisticated stream orchestration and coordination
- Apply stream composition for real-world data processing scenarios
- Design maintainable and testable complex reactive systems

## Table of Contents
1. [Stream Composition Fundamentals](#stream-composition-fundamentals)
2. [Multi-Source Stream Coordination](#multi-source-stream-coordination)
3. [Dynamic Stream Creation](#dynamic-stream-creation)
4. [Stream Branching and Merging](#stream-branching-and-merging)
5. [Conditional Stream Processing](#conditional-stream-processing)
6. [Stream State Management](#stream-state-management)
7. [Complex Data Transformation Pipelines](#complex-data-transformation-pipelines)
8. [Stream Synchronization Patterns](#stream-synchronization-patterns)
9. [Angular Examples](#angular-examples)
10. [Performance Optimization](#performance-optimization)
11. [Testing Complex Compositions](#testing-complex-compositions)
12. [Exercises](#exercises)

## Stream Composition Fundamentals

### Advanced Composition Patterns

```typescript
import { 
  Observable, Subject, BehaviorSubject, merge, combineLatest, 
  of, from, interval, timer, EMPTY, NEVER, throwError 
} from 'rxjs';
import { 
  map, filter, switchMap, mergeMap, concatMap, exhaustMap,
  scan, reduce, tap, catchError, retry, retryWhen, delay,
  take, takeUntil, takeWhile, skip, startWith, share, shareReplay,
  distinctUntilChanged, debounceTime, throttleTime, buffer,
  windowTime, groupBy, partition, pluck, mapTo
} from 'rxjs/operators';

// Stream composition builder for complex pipelines
class StreamComposer<T> {
  constructor(private source$: Observable<T>) {}

  // Add conditional branching
  branch<U, V>(
    condition: (value: T) => boolean,
    trueStream: (value: T) => Observable<U>,
    falseStream: (value: T) => Observable<V>
  ): StreamComposer<U | V> {
    const composed$ = this.source$.pipe(
      switchMap(value => 
        condition(value) 
          ? trueStream(value) 
          : falseStream(value)
      )
    );
    return new StreamComposer(composed$);
  }

  // Add stream with fallback
  withFallback<U>(
    primaryStream: (value: T) => Observable<U>,
    fallbackStream: (value: T) => Observable<U>,
    shouldFallback: (error: any) => boolean = () => true
  ): StreamComposer<U> {
    const composed$ = this.source$.pipe(
      switchMap(value => 
        primaryStream(value).pipe(
          catchError(error => 
            shouldFallback(error) 
              ? fallbackStream(value)
              : throwError(error)
          )
        )
      )
    );
    return new StreamComposer(composed$);
  }

  // Add parallel processing
  parallel<U>(
    streamFactories: Array<(value: T) => Observable<U>>,
    combiner: (results: U[]) => U = results => results[0]
  ): StreamComposer<U> {
    const composed$ = this.source$.pipe(
      switchMap(value => {
        const parallelStreams = streamFactories.map(factory => factory(value));
        return combineLatest(parallelStreams).pipe(
          map(combiner)
        );
      })
    );
    return new StreamComposer(composed$);
  }

  // Add sequential processing
  sequential<U>(
    streamFactories: Array<(value: any) => Observable<U>>
  ): StreamComposer<U> {
    const composed$ = this.source$.pipe(
      switchMap(initialValue => 
        streamFactories.reduce(
          (acc$, factory) => acc$.pipe(switchMap(factory)),
          of(initialValue)
        )
      )
    );
    return new StreamComposer(composed$);
  }

  // Add caching layer
  withCache<K>(
    keyExtractor: (value: T) => K,
    ttl: number = 60000 // 1 minute default
  ): StreamComposer<T> {
    const cache = new Map<K, { value: T; timestamp: number }>();
    
    const composed$ = this.source$.pipe(
      map(value => {
        const key = keyExtractor(value);
        const cached = cache.get(key);
        const now = Date.now();
        
        if (cached && (now - cached.timestamp) < ttl) {
          return cached.value;
        }
        
        cache.set(key, { value, timestamp: now });
        return value;
      })
    );
    return new StreamComposer(composed$);
  }

  // Build final observable
  build(): Observable<T> {
    return this.source$;
  }
}

// Factory function for stream composition
function composeStream<T>(source$: Observable<T>): StreamComposer<T> {
  return new StreamComposer(source$);
}
```

### Marble Diagram: Stream Composition
```
Source:      --a----b----c----d------>
Branch True: ----A--------C---------->  (when value > threshold)
Branch False:------B----D----------->  (when value <= threshold)
Combined:    ----A-B----C-D---------->
```

## Multi-Source Stream Coordination

### Advanced Coordination Patterns

```typescript
// Stream coordinator for complex multi-source scenarios
class StreamCoordinator {
  private sources = new Map<string, Observable<any>>();
  private dependencies = new Map<string, string[]>();
  private results = new Map<string, any>();

  // Register stream source
  addSource<T>(name: string, source$: Observable<T>, dependencies: string[] = []): this {
    this.sources.set(name, source$);
    this.dependencies.set(name, dependencies);
    return this;
  }

  // Execute streams with dependency resolution
  execute(): Observable<Map<string, any>> {
    const executionOrder = this.resolveExecutionOrder();
    const resultSubject = new BehaviorSubject(new Map<string, any>());

    this.executeInOrder(executionOrder, resultSubject);
    
    return resultSubject.asObservable();
  }

  // Execute with timeout and error handling
  executeWithTimeout(timeout: number): Observable<Map<string, any>> {
    return this.execute().pipe(
      take(1),
      timeout(timeout),
      catchError(error => {
        console.error('Stream coordination failed:', error);
        return of(new Map());
      })
    );
  }

  private resolveExecutionOrder(): string[] {
    const visited = new Set<string>();
    const visiting = new Set<string>();
    const order: string[] = [];

    const visit = (name: string) => {
      if (visiting.has(name)) {
        throw new Error(`Circular dependency detected: ${name}`);
      }
      if (visited.has(name)) return;

      visiting.add(name);
      const deps = this.dependencies.get(name) || [];
      deps.forEach(visit);
      visiting.delete(name);
      visited.add(name);
      order.push(name);
    };

    Array.from(this.sources.keys()).forEach(visit);
    return order;
  }

  private executeInOrder(order: string[], resultSubject: BehaviorSubject<Map<string, any>>): void {
    order.reduce((acc$, streamName) => {
      return acc$.pipe(
        switchMap(() => {
          const source$ = this.sources.get(streamName)!;
          const deps = this.dependencies.get(streamName) || [];
          const depValues = deps.map(dep => this.results.get(dep));

          return this.executeStream(source$, depValues).pipe(
            tap(result => {
              this.results.set(streamName, result);
              resultSubject.next(new Map(this.results));
            })
          );
        })
      );
    }, of(null)).subscribe();
  }

  private executeStream(source$: Observable<any>, dependencies: any[]): Observable<any> {
    if (dependencies.length === 0) {
      return source$.pipe(take(1));
    }

    // Inject dependencies into stream
    return source$.pipe(
      map(value => ({ value, dependencies })),
      take(1)
    );
  }
}

// Example: Complex data loading coordination
class DataLoadingCoordinator {
  private coordinator = new StreamCoordinator();

  constructor(
    private userService: UserService,
    private permissionsService: PermissionsService,
    private preferencesService: PreferencesService,
    private contentService: ContentService
  ) {
    this.setupStreams();
  }

  private setupStreams(): void {
    // User data (no dependencies)
    this.coordinator.addSource(
      'user',
      this.userService.getCurrentUser()
    );

    // Permissions (depends on user)
    this.coordinator.addSource(
      'permissions',
      this.permissionsService.getUserPermissions(),
      ['user']
    );

    // Preferences (depends on user)
    this.coordinator.addSource(
      'preferences',
      this.preferencesService.getUserPreferences(),
      ['user']
    );

    // Content (depends on user and permissions)
    this.coordinator.addSource(
      'content',
      this.contentService.getContent(),
      ['user', 'permissions']
    );
  }

  loadApplicationData(): Observable<ApplicationData> {
    return this.coordinator.executeWithTimeout(10000).pipe(
      map(results => ({
        user: results.get('user'),
        permissions: results.get('permissions'),
        preferences: results.get('preferences'),
        content: results.get('content')
      }))
    );
  }
}
```

## Dynamic Stream Creation

### Runtime Stream Generation

```typescript
// Dynamic stream factory
class DynamicStreamFactory {
  private streamCache = new Map<string, Observable<any>>();

  // Create stream from configuration
  createFromConfig<T>(config: StreamConfig): Observable<T> {
    const cacheKey = this.generateCacheKey(config);
    
    if (this.streamCache.has(cacheKey)) {
      return this.streamCache.get(cacheKey)!;
    }

    const stream$ = this.buildStream<T>(config);
    this.streamCache.set(cacheKey, stream$);
    
    return stream$;
  }

  // Create conditional stream
  createConditional<T>(
    condition: () => boolean,
    trueStreamFactory: () => Observable<T>,
    falseStreamFactory: () => Observable<T>
  ): Observable<T> {
    return new Observable<T>(observer => {
      const stream$ = condition() ? trueStreamFactory() : falseStreamFactory();
      return stream$.subscribe(observer);
    });
  }

  // Create stream with dynamic operators
  createWithDynamicOperators<T>(
    source$: Observable<T>,
    operatorConfigs: OperatorConfig[]
  ): Observable<any> {
    return operatorConfigs.reduce((stream$, config) => {
      return this.applyOperator(stream$, config);
    }, source$);
  }

  private buildStream<T>(config: StreamConfig): Observable<T> {
    let stream$: Observable<any>;

    switch (config.type) {
      case 'interval':
        stream$ = interval(config.interval || 1000);
        break;
      case 'timer':
        stream$ = timer(config.delay || 0, config.interval);
        break;
      case 'http':
        stream$ = this.createHttpStream(config.url!, config.method);
        break;
      case 'websocket':
        stream$ = this.createWebSocketStream(config.url!);
        break;
      default:
        stream$ = of(config.staticValue);
    }

    // Apply transformations
    if (config.transformations) {
      stream$ = config.transformations.reduce(
        (acc$, transformation) => this.applyTransformation(acc$, transformation),
        stream$
      );
    }

    return stream$;
  }

  private applyOperator(stream$: Observable<any>, config: OperatorConfig): Observable<any> {
    switch (config.type) {
      case 'map':
        return stream$.pipe(map(config.mapFunction!));
      case 'filter':
        return stream$.pipe(filter(config.filterFunction!));
      case 'debounce':
        return stream$.pipe(debounceTime(config.time!));
      case 'throttle':
        return stream$.pipe(throttleTime(config.time!));
      case 'take':
        return stream$.pipe(take(config.count!));
      case 'skip':
        return stream$.pipe(skip(config.count!));
      default:
        return stream$;
    }
  }

  private applyTransformation(stream$: Observable<any>, transformation: any): Observable<any> {
    // Apply custom transformation logic
    return stream$.pipe(map(transformation));
  }

  private createHttpStream(url: string, method: string = 'GET'): Observable<any> {
    // Implementation depends on HTTP client
    return new Observable(observer => {
      fetch(url, { method })
        .then(response => response.json())
        .then(data => {
          observer.next(data);
          observer.complete();
        })
        .catch(error => observer.error(error));
    });
  }

  private createWebSocketStream(url: string): Observable<any> {
    return new Observable(observer => {
      const ws = new WebSocket(url);
      
      ws.onmessage = event => observer.next(JSON.parse(event.data));
      ws.onerror = error => observer.error(error);
      ws.onclose = () => observer.complete();
      
      return () => ws.close();
    });
  }

  private generateCacheKey(config: StreamConfig): string {
    return JSON.stringify(config);
  }
}

interface StreamConfig {
  type: 'interval' | 'timer' | 'http' | 'websocket' | 'static';
  interval?: number;
  delay?: number;
  url?: string;
  method?: string;
  staticValue?: any;
  transformations?: any[];
}

interface OperatorConfig {
  type: string;
  mapFunction?: (value: any) => any;
  filterFunction?: (value: any) => boolean;
  time?: number;
  count?: number;
}
```

## Stream Branching and Merging

### Advanced Branching Patterns

```typescript
// Stream branching utility
class StreamBrancher<T> {
  constructor(private source$: Observable<T>) {}

  // Partition stream based on predicate
  partition(predicate: (value: T) => boolean): [Observable<T>, Observable<T>] {
    const [truthyStream$, falsyStream$] = partition(this.source$, predicate);
    return [truthyStream$, falsyStream$];
  }

  // Multiple branching with routing
  route<K extends string | number>(
    router: (value: T) => K
  ): Map<K, Observable<T>> {
    const branches = new Map<K, Subject<T>>();
    const result = new Map<K, Observable<T>>();

    this.source$.subscribe(value => {
      const route = router(value);
      
      if (!branches.has(route)) {
        branches.set(route, new Subject<T>());
        result.set(route, branches.get(route)!.asObservable());
      }
      
      branches.get(route)!.next(value);
    });

    return result;
  }

  // Conditional multicast
  multicast<U>(
    transformers: Array<{
      condition: (value: T) => boolean;
      transform: (value: T) => U;
    }>
  ): Observable<U>[] {
    return transformers.map(({ condition, transform }) =>
      this.source$.pipe(
        filter(condition),
        map(transform)
      )
    );
  }

  // Dynamic branching based on stream content
  dynamicBranch<U>(
    branchFactory: (value: T) => Observable<U>
  ): Observable<U> {
    return this.source$.pipe(
      switchMap(branchFactory)
    );
  }
}

// Stream merger utility
class StreamMerger {
  // Merge with priority
  static mergeWithPriority<T>(
    streams: Array<{ stream: Observable<T>; priority: number }>
  ): Observable<T> {
    const sortedStreams = streams
      .sort((a, b) => b.priority - a.priority)
      .map(({ stream }) => stream);

    return merge(...sortedStreams);
  }

  // Merge with backpressure handling
  static mergeWithBackpressure<T>(
    streams: Observable<T>[],
    bufferSize: number = 100
  ): Observable<T> {
    return merge(...streams).pipe(
      buffer(timer(0, 100)), // Buffer for 100ms
      filter(buffer => buffer.length > 0),
      switchMap(buffer => 
        buffer.length > bufferSize 
          ? from(buffer.slice(0, bufferSize))
          : from(buffer)
      )
    );
  }

  // Intelligent merging with conflict resolution
  static mergeWithConflictResolution<T>(
    streams: Observable<T>[],
    resolver: (values: T[]) => T
  ): Observable<T> {
    return combineLatest(streams).pipe(
      debounceTime(50), // Wait for all simultaneous values
      map(resolver),
      distinctUntilChanged()
    );
  }
}

// Example: Complex branching scenario
class DataProcessingPipeline<T> {
  private brancher: StreamBrancher<T>;
  private processingResults: Map<string, Observable<any>> = new Map();

  constructor(source$: Observable<T>) {
    this.brancher = new StreamBrancher(source$);
    this.setupProcessingBranches();
  }

  private setupProcessingBranches(): void {
    // Branch by data type
    const branches = this.brancher.route(this.getDataType);

    // Process each branch differently
    branches.forEach((branch$, type) => {
      switch (type) {
        case 'user-data':
          this.processingResults.set(type, this.processUserData(branch$));
          break;
        case 'analytics-data':
          this.processingResults.set(type, this.processAnalyticsData(branch$));
          break;
        case 'system-data':
          this.processingResults.set(type, this.processSystemData(branch$));
          break;
      }
    });
  }

  private getDataType(data: T): string {
    // Implementation depends on data structure
    return (data as any).type || 'unknown';
  }

  private processUserData(stream$: Observable<T>): Observable<any> {
    return stream$.pipe(
      map(data => ({ ...data, processed: true, processedAt: Date.now() })),
      filter(data => this.validateUserData(data))
    );
  }

  private processAnalyticsData(stream$: Observable<T>): Observable<any> {
    return stream$.pipe(
      buffer(timer(0, 5000)), // Batch every 5 seconds
      filter(batch => batch.length > 0),
      map(batch => this.aggregateAnalytics(batch))
    );
  }

  private processSystemData(stream$: Observable<T>): Observable<any> {
    return stream$.pipe(
      throttleTime(1000), // Rate limit system data
      map(data => this.enrichSystemData(data))
    );
  }

  private validateUserData(data: any): boolean {
    return data && data.userId && data.action;
  }

  private aggregateAnalytics(batch: T[]): any {
    return {
      type: 'analytics-batch',
      count: batch.length,
      timestamp: Date.now(),
      data: batch
    };
  }

  private enrichSystemData(data: any): any {
    return {
      ...data,
      enriched: true,
      systemMetadata: {
        processedAt: Date.now(),
        version: '1.0'
      }
    };
  }

  getProcessedResults(): Observable<any> {
    const allResults = Array.from(this.processingResults.values());
    return merge(...allResults);
  }
}
```

### Marble Diagram: Complex Branching
```
Source:     --a--b--c--d--e--f--g-->
Type Check: --U--A--S--U--A--S--U-->  (U=User, A=Analytics, S=System)

User:       --a-----d--------g---->  (filter by type)
Analytics:  -----b-----e---------->  (batch every 5s)
System:     --------c-----f------->  (throttle 1s)

Merged:     --a--B--c--d--E--f--g-->  (B,E = batched analytics)
```

## Conditional Stream Processing

### Advanced Conditional Patterns

```typescript
// Conditional stream processor
class ConditionalStreamProcessor<T> {
  constructor(private source$: Observable<T>) {}

  // Process with multiple conditions
  processWithConditions<U>(
    processors: Array<{
      condition: (value: T) => boolean;
      processor: (value: T) => Observable<U>;
      priority?: number;
    }>
  ): Observable<U> {
    // Sort by priority (higher first)
    const sortedProcessors = processors.sort((a, b) => 
      (b.priority || 0) - (a.priority || 0)
    );

    return this.source$.pipe(
      switchMap(value => {
        // Find first matching processor
        const processor = sortedProcessors.find(p => p.condition(value));
        return processor 
          ? processor.processor(value)
          : EMPTY;
      })
    );
  }

  // Switch stream based on condition changes
  switchOnCondition<U, V>(
    condition: (value: T) => boolean,
    trueProcessor: (stream$: Observable<T>) => Observable<U>,
    falseProcessor: (stream$: Observable<T>) => Observable<V>
  ): Observable<U | V> {
    return this.source$.pipe(
      groupBy(condition),
      mergeMap(group$ => 
        group$.key 
          ? trueProcessor(group$)
          : falseProcessor(group$)
      )
    );
  }

  // Conditional buffering
  bufferWhen<U>(
    shouldBuffer: (value: T) => boolean,
    processor: (buffer: T[]) => Observable<U>
  ): Observable<U> {
    return this.source$.pipe(
      scan((acc, value) => {
        if (shouldBuffer(value)) {
          return { buffer: [...acc.buffer, value], shouldFlush: false };
        } else {
          return { buffer: [value], shouldFlush: true };
        }
      }, { buffer: [] as T[], shouldFlush: false }),
      filter(({ shouldFlush }) => shouldFlush),
      switchMap(({ buffer }) => processor(buffer))
    );
  }

  // Conditional retry with escalation
  retryWithEscalation<U>(
    processor: (value: T) => Observable<U>,
    retryStrategies: Array<{
      condition: (error: any, attempt: number) => boolean;
      delay: number;
      maxAttempts: number;
    }>
  ): Observable<U> {
    return this.source$.pipe(
      switchMap(value => 
        processor(value).pipe(
          retryWhen(errors$ => 
            errors$.pipe(
              scan((acc, error) => ({ ...acc, attempt: acc.attempt + 1, error }), 
                   { attempt: 0, error: null }),
              switchMap(({ attempt, error }) => {
                const strategy = retryStrategies.find(s => 
                  s.condition(error, attempt) && attempt < s.maxAttempts
                );
                
                return strategy 
                  ? timer(strategy.delay)
                  : throwError(error);
              })
            )
          )
        )
      )
    );
  }
}

// Example: E-commerce order processing
class OrderProcessingStream {
  private processor: ConditionalStreamProcessor<Order>;

  constructor(orders$: Observable<Order>) {
    this.processor = new ConditionalStreamProcessor(orders$);
  }

  processOrders(): Observable<ProcessedOrder> {
    return this.processor.processWithConditions([
      {
        condition: order => order.priority === 'urgent',
        processor: order => this.processUrgentOrder(order),
        priority: 100
      },
      {
        condition: order => order.amount > 1000,
        processor: order => this.processHighValueOrder(order),
        priority: 90
      },
      {
        condition: order => order.isInternational,
        processor: order => this.processInternationalOrder(order),
        priority: 80
      },
      {
        condition: () => true, // default case
        processor: order => this.processStandardOrder(order),
        priority: 0
      }
    ]);
  }

  private processUrgentOrder(order: Order): Observable<ProcessedOrder> {
    return this.validateOrder(order).pipe(
      switchMap(() => this.priorityQueue(order)),
      switchMap(() => this.expeditedShipping(order)),
      map(result => ({ ...order, processed: true, priority: 'urgent', result }))
    );
  }

  private processHighValueOrder(order: Order): Observable<ProcessedOrder> {
    return this.validateOrder(order).pipe(
      switchMap(() => this.fraudCheck(order)),
      switchMap(() => this.managerApproval(order)),
      switchMap(() => this.specialHandling(order)),
      map(result => ({ ...order, processed: true, highValue: true, result }))
    );
  }

  private processInternationalOrder(order: Order): Observable<ProcessedOrder> {
    return this.validateOrder(order).pipe(
      switchMap(() => this.customsDocumentation(order)),
      switchMap(() => this.currencyConversion(order)),
      switchMap(() => this.internationalShipping(order)),
      map(result => ({ ...order, processed: true, international: true, result }))
    );
  }

  private processStandardOrder(order: Order): Observable<ProcessedOrder> {
    return this.validateOrder(order).pipe(
      switchMap(() => this.standardProcessing(order)),
      map(result => ({ ...order, processed: true, standard: true, result }))
    );
  }

  // Helper methods
  private validateOrder(order: Order): Observable<boolean> {
    return of(true).pipe(delay(100)); // Simulate validation
  }

  private priorityQueue(order: Order): Observable<any> {
    return of('queued').pipe(delay(50));
  }

  private expeditedShipping(order: Order): Observable<any> {
    return of('expedited').pipe(delay(200));
  }

  private fraudCheck(order: Order): Observable<any> {
    return of('cleared').pipe(delay(500));
  }

  private managerApproval(order: Order): Observable<any> {
    return of('approved').pipe(delay(1000));
  }

  private specialHandling(order: Order): Observable<any> {
    return of('handled').pipe(delay(300));
  }

  private customsDocumentation(order: Order): Observable<any> {
    return of('documented').pipe(delay(800));
  }

  private currencyConversion(order: Order): Observable<any> {
    return of('converted').pipe(delay(200));
  }

  private internationalShipping(order: Order): Observable<any> {
    return of('shipped').pipe(delay(400));
  }

  private standardProcessing(order: Order): Observable<any> {
    return of('processed').pipe(delay(300));
  }
}

interface Order {
  id: string;
  amount: number;
  priority: 'normal' | 'urgent';
  isInternational: boolean;
  customerId: string;
}

interface ProcessedOrder extends Order {
  processed: boolean;
  result: any;
  priority?: string;
  highValue?: boolean;
  international?: boolean;
  standard?: boolean;
}
```

## Stream State Management

### Stateful Stream Composition

```typescript
// Stateful stream manager
class StatefulStreamManager<T, S> {
  private state$ = new BehaviorSubject<S>(this.initialState);
  private stateHistory: S[] = [];

  constructor(
    private source$: Observable<T>,
    private initialState: S,
    private reducer: (state: S, value: T) => S
  ) {
    this.setupStateManagement();
  }

  // Get current state
  getCurrentState(): S {
    return this.state$.value;
  }

  // Get state stream
  getStateStream(): Observable<S> {
    return this.state$.asObservable();
  }

  // Get state history
  getStateHistory(): S[] {
    return [...this.stateHistory];
  }

  // Reset state
  resetState(): void {
    this.state$.next(this.initialState);
    this.stateHistory = [this.initialState];
  }

  // Time travel (undo/redo)
  timeTravel(steps: number): void {
    const targetIndex = this.stateHistory.length - 1 + steps;
    if (targetIndex >= 0 && targetIndex < this.stateHistory.length) {
      this.state$.next(this.stateHistory[targetIndex]);
    }
  }

  // Create derived state stream
  derive<U>(selector: (state: S) => U): Observable<U> {
    return this.state$.pipe(
      map(selector),
      distinctUntilChanged()
    );
  }

  // Add state middleware
  addMiddleware(middleware: (state: S, value: T) => S): void {
    // Composition of middlewares can be added here
  }

  private setupStateManagement(): void {
    this.source$.pipe(
      scan((state, value) => {
        const newState = this.reducer(state, value);
        this.stateHistory.push(newState);
        
        // Limit history size to prevent memory leaks
        if (this.stateHistory.length > 100) {
          this.stateHistory = this.stateHistory.slice(-50);
        }
        
        return newState;
      }, this.initialState)
    ).subscribe(this.state$);
  }
}

// Complex state aggregation
class MultiStreamStateAggregator {
  private aggregatedState$ = new BehaviorSubject<AggregatedState>({});
  private sourceStates = new Map<string, any>();

  // Register state stream
  registerStateStream<T>(
    key: string, 
    stream$: Observable<T>, 
    transformer?: (value: T) => any
  ): void {
    stream$.pipe(
      map(value => transformer ? transformer(value) : value),
      tap(transformedValue => {
        this.sourceStates.set(key, transformedValue);
        this.updateAggregatedState();
      })
    ).subscribe();
  }

  // Get aggregated state
  getAggregatedState(): Observable<AggregatedState> {
    return this.aggregatedState$.asObservable();
  }

  // Get specific state slice
  getStateSlice<T>(key: string): Observable<T> {
    return this.aggregatedState$.pipe(
      map(state => state[key]),
      filter(value => value !== undefined),
      distinctUntilChanged()
    );
  }

  private updateAggregatedState(): void {
    const newState: AggregatedState = {};
    this.sourceStates.forEach((value, key) => {
      newState[key] = value;
    });
    this.aggregatedState$.next(newState);
  }
}

interface AggregatedState {
  [key: string]: any;
}

// Example: Real-time dashboard state management
class DashboardStateManager {
  private stateManager: StatefulStreamManager<DashboardEvent, DashboardState>;
  private aggregator: MultiStreamStateAggregator;

  constructor(
    events$: Observable<DashboardEvent>,
    userActivity$: Observable<UserActivity>,
    systemMetrics$: Observable<SystemMetrics>
  ) {
    // Initialize state manager
    this.stateManager = new StatefulStreamManager(
      events$,
      this.getInitialState(),
      this.reducer.bind(this)
    );

    // Setup aggregator
    this.aggregator = new MultiStreamStateAggregator();
    this.setupAggregation(userActivity$, systemMetrics$);
  }

  private setupAggregation(
    userActivity$: Observable<UserActivity>,
    systemMetrics$: Observable<SystemMetrics>
  ): void {
    // Register dashboard state
    this.aggregator.registerStateStream(
      'dashboard',
      this.stateManager.getStateStream()
    );

    // Register user activity
    this.aggregator.registerStateStream(
      'userActivity',
      userActivity$,
      activity => ({
        activeUsers: activity.activeUsers,
        lastActivity: activity.timestamp
      })
    );

    // Register system metrics
    this.aggregator.registerStateStream(
      'systemMetrics',
      systemMetrics$.pipe(
        scan((acc, metrics) => ({
          ...acc,
          cpu: metrics.cpu,
          memory: metrics.memory,
          requests: acc.requests + metrics.newRequests
        }), { cpu: 0, memory: 0, requests: 0 })
      )
    );
  }

  private getInitialState(): DashboardState {
    return {
      widgets: [],
      layout: 'grid',
      filters: {},
      selectedTimeRange: '1h'
    };
  }

  private reducer(state: DashboardState, event: DashboardEvent): DashboardState {
    switch (event.type) {
      case 'ADD_WIDGET':
        return {
          ...state,
          widgets: [...state.widgets, event.widget]
        };
      case 'REMOVE_WIDGET':
        return {
          ...state,
          widgets: state.widgets.filter(w => w.id !== event.widgetId)
        };
      case 'UPDATE_LAYOUT':
        return {
          ...state,
          layout: event.layout
        };
      case 'SET_FILTER':
        return {
          ...state,
          filters: { ...state.filters, [event.filterKey]: event.filterValue }
        };
      default:
        return state;
    }
  }

  // Public interface
  getDashboardState(): Observable<DashboardState> {
    return this.stateManager.getStateStream();
  }

  getFullApplicationState(): Observable<AggregatedState> {
    return this.aggregator.getAggregatedState();
  }

  getWidgets(): Observable<Widget[]> {
    return this.stateManager.derive(state => state.widgets);
  }

  addWidget(widget: Widget): void {
    // This would be connected to an input stream
  }

  resetDashboard(): void {
    this.stateManager.resetState();
  }
}

interface DashboardState {
  widgets: Widget[];
  layout: string;
  filters: { [key: string]: any };
  selectedTimeRange: string;
}

interface DashboardEvent {
  type: string;
  widget?: Widget;
  widgetId?: string;
  layout?: string;
  filterKey?: string;
  filterValue?: any;
}

interface Widget {
  id: string;
  type: string;
  config: any;
}

interface UserActivity {
  activeUsers: number;
  timestamp: Date;
}

interface SystemMetrics {
  cpu: number;
  memory: number;
  newRequests: number;
}
```

## Angular Examples

### Complex Angular Stream Composition

```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { FormBuilder, FormGroup } from '@angular/forms';
import { 
  Observable, Subject, BehaviorSubject, combineLatest, merge 
} from 'rxjs';
import { 
  map, switchMap, debounceTime, distinctUntilChanged, 
  startWith, takeUntil, shareReplay, catchError 
} from 'rxjs/operators';

@Component({
  selector: 'app-complex-search',
  template: `
    <div class="search-interface">
      <!-- Search Form -->
      <form [formGroup]="searchForm">
        <input formControlName="query" placeholder="Search...">
        <select formControlName="category">
          <option value="">All Categories</option>
          <option *ngFor="let cat of categories$ | async" [value]="cat.id">
            {{ cat.name }}
          </option>
        </select>
        <input type="range" formControlName="priceRange" min="0" max="1000">
      </form>

      <!-- Filters -->
      <div class="filters">
        <div class="filter-chip" 
             *ngFor="let filter of activeFilters$ | async"
             (click)="removeFilter(filter.key)">
          {{ filter.label }} ✕
        </div>
      </div>

      <!-- Results -->
      <div class="results" *ngIf="searchResults$ | async as results">
        <div class="result-stats">
          {{ results.total }} results ({{ results.duration }}ms)
        </div>
        
        <div class="result-item" *ngFor="let item of results.items">
          <h3>{{ item.title }}</h3>
          <p>{{ item.description }}</p>
          <span class="price">${{ item.price }}</span>
        </div>
      </div>

      <!-- Loading & Error States -->
      <div *ngIf="isLoading$ | async" class="loading">Searching...</div>
      <div *ngIf="error$ | async as error" class="error">{{ error }}</div>
    </div>
  `
})
export class ComplexSearchComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  
  // Form and input streams
  searchForm: FormGroup;
  private externalFilters$ = new Subject<ExternalFilter>();
  private userInteractions$ = new Subject<UserInteraction>();
  
  // State streams
  categories$: Observable<Category[]>;
  activeFilters$: Observable<ActiveFilter[]>;
  searchResults$: Observable<SearchResults>;
  isLoading$: Observable<boolean>;
  error$: Observable<string | null>;
  
  // Composed streams
  private searchParams$: Observable<SearchParams>;
  private debouncedSearch$: Observable<SearchParams>;
  
  constructor(
    private fb: FormBuilder,
    private searchService: SearchService,
    private categoryService: CategoryService,
    private analyticsService: AnalyticsService
  ) {
    this.initializeForm();
    this.setupStreams();
  }
  
  ngOnInit(): void {
    this.categories$ = this.categoryService.getCategories();
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
  
  private initializeForm(): void {
    this.searchForm = this.fb.group({
      query: [''],
      category: [''],
      priceRange: [500]
    });
  }
  
  private setupStreams(): void {
    // Form value changes
    const formChanges$ = this.searchForm.valueChanges.pipe(
      startWith(this.searchForm.value)
    );
    
    // Combine all search parameters
    this.searchParams$ = combineLatest([
      formChanges$,
      this.externalFilters$.pipe(startWith(null)),
      this.getLocationParams()
    ]).pipe(
      map(([formValue, externalFilter, locationParams]) => 
        this.combineSearchParams(formValue, externalFilter, locationParams)
      ),
      distinctUntilChanged((prev, curr) => 
        JSON.stringify(prev) === JSON.stringify(curr)
      ),
      shareReplay(1),
      takeUntil(this.destroy$)
    );
    
    // Debounced search for query changes
    this.debouncedSearch$ = this.searchParams$.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      takeUntil(this.destroy$)
    );
    
    // Search execution with loading and error handling
    const searchExecution$ = this.debouncedSearch$.pipe(
      switchMap(params => {
        // Track analytics
        this.analyticsService.trackSearch(params);
        
        return this.searchService.search(params).pipe(
          map(results => ({ type: 'success', data: results })),
          catchError(error => of({ type: 'error', error: error.message })),
          startWith({ type: 'loading' })
        );
      }),
      shareReplay(1),
      takeUntil(this.destroy$)
    );
    
    // Extract different states from search execution
    this.searchResults$ = searchExecution$.pipe(
      filter(result => result.type === 'success'),
      map(result => result.data),
      takeUntil(this.destroy$)
    );
    
    this.isLoading$ = searchExecution$.pipe(
      map(result => result.type === 'loading'),
      startWith(false),
      takeUntil(this.destroy$)
    );
    
    this.error$ = searchExecution$.pipe(
      filter(result => result.type === 'error'),
      map(result => result.error),
      startWith(null),
      takeUntil(this.destroy$)
    );
    
    // Active filters derived from search params
    this.activeFilters$ = this.searchParams$.pipe(
      map(params => this.extractActiveFilters(params)),
      takeUntil(this.destroy$)
    );
    
    // Side effects
    this.setupSideEffects();
  }
  
  private setupSideEffects(): void {
    // Auto-save search preferences
    this.searchParams$.pipe(
      debounceTime(2000),
      takeUntil(this.destroy$)
    ).subscribe(params => {
      this.saveSearchPreferences(params);
    });
    
    // Track user interactions
    this.userInteractions$.pipe(
      takeUntil(this.destroy$)
    ).subscribe(interaction => {
      this.analyticsService.trackInteraction(interaction);
    });
    
    // Update URL with search params
    this.searchParams$.pipe(
      debounceTime(500),
      takeUntil(this.destroy$)
    ).subscribe(params => {
      this.updateUrlParams(params);
    });
  }
  
  private combineSearchParams(
    formValue: any, 
    externalFilter: ExternalFilter | null, 
    locationParams: any
  ): SearchParams {
    return {
      query: formValue.query || '',
      category: formValue.category || '',
      priceRange: {
        min: 0,
        max: formValue.priceRange || 1000
      },
      externalFilters: externalFilter ? [externalFilter] : [],
      location: locationParams,
      timestamp: Date.now()
    };
  }
  
  private extractActiveFilters(params: SearchParams): ActiveFilter[] {
    const filters: ActiveFilter[] = [];
    
    if (params.category) {
      filters.push({
        key: 'category',
        label: `Category: ${params.category}`,
        value: params.category
      });
    }
    
    if (params.priceRange.max < 1000) {
      filters.push({
        key: 'priceRange',
        label: `Max Price: $${params.priceRange.max}`,
        value: params.priceRange
      });
    }
    
    return filters;
  }
  
  private getLocationParams(): Observable<any> {
    // Simulate location-based parameters
    return of({ country: 'US', timezone: 'PST' });
  }
  
  private saveSearchPreferences(params: SearchParams): void {
    localStorage.setItem('searchPreferences', JSON.stringify(params));
  }
  
  private updateUrlParams(params: SearchParams): void {
    // Update URL without triggering navigation
    const url = new URL(window.location.href);
    url.searchParams.set('q', params.query);
    url.searchParams.set('cat', params.category);
    window.history.replaceState({}, '', url.toString());
  }
  
  // Public methods for template
  removeFilter(filterKey: string): void {
    switch (filterKey) {
      case 'category':
        this.searchForm.patchValue({ category: '' });
        break;
      case 'priceRange':
        this.searchForm.patchValue({ priceRange: 1000 });
        break;
    }
    
    this.userInteractions$.next({
      type: 'filter_removed',
      filterKey,
      timestamp: Date.now()
    });
  }
  
  addExternalFilter(filter: ExternalFilter): void {
    this.externalFilters$.next(filter);
  }
}

// Interfaces
interface SearchParams {
  query: string;
  category: string;
  priceRange: { min: number; max: number };
  externalFilters: ExternalFilter[];
  location: any;
  timestamp: number;
}

interface SearchResults {
  items: SearchItem[];
  total: number;
  duration: number;
}

interface SearchItem {
  id: string;
  title: string;
  description: string;
  price: number;
}

interface Category {
  id: string;
  name: string;
}

interface ActiveFilter {
  key: string;
  label: string;
  value: any;
}

interface ExternalFilter {
  type: string;
  value: any;
}

interface UserInteraction {
  type: string;
  filterKey?: string;
  timestamp: number;
}
```

## Performance Optimization

### Stream Performance Patterns

```typescript
// Performance optimization utilities
class StreamPerformanceOptimizer {
  // Lazy stream creation
  static lazy<T>(factory: () => Observable<T>): Observable<T> {
    return new Observable(observer => {
      const subscription = factory().subscribe(observer);
      return () => subscription.unsubscribe();
    });
  }
  
  // Memoized stream results
  static memoize<T, U>(
    stream$: Observable<T>,
    keySelector: (value: T) => string,
    ttl: number = 60000
  ): Observable<T> {
    const cache = new Map<string, { value: T; timestamp: number }>();
    
    return stream$.pipe(
      map(value => {
        const key = keySelector(value);
        const cached = cache.get(key);
        const now = Date.now();
        
        if (cached && (now - cached.timestamp) < ttl) {
          return cached.value;
        }
        
        cache.set(key, { value, timestamp: now });
        
        // Clean expired entries
        cache.forEach((entry, k) => {
          if (now - entry.timestamp > ttl) {
            cache.delete(k);
          }
        });
        
        return value;
      })
    );
  }
  
  // Batching for efficiency
  static batch<T>(
    stream$: Observable<T>,
    batchSize: number,
    timeWindow: number
  ): Observable<T[]> {
    return stream$.pipe(
      buffer(
        merge(
          stream$.pipe(bufferCount(batchSize), mapTo(null)),
          timer(timeWindow, timeWindow)
        )
      ),
      filter(batch => batch.length > 0)
    );
  }
  
  // Memory-efficient windowing
  static slidingWindow<T>(
    stream$: Observable<T>,
    windowSize: number
  ): Observable<T[]> {
    return stream$.pipe(
      scan((window, value) => {
        const newWindow = [...window, value];
        return newWindow.length > windowSize 
          ? newWindow.slice(-windowSize)
          : newWindow;
      }, [] as T[])
    );
  }
  
  // Adaptive backpressure
  static adaptiveBackpressure<T>(
    stream$: Observable<T>,
    maxBufferSize: number = 1000
  ): Observable<T> {
    let bufferSize = 0;
    
    return stream$.pipe(
      tap(() => bufferSize++),
      filter(() => {
        if (bufferSize > maxBufferSize) {
          console.warn('Backpressure detected, dropping items');
          bufferSize = Math.floor(maxBufferSize * 0.8); // Reset to 80%
          return false;
        }
        return true;
      }),
      tap(() => bufferSize--)
    );
  }
}

// Performance monitoring
class StreamPerformanceMonitor {
  private metrics = new Map<string, PerformanceMetrics>();
  
  monitor<T>(
    stream$: Observable<T>,
    streamName: string
  ): Observable<T> {
    const startTime = performance.now();
    let itemCount = 0;
    let errors = 0;
    
    return stream$.pipe(
      tap(() => itemCount++),
      catchError(error => {
        errors++;
        this.updateMetrics(streamName, startTime, itemCount, errors);
        return throwError(error);
      }),
      finalize(() => {
        this.updateMetrics(streamName, startTime, itemCount, errors);
      })
    );
  }
  
  private updateMetrics(
    streamName: string, 
    startTime: number, 
    itemCount: number, 
    errors: number
  ): void {
    const duration = performance.now() - startTime;
    const throughput = itemCount / (duration / 1000); // items per second
    
    this.metrics.set(streamName, {
      duration,
      itemCount,
      errors,
      throughput,
      timestamp: Date.now()
    });
  }
  
  getMetrics(streamName: string): PerformanceMetrics | undefined {
    return this.metrics.get(streamName);
  }
  
  getAllMetrics(): Map<string, PerformanceMetrics> {
    return new Map(this.metrics);
  }
}

interface PerformanceMetrics {
  duration: number;
  itemCount: number;
  errors: number;
  throughput: number;
  timestamp: number;
}
```

## Testing Complex Compositions

### Advanced Testing Patterns

```typescript
import { TestScheduler } from 'rxjs/testing';
import { cold, hot } from 'jasmine-marbles';

describe('Complex Stream Composition', () => {
  let testScheduler: TestScheduler;
  let streamComposer: StreamComposer<any>;
  
  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });
  
  describe('StreamComposer', () => {
    it('should handle complex branching correctly', () => {
      testScheduler.run(({ cold, hot, expectObservable }) => {
        const source$ = hot('  -a-b-c-d-e-|', {
          a: { type: 'A', value: 1 },
          b: { type: 'B', value: 2 },
          c: { type: 'A', value: 3 },
          d: { type: 'B', value: 4 },
          e: { type: 'A', value: 5 }
        });
        
        const result$ = composeStream(source$)
          .branch(
            item => item.type === 'A',
            item => cold('--x|', { x: { ...item, processed: 'A' } }),
            item => cold('-y|', { y: { ...item, processed: 'B' } })
          )
          .build();
        
        expectObservable(result$).toBe('---x-y---x-y---x|', {
          x: { type: 'A', value: 1, processed: 'A' },
          y: { type: 'B', value: 2, processed: 'B' }
        });
      });
    });
    
    it('should handle parallel processing', () => {
      testScheduler.run(({ cold, hot, expectObservable }) => {
        const source$ = hot('-a----b----|', {
          a: { value: 1 },
          b: { value: 2 }
        });
        
        const result$ = composeStream(source$)
          .parallel([
            item => cold('--x|', { x: item.value * 2 }),
            item => cold('---y|', { y: item.value * 3 }),
            item => cold('-z|', { z: item.value * 4 })
          ], results => results.reduce((sum, val) => sum + val, 0))
          .build();
        
        expectObservable(result$).toBe('----a-----b----|', {
          a: 1 * 2 + 1 * 3 + 1 * 4, // 9
          b: 2 * 2 + 2 * 3 + 2 * 4  // 18
        });
      });
    });
  });
  
  describe('ConditionalStreamProcessor', () => {
    it('should process with priority-based conditions', () => {
      testScheduler.run(({ cold, hot, expectObservable }) => {
        const source$ = hot('-a-b-c-|', {
          a: { priority: 'high', value: 1 },
          b: { priority: 'low', value: 2 },
          c: { priority: 'medium', value: 3 }
        });
        
        const processor = new ConditionalStreamProcessor(source$);
        
        const result$ = processor.processWithConditions([
          {
            condition: item => item.priority === 'high',
            processor: item => cold('--x|', { x: `high:${item.value}` }),
            priority: 100
          },
          {
            condition: item => item.priority === 'medium',
            processor: item => cold('---y|', { y: `medium:${item.value}` }),
            priority: 50
          },
          {
            condition: () => true,
            processor: item => cold('-z|', { z: `default:${item.value}` }),
            priority: 0
          }
        ]);
        
        expectObservable(result$).toBe('---x-z----y|', {
          x: 'high:1',
          z: 'default:2',
          y: 'medium:3'
        });
      });
    });
  });
  
  describe('StatefulStreamManager', () => {
    it('should manage state correctly with time travel', () => {
      testScheduler.run(({ cold, hot, expectObservable }) => {
        const events$ = hot('-a-b-c-|', {
          a: { type: 'ADD', value: 1 },
          b: { type: 'ADD', value: 2 },
          c: { type: 'SUBTRACT', value: 1 }
        });
        
        const stateManager = new StatefulStreamManager(
          events$,
          { total: 0 },
          (state, event) => {
            switch (event.type) {
              case 'ADD':
                return { total: state.total + event.value };
              case 'SUBTRACT':
                return { total: state.total - event.value };
              default:
                return state;
            }
          }
        );
        
        const result$ = stateManager.getStateStream();
        
        expectObservable(result$).toBe('aa-b-c-|', {
          a: { total: 0 },
          b: { total: 1 },
          c: { total: 3 }
        });
      });
    });
  });
});

// Integration testing for complex compositions
describe('Integration: Complex Data Pipeline', () => {
  let pipeline: DataProcessingPipeline<any>;
  let testScheduler: TestScheduler;
  
  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });
  
  it('should process mixed data types correctly', () => {
    testScheduler.run(({ cold, hot, expectObservable }) => {
      const source$ = hot('-a-b-c-d-e-|', {
        a: { type: 'user-data', userId: '1', action: 'login' },
        b: { type: 'analytics-data', event: 'page-view', page: '/home' },
        c: { type: 'system-data', metric: 'cpu', value: 80 },
        d: { type: 'user-data', userId: '2', action: 'logout' },
        e: { type: 'analytics-data', event: 'click', element: 'button' }
      });
      
      pipeline = new DataProcessingPipeline(source$);
      const result$ = pipeline.getProcessedResults();
      
      // Verify that each data type is processed according to its rules
      expectObservable(result$).toBe('-a---c-d----|', {
        a: jasmine.objectContaining({ processed: true, userId: '1' }),
        c: jasmine.objectContaining({ enriched: true, metric: 'cpu' }),
        d: jasmine.objectContaining({ processed: true, userId: '2' })
      });
    });
  });
});
```

## Exercises

### Exercise 1: Build a Complex Stream Pipeline
Create a data processing pipeline that handles multiple data sources with different processing requirements:

```typescript
// TODO: Implement a pipeline that:
// 1. Accepts multiple input streams (user events, system metrics, external APIs)
// 2. Routes data based on type and priority
// 3. Applies different transformations per data type
// 4. Handles errors and retries appropriately
// 5. Aggregates results into a unified output

class MultiSourceDataPipeline {
  // Your implementation here
}

// Test your implementation
const userEvents$ = interval(1000).pipe(
  map(i => ({ type: 'user-event', data: `event-${i}` }))
);

const systemMetrics$ = interval(2000).pipe(
  map(i => ({ type: 'system-metric', data: { cpu: Math.random() * 100 } }))
);

const pipeline = new MultiSourceDataPipeline();
// Configure and test your pipeline
```

### Exercise 2: Implement Adaptive Stream Processing
Build a system that adapts its processing strategy based on load and performance:

```typescript
// TODO: Create an adaptive processor that:
// 1. Monitors stream throughput and latency
// 2. Switches between different processing strategies based on load
// 3. Implements backpressure when overwhelmed
// 4. Provides metrics and monitoring

class AdaptiveStreamProcessor<T> {
  // Your implementation here
}
```

### Exercise 3: Create a State-Aware Stream Orchestrator
Implement a complex orchestrator that manages multiple interdependent streams:

```typescript
// TODO: Build an orchestrator that:
// 1. Manages dependencies between multiple streams
// 2. Handles stream lifecycle (start, pause, resume, stop)
// 3. Maintains shared state across streams
// 4. Provides rollback capabilities for failed operations

class StreamOrchestrator {
  // Your implementation here
}
```

## Summary

In this lesson, we explored sophisticated stream composition techniques that enable building complex, maintainable reactive systems:

### Key Concepts Covered:
1. **Advanced Composition Patterns** - Building reusable stream composition utilities
2. **Multi-Source Coordination** - Managing dependencies and execution order
3. **Dynamic Stream Creation** - Runtime stream generation and configuration
4. **Branching and Merging** - Complex routing and aggregation patterns
5. **Conditional Processing** - Priority-based and adaptive processing
6. **Stateful Composition** - Managing state across complex stream networks
7. **Performance Optimization** - Techniques for high-performance stream processing

### Best Practices:
- Use composition patterns to build reusable stream utilities
- Implement proper error handling and fallback strategies
- Monitor performance and implement adaptive backpressure
- Design for testability with clear separation of concerns
- Document complex compositions for maintainability

### Angular Integration:
- Reactive form handling with complex validation
- Multi-source data coordination in components
- Performance-optimized search and filtering
- State management across component hierarchies

### Next Steps:
- Practice building complex data pipelines
- Explore real-world stream orchestration scenarios
- Study performance optimization techniques
- Learn advanced error handling and recovery patterns

These techniques form the foundation for building sophisticated reactive applications that can handle complex business requirements while maintaining performance and reliability.
