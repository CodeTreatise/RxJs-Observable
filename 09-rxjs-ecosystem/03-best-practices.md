# RxJS Best Practices & Guidelines

## Introduction

This lesson establishes comprehensive best practices and guidelines for writing maintainable, performant, and reliable RxJS code. We'll cover coding standards, architectural patterns, performance optimization, error handling, testing strategies, and team guidelines.

## Learning Objectives

- Master RxJS coding standards and conventions
- Understand performance optimization techniques
- Learn proper error handling and resource management
- Establish testing and debugging strategies
- Create team guidelines and code review standards
- Implement architectural patterns for scalable reactive systems

## Core RxJS Best Practices

### 1. Observable Creation and Naming

```typescript
// ✅ GOOD: Clear naming conventions
// Use $ suffix for observables
const userProfile$ = this.userService.getCurrentUser();
const searchResults$ = this.searchService.search(query);
const isLoading$ = new BehaviorSubject<boolean>(false);

// Use descriptive names that indicate the data type and purpose
const temperatureSensorData$ = this.sensorService.getTemperatureData();
const validationErrors$ = this.formValidationService.getErrors();
const authenticatedUser$ = this.authService.getAuthenticatedUser();

// ❌ BAD: Poor naming
const data = this.service.getData(); // No $ suffix, unclear purpose
const obs = new Subject(); // Unclear what it contains
const stream = this.api.fetch(); // Generic name

// ✅ GOOD: Proper observable creation patterns
class UserService {
  private readonly apiUrl = 'https://api.example.com';
  
  // Use readonly for observables that shouldn't be reassigned
  readonly users$ = this.http.get<User[]>(`${this.apiUrl}/users`).pipe(
    shareReplay(1) // Cache the result
  );
  
  // Private subjects with public observables
  private readonly userUpdates$ = new Subject<User>();
  readonly userUpdates = this.userUpdates$.asObservable();
  
  // Proper factory methods
  getUserById(id: string): Observable<User> {
    return this.http.get<User>(`${this.apiUrl}/users/${id}`).pipe(
      catchError(error => {
        console.error(`Failed to fetch user ${id}:`, error);
        return throwError(() => new UserNotFoundError(id));
      })
    );
  }
  
  // ❌ BAD: Exposing subjects directly
  // userUpdates = new Subject<User>(); // Can be abused by external code
}
```

### 2. Operator Usage and Chaining

```typescript
// ✅ GOOD: Proper operator chaining and formatting
const searchResults$ = this.searchInput$.pipe(
  // One operator per line for readability
  debounceTime(300),
  distinctUntilChanged(),
  filter(query => query.length >= 2),
  tap(query => console.log('Searching for:', query)),
  switchMap(query => 
    this.searchService.search(query).pipe(
      catchError(error => {
        console.error('Search failed:', error);
        return of([]); // Return empty results on error
      })
    )
  ),
  shareReplay(1)
);

// ✅ GOOD: Complex transformations with clear intermediate steps
const processedData$ = this.rawData$.pipe(
  // Data validation
  filter(data => this.isValidData(data)),
  
  // Data transformation
  map(data => this.normalizeData(data)),
  
  // Enrichment
  switchMap(data => 
    this.enrichmentService.enrichData(data).pipe(
      startWith(data) // Show original data while enriching
    )
  ),
  
  // Caching
  shareReplay({ bufferSize: 1, refCount: true })
);

// ❌ BAD: Poor operator usage
const badExample$ = this.data$.pipe(
  map(x => x), // Unnecessary identity mapping
  filter(() => true), // Unnecessary filter
  switchMap(x => of(x)), // Unnecessary switchMap
  tap(() => {}), // Empty tap
  distinctUntilChanged(),
  distinctUntilChanged() // Duplicate operators
);

// ✅ GOOD: Custom operators for reusability
export function retryWithBackoff(
  maxRetries: number = 3,
  initialDelay: number = 1000,
  delayFactor: number = 2
): <T>(source: Observable<T>) => Observable<T> {
  return <T>(source: Observable<T>) => source.pipe(
    retryWhen(errors => 
      errors.pipe(
        scan((retryCount, error) => {
          if (retryCount >= maxRetries) {
            throw error;
          }
          return retryCount + 1;
        }, 0),
        delayWhen(retryCount => 
          timer(initialDelay * Math.pow(delayFactor, retryCount - 1))
        )
      )
    )
  );
}

// Usage
const resilientApiCall$ = this.apiService.getData().pipe(
  retryWithBackoff(3, 1000, 2)
);
```

### 3. Memory Management and Subscription Handling

