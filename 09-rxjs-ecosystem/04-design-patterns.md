# Complete RxJS Design Patterns Collection

## Introduction

This lesson provides a comprehensive collection of proven RxJS design patterns for building robust, scalable, and maintainable reactive applications. We'll explore creational, structural, and behavioral patterns that solve common challenges in reactive programming.

## Learning Objectives

- Master essential RxJS design patterns and their use cases
- Understand when and how to apply each pattern effectively
- Learn to combine patterns for complex reactive architectures
- Implement reusable pattern-based solutions
- Create custom patterns for specific domain requirements

## Creational Patterns

### 1. Factory Pattern for Observables

```typescript
// ✅ Observable Factory Pattern
export class ObservableFactory {
  
  // HTTP request factory with different configurations
  static createHttpRequest<T>(config: HttpRequestConfig): Observable<T> {
    const { url, method = 'GET', retries = 3, timeout = 30000 } = config;
    
    return defer(() => {
      const request$ = this.createBaseRequest<T>(url, method);
      
      return request$.pipe(
        timeout(timeout),
        retryWhen(this.createRetryStrategy(retries)),
        catchError(this.handleHttpError)
      );
    });
  }
  
  // WebSocket connection factory
  static createWebSocketConnection(
    url: string, 
    options: WebSocketOptions = {}
  ): Observable<WebSocketConnection> {
    return new Observable<WebSocketConnection>(subscriber => {
      const ws = new WebSocket(url, options.protocols);
      const connection = new WebSocketConnection(ws);
      
      ws.onopen = () => subscriber.next(connection);
      ws.onerror = (error) => subscriber.error(error);
      ws.onclose = () => subscriber.complete();
      
      return () => {
        if (ws.readyState === WebSocket.OPEN) {
          ws.close();
        }
      };
    }).pipe(
      shareReplay({ bufferSize: 1, refCount: true })
    );
  }
  
  // Polling factory with different strategies
  static createPolling<T>(
    source: () => Observable<T>,
    strategy: PollingStrategy
  ): Observable<T> {
    switch (strategy.type) {
      case 'fixed':
        return timer(0, strategy.interval).pipe(
          switchMap(() => source())
        );
        
      case 'exponential':
        return this.createExponentialPolling(source, strategy);
        
      case 'adaptive':
        return this.createAdaptivePolling(source, strategy);
        
      default:
        throw new Error(`Unknown polling strategy: ${strategy.type}`);
    }
  }
  
  // Stream factory for different data sources
  static createDataStream<T>(
    source: DataSource,
    options: StreamOptions = {}
  ): Observable<T> {
    const { bufferSize = 100, backpressureStrategy = 'drop' } = options;
    
    switch (source.type) {
      case 'file':
        return this.createFileStream<T>(source, options);
        
      case 'database':
        return this.createDatabaseStream<T>(source, options);
        
      case 'api':
        return this.createApiStream<T>(source, options);
        
      case 'realtime':
        return this.createRealtimeStream<T>(source, options);
        
      default:
        throw new Error(`Unsupported data source: ${source.type}`);
    }
  }
}

// Usage examples
const apiCall$ = ObservableFactory.createHttpRequest<User[]>({
  url: '/api/users',
  method: 'GET',
  retries: 3,
  timeout: 5000
});

const wsConnection$ = ObservableFactory.createWebSocketConnection(
  'wss://api.example.com/ws',
  { protocols: ['json'] }
);

const polling$ = ObservableFactory.createPolling(
  () => this.statusService.getStatus(),
  { type: 'exponential', initialInterval: 1000, maxInterval: 30000 }
);
```

### 2. Builder Pattern for Complex Observables

```typescript
// ✅ Observable Builder Pattern
export class ObservableBuilder<T> {
  private source$: Observable<T>;
  private operators: OperatorFunction<any, any>[] = [];
  
  constructor(source: Observable<T>) {
    this.source$ = source;
  }
  
  // Transformation builders
  transform<R>(mapper: (value: T) => R): ObservableBuilder<R> {
    this.operators.push(map(mapper));
    return this as any;
  }
  
  filter(predicate: (value: T) => boolean): ObservableBuilder<T> {
    this.operators.push(filter(predicate));
    return this;
  }
  
  debounce(dueTime: number): ObservableBuilder<T> {
    this.operators.push(debounceTime(dueTime));
    return this;
  }
  
  distinctValues(): ObservableBuilder<T> {
    this.operators.push(distinctUntilChanged());
    return this;
  }
  
  // Error handling builders
  retryOnError(config: RetryConfig): ObservableBuilder<T> {
    this.operators.push(
      retryWhen(errors => this.createRetryStrategy(errors, config))
    );
    return this;
  }
  
  fallbackTo(fallbackValue: T): ObservableBuilder<T> {
    this.operators.push(
      catchError(() => of(fallbackValue))
    );
    return this;
  }
  
  // Timing builders
  timeoutAfter(ms: number): ObservableBuilder<T> {
    this.operators.push(timeout(ms));
    return this;
  }
  
  delayBy(ms: number): ObservableBuilder<T> {
    this.operators.push(delay(ms));
    return this;
  }
  
  // Caching builders
  cache(config: CacheConfig = {}): ObservableBuilder<T> {
    const { bufferSize = 1, refCount = true } = config;
    this.operators.push(shareReplay({ bufferSize, refCount }));
    return this;
  }
  
  // Lifecycle builders
  takeUntilDestroy(destroy$: Observable<any>): ObservableBuilder<T> {
    this.operators.push(takeUntil(destroy$));
    return this;
  }
  
  // Build final observable
  build(): Observable<T> {
    return this.source$.pipe(...this.operators as any);
  }
  
  // Build with subscription
  subscribe(observer?: Partial<Observer<T>>): Subscription {
    return this.build().subscribe(observer);
  }
  
  private createRetryStrategy(
    errors: Observable<any>, 
    config: RetryConfig
  ): Observable<any> {
    return errors.pipe(
      scan((retryCount, error) => {
        if (retryCount >= config.maxRetries) {
          throw error;
        }
        return retryCount + 1;
      }, 0),
      delayWhen(retryCount => 
        timer(config.delay * Math.pow(config.backoffFactor || 1, retryCount - 1))
      )
    );
  }
}

// Fluent API usage
const searchResults$ = new ObservableBuilder(this.searchInput$)
  .debounce(300)
  .distinctValues()
  .filter(query => query.length >= 2)
  .transform(query => ({ query, timestamp: Date.now() }))
  .timeoutAfter(5000)
  .retryOnError({ maxRetries: 3, delay: 1000, backoffFactor: 2 })
  .fallbackTo({ query: '', timestamp: 0 })
  .cache({ bufferSize: 1, refCount: true })
  .takeUntilDestroy(this.destroy$)
  .build();
```