```typescript
// ✅ GOOD: Automatic subscription management in Angular
@Component({
  template: `<div>{{ data$ | async }}</div>`
})
export class GoodComponent implements OnDestroy {
  private readonly destroy$ = new Subject<void>();
  
  readonly data$ = this.dataService.getData().pipe(
    takeUntil(this.destroy$),
    shareReplay(1)
  );
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// ✅ GOOD: Subscription management utility
export class SubscriptionManager {
  private subscriptions = new Set<Subscription>();
  
  add(subscription: Subscription): void {
    this.subscriptions.add(subscription);
  }
  
  addObservable<T>(observable$: Observable<T>, observer: Partial<Observer<T>>): void {
    const subscription = observable$.subscribe(observer);
    this.add(subscription);
  }
  
  unsubscribeAll(): void {
    this.subscriptions.forEach(sub => sub.unsubscribe());
    this.subscriptions.clear();
  }
  
  get activeCount(): number {
    return Array.from(this.subscriptions).filter(sub => !sub.closed).length;
  }
}

// ✅ GOOD: Resource cleanup patterns
export class ResourceCleanupExample {
  private subscriptionManager = new SubscriptionManager();
  
  initialize(): void {
    // Pattern 1: Use takeUntil for component lifecycle
    const componentDestroy$ = new Subject<void>();
    
    this.dataService.getData().pipe(
      takeUntil(componentDestroy$)
    ).subscribe(data => this.processData(data));
    
    // Pattern 2: Use finalize for cleanup
    this.fileService.uploadFile(file).pipe(
      finalize(() => this.hideProgressIndicator())
    ).subscribe();
    
    // Pattern 3: Manual subscription management
    this.subscriptionManager.addObservable(
      this.userService.getCurrentUser(),
      {
        next: user => this.updateUserProfile(user),
        error: error => this.handleError(error)
      }
    );
  }
  
  cleanup(): void {
    this.subscriptionManager.unsubscribeAll();
  }
}

// ❌ BAD: Memory leaks and poor subscription management
export class BadComponent {
  ngOnInit(): void {
    // Memory leak - no unsubscription
    this.dataService.getData().subscribe(data => {
      this.processData(data);
    });
    
    // Multiple subscriptions without management
    interval(1000).subscribe();
    fromEvent(window, 'resize').subscribe();
    this.userService.getUsers().subscribe();
  }
  
  // Missing ngOnDestroy - subscriptions never cleaned up
}
```

## Error Handling Best Practices

### 1. Comprehensive Error Handling Strategies

```typescript
// ✅ GOOD: Layered error handling approach
export enum ErrorSeverity {
  LOW = 'low',
  MEDIUM = 'medium',
  HIGH = 'high',
  CRITICAL = 'critical'
}

export interface AppError {
  code: string;
  message: string;
  severity: ErrorSeverity;
  context?: any;
  timestamp: Date;
  recoverable: boolean;
}

export class ErrorHandlingService {
  private errorLog$ = new Subject<AppError>();
  
  // Global error handling
  handleError(error: any, context?: string): Observable<never> {
    const appError: AppError = {
      code: error.code || 'UNKNOWN_ERROR',
      message: error.message || 'An unexpected error occurred',
      severity: this.determineSeverity(error),
      context,
      timestamp: new Date(),
      recoverable: this.isRecoverable(error)
    };
    
    // Log error
    this.errorLog$.next(appError);
    
    // Send to monitoring service
    this.sendToMonitoring(appError);
    
    // Show user notification if appropriate
    if (appError.severity !== ErrorSeverity.LOW) {
      this.showUserNotification(appError);
    }
    
    return throwError(() => appError);
  }
  
  // Retry strategies
  createRetryStrategy(config: RetryConfig) {
    return (errors: Observable<any>) => errors.pipe(
      scan((retryCount, error) => {
        if (retryCount >= config.maxRetries) {
          throw error;
        }
        
        if (!this.shouldRetry(error, config)) {
          throw error;
        }
        
        return retryCount + 1;
      }, 0),
      delayWhen(retryCount => {
        const delay = config.baseDelay * Math.pow(config.backoffFactor, retryCount - 1);
        return timer(Math.min(delay, config.maxDelay));
      })
    );
  }
  
  private shouldRetry(error: any, config: RetryConfig): boolean {
    // Don't retry on authentication errors
    if (error.status === 401 || error.status === 403) {
      return false;
    }
    
    // Don't retry on client errors (4xx)
    if (error.status >= 400 && error.status < 500) {
      return false;
    }
    
    // Retry on network errors and server errors
    return true;
  }
}

// ✅ GOOD: Service-level error handling
export class ApiService {
  constructor(
    private http: HttpClient,
    private errorHandler: ErrorHandlingService
  ) {}
  
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>('/api/users').pipe(
      retryWhen(this.errorHandler.createRetryStrategy({
        maxRetries: 3,
        baseDelay: 1000,
        backoffFactor: 2,
        maxDelay: 10000
      })),
      catchError(error => 
        this.errorHandler.handleError(error, 'ApiService.getUsers')
      )
    );
  }
  
  createUser(user: CreateUserRequest): Observable<User> {
    return this.http.post<User>('/api/users', user).pipe(
      catchError(error => {
        if (error.status === 409) {
          // Handle specific error case
          return throwError(() => new UserAlreadyExistsError(user.email));
        }
        return this.errorHandler.handleError(error, 'ApiService.createUser');
      })
    );
  }
}

// ✅ GOOD: Component-level error handling
export class UserListComponent {
  users$ = this.userService.getUsers().pipe(
    catchError(error => {
      this.showErrorMessage('Failed to load users');
      return of([]); // Return empty array to keep UI functional
    })
  );
  
  deleteUser(userId: string): void {
    this.userService.deleteUser(userId).pipe(
      catchError(error => {
        if (error instanceof UserHasActiveOrdersError) {
          this.showConfirmationDialog(
            'User has active orders. Delete anyway?',
            () => this.forceDeleteUser(userId)
          );
          return EMPTY;
        }
        
        this.showErrorMessage('Failed to delete user');
        return EMPTY;
      })
    ).subscribe();
  }
}
```

### 2. Custom Error Types and Recovery