### 3. Singleton Pattern for Shared Resources

```typescript
// ✅ Observable Singleton Pattern
export class SharedResourceManager {
  private static instances = new Map<string, Observable<any>>();
  
  // Singleton database connection
  static getDatabaseConnection(): Observable<DatabaseConnection> {
    const key = 'database-connection';
    
    if (!this.instances.has(key)) {
      const connection$ = defer(() => 
        this.createDatabaseConnection()
      ).pipe(
        retry({ count: 3, delay: 1000 }),
        shareReplay({ bufferSize: 1, refCount: false }), // Keep alive
        finalize(() => {
          console.log('Database connection closed');
          this.instances.delete(key);
        })
      );
      
      this.instances.set(key, connection$);
    }
    
    return this.instances.get(key)!;
  }
  
  // Singleton configuration service
  static getConfiguration(): Observable<AppConfig> {
    const key = 'app-configuration';
    
    if (!this.instances.has(key)) {
      const config$ = this.loadConfiguration().pipe(
        shareReplay({ bufferSize: 1, refCount: false })
      );
      
      this.instances.set(key, config$);
    }
    
    return this.instances.get(key)!;
  }
  
  // Singleton user session
  static getUserSession(): Observable<UserSession> {
    const key = 'user-session';
    
    if (!this.instances.has(key)) {
      const session$ = merge(
        this.loadStoredSession(),
        this.authEvents$.pipe(
          filter(event => event.type === 'login'),
          switchMap(() => this.createNewSession())
        )
      ).pipe(
        distinctUntilChanged((a, b) => a.sessionId === b.sessionId),
        shareReplay({ bufferSize: 1, refCount: false })
      );
      
      this.instances.set(key, session$);
    }
    
    return this.instances.get(key)!;
  }
  
  // Clear specific singleton
  static clearInstance(key: string): void {
    this.instances.delete(key);
  }
  
  // Clear all singletons
  static clearAll(): void {
    this.instances.clear();
  }
}

// Usage
const dbConnection$ = SharedResourceManager.getDatabaseConnection();
const config$ = SharedResourceManager.getConfiguration();
const session$ = SharedResourceManager.getUserSession();
```

## Structural Patterns

### 1. Adapter Pattern for Data Transformation

```typescript
// ✅ Observable Adapter Pattern
export interface DataAdapter<TInput, TOutput> {
  adapt(source: Observable<TInput>): Observable<TOutput>;
}

// Legacy API adapter
export class LegacyApiAdapter implements DataAdapter<LegacyUser, User> {
  adapt(source: Observable<LegacyUser>): Observable<User> {
    return source.pipe(
      map(legacyUser => this.transformUser(legacyUser)),
      catchError(error => {
        console.warn('Legacy API error, attempting fallback');
        return this.getFallbackUser(legacyUser.id);
      })
    );
  }
  
  private transformUser(legacy: LegacyUser): User {
    return {
      id: legacy.userId,
      email: legacy.emailAddress,
      name: `${legacy.firstName} ${legacy.lastName}`,
      createdAt: new Date(legacy.creationTimestamp),
      isActive: legacy.status === 'ACTIVE',
      profile: {
        avatar: legacy.profilePicture,
        bio: legacy.description,
        preferences: this.adaptPreferences(legacy.settings)
      }
    };
  }
}

// External service adapter
export class ExternalServiceAdapter implements DataAdapter<ExternalData, InternalData> {
  constructor(private mappingConfig: MappingConfig) {}
  
  adapt(source: Observable<ExternalData>): Observable<InternalData> {
    return source.pipe(
      map(data => this.applyMapping(data)),
      filter(data => this.isValidData(data)),
      tap(data => this.logTransformation(data)),
      catchError(error => this.handleAdapterError(error))
    );
  }
  
  private applyMapping(external: ExternalData): InternalData {
    const mapping = this.mappingConfig;
    const internal: Partial<InternalData> = {};
    
    for (const [internalKey, externalPath] of Object.entries(mapping.fieldMappings)) {
      internal[internalKey] = this.getNestedValue(external, externalPath);
    }
    
    return internal as InternalData;
  }
}

// Composite adapter for multiple sources
export class CompositeAdapter<T> implements DataAdapter<any, T> {
  private adapters = new Map<string, DataAdapter<any, T>>();
  
  registerAdapter(sourceType: string, adapter: DataAdapter<any, T>): void {
    this.adapters.set(sourceType, adapter);
  }
  
  adapt(source: Observable<{ type: string; data: any }>): Observable<T> {
    return source.pipe(
      switchMap(({ type, data }) => {
        const adapter = this.adapters.get(type);
        
        if (!adapter) {
          return throwError(() => new Error(`No adapter for type: ${type}`));
        }
        
        return adapter.adapt(of(data));
      })
    );
  }
}
```

### 2. Decorator Pattern for Observable Enhancement

```typescript
// ✅ Observable Decorator Pattern
export abstract class ObservableDecorator<T> {
  constructor(protected source: Observable<T>) {}
  
  abstract pipe(...operators: OperatorFunction<T, any>[]): Observable<any>;
}

// Logging decorator
export class LoggingDecorator<T> extends ObservableDecorator<T> {
  constructor(
    source: Observable<T>, 
    private logger: Logger,
    private context: string
  ) {
    super(source);
  }
  
  pipe(...operators: OperatorFunction<T, any>[]): Observable<any> {
    return this.source.pipe(
      tap({
        next: value => this.logger.debug(`${this.context} - Next:`, value),
        error: error => this.logger.error(`${this.context} - Error:`, error),
        complete: () => this.logger.debug(`${this.context} - Complete`)
      }),
      ...operators
    );
  }
}

// Metrics decorator
export class MetricsDecorator<T> extends ObservableDecorator<T> {
  constructor(
    source: Observable<T>,
    private metricsService: MetricsService,
    private operationName: string
  ) {
    super(source);
  }
  
  pipe(...operators: OperatorFunction<T, any>[]): Observable<any> {
    const startTime = performance.now();
    let itemCount = 0;
    
    return this.source.pipe(
      tap({
        next: () => {
          itemCount++;
          this.metricsService.incrementCounter(`${this.operationName}.items`);
        },
        error: error => {
          const duration = performance.now() - startTime;
          this.metricsService.recordDuration(`${this.operationName}.error`, duration);
          this.metricsService.incrementCounter(`${this.operationName}.errors`);
        },
        complete: () => {
          const duration = performance.now() - startTime;
          this.metricsService.recordDuration(`${this.operationName}.success`, duration);
          this.metricsService.recordGauge(`${this.operationName}.item_count`, itemCount);
        }
      }),
      ...operators
    );
  }
}

// Cache decorator
export class CacheDecorator<T> extends ObservableDecorator<T> {
  private cache = new Map<string, Observable<T>>();
  
  constructor(
    source: Observable<T>,
    private cacheKeyFn: (value: any) => string,
    private ttl: number = 300000 // 5 minutes
  ) {
    super(source);
  }
  
  pipe(...operators: OperatorFunction<T, any>[]): Observable<any> {
    const cacheKey = this.cacheKeyFn(this.source);
    
    if (this.cache.has(cacheKey)) {
      return this.cache.get(cacheKey)!.pipe(...operators);
    }
    
    const cached$ = this.source.pipe(
      shareReplay({ bufferSize: 1, refCount: true }),
      finalize(() => {
        // Remove from cache after TTL
        setTimeout(() => this.cache.delete(cacheKey), this.ttl);
      })
    );
    
    this.cache.set(cacheKey, cached$);
    return cached$.pipe(...operators);
  }
}

// Decorator factory
export class ObservableDecoratorFactory {
  static withLogging<T>(
    source: Observable<T>, 
    logger: Logger, 
    context: string
  ): LoggingDecorator<T> {
    return new LoggingDecorator(source, logger, context);
  }
  
  static withMetrics<T>(
    source: Observable<T>, 
    metricsService: MetricsService, 
    operationName: string
  ): MetricsDecorator<T> {
    return new MetricsDecorator(source, metricsService, operationName);
  }
  
  static withCache<T>(
    source: Observable<T>, 
    cacheKeyFn: (value: any) => string, 
    ttl?: number
  ): CacheDecorator<T> {
    return new CacheDecorator(source, cacheKeyFn, ttl);
  }
  
  // Compose multiple decorators
  static compose<T>(
    source: Observable<T>,
    ...decorators: Array<(source: Observable<T>) => ObservableDecorator<T>>
  ): Observable<T> {
    return decorators.reduce(
      (decorated, decorator) => decorator(decorated).pipe(),
      source
    );
  }
}
```

### 3. Facade Pattern for Complex Operations