```typescript
// ✅ GOOD: Custom error hierarchy
export abstract class DomainError extends Error {
  abstract readonly code: string;
  abstract readonly severity: ErrorSeverity;
  abstract readonly recoverable: boolean;
  
  constructor(message: string, public readonly context?: any) {
    super(message);
    this.name = this.constructor.name;
  }
}

export class NetworkError extends DomainError {
  readonly code = 'NETWORK_ERROR';
  readonly severity = ErrorSeverity.MEDIUM;
  readonly recoverable = true;
}

export class ValidationError extends DomainError {
  readonly code = 'VALIDATION_ERROR';
  readonly severity = ErrorSeverity.LOW;
  readonly recoverable = true;
  
  constructor(
    message: string,
    public readonly field: string,
    public readonly violations: string[]
  ) {
    super(message, { field, violations });
  }
}

export class AuthenticationError extends DomainError {
  readonly code = 'AUTHENTICATION_ERROR';
  readonly severity = ErrorSeverity.HIGH;
  readonly recoverable = false;
}

// ✅ GOOD: Error recovery operators
export function recoverWithFallback<T>(
  fallbackValue: T,
  predicate?: (error: any) => boolean
): MonoTypeOperatorFunction<T> {
  return catchError(error => {
    if (predicate && !predicate(error)) {
      return throwError(() => error);
    }
    
    console.warn('Recovering with fallback value:', fallbackValue);
    return of(fallbackValue);
  });
}

export function recoverWithRetry<T>(
  maxRetries: number,
  retryPredicate?: (error: any) => boolean
): MonoTypeOperatorFunction<T> {
  return (source: Observable<T>) => source.pipe(
    retryWhen(errors => errors.pipe(
      scan((retryCount, error) => {
        if (retryCount >= maxRetries) {
          throw error;
        }
        
        if (retryPredicate && !retryPredicate(error)) {
          throw error;
        }
        
        return retryCount + 1;
      }, 0),
      delay(1000)
    ))
  );
}

// Usage examples
const resilientDataFetch$ = this.apiService.getData().pipe(
  recoverWithRetry(3, error => error instanceof NetworkError),
  recoverWithFallback([], error => error instanceof ValidationError)
);
```

## Performance Optimization Guidelines

### 1. Operator Selection and Optimization

```typescript
// ✅ GOOD: Choose appropriate operators for performance
export class PerformanceOptimizedService {
  
  // Use shareReplay for expensive computations
  readonly expensiveComputation$ = this.dataService.getRawData().pipe(
    map(data => this.performExpensiveCalculation(data)),
    shareReplay({ bufferSize: 1, refCount: true })
  );
  
  // Use distinctUntilChanged with custom comparators
  readonly optimizedUserData$ = this.userService.getUser().pipe(
    distinctUntilChanged((prev, curr) => {
      // Custom equality check for better performance
      return prev.id === curr.id && prev.lastModified === curr.lastModified;
    }),
    shareReplay(1)
  );
  
  // Use switchMap for cancellable operations
  search(query$: Observable<string>): Observable<SearchResult[]> {
    return query$.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      filter(query => query.length >= 2),
      switchMap(query => 
        this.searchService.search(query).pipe(
          catchError(() => of([]))
        )
      )
    );
  }
  
  // Use mergeMap with concurrency limit for parallel operations
  processItems(items: Observable<Item[]>): Observable<ProcessedItem> {
    return items.pipe(
      switchMap(itemList => from(itemList)),
      mergeMap(
        item => this.processItem(item),
        4 // Limit concurrency to 4 parallel operations
      )
    );
  }
  
  // ❌ BAD: Performance anti-patterns
  badPerformanceExample(): Observable<any> {
    return this.data$.pipe(
      // Don't use nested subscriptions
      switchMap(data => {
        // This creates a memory leak
        this.otherService.process(data).subscribe();
        return of(data);
      }),
      
      // Don't use unnecessary operators
      map(x => x), // Identity mapping
      filter(() => true), // Always true filter
      
      // Don't chain expensive operations without caching
      map(data => this.expensiveOperation(data)),
      map(data => this.anotherExpensiveOperation(data)),
      // Should use shareReplay to cache expensive results
    );
  }
}

// ✅ GOOD: Performance monitoring
export function measurePerformance<T>(
  operationName: string
): MonoTypeOperatorFunction<T> {
  return (source: Observable<T>) => {
    return new Observable<T>(subscriber => {
      const startTime = performance.now();
      let itemCount = 0;
      
      return source.subscribe({
        next: value => {
          itemCount++;
          subscriber.next(value);
        },
        error: error => {
          const duration = performance.now() - startTime;
          console.log(`${operationName} failed after ${duration}ms, ${itemCount} items processed`);
          subscriber.error(error);
        },
        complete: () => {
          const duration = performance.now() - startTime;
          console.log(`${operationName} completed in ${duration}ms, ${itemCount} items processed`);
          subscriber.complete();
        }
      });
    });
  };
}
```

### 2. Memory Management and Bundle Size