```typescript
// ✅ Observable Facade Pattern
export class UserManagementFacade {
  constructor(
    private userService: UserService,
    private profileService: ProfileService,
    private authService: AuthService,
    private notificationService: NotificationService,
    private auditService: AuditService
  ) {}
  
  // Simplified interface for complex user creation
  createCompleteUser(userData: CreateUserRequest): Observable<CompleteUser> {
    return this.validateUserData(userData).pipe(
      switchMap(validData => this.executeUserCreation(validData)),
      tap(user => this.postCreationActions(user)),
      catchError(error => this.handleCreationError(error, userData))
    );
  }
  
  // Simplified interface for user profile updates
  updateUserProfile(
    userId: string, 
    updates: ProfileUpdateRequest
  ): Observable<UpdateResult> {
    return combineLatest([
      this.userService.getUser(userId),
      this.authService.getCurrentUser(),
      this.profileService.getProfile(userId)
    ]).pipe(
      switchMap(([user, currentUser, profile]) => 
        this.validateUpdatePermissions(user, currentUser).pipe(
          switchMap(() => this.executeProfileUpdate(profile, updates))
        )
      ),
      switchMap(result => this.notifyProfileUpdate(userId, result)),
      tap(result => this.auditProfileUpdate(userId, updates, result))
    );
  }
  
  // Simplified interface for user search with filters
  searchUsersWithFilters(
    query: string, 
    filters: UserFilters
  ): Observable<PaginatedUsers> {
    return this.buildSearchRequest(query, filters).pipe(
      switchMap(request => 
        combineLatest([
          this.userService.searchUsers(request),
          this.profileService.getPublicProfiles(request.userIds),
          this.authService.getUserPermissions()
        ])
      ),
      map(([users, profiles, permissions]) => 
        this.combineUserData(users, profiles, permissions)
      ),
      shareReplay({ bufferSize: 1, refCount: true })
    );
  }
  
  // Simplified interface for bulk operations
  performBulkUserOperation(
    userIds: string[], 
    operation: BulkOperation
  ): Observable<BulkOperationResult> {
    return from(userIds).pipe(
      mergeMap(
        userId => this.executeSingleOperation(userId, operation).pipe(
          map(result => ({ userId, success: true, result })),
          catchError(error => of({ 
            userId, 
            success: false, 
            error: error.message 
          }))
        ),
        5 // Limit concurrency to 5 operations at a time
      ),
      scan((acc, result) => {
        acc.results.push(result);
        if (result.success) {
          acc.successCount++;
        } else {
          acc.errorCount++;
        }
        return acc;
      }, { results: [], successCount: 0, errorCount: 0 } as BulkOperationResult),
      last(),
      tap(result => this.auditBulkOperation(operation, result))
    );
  }
  
  private validateUserData(userData: CreateUserRequest): Observable<CreateUserRequest> {
    return this.userService.validateEmail(userData.email).pipe(
      switchMap(isValid => {
        if (!isValid) {
          return throwError(() => new ValidationError('Invalid email'));
        }
        return of(userData);
      })
    );
  }
  
  private executeUserCreation(userData: CreateUserRequest): Observable<CompleteUser> {
    return this.userService.createUser(userData).pipe(
      switchMap(user => 
        this.profileService.createProfile(user.id, userData.profile).pipe(
          map(profile => ({ user, profile }))
        )
      ),
      switchMap(({ user, profile }) =>
        this.authService.createUserCredentials(user.id, userData.password).pipe(
          map(credentials => ({ user, profile, credentials }))
        )
      )
    );
  }
}
```

## Behavioral Patterns

### 1. Observer Pattern with Subject Management

```typescript
// ✅ Advanced Observer Pattern
export class EventBus {
  private subjects = new Map<string, Subject<any>>();
  private globalSubject = new Subject<BusEvent>();
  
  // Type-safe event subscription
  on<T>(eventType: string): Observable<T> {
    if (!this.subjects.has(eventType)) {
      this.subjects.set(eventType, new Subject<T>());
    }
    
    return this.subjects.get(eventType)!.asObservable();
  }
  
  // Emit events with metadata
  emit<T>(eventType: string, data: T, metadata?: EventMetadata): void {
    const event: BusEvent = {
      type: eventType,
      data,
      timestamp: new Date(),
      id: this.generateEventId(),
      metadata
    };
    
    // Emit to specific subscribers
    if (this.subjects.has(eventType)) {
      this.subjects.get(eventType)!.next(data);
    }
    
    // Emit to global stream
    this.globalSubject.next(event);
  }
  
  // Subscribe to all events
  onAll(): Observable<BusEvent> {
    return this.globalSubject.asObservable();
  }
  
  // Subscribe to multiple event types
  onAny(eventTypes: string[]): Observable<BusEvent> {
    return this.globalSubject.pipe(
      filter(event => eventTypes.includes(event.type))
    );
  }
  
  // Subscribe with filters
  onWhere<T>(
    eventType: string, 
    predicate: (data: T) => boolean
  ): Observable<T> {
    return this.on<T>(eventType).pipe(
      filter(predicate)
    );
  }
  
  // Buffered events
  onBuffered<T>(
    eventType: string, 
    bufferTime: number = 1000
  ): Observable<T[]> {
    return this.on<T>(eventType).pipe(
      bufferTime(bufferTime),
      filter(buffer => buffer.length > 0)
    );
  }
  
  // Request-response pattern
  request<TRequest, TResponse>(
    requestType: string, 
    data: TRequest, 
    timeout: number = 30000
  ): Observable<TResponse> {
    const responseType = `${requestType}:response`;
    const requestId = this.generateEventId();
    
    // Listen for response first
    const response$ = this.on<{ requestId: string; data: TResponse }>(responseType).pipe(
      filter(response => response.requestId === requestId),
      map(response => response.data),
      take(1),
      timeout(timeout)
    );
    
    // Send request
    this.emit(requestType, { requestId, data });
    
    return response$;
  }
  
  // Cleanup
  destroy(): void {
    this.subjects.forEach(subject => subject.complete());
    this.subjects.clear();
    this.globalSubject.complete();
  }
}

// Usage examples
const eventBus = new EventBus();

// Subscribe to user events
eventBus.on<UserCreatedEvent>('user.created').subscribe(user => {
  console.log('New user created:', user);
});

// Subscribe to multiple events
eventBus.onAny(['user.created', 'user.updated', 'user.deleted']).subscribe(event => {
  console.log('User event:', event);
});

// Filtered subscription
eventBus.onWhere<UserUpdatedEvent>('user.updated', 
  user => user.isActive
).subscribe(user => {
  console.log('Active user updated:', user);
});

// Request-response
eventBus.request<string, User>('user.get', userId).subscribe(user => {
  console.log('Received user:', user);
});
```

### 2. Strategy Pattern for Operator Selection

```typescript
// ✅ Strategy Pattern for RxJS Operations
export interface OperatorStrategy<T, R> {
  execute(source: Observable<T>): Observable<R>;
}

// Retry strategies
export class FixedRetryStrategy<T> implements OperatorStrategy<T, T> {
  constructor(
    private maxRetries: number, 
    private delay: number
  ) {}
  
  execute(source: Observable<T>): Observable<T> {
    return source.pipe(
      retryWhen(errors => errors.pipe(
        scan((retryCount, error) => {
          if (retryCount >= this.maxRetries) {
            throw error;
          }
          return retryCount + 1;
        }, 0),
        delay(this.delay)
      ))
    );
  }
}

export class ExponentialBackoffStrategy<T> implements OperatorStrategy<T, T> {
  constructor(
    private maxRetries: number,
    private baseDelay: number,
    private maxDelay: number = 30000
  ) {}
  
  execute(source: Observable<T>): Observable<T> {
    return source.pipe(
      retryWhen(errors => errors.pipe(
        scan((retryCount, error) => {
          if (retryCount >= this.maxRetries) {
            throw error;
          }
          return retryCount + 1;
        }, 0),
        delayWhen(retryCount => {
          const delayTime = Math.min(
            this.baseDelay * Math.pow(2, retryCount - 1),
            this.maxDelay
          );
          return timer(delayTime);
        })
      ))
    );
  }
}

// Error handling strategies
export class FallbackStrategy<T> implements OperatorStrategy<T, T> {
  constructor(private fallbackValue: T) {}
  
  execute(source: Observable<T>): Observable<T> {
    return source.pipe(
      catchError(() => of(this.fallbackValue))
    );
  }
}

export class CircuitBreakerStrategy<T> implements OperatorStrategy<T, T> {
  private failureCount = 0;
  private isOpen = false;
  private lastFailureTime = 0;
  
  constructor(
    private failureThreshold: number,
    private resetTimeoutMs: number
  ) {}
  
  execute(source: Observable<T>): Observable<T> {
    return defer(() => {
      if (this.isOpen && this.shouldReset()) {
        this.reset();
      }
      
      if (this.isOpen) {
        return throwError(() => new Error('Circuit breaker is open'));
      }
      
      return source.pipe(
        tap(() => this.onSuccess()),
        catchError(error => {
          this.onFailure();
          return throwError(() => error);
        })
      );
    });
  }
  
  private onSuccess(): void {
    this.failureCount = 0;
  }
  
  private onFailure(): void {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    
    if (this.failureCount >= this.failureThreshold) {
      this.isOpen = true;
    }
  }
  
  private shouldReset(): boolean {
    return Date.now() - this.lastFailureTime >= this.resetTimeoutMs;
  }
  
  private reset(): void {
    this.isOpen = false;
    this.failureCount = 0;
  }
}

// Strategy context
export class ResilientObservable<T> {
  private retryStrategy?: OperatorStrategy<T, T>;
  private errorStrategy?: OperatorStrategy<T, T>;
  
  constructor(private source: Observable<T>) {}
  
  withRetryStrategy(strategy: OperatorStrategy<T, T>): ResilientObservable<T> {
    this.retryStrategy = strategy;
    return this;
  }
  
  withErrorStrategy(strategy: OperatorStrategy<T, T>): ResilientObservable<T> {
    this.errorStrategy = strategy;
    return this;
  }
  
  build(): Observable<T> {
    let result$ = this.source;
    
    if (this.retryStrategy) {
      result$ = this.retryStrategy.execute(result$);
    }
    
    if (this.errorStrategy) {
      result$ = this.errorStrategy.execute(result$);
    }
    
    return result$;
  }
}

// Usage
const resilientApiCall$ = new ResilientObservable(
  this.http.get<Data>('/api/data')
)
  .withRetryStrategy(new ExponentialBackoffStrategy(3, 1000, 10000))
  .withErrorStrategy(new CircuitBreakerStrategy(5, 60000))
  .build();
```

### 3. Command Pattern for Reactive Operations