```typescript
// ✅ GOOD: Bundle size optimization
// Import only what you need
import { map, filter, debounceTime } from 'rxjs/operators';
import { Observable, Subject } from 'rxjs';

// ❌ BAD: Importing entire library
// import * as rxjs from 'rxjs';
// import 'rxjs/add/operator/map'; // Deprecated patch imports

// ✅ GOOD: Lazy loading for large operators
export async function loadAdvancedOperators() {
  const { audit, buffer, window } = await import('rxjs/operators');
  return { audit, buffer, window };
}

// ✅ GOOD: Memory-efficient patterns
export class MemoryEfficientService {
  private readonly cache = new Map<string, Observable<any>>();
  private readonly cacheSize = 100;
  
  // Implement LRU cache for observables
  getCachedData(key: string): Observable<any> {
    if (this.cache.has(key)) {
      // Move to end (most recently used)
      const value = this.cache.get(key)!;
      this.cache.delete(key);
      this.cache.set(key, value);
      return value;
    }
    
    // Create new observable
    const data$ = this.fetchData(key).pipe(
      shareReplay({ bufferSize: 1, refCount: true })
    );
    
    // Add to cache
    this.cache.set(key, data$);
    
    // Evict oldest if cache is full
    if (this.cache.size > this.cacheSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    
    return data$;
  }
  
  // Use weak references for temporary data
  private weakCache = new WeakMap<object, Observable<any>>();
  
  getWeakCachedData(keyObject: object): Observable<any> {
    let data$ = this.weakCache.get(keyObject);
    
    if (!data$) {
      data$ = this.fetchData(keyObject.toString()).pipe(
        shareReplay(1)
      );
      this.weakCache.set(keyObject, data$);
    }
    
    return data$;
  }
}

// ✅ GOOD: Resource pooling
export class ConnectionPool {
  private availableConnections: Observable<Connection>[] = [];
  private readonly maxConnections = 10;
  
  getConnection(): Observable<Connection> {
    if (this.availableConnections.length > 0) {
      return this.availableConnections.pop()!;
    }
    
    if (this.getTotalConnections() < this.maxConnections) {
      return this.createConnection();
    }
    
    // Wait for available connection
    return this.waitForAvailableConnection();
  }
  
  releaseConnection(connection$: Observable<Connection>): void {
    this.availableConnections.push(connection$);
  }
  
  private createConnection(): Observable<Connection> {
    return new Observable<Connection>(subscriber => {
      const connection = new Connection();
      subscriber.next(connection);
      
      return () => {
        connection.close();
      };
    }).pipe(
      shareReplay(1)
    );
  }
}
```

## Testing Best Practices

### 1. Comprehensive Testing Strategies

```typescript
// ✅ GOOD: Marble testing for reactive streams
import { TestScheduler } from 'rxjs/testing';
import { cold, hot } from 'jasmine-marbles';

describe('UserService', () => {
  let testScheduler: TestScheduler;
  let userService: UserService;
  let httpMock: jasmine.SpyObj<HttpClient>;
  
  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
    
    httpMock = jasmine.createSpyObj('HttpClient', ['get', 'post', 'put', 'delete']);
    userService = new UserService(httpMock);
  });
  
  it('should debounce search queries', () => {
    testScheduler.run(({ hot, cold, expectObservable }) => {
      const searchInput$ = hot('  --a-b-c---|');
      const expected = cold('     ------c---|');
      
      const result$ = searchInput$.pipe(
        debounceTime(50, testScheduler),
        distinctUntilChanged()
      );
      
      expectObservable(result$).toBe('------c---|');
    });
  });
  
  it('should handle API errors with retry', () => {
    testScheduler.run(({ cold, expectObservable }) => {
      const httpResponse$ = cold('#', {}, new Error('Network error'));
      httpMock.get.and.returnValue(httpResponse$);
      
      const result$ = userService.getUsers();
      const expected = cold('     #', {}, jasmine.any(Error));
      
      expectObservable(result$).toBe(expected);
    });
  });
  
  it('should cache successful responses', () => {
    testScheduler.run(({ cold, expectObservable }) => {
      const users = [{ id: 1, name: 'John' }];
      const httpResponse$ = cold('--a|', { a: users });
      httpMock.get.and.returnValue(httpResponse$);
      
      const firstCall$ = userService.getUsers();
      const secondCall$ = userService.getUsers();
      
      // Both calls should receive the same cached result
      expectObservable(firstCall$).toBe('--a|', { a: users });
      expectObservable(secondCall$).toBe('(a|)', { a: users }); // Synchronous from cache
      
      expect(httpMock.get).toHaveBeenCalledTimes(1);
    });
  });
});

// ✅ GOOD: Integration testing with real observables
describe('SearchComponent Integration', () => {
  let component: SearchComponent;
  let searchService: jasmine.SpyObj<SearchService>;
  
  beforeEach(() => {
    searchService = jasmine.createSpyObj('SearchService', ['search']);
    component = new SearchComponent(searchService);
  });
  
  it('should perform search with debouncing', fakeAsync(() => {
    const searchResults = [{ id: 1, title: 'Result 1' }];
    searchService.search.and.returnValue(of(searchResults));
    
    const results: any[] = [];
    component.searchResults$.subscribe(result => results.push(result));
    
    // Simulate rapid typing
    component.onSearchInput('a');
    tick(100);
    component.onSearchInput('ab');
    tick(100);
    component.onSearchInput('abc');
    tick(300); // Wait for debounce
    
    expect(searchService.search).toHaveBeenCalledTimes(1);
    expect(searchService.search).toHaveBeenCalledWith('abc');
    expect(results).toEqual([[], searchResults]);
  }));
});

// ✅ GOOD: Custom testing utilities
export class RxJSTestUtils {
  static expectSequence<T>(
    observable$: Observable<T>,
    expectedValues: T[],
    done: jasmine.DoneCallback
  ): void {
    const actualValues: T[] = [];
    
    observable$.pipe(
      take(expectedValues.length)
    ).subscribe({
      next: value => actualValues.push(value),
      error: done.fail,
      complete: () => {
        expect(actualValues).toEqual(expectedValues);
        done();
      }
    });
  }
  
  static expectError<T>(
    observable$: Observable<T>,
    expectedErrorType: any,
    done: jasmine.DoneCallback
  ): void {
    observable$.subscribe({
      next: () => done.fail('Expected error but got value'),
      error: error => {
        expect(error).toBeInstanceOf(expectedErrorType);
        done();
      },
      complete: () => done.fail('Expected error but completed successfully')
    });
  }
  
  static createMockObservable<T>(
    values: T[],
    delayMs: number = 0
  ): Observable<T> {
    return from(values).pipe(
      concatMap((value, index) => 
        of(value).pipe(delay(index * delayMs))
      )
    );
  }
}
```