```typescript
// ✅ Command Pattern for Observable Operations
export interface Command<T = any> {
  execute(): Observable<T>;
  undo?(): Observable<void>;
  canExecute?(): Observable<boolean>;
  description: string;
}

// Basic commands
export class ApiCommand<T> implements Command<T> {
  constructor(
    public description: string,
    private apiCall: () => Observable<T>,
    private undoCall?: () => Observable<void>
  ) {}
  
  execute(): Observable<T> {
    return defer(() => {
      console.log(`Executing: ${this.description}`);
      return this.apiCall();
    });
  }
  
  undo(): Observable<void> {
    if (!this.undoCall) {
      return throwError(() => new Error('Undo not supported'));
    }
    
    return defer(() => {
      console.log(`Undoing: ${this.description}`);
      return this.undoCall!();
    });
  }
}

// Composite command for multiple operations
export class CompositeCommand implements Command<any[]> {
  private commands: Command[] = [];
  
  constructor(public description: string) {}
  
  add(command: Command): CompositeCommand {
    this.commands.push(command);
    return this;
  }
  
  execute(): Observable<any[]> {
    return from(this.commands).pipe(
      mergeMap(command => command.execute().pipe(
        map(result => ({ command, result, success: true })),
        catchError(error => of({ command, error, success: false }))
      ), 3), // Limit concurrency
      scan((acc, result) => [...acc, result], [] as any[]),
      last()
    );
  }
  
  undo(): Observable<void> {
    // Undo in reverse order
    const undoableCommands = this.commands
      .filter(cmd => cmd.undo)
      .reverse();
    
    return from(undoableCommands).pipe(
      concatMap(command => command.undo!()),
      ignoreElements(),
      defaultIfEmpty(undefined),
      map(() => void 0)
    );
  }
}

// Command queue with execution control
export class CommandQueue {
  private queue$ = new Subject<Command>();
  private isExecuting$ = new BehaviorSubject<boolean>(false);
  private executionHistory: Array<{ command: Command; result: any; timestamp: Date }> = [];
  
  constructor(private concurrency: number = 1) {
    this.setupExecution();
  }
  
  enqueue<T>(command: Command<T>): Observable<T> {
    return new Observable<T>(subscriber => {
      const executionId = this.generateExecutionId();
      
      // Add to queue
      this.queue$.next(command);
      
      // Wait for result
      const subscription = this.onCommandComplete(executionId).subscribe({
        next: result => {
          if (result.success) {
            subscriber.next(result.data);
            subscriber.complete();
          } else {
            subscriber.error(result.error);
          }
        },
        error: error => subscriber.error(error)
      });
      
      return () => subscription.unsubscribe();
    });
  }
  
  private setupExecution(): void {
    this.queue$.pipe(
      tap(() => this.isExecuting$.next(true)),
      mergeMap(
        command => this.executeCommand(command),
        this.concurrency
      ),
      tap(() => {
        if (this.queue$.observers.length === 0) {
          this.isExecuting$.next(false);
        }
      })
    ).subscribe();
  }
  
  private executeCommand(command: Command): Observable<any> {
    const startTime = new Date();
    
    return command.execute().pipe(
      map(result => {
        this.executionHistory.push({
          command,
          result,
          timestamp: startTime
        });
        return result;
      }),
      catchError(error => {
        this.executionHistory.push({
          command,
          result: error,
          timestamp: startTime
        });
        return throwError(() => error);
      })
    );
  }
  
  getExecutionHistory(): Array<{ command: Command; result: any; timestamp: Date }> {
    return [...this.executionHistory];
  }
  
  isExecuting(): Observable<boolean> {
    return this.isExecuting$.asObservable();
  }
}

// Usage examples
const createUserCommand = new ApiCommand(
  'Create User',
  () => this.userService.createUser(userData),
  () => this.userService.deleteUser(userId)
);

const updateProfileCommand = new ApiCommand(
  'Update Profile',
  () => this.profileService.updateProfile(profileData)
);

const batchOperation = new CompositeCommand('Batch User Operations')
  .add(createUserCommand)
  .add(updateProfileCommand);

const commandQueue = new CommandQueue(3);

// Execute commands
commandQueue.enqueue(batchOperation).subscribe({
  next: results => console.log('Batch completed:', results),
  error: error => console.error('Batch failed:', error)
});
```

## Advanced Composition Patterns

### 1. Pipeline Pattern for Data Processing