### 2. Test Data Management

```typescript
// ✅ GOOD: Test data factories and builders
export class TestDataBuilder {
  static user(overrides: Partial<User> = {}): User {
    return {
      id: 'test-user-id',
      email: 'test@example.com',
      name: 'Test User',
      createdAt: new Date('2023-01-01'),
      isActive: true,
      ...overrides
    };
  }
  
  static userList(count: number = 3): User[] {
    return Array.from({ length: count }, (_, index) => 
      this.user({
        id: `user-${index}`,
        email: `user${index}@example.com`,
        name: `User ${index}`
      })
    );
  }
  
  static apiResponse<T>(data: T, overrides: Partial<ApiResponse<T>> = {}): ApiResponse<T> {
    return {
      data,
      status: 'success',
      timestamp: new Date().toISOString(),
      ...overrides
    };
  }
}

// ✅ GOOD: Mock observables with realistic behavior
export class MockObservableFactory {
  static successfulResponse<T>(data: T, delayMs: number = 100): Observable<T> {
    return of(data).pipe(delay(delayMs));
  }
  
  static errorResponse(error: any, delayMs: number = 100): Observable<never> {
    return throwError(() => error).pipe(delay(delayMs));
  }
  
  static slowResponse<T>(data: T, delayMs: number = 5000): Observable<T> {
    return of(data).pipe(delay(delayMs));
  }
  
  static intermittentResponse<T>(
    data: T,
    successRate: number = 0.8
  ): Observable<T> {
    return defer(() => 
      Math.random() < successRate
        ? of(data)
        : throwError(() => new Error('Random failure'))
    );
  }
  
  static paginatedResponse<T>(
    allData: T[],
    pageSize: number = 10
  ): (page: number) => Observable<PaginatedResponse<T>> {
    return (page: number) => {
      const startIndex = (page - 1) * pageSize;
      const endIndex = startIndex + pageSize;
      const pageData = allData.slice(startIndex, endIndex);
      
      return of({
        data: pageData,
        page,
        pageSize,
        total: allData.length,
        hasNext: endIndex < allData.length,
        hasPrevious: page > 1
      }).pipe(delay(100));
    };
  }
}
```

## Architectural Patterns and Guidelines

### 1. Service Design Patterns

```typescript
// ✅ GOOD: Repository pattern with reactive streams
export abstract class BaseRepository<T, ID> {
  protected abstract apiUrl: string;
  
  constructor(protected http: HttpClient) {}
  
  // Standard CRUD operations
  findAll(): Observable<T[]> {
    return this.http.get<T[]>(this.apiUrl).pipe(
      shareReplay({ bufferSize: 1, refCount: true })
    );
  }
  
  findById(id: ID): Observable<T | null> {
    return this.http.get<T>(`${this.apiUrl}/${id}`).pipe(
      catchError(error => {
        if (error.status === 404) {
          return of(null);
        }
        return throwError(() => error);
      })
    );
  }
  
  create(entity: Omit<T, 'id'>): Observable<T> {
    return this.http.post<T>(this.apiUrl, entity);
  }
  
  update(id: ID, entity: Partial<T>): Observable<T> {
    return this.http.put<T>(`${this.apiUrl}/${id}`, entity);
  }
  
  delete(id: ID): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`);
  }
}

// ✅ GOOD: Facade pattern for complex operations
export class UserFacadeService {
  constructor(
    private userRepository: UserRepository,
    private profileService: ProfileService,
    private notificationService: NotificationService,
    private auditService: AuditService
  ) {}
  
  // Complex operation combining multiple services
  createUserWithProfile(
    userData: CreateUserRequest,
    profileData: CreateProfileRequest
  ): Observable<UserWithProfile> {
    return this.userRepository.create(userData).pipe(
      switchMap(user => 
        this.profileService.create(user.id, profileData).pipe(
          map(profile => ({ user, profile }))
        )
      ),
      tap(({ user, profile }) => {
        // Side effects
        this.notificationService.sendWelcomeEmail(user.email);
        this.auditService.logUserCreation(user.id);
      }),
      catchError(error => {
        // Rollback on failure
        return this.handleUserCreationFailure(error);
      })
    );
  }
  
  private handleUserCreationFailure(error: any): Observable<never> {
    // Implement rollback logic
    this.auditService.logUserCreationFailure(error);
    return throwError(() => error);
  }
}

// ✅ GOOD: State management with reactive patterns
export class ReactiveStateManager<T> {
  private readonly state$ = new BehaviorSubject<T>(this.initialState);
  private readonly actions$ = new Subject<StateAction<T>>();
  
  readonly currentState$ = this.state$.asObservable();
  
  constructor(private initialState: T) {
    // Process actions and update state
    this.actions$.pipe(
      scan((state, action) => this.reducer(state, action), this.initialState)
    ).subscribe(this.state$);
  }
  
  dispatch(action: StateAction<T>): void {
    this.actions$.next(action);
  }
  
  select<K extends keyof T>(key: K): Observable<T[K]> {
    return this.state$.pipe(
      map(state => state[key]),
      distinctUntilChanged()
    );
  }
  
  selectComputed<R>(selector: (state: T) => R): Observable<R> {
    return this.state$.pipe(
      map(selector),
      distinctUntilChanged()
    );
  }
  
  private reducer(state: T, action: StateAction<T>): T {
    switch (action.type) {
      case 'UPDATE':
        return { ...state, ...action.payload };
      case 'RESET':
        return this.initialState;
      default:
        return state;
    }
  }
}
```

### 2. Component Architecture Patterns

```typescript
// ✅ GOOD: Smart/Dumb component pattern with reactive data flow
@Component({
  selector: 'app-user-list-container',
  template: `
    <app-user-list
      [users]="users$ | async"
      [loading]="loading$ | async"
      [error]="error$ | async"
      (userSelected)="onUserSelected($event)"
      (refreshRequested)="onRefreshRequested()"
    ></app-user-list>
  `
})
export class UserListContainerComponent {
  private readonly refresh$ = new Subject<void>();
  private readonly userSelected$ = new Subject<User>();
  
  readonly users$ = this.refresh$.pipe(
    startWith(null),
    switchMap(() => this.userService.getUsers()),
    shareReplay(1)
  );
  
  readonly loading$ = merge(
    this.refresh$.pipe(map(() => true)),
    this.users$.pipe(map(() => false))
  ).pipe(distinctUntilChanged());
  
  readonly error$ = this.users$.pipe(
    map(() => null),
    catchError(error => of(error))
  );
  
  constructor(
    private userService: UserService,
    private router: Router
  ) {
    // Handle user selection
    this.userSelected$.subscribe(user => {
      this.router.navigate(['/users', user.id]);
    });
  }
  
  onUserSelected(user: User): void {
    this.userSelected$.next(user);
  }
  
  onRefreshRequested(): void {
    this.refresh$.next();
  }
}

// ✅ GOOD: Reactive form patterns
@Component({
  selector: 'app-user-form',
  template: `
    <form [formGroup]="userForm">
      <input formControlName="email" placeholder="Email">
      <div *ngIf="emailErrors$ | async as errors">
        <span *ngFor="let error of errors">{{ error }}</span>
      </div>
      
      <input formControlName="name" placeholder="Name">
      <button 
        type="submit" 
        [disabled]="!canSubmit$ | async"
        (click)="onSubmit()"
      >
        {{ submitButtonText$ | async }}
      </button>
    </form>
  `
})
export class UserFormComponent {
  readonly userForm = this.fb.group({
    email: ['', [Validators.required, Validators.email]],
    name: ['', [Validators.required, Validators.minLength(2)]]
  });
  
  private readonly submitAttempted$ = new Subject<void>();
  
  readonly emailErrors$ = combineLatest([
    this.userForm.get('email')!.statusChanges.pipe(startWith('INVALID')),
    this.userForm.get('email')!.valueChanges.pipe(startWith('')),
    this.submitAttempted$.pipe(startWith(false), map(() => true))
  ]).pipe(
    map(([status, value, attempted]) => {
      const control = this.userForm.get('email')!;
      if (control.valid || (!attempted && !control.touched)) {
        return [];
      }
      
      const errors: string[] = [];
      if (control.errors?.['required']) errors.push('Email is required');
      if (control.errors?.['email']) errors.push('Invalid email format');
      return errors;
    })
  );
  
  readonly canSubmit$ = this.userForm.statusChanges.pipe(
    startWith(this.userForm.status),
    map(status => status === 'VALID')
  );
  
  readonly submitButtonText$ = this.canSubmit$.pipe(
    map(canSubmit => canSubmit ? 'Submit' : 'Please fix errors')
  );
  
  constructor(private fb: FormBuilder) {}
  
  onSubmit(): void {
    this.submitAttempted$.next();
    
    if (this.userForm.valid) {
      const userData = this.userForm.value as CreateUserRequest;
      // Handle submission
    }
  }
}
```

## Team Guidelines and Code Review

### 1. Code Review Checklist

```typescript
// ✅ RxJS Code Review Checklist

export const rxjsCodeReviewChecklist = {
  subscription_management: [
    'Are all subscriptions properly unsubscribed?',
    'Is takeUntil used correctly for component lifecycle?',
    'Are manual subscriptions avoided where async pipe could be used?'
  ],
  
  operator_usage: [
    'Are operators used appropriately (switchMap vs mergeMap vs concatMap)?',
    'Is shareReplay used for expensive operations?',
    'Are unnecessary operators removed?',
    'Is error handling present where needed?'
  ],
  
  performance: [
    'Are there any potential memory leaks?',
    'Is caching implemented for expensive operations?',
    'Are observables created unnecessarily in templates or loops?',
    'Is distinctUntilChanged used where appropriate?'
  ],
  
  error_handling: [
    'Are errors handled gracefully?',
    'Is fallback behavior implemented?',
    'Are user-friendly error messages provided?',
    'Is retry logic appropriate for the use case?'
  ],
  
  testing: [
    'Are observables properly tested with marble diagrams?',
    'Are error scenarios tested?',
    'Are async operations tested with proper timing?',
    'Are subscriptions tested for proper cleanup?'
  ],
  
  readability: [
    'Are observables named with $ suffix?',
    'Is the code properly formatted and indented?',
    'Are complex operations broken down into smaller functions?',
    'Are comments provided for complex reactive logic?'
  ]
};