```typescript
// ✅ Pipeline Pattern
export interface PipelineStage<TInput, TOutput> {
  name: string;
  process(input: Observable<TInput>): Observable<TOutput>;
  canProcess?(input: TInput): boolean;
}

export class DataPipeline<TInput, TOutput> {
  private stages: PipelineStage<any, any>[] = [];
  
  constructor(private name: string) {}
  
  addStage<TNext>(stage: PipelineStage<TOutput, TNext>): DataPipeline<TInput, TNext> {
    this.stages.push(stage);
    return this as any;
  }
  
  process(input: Observable<TInput>): Observable<TOutput> {
    return this.stages.reduce(
      (stream, stage) => {
        return stream.pipe(
          tap(() => console.log(`Processing stage: ${stage.name}`)),
          switchMap(data => stage.process(of(data))),
          catchError(error => {
            console.error(`Stage ${stage.name} failed:`, error);
            return throwError(() => new PipelineError(stage.name, error));
          })
        );
      },
      input
    ) as Observable<TOutput>;
  }
  
  processParallel(input: Observable<TInput>): Observable<TOutput> {
    // Process independent stages in parallel
    const parallelStages = this.getParallelStages();
    const sequentialStages = this.getSequentialStages();
    
    let result$ = input;
    
    // Execute parallel stages
    if (parallelStages.length > 0) {
      result$ = result$.pipe(
        switchMap(data => 
          combineLatest(
            parallelStages.map(stage => stage.process(of(data)))
          ).pipe(
            map(results => this.mergeParallelResults(data, results))
          )
        )
      );
    }
    
    // Execute sequential stages
    result$ = sequentialStages.reduce(
      (stream, stage) => stream.pipe(
        switchMap(data => stage.process(of(data)))
      ),
      result$
    );
    
    return result$ as Observable<TOutput>;
  }
}

// Example pipeline stages
export class ValidationStage implements PipelineStage<RawData, ValidatedData> {
  name = 'Validation';
  
  process(input: Observable<RawData>): Observable<ValidatedData> {
    return input.pipe(
      map(data => this.validate(data)),
      filter(data => data.isValid),
      map(data => data.validatedData)
    );
  }
  
  private validate(data: RawData): { isValid: boolean; validatedData: ValidatedData } {
    // Validation logic
    return {
      isValid: this.isDataValid(data),
      validatedData: this.transformToValidated(data)
    };
  }
}

export class EnrichmentStage implements PipelineStage<ValidatedData, EnrichedData> {
  name = 'Enrichment';
  
  constructor(private enrichmentService: EnrichmentService) {}
  
  process(input: Observable<ValidatedData>): Observable<EnrichedData> {
    return input.pipe(
      mergeMap(data => 
        this.enrichmentService.enrich(data).pipe(
          map(enrichment => ({ ...data, enrichment }))
        ),
        5 // Limit concurrent enrichments
      )
    );
  }
}
```

### 2. Event Sourcing Pattern

```typescript
// ✅ Event Sourcing Pattern
export interface DomainEvent {
  id: string;
  type: string;
  aggregateId: string;
  aggregateType: string;
  version: number;
  timestamp: Date;
  data: any;
  metadata?: any;
}

export abstract class EventSourcedAggregate {
  private events: DomainEvent[] = [];
  private version = 0;
  
  constructor(protected id: string) {}
  
  // Apply events to rebuild state
  replayEvents(events: DomainEvent[]): void {
    events.forEach(event => {
      this.applyEvent(event);
      this.version = event.version;
    });
  }
  
  // Get uncommitted events
  getUncommittedEvents(): DomainEvent[] {
    return [...this.events];
  }
  
  // Mark events as committed
  markEventsAsCommitted(): void {
    this.events = [];
  }
  
  // Raise new event
  protected raiseEvent(eventType: string, data: any, metadata?: any): void {
    const event: DomainEvent = {
      id: this.generateEventId(),
      type: eventType,
      aggregateId: this.id,
      aggregateType: this.constructor.name,
      version: this.version + 1,
      timestamp: new Date(),
      data,
      metadata
    };
    
    this.events.push(event);
    this.applyEvent(event);
    this.version = event.version;
  }
  
  protected abstract applyEvent(event: DomainEvent): void;
  
  private generateEventId(): string {
    return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
}

// Event store
export class EventStore {
  private events$ = new Subject<DomainEvent>();
  private eventStorage = new Map<string, DomainEvent[]>();
  
  // Save events
  saveEvents(
    aggregateId: string, 
    events: DomainEvent[], 
    expectedVersion: number
  ): Observable<void> {
    return defer(() => {
      const existingEvents = this.eventStorage.get(aggregateId) || [];
      
      // Optimistic concurrency check
      if (existingEvents.length !== expectedVersion) {
        return throwError(() => new ConcurrencyError(expectedVersion, existingEvents.length));
      }
      
      // Append events
      const allEvents = [...existingEvents, ...events];
      this.eventStorage.set(aggregateId, allEvents);
      
      // Publish events
      events.forEach(event => this.events$.next(event));
      
      return of(void 0);
    });
  }
  
  // Get events for aggregate
  getEvents(aggregateId: string, fromVersion?: number): Observable<DomainEvent[]> {
    return defer(() => {
      const events = this.eventStorage.get(aggregateId) || [];
      const filteredEvents = fromVersion 
        ? events.filter(e => e.version > fromVersion)
        : events;
      
      return of(filteredEvents);
    });
  }
  
  // Subscribe to all events
  getAllEvents(): Observable<DomainEvent> {
    return this.events$.asObservable();
  }
  
  // Subscribe to events by type
  getEventsByType(eventType: string): Observable<DomainEvent> {
    return this.events$.pipe(
      filter(event => event.type === eventType)
    );
  }
  
  // Subscribe to events by aggregate
  getEventsForAggregate(aggregateId: string): Observable<DomainEvent> {
    return this.events$.pipe(
      filter(event => event.aggregateId === aggregateId)
    );
  }
}

// Example aggregate
export class UserAggregate extends EventSourcedAggregate {
  private email = '';
  private name = '';
  private isActive = false;
  
  static create(id: string, email: string, name: string): UserAggregate {
    const user = new UserAggregate(id);
    user.raiseEvent('UserCreated', { email, name });
    return user;
  }
  
  updateEmail(newEmail: string): void {
    if (this.email !== newEmail) {
      this.raiseEvent('EmailUpdated', { 
        oldEmail: this.email, 
        newEmail 
      });
    }
  }
  
  activate(): void {
    if (!this.isActive) {
      this.raiseEvent('UserActivated', {});
    }
  }
  
  deactivate(): void {
    if (this.isActive) {
      this.raiseEvent('UserDeactivated', {});
    }
  }
  
  protected applyEvent(event: DomainEvent): void {
    switch (event.type) {
      case 'UserCreated':
        this.email = event.data.email;
        this.name = event.data.name;
        this.isActive = true;
        break;
        
      case 'EmailUpdated':
        this.email = event.data.newEmail;
        break;
        
      case 'UserActivated':
        this.isActive = true;
        break;
        
      case 'UserDeactivated':
        this.isActive = false;
        break;
    }
  }
  
  // Getters
  getEmail(): string { return this.email; }
  getName(): string { return this.name; }
  getIsActive(): boolean { return this.isActive; }
}
```