// ✅ GOOD: Code review automation
export class RxJSLintRules {
  static rules = {
    'no-exposed-subjects': {
      severity: 'error',
      message: 'Subjects should not be exposed directly. Use asObservable().'
    },
    
    'prefer-async-pipe': {
      severity: 'warning',
      message: 'Consider using async pipe instead of manual subscription.'
    },
    
    'require-takeuntil': {
      severity: 'error',
      message: 'Long-lived subscriptions must use takeUntil for cleanup.'
    },
    
    'no-nested-subscribe': {
      severity: 'error',
      message: 'Avoid nested subscribe calls. Use operators like switchMap.'
    },
    
    'prefer-operators': {
      severity: 'warning',
      message: 'Use RxJS operators instead of imperative code in subscribe.'
    }
  };
  
  static checkCode(code: string): LintResult[] {
    const results: LintResult[] = [];
    
    // Check for exposed subjects
    if (/public\s+\w+\s*:\s*Subject/.test(code)) {
      results.push({
        rule: 'no-exposed-subjects',
        severity: 'error',
        message: this.rules['no-exposed-subjects'].message
      });
    }
    
    // Check for nested subscriptions
    if (/subscribe\s*\([^}]*subscribe/.test(code)) {
      results.push({
        rule: 'no-nested-subscribe',
        severity: 'error',
        message: this.rules['no-nested-subscribe'].message
      });
    }
    
    return results;
  }
}
```

### 2. Team Training and Documentation

```typescript
// ✅ GOOD: Team onboarding materials
export const rxjsTeamGuidelines = {
  beginnerPath: {
    week1: [
      'Learn Observable basics and creation',
      'Master map, filter, and tap operators',
      'Understand subscription and unsubscription'
    ],
    week2: [
      'Learn async pipe usage',
      'Master debounceTime and distinctUntilChanged',
      'Understand switchMap vs mergeMap'
    ],
    week3: [
      'Learn error handling with catchError',
      'Master shareReplay and caching',
      'Understand testing with marble diagrams'
    ],
    week4: [
      'Practice with real project features',
      'Learn debugging techniques',
      'Review and refactor existing code'
    ]
  ],
  
  commonMistakes: [
    {
      mistake: 'Not unsubscribing from observables',
      solution: 'Use takeUntil pattern or async pipe',
      example: `
        // ❌ Bad
        ngOnInit() {
          this.service.getData().subscribe(data => this.data = data);
        }
        
        // ✅ Good
        data$ = this.service.getData();
        // Use: {{ data$ | async }} in template
      `
    },
    {
      mistake: 'Using nested subscriptions',
      solution: 'Use flatMap operators (switchMap, mergeMap, concatMap)',
      example: `
        // ❌ Bad
        this.userService.getUser().subscribe(user => {
          this.orderService.getOrders(user.id).subscribe(orders => {
            this.orders = orders;
          });
        });
        
        // ✅ Good
        this.orders$ = this.userService.getUser().pipe(
          switchMap(user => this.orderService.getOrders(user.id))
        );
      `
    }
  ],
  
  codeTemplates: {
    serviceTemplate: `
      @Injectable({ providedIn: 'root' })
      export class ExampleService {
        private readonly baseUrl = 'https://api.example.com';
        
        constructor(private http: HttpClient) {}
        
        getData(): Observable<Data[]> {
          return this.http.get<Data[]>(\`\${this.baseUrl}/data\`).pipe(
            shareReplay({ bufferSize: 1, refCount: true }),
            catchError(error => {
              console.error('Failed to fetch data:', error);
              return throwError(() => error);
            })
          );
        }
      }
    `,
    
    componentTemplate: `
      @Component({
        template: \`
          <div *ngIf="loading$ | async">Loading...</div>
          <div *ngIf="error$ | async as error">Error: {{ error.message }}</div>
          <div *ngFor="let item of data$ | async">{{ item.name }}</div>
        \`
      })
      export class ExampleComponent implements OnDestroy {
        private destroy$ = new Subject<void>();
        
        data$ = this.service.getData().pipe(
          takeUntil(this.destroy$)
        );
        
        loading$ = this.loadingService.isLoading$;
        error$ = this.errorService.error$;
        
        constructor(private service: ExampleService) {}
        
        ngOnDestroy(): void {
          this.destroy$.next();
          this.destroy$.complete();
        }
      }
    `
  }
};
```

## Security Best Practices

### 1. Secure Observable Patterns

```typescript
// ✅ GOOD: Input validation and sanitization
export class SecureDataService {
  constructor(
    private http: HttpClient,
    private sanitizer: DomSanitizer
  ) {}
  
  // Validate and sanitize input data
  searchUsers(query: string): Observable<User[]> {
    // Input validation
    if (!this.isValidSearchQuery(query)) {
      return throwError(() => new Error('Invalid search query'));
    }
    
    // Sanitize input
    const sanitizedQuery = this.sanitizer.sanitize(SecurityContext.URL, query);
    
    return this.http.get<User[]>(`/api/users/search?q=${encodeURIComponent(sanitizedQuery)}`).pipe(
      map(users => users.map(user => this.sanitizeUser(user))),
      catchError(this.handleSearchError)
    );
  }
  