## Pattern Composition and Best Practices

### 1. Combining Multiple Patterns

```typescript
// ✅ Pattern Composition Example
export class AdvancedDataService {
  private eventBus = new EventBus();
  private commandQueue = new CommandQueue(3);
  private cache = new Map<string, Observable<any>>();
  
  constructor(
    private apiService: ApiService,
    private eventStore: EventStore
  ) {
    this.setupEventHandlers();
  }
  
  // Combine Factory + Strategy + Observer patterns
  createDataStream<T>(
    config: StreamConfig<T>
  ): Observable<T> {
    // Factory pattern for stream creation
    const baseStream$ = ObservableFactory.createDataStream<T>(
      config.source, 
      config.options
    );
    
    // Strategy pattern for processing
    const processedStream$ = new ResilientObservable(baseStream$)
      .withRetryStrategy(config.retryStrategy)
      .withErrorStrategy(config.errorStrategy)
      .build();
    
    // Observer pattern for notifications
    const enhancedStream$ = processedStream$.pipe(
      tap(data => this.eventBus.emit('data.received', data)),
      catchError(error => {
        this.eventBus.emit('data.error', error);
        return throwError(() => error);
      }),
      finalize(() => this.eventBus.emit('stream.completed', config.id))
    );
    
    // Decorator pattern for additional features
    return ObservableDecoratorFactory.compose(
      enhancedStream$,
      source => ObservableDecoratorFactory.withLogging(source, this.logger, config.id),
      source => ObservableDecoratorFactory.withMetrics(source, this.metrics, config.operation),
      source => ObservableDecoratorFactory.withCache(source, () => config.cacheKey, config.cacheTtl)
    );
  }
  
  // Combine Command + Event Sourcing patterns
  executeBusinessOperation<T>(
    operation: BusinessOperation<T>
  ): Observable<T> {
    // Create command
    const command = new ApiCommand(
      operation.description,
      () => this.executeOperation(operation),
      operation.undoOperation ? () => operation.undoOperation!() : undefined
    );
    
    // Execute through command queue
    return this.commandQueue.enqueue(command).pipe(
      // Store events for audit trail
      tap(result => this.storeOperationEvents(operation, result)),
      // Publish domain events
      tap(result => this.publishDomainEvents(operation, result)),
      // Handle side effects
      switchMap(result => this.handleSideEffects(operation, result))
    );
  }
  
  private setupEventHandlers(): void {
    // React to domain events
    this.eventBus.on<DomainEvent>('domain.event').subscribe(event => {
      this.eventStore.saveEvents(event.aggregateId, [event], event.version - 1);
    });
    
    // Handle cache invalidation
    this.eventBus.on<CacheInvalidationEvent>('cache.invalidate').subscribe(event => {
      this.cache.delete(event.key);
    });
  }
}

// Pattern configuration
export interface PatternConfig {
  patterns: {
    factory: FactoryConfig;
    strategy: StrategyConfig;
    observer: ObserverConfig;
    decorator: DecoratorConfig;
    command: CommandConfig;
  };
  
  features: {
    caching: boolean;
    metrics: boolean;
    logging: boolean;
    eventSourcing: boolean;
    resilience: boolean;
  };
}

// Pattern registry for discoverability
export class PatternRegistry {
  private patterns = new Map<string, PatternMetadata>();
  
  register<T>(
    name: string, 
    pattern: PatternFactory<T>, 
    metadata: PatternMetadata
  ): void {
    this.patterns.set(name, {
      ...metadata,
      factory: pattern,
      registeredAt: new Date()
    });
  }
  
  get<T>(name: string): PatternFactory<T> | undefined {
    const metadata = this.patterns.get(name);
    return metadata?.factory as PatternFactory<T>;
  }
  
  list(): PatternMetadata[] {
    return Array.from(this.patterns.values());
  }
  
  findByCategory(category: PatternCategory): PatternMetadata[] {
    return this.list().filter(p => p.category === category);
  }
  
  findByComplexity(complexity: PatternComplexity): PatternMetadata[] {
    return this.list().filter(p => p.complexity === complexity);
  }
}
```

## Summary

This comprehensive collection covers essential RxJS design patterns:

- ✅ **Creational Patterns**: Factory, Builder, and Singleton patterns for observable creation
- ✅ **Structural Patterns**: Adapter, Decorator, and Facade patterns for observable composition
- ✅ **Behavioral Patterns**: Observer, Strategy, and Command patterns for reactive behavior
- ✅ **Advanced Patterns**: Pipeline, Event Sourcing, and composition patterns for complex scenarios

These patterns provide reusable solutions for common reactive programming challenges and enable the creation of maintainable, scalable RxJS applications.

## Next Steps

Next, we'll explore the **Future of RxJS & Upcoming Features**, covering emerging trends, new features, and the evolution of reactive programming.