  private isValidSearchQuery(query: string): boolean {
    // Validate query length and characters
    return query.length >= 2 && 
           query.length <= 100 && 
           /^[a-zA-Z0-9\s\-_.@]+$/.test(query);
  }
  
  private sanitizeUser(user: User): User {
    return {
      ...user,
      // Sanitize HTML content
      bio: this.sanitizer.sanitize(SecurityContext.HTML, user.bio) || '',
      // Remove sensitive fields
      passwordHash: undefined,
      internalNotes: undefined
    };
  }
  
  private handleSearchError = (error: any): Observable<User[]> => {
    // Log error without exposing sensitive information
    console.error('Search failed:', {
      timestamp: new Date().toISOString(),
      error: error.message,
      // Don't log the actual query for privacy
    });
    
    return of([]); // Return empty results on error
  };
}

// ✅ GOOD: Authentication and authorization
export class SecureApiService {
  constructor(
    private http: HttpClient,
    private authService: AuthService
  ) {}
  
  // Ensure authenticated requests
  makeAuthenticatedRequest<T>(url: string): Observable<T> {
    return this.authService.getValidToken().pipe(
      switchMap(token => {
        if (!token) {
          return throwError(() => new Error('Authentication required'));
        }
        
        const headers = new HttpHeaders({
          'Authorization': `Bearer ${token}`,
          'X-Requested-With': 'XMLHttpRequest', // CSRF protection
        });
        
        return this.http.get<T>(url, { headers });
      }),
      timeout(30000), // Prevent hanging requests
      retry({
        count: 1,
        delay: (error) => {
          // Only retry on network errors, not auth errors
          if (error.status === 401 || error.status === 403) {
            return throwError(() => error);
          }
          return timer(1000);
        }
      })
    );
  }
  
  // Rate limiting for sensitive operations
  private rateLimitedOperations = new Map<string, number>();
  
  rateLimitedRequest<T>(
    operation: string,
    request: () => Observable<T>,
    maxRequestsPerMinute: number = 10
  ): Observable<T> {
    const now = Date.now();
    const lastRequest = this.rateLimitedOperations.get(operation) || 0;
    const timeSinceLastRequest = now - lastRequest;
    const minInterval = 60000 / maxRequestsPerMinute; // ms between requests
    
    if (timeSinceLastRequest < minInterval) {
      const delay = minInterval - timeSinceLastRequest;
      return timer(delay).pipe(
        switchMap(() => {
          this.rateLimitedOperations.set(operation, Date.now());
          return request();
        })
      );
    }
    
    this.rateLimitedOperations.set(operation, now);
    return request();
  }
}
```

### 2. Data Privacy and Protection

```typescript
// ✅ GOOD: Data privacy patterns
export class PrivacyAwareService {
  // PII (Personally Identifiable Information) handling
  getUserData(userId: string): Observable<PublicUserData> {
    return this.http.get<User>(`/api/users/${userId}`).pipe(
      map(user => this.stripPII(user)),
      catchError(error => {
        // Don't expose whether user exists
        if (error.status === 404) {
          return throwError(() => new Error('Access denied'));
        }
        return throwError(() => error);
      })
    );
  }
  
  private stripPII(user: User): PublicUserData {
    const { 
      id, 
      username, 
      displayName, 
      avatar,
      // Remove PII fields
      email, 
      phoneNumber, 
      address, 
      ssn,
      ...publicData 
    } = user;
    
    return {
      id,
      username,
      displayName,
      avatar,
      // Only include explicitly allowed fields
      joinDate: user.joinDate,
      isVerified: user.isVerified
    };
  }
  
  // Consent-aware data collection
  collectAnalytics(event: AnalyticsEvent): Observable<void> {
    return this.consentService.hasConsent('analytics').pipe(
      switchMap(hasConsent => {
        if (!hasConsent) {
          // Don't collect data without consent
          return of(void 0);
        }
        
        // Anonymize data before sending
        const anonymizedEvent = this.anonymizeEvent(event);
        return this.analyticsService.track(anonymizedEvent);
      })
    );
  }
  
  private anonymizeEvent(event: AnalyticsEvent): AnonymizedEvent {
    return {
      type: event.type,
      timestamp: event.timestamp,
      // Remove or hash identifying information
      sessionId: this.hashValue(event.sessionId),
      // Remove IP address, user agent, etc.
      metadata: this.filterMetadata(event.metadata)
    };
  }
}
```

## Summary

This comprehensive guide covers essential RxJS best practices:

- ✅ **Core Best Practices**: Naming conventions, operator usage, and memory management
- ✅ **Error Handling**: Comprehensive strategies, custom errors, and recovery patterns
- ✅ **Performance Optimization**: Operator selection, caching, and bundle optimization
- ✅ **Testing Strategies**: Marble testing, utilities, and integration testing
- ✅ **Architectural Patterns**: Service design, state management, and component architecture
- ✅ **Team Guidelines**: Code review checklists, training materials, and documentation
- ✅ **Security Practices**: Input validation, authentication, and data privacy

These practices ensure maintainable, performant, and secure RxJS applications while establishing clear standards for development teams.

## Next Steps

Next, we'll explore the **Complete RxJS Design Patterns Collection**, covering advanced patterns and architectural approaches for complex reactive systems.
