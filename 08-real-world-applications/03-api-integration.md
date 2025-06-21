# Advanced API Integration Patterns

## Learning Objectives
By the end of this lesson, you will be able to:
- Implement sophisticated API integration patterns using RxJS
- Handle complex API scenarios like pagination, caching, and retries
- Build resilient data layers with proper error handling
- Optimize API calls with advanced RxJS operators
- Implement real-time API synchronization patterns

## Table of Contents
1. [API Integration Fundamentals](#api-integration-fundamentals)
2. [Advanced HTTP Patterns](#advanced-http-patterns)
3. [Caching Strategies](#caching-strategies)
4. [Pagination & Infinite Scroll](#pagination--infinite-scroll)
5. [Error Handling & Resilience](#error-handling--resilience)
6. [Real-time Synchronization](#real-time-synchronization)
7. [Batch Operations](#batch-operations)
8. [Best Practices](#best-practices)

## API Integration Fundamentals

### Service Architecture Pattern

```typescript
// api.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { Observable, throwError, BehaviorSubject, timer } from 'rxjs';
import { 
  map, 
  catchError, 
  retry, 
  retryWhen, 
  delay,
  mergeMap,
  shareReplay,
  switchMap,
  debounceTime,
  distinctUntilChanged
} from 'rxjs/operators';

export interface ApiResponse<T> {
  data: T;
  message?: string;
  status: 'success' | 'error';
  pagination?: {
    total: number;
    page: number;
    pageSize: number;
    hasNext: boolean;
  };
}

@Injectable({
  providedIn: 'root'
})
export class ApiService {
  private readonly baseUrl = 'https://api.example.com';
  private cache = new Map<string, Observable<any>>();
  private readonly loadingSubject = new BehaviorSubject<Set<string>>(new Set());
  
  public loading$ = this.loadingSubject.asObservable().pipe(
    map(loadingSet => loadingSet.size > 0)
  );

  constructor(private http: HttpClient) {}

  private addToLoading(key: string): void {
    const current = this.loadingSubject.value;
    current.add(key);
    this.loadingSubject.next(new Set(current));
  }

  private removeFromLoading(key: string): void {
    const current = this.loadingSubject.value;
    current.delete(key);
    this.loadingSubject.next(new Set(current));
  }

  private handleError(error: HttpErrorResponse): Observable<never> {
    let errorMessage = 'An unknown error occurred';
    
    if (error.error instanceof ErrorEvent) {
      // Client-side error
      errorMessage = error.error.message;
    } else {
      // Server-side error
      switch (error.status) {
        case 400:
          errorMessage = 'Bad request. Please check your input.';
          break;
        case 401:
          errorMessage = 'Unauthorized. Please log in again.';
          break;
        case 403:
          errorMessage = 'Forbidden. You don\'t have permission.';
          break;
        case 404:
          errorMessage = 'Resource not found.';
          break;
        case 500:
          errorMessage = 'Server error. Please try again later.';
          break;
        default:
          errorMessage = `Error ${error.status}: ${error.message}`;
      }
    }
    
    return throwError({ message: errorMessage, status: error.status });
  }

  private retryStrategy(maxRetries: number = 3, delayMs: number = 1000) {
    return retryWhen(errors =>
      errors.pipe(
        mergeMap((error, index) => {
          if (index >= maxRetries) {
            return throwError(error);
          }
          
          // Only retry on server errors (5xx) or network errors
          if (error.status >= 500 || error.status === 0) {
            return timer(delayMs * Math.pow(2, index)); // Exponential backoff
          }
          
          return throwError(error);
        })
      )
    );
  }
}
```

## Advanced HTTP Patterns

### Concurrent Request Management

```typescript
// data.service.ts
@Injectable({
  providedIn: 'root'
})
export class DataService {
  constructor(private apiService: ApiService) {}

  // Sequential API calls with dependency
  getUserWithProfile(userId: string): Observable<UserWithProfile> {
    return this.apiService.get<User>(`/users/${userId}`).pipe(
      switchMap(user => 
        this.apiService.get<Profile>(`/profiles/${user.profileId}`).pipe(
          map(profile => ({ ...user, profile }))
        )
      ),
      catchError(this.handleUserError)
    );
  }

  // Parallel API calls
  getDashboardData(): Observable<DashboardData> {
    const user$ = this.apiService.get<User>('/user/current');
    const stats$ = this.apiService.get<Stats>('/dashboard/stats');
    const notifications$ = this.apiService.get<Notification[]>('/notifications');
    
    return combineLatest([user$, stats$, notifications$]).pipe(
      map(([user, stats, notifications]) => ({
        user,
        stats,
        notifications,
        lastUpdated: new Date()
      })),
      shareReplay(1)
    );
  }

  // Conditional API calls
  getOptionalData(includeDetails: boolean): Observable<DataResponse> {
    return this.apiService.get<BasicData>('/data/basic').pipe(
      switchMap(basicData => {
        if (includeDetails) {
          return this.apiService.get<DetailedData>(`/data/details/${basicData.id}`).pipe(
            map(details => ({ ...basicData, details }))
          );
        }
        return of(basicData);
      })
    );
  }

  private handleUserError = (error: any): Observable<UserWithProfile> => {
    // Custom error handling for user-specific errors
    if (error.status === 404) {
      return of({
        id: '',
        name: 'Unknown User',
        profile: { bio: 'Profile not found' }
      } as UserWithProfile);
    }
    return throwError(error);
  };
}
```

### Request Deduplication

```typescript
// request-cache.service.ts
@Injectable({
  providedIn: 'root'
})
export class RequestCacheService {
  private cache = new Map<string, Observable<any>>();
  private readonly cacheExpiry = 5 * 60 * 1000; // 5 minutes

  constructor(private http: HttpClient) {}

  get<T>(url: string, options?: any): Observable<T> {
    const cacheKey = this.createCacheKey(url, options);
    
    if (this.cache.has(cacheKey)) {
      return this.cache.get(cacheKey)!;
    }

    const request$ = this.http.get<T>(url, options).pipe(
      shareReplay(1),
      tap(() => {
        // Auto-expire cache entry
        timer(this.cacheExpiry).subscribe(() => {
          this.cache.delete(cacheKey);
        });
      }),
      catchError(error => {
        this.cache.delete(cacheKey);
        return throwError(error);
      })
    );

    this.cache.set(cacheKey, request$);
    return request$;
  }

  private createCacheKey(url: string, options?: any): string {
    return `${url}${options ? JSON.stringify(options) : ''}`;
  }

  clearCache(pattern?: string): void {
    if (pattern) {
      const keysToDelete = Array.from(this.cache.keys())
        .filter(key => key.includes(pattern));
      keysToDelete.forEach(key => this.cache.delete(key));
    } else {
      this.cache.clear();
    }
  }
}
```

## Caching Strategies

### Multi-Level Caching

```typescript
// cache.service.ts
export interface CacheConfig {
  ttl: number;
  maxSize: number;
  strategy: 'LRU' | 'FIFO' | 'TTL';
}

@Injectable({
  providedIn: 'root'
})
export class CacheService {
  private memoryCache = new Map<string, CacheEntry>();
  private accessOrder = new Map<string, number>();
  private accessCounter = 0;

  constructor(private config: CacheConfig) {}

  get<T>(key: string): T | null {
    const entry = this.memoryCache.get(key);
    
    if (!entry) {
      return null;
    }

    if (this.isExpired(entry)) {
      this.memoryCache.delete(key);
      this.accessOrder.delete(key);
      return null;
    }

    // Update access order for LRU
    if (this.config.strategy === 'LRU') {
      this.accessOrder.set(key, ++this.accessCounter);
    }

    return entry.data;
  }

  set<T>(key: string, data: T): void {
    this.enforceMaxSize();

    const entry: CacheEntry = {
      data,
      timestamp: Date.now(),
      ttl: this.config.ttl
    };

    this.memoryCache.set(key, entry);
    this.accessOrder.set(key, ++this.accessCounter);
  }

  private isExpired(entry: CacheEntry): boolean {
    return Date.now() - entry.timestamp > entry.ttl;
  }

  private enforceMaxSize(): void {
    if (this.memoryCache.size >= this.config.maxSize) {
      const oldestKey = this.getOldestKey();
      if (oldestKey) {
        this.memoryCache.delete(oldestKey);
        this.accessOrder.delete(oldestKey);
      }
    }
  }

  private getOldestKey(): string | null {
    let oldestKey: string | null = null;
    let oldestAccess = Infinity;

    for (const [key, access] of this.accessOrder) {
      if (access < oldestAccess) {
        oldestAccess = access;
        oldestKey = key;
      }
    }

    return oldestKey;
  }
}

interface CacheEntry {
  data: any;
  timestamp: number;
  ttl: number;
}
```

### HTTP Cache with Observable Patterns

```typescript
// http-cache.service.ts
@Injectable({
  providedIn: 'root'
})
export class HttpCacheService {
  private cache = new Map<string, Observable<any>>();

  constructor(
    private http: HttpClient,
    private cacheService: CacheService
  ) {}

  get<T>(url: string, options: CacheOptions = {}): Observable<T> {
    const cacheKey = this.createCacheKey(url, options);
    
    // Check memory cache first
    const cachedData = this.cacheService.get<T>(cacheKey);
    if (cachedData && !options.forceRefresh) {
      return of(cachedData);
    }

    // Check if request is already in flight
    if (this.cache.has(cacheKey)) {
      return this.cache.get(cacheKey)!;
    }

    // Make new request
    const request$ = this.http.get<T>(url).pipe(
      tap(data => this.cacheService.set(cacheKey, data)),
      shareReplay(1),
      finalize(() => this.cache.delete(cacheKey))
    );

    this.cache.set(cacheKey, request$);
    return request$;
  }

  // Cache with refresh strategy
  getWithRefresh<T>(url: string, refreshInterval: number): Observable<T> {
    return timer(0, refreshInterval).pipe(
      switchMap(() => this.get<T>(url, { forceRefresh: true })),
      shareReplay(1)
    );
  }

  private createCacheKey(url: string, options: any): string {
    return `${url}_${JSON.stringify(options)}`;
  }
}

interface CacheOptions {
  forceRefresh?: boolean;
  ttl?: number;
}
```

## Pagination & Infinite Scroll

### Advanced Pagination Service

```typescript
// pagination.service.ts
export interface PaginationState<T> {
  items: T[];
  currentPage: number;
  totalPages: number;
  totalItems: number;
  pageSize: number;
  loading: boolean;
  hasNext: boolean;
  hasPrevious: boolean;
}

@Injectable({
  providedIn: 'root'
})
export class PaginationService<T> {
  private stateSubject = new BehaviorSubject<PaginationState<T>>({
    items: [],
    currentPage: 1,
    totalPages: 0,
    totalItems: 0,
    pageSize: 20,
    loading: false,
    hasNext: false,
    hasPrevious: false
  });

  public state$ = this.stateSubject.asObservable();
  public items$ = this.state$.pipe(map(state => state.items));
  public loading$ = this.state$.pipe(map(state => state.loading));

  constructor(private apiService: ApiService) {}

  loadPage(page: number, append: boolean = false): Observable<T[]> {
    const currentState = this.stateSubject.value;
    
    this.updateState({ loading: true });

    return this.apiService.get<ApiResponse<T[]>>(`/items?page=${page}&size=${currentState.pageSize}`).pipe(
      map(response => {
        const newItems = append 
          ? [...currentState.items, ...response.data]
          : response.data;

        this.updateState({
          items: newItems,
          currentPage: page,
          totalPages: Math.ceil(response.pagination!.total / currentState.pageSize),
          totalItems: response.pagination!.total,
          loading: false,
          hasNext: response.pagination!.hasNext,
          hasPrevious: page > 1
        });

        return response.data;
      }),
      catchError(error => {
        this.updateState({ loading: false });
        return throwError(error);
      })
    );
  }

  loadNext(): Observable<T[]> {
    const currentState = this.stateSubject.value;
    if (currentState.hasNext && !currentState.loading) {
      return this.loadPage(currentState.currentPage + 1, true);
    }
    return EMPTY;
  }

  refresh(): Observable<T[]> {
    return this.loadPage(1, false);
  }

  private updateState(partial: Partial<PaginationState<T>>): void {
    const currentState = this.stateSubject.value;
    this.stateSubject.next({ ...currentState, ...partial });
  }
}
```

### Infinite Scroll Implementation

```typescript
// infinite-scroll.directive.ts
@Directive({
  selector: '[appInfiniteScroll]'
})
export class InfiniteScrollDirective implements OnInit, OnDestroy {
  @Output() scrolled = new EventEmitter<void>();
  @Input() threshold = 100;
  @Input() enabled = true;

  private destroy$ = new Subject<void>();

  constructor(private elementRef: ElementRef) {}

  ngOnInit(): void {
    if (this.enabled) {
      fromEvent(this.elementRef.nativeElement, 'scroll')
        .pipe(
          debounceTime(50),
          map(() => this.calculateDistance()),
          filter(distance => distance <= this.threshold),
          takeUntil(this.destroy$)
        )
        .subscribe(() => this.scrolled.emit());
    }
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }

  private calculateDistance(): number {
    const element = this.elementRef.nativeElement;
    return element.scrollHeight - element.scrollTop - element.clientHeight;
  }
}

// Usage in component
@Component({
  template: `
    <div 
      appInfiniteScroll 
      (scrolled)="loadMore()"
      [enabled]="!loading"
      class="scrollable-container">
      
      <div *ngFor="let item of items$ | async" class="item">
        {{ item.name }}
      </div>
      
      <div *ngIf="loading$ | async" class="loading">
        Loading more items...
      </div>
    </div>
  `
})
export class InfiniteScrollComponent {
  items$ = this.paginationService.items$;
  loading$ = this.paginationService.loading$;

  constructor(private paginationService: PaginationService<any>) {
    this.paginationService.loadPage(1).subscribe();
  }

  loadMore(): void {
    this.paginationService.loadNext().subscribe();
  }
}
```

## Error Handling & Resilience

### Comprehensive Error Strategy

```typescript
// error-handler.service.ts
export interface ErrorContext {
  operation: string;
  url?: string;
  user?: string;
  timestamp: Date;
  attemptNumber?: number;
}

@Injectable({
  providedIn: 'root'
})
export class ErrorHandlerService {
  private errorLog: ErrorLog[] = [];
  private readonly maxRetries = 3;

  constructor(
    private notificationService: NotificationService,
    private logger: LoggerService
  ) {}

  handleError<T>(
    operation: string,
    result?: T
  ): (error: any) => Observable<T> {
    return (error: any): Observable<T> => {
      const context: ErrorContext = {
        operation,
        url: error.url,
        timestamp: new Date()
      };

      this.logError(error, context);
      
      // Determine if error is recoverable
      if (this.isRecoverableError(error)) {
        return this.attemptRecovery(error, context, result);
      }

      // Show user-friendly message
      this.showErrorToUser(error, context);
      
      // Return safe fallback
      return of(result as T);
    };
  }

  private isRecoverableError(error: any): boolean {
    return error.status >= 500 || 
           error.status === 0 || 
           error.status === 408 ||
           error.name === 'TimeoutError';
  }

  private attemptRecovery<T>(
    error: any, 
    context: ErrorContext, 
    fallback?: T
  ): Observable<T> {
    return timer(this.getRetryDelay(context.attemptNumber || 0)).pipe(
      switchMap(() => {
        if ((context.attemptNumber || 0) < this.maxRetries) {
          // Retry the original operation
          return throwError(error);
        }
        
        // Max retries reached, return fallback
        return of(fallback as T);
      })
    );
  }

  private getRetryDelay(attempt: number): number {
    // Exponential backoff with jitter
    const baseDelay = 1000;
    const maxDelay = 30000;
    const delay = Math.min(baseDelay * Math.pow(2, attempt), maxDelay);
    const jitter = Math.random() * 0.1 * delay;
    return delay + jitter;
  }

  private logError(error: any, context: ErrorContext): void {
    const errorLog: ErrorLog = {
      id: this.generateId(),
      error: {
        message: error.message,
        status: error.status,
        stack: error.stack
      },
      context,
      severity: this.determineSeverity(error)
    };

    this.errorLog.push(errorLog);
    this.logger.error('API Error', errorLog);
  }

  private showErrorToUser(error: any, context: ErrorContext): void {
    const userMessage = this.getUserFriendlyMessage(error);
    this.notificationService.showError(userMessage);
  }

  private getUserFriendlyMessage(error: any): string {
    const messages: { [key: number]: string } = {
      400: 'Please check your input and try again.',
      401: 'Please log in to continue.',
      403: 'You don\'t have permission to perform this action.',
      404: 'The requested information could not be found.',
      408: 'The request timed out. Please try again.',
      429: 'Too many requests. Please wait a moment.',
      500: 'A server error occurred. Please try again later.',
      503: 'The service is temporarily unavailable.'
    };

    return messages[error.status] || 'An unexpected error occurred.';
  }

  private determineSeverity(error: any): 'low' | 'medium' | 'high' | 'critical' {
    if (error.status >= 500) return 'critical';
    if (error.status === 401 || error.status === 403) return 'high';
    if (error.status >= 400) return 'medium';
    return 'low';
  }

  private generateId(): string {
    return Date.now().toString(36) + Math.random().toString(36).substr(2);
  }
}

interface ErrorLog {
  id: string;
  error: {
    message: string;
    status?: number;
    stack?: string;
  };
  context: ErrorContext;
  severity: 'low' | 'medium' | 'high' | 'critical';
}
```

### Circuit Breaker Pattern

```typescript
// circuit-breaker.service.ts
enum CircuitState {
  CLOSED,
  OPEN,
  HALF_OPEN
}

@Injectable({
  providedIn: 'root'
})
export class CircuitBreakerService {
  private state = CircuitState.CLOSED;
  private failureCount = 0;
  private lastFailureTime: number = 0;
  private readonly failureThreshold = 5;
  private readonly recoveryTimeout = 60000; // 1 minute
  private readonly monitoringWindow = 300000; // 5 minutes

  execute<T>(operation: () => Observable<T>): Observable<T> {
    if (this.state === CircuitState.OPEN) {
      if (this.shouldAttemptReset()) {
        this.state = CircuitState.HALF_OPEN;
      } else {
        return throwError(new Error('Circuit breaker is OPEN'));
      }
    }

    return operation().pipe(
      tap(() => this.onSuccess()),
      catchError(error => {
        this.onFailure();
        return throwError(error);
      })
    );
  }

  private onSuccess(): void {
    this.failureCount = 0;
    this.state = CircuitState.CLOSED;
  }

  private onFailure(): void {
    this.failureCount++;
    this.lastFailureTime = Date.now();

    if (this.failureCount >= this.failureThreshold) {
      this.state = CircuitState.OPEN;
    }
  }

  private shouldAttemptReset(): boolean {
    return Date.now() - this.lastFailureTime >= this.recoveryTimeout;
  }

  getState(): CircuitState {
    return this.state;
  }
}
```

## Real-time Synchronization

### Real-time Data Service

```typescript
// realtime-data.service.ts
@Injectable({
  providedIn: 'root'
})
export class RealtimeDataService {
  private wsConnection: WebSocketSubject<any> | null = null;
  private reconnectInterval = 5000;
  private maxReconnectAttempts = 10;
  private reconnectAttempts = 0;

  private dataSubject = new BehaviorSubject<any[]>([]);
  public data$ = this.dataSubject.asObservable();

  private connectionStatus = new BehaviorSubject<'connected' | 'disconnected' | 'reconnecting'>('disconnected');
  public connectionStatus$ = this.connectionStatus.asObservable();

  constructor(
    private apiService: ApiService,
    private notificationService: NotificationService
  ) {}

  connect(): void {
    if (this.wsConnection) {
      return;
    }

    this.connectionStatus.next('reconnecting');
    
    this.wsConnection = webSocket({
      url: 'wss://api.example.com/realtime',
      openObserver: {
        next: () => {
          this.connectionStatus.next('connected');
          this.reconnectAttempts = 0;
          this.loadInitialData();
        }
      },
      closeObserver: {
        next: () => {
          this.connectionStatus.next('disconnected');
          this.wsConnection = null;
          this.scheduleReconnect();
        }
      }
    });

    this.wsConnection.subscribe({
      next: (message) => this.handleMessage(message),
      error: (error) => this.handleError(error)
    });
  }

  private loadInitialData(): void {
    this.apiService.get<any[]>('/data').subscribe({
      next: (data) => this.dataSubject.next(data),
      error: (error) => console.error('Failed to load initial data:', error)
    });
  }

  private handleMessage(message: any): void {
    switch (message.type) {
      case 'DATA_UPDATE':
        this.updateData(message.payload);
        break;
      case 'DATA_DELETE':
        this.deleteData(message.payload.id);
        break;
      case 'DATA_CREATE':
        this.addData(message.payload);
        break;
      case 'BULK_UPDATE':
        this.handleBulkUpdate(message.payload);
        break;
    }
  }

  private updateData(updatedItem: any): void {
    const currentData = this.dataSubject.value;
    const index = currentData.findIndex(item => item.id === updatedItem.id);
    
    if (index !== -1) {
      const newData = [...currentData];
      newData[index] = { ...newData[index], ...updatedItem };
      this.dataSubject.next(newData);
    }
  }

  private deleteData(id: string): void {
    const currentData = this.dataSubject.value;
    const newData = currentData.filter(item => item.id !== id);
    this.dataSubject.next(newData);
  }

  private addData(newItem: any): void {
    const currentData = this.dataSubject.value;
    this.dataSubject.next([...currentData, newItem]);
  }

  private handleBulkUpdate(updates: any[]): void {
    const currentData = this.dataSubject.value;
    const newData = currentData.map(item => {
      const update = updates.find(u => u.id === item.id);
      return update ? { ...item, ...update } : item;
    });
    this.dataSubject.next(newData);
  }

  private scheduleReconnect(): void {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      timer(this.reconnectInterval * Math.pow(2, this.reconnectAttempts)).subscribe(() => {
        this.reconnectAttempts++;
        this.connect();
      });
    } else {
      this.notificationService.showError('Unable to maintain real-time connection');
    }
  }

  private handleError(error: any): void {
    console.error('WebSocket error:', error);
    this.connectionStatus.next('disconnected');
  }

  disconnect(): void {
    if (this.wsConnection) {
      this.wsConnection.complete();
      this.wsConnection = null;
    }
    this.connectionStatus.next('disconnected');
  }
}
```

## Batch Operations

### Efficient Batch Processing

```typescript
// batch-processor.service.ts
@Injectable({
  providedIn: 'root'
})
export class BatchProcessorService {
  private batchQueue: BatchOperation[] = [];
  private readonly batchSize = 10;
  private readonly batchDelay = 1000;
  private processingSubject = new Subject<BatchOperation[]>();

  constructor(private apiService: ApiService) {
    this.initializeBatchProcessor();
  }

  private initializeBatchProcessor(): void {
    this.processingSubject.pipe(
      debounceTime(this.batchDelay),
      map(() => this.createBatches()),
      mergeMap(batches => this.processBatches(batches))
    ).subscribe();
  }

  addToBatch(operation: BatchOperation): Observable<any> {
    return new Observable(observer => {
      operation.observer = observer;
      this.batchQueue.push(operation);
      this.processingSubject.next(this.batchQueue);
    });
  }

  // Batch create operations
  batchCreate<T>(items: T[]): Observable<T[]> {
    const operations = items.map(item => ({
      type: 'CREATE' as const,
      data: item,
      endpoint: '/batch/create'
    }));

    return this.processBatchOperations(operations);
  }

  // Batch update operations
  batchUpdate<T>(items: T[]): Observable<T[]> {
    const operations = items.map(item => ({
      type: 'UPDATE' as const,
      data: item,
      endpoint: '/batch/update'
    }));

    return this.processBatchOperations(operations);
  }

  // Batch delete operations
  batchDelete(ids: string[]): Observable<string[]> {
    const operations = ids.map(id => ({
      type: 'DELETE' as const,
      data: { id },
      endpoint: '/batch/delete'
    }));

    return this.processBatchOperations(operations);
  }

  private processBatchOperations<T>(operations: BatchOperation[]): Observable<T[]> {
    const batches = this.chunkArray(operations, this.batchSize);
    
    return from(batches).pipe(
      mergeMap(batch => this.apiService.post('/batch', { operations: batch }), 3),
      reduce((acc: T[], result: T[]) => [...acc, ...result], [])
    );
  }

  private createBatches(): BatchOperation[][] {
    const batches = this.chunkArray([...this.batchQueue], this.batchSize);
    this.batchQueue.length = 0; // Clear the queue
    return batches;
  }

  private processBatches(batches: BatchOperation[][]): Observable<any> {
    return from(batches).pipe(
      mergeMap(batch => this.processSingleBatch(batch), 2)
    );
  }

  private processSingleBatch(batch: BatchOperation[]): Observable<any> {
    const batchPayload = {
      operations: batch.map(op => ({
        type: op.type,
        data: op.data,
        endpoint: op.endpoint
      }))
    };

    return this.apiService.post('/batch', batchPayload).pipe(
      tap(results => {
        batch.forEach((operation, index) => {
          if (operation.observer) {
            if (results[index].success) {
              operation.observer.next(results[index].data);
              operation.observer.complete();
            } else {
              operation.observer.error(results[index].error);
            }
          }
        });
      }),
      catchError(error => {
        batch.forEach(operation => {
          if (operation.observer) {
            operation.observer.error(error);
          }
        });
        return EMPTY;
      })
    );
  }

  private chunkArray<T>(array: T[], size: number): T[][] {
    const chunks: T[][] = [];
    for (let i = 0; i < array.length; i += size) {
      chunks.push(array.slice(i, i + size));
    }
    return chunks;
  }
}

interface BatchOperation {
  type: 'CREATE' | 'UPDATE' | 'DELETE';
  data: any;
  endpoint: string;
  observer?: Observer<any>;
}
```

## Best Practices

### API Integration Checklist

```typescript
// api-best-practices.service.ts
@Injectable({
  providedIn: 'root'
})
export class ApiBestPracticesService {
  private readonly guidelines = {
    // 1. Always handle errors gracefully
    errorHandling: {
      useRetryStrategy: true,
      implementCircuitBreaker: true,
      provideUserFriendlyMessages: true,
      logErrorsForDebugging: true
    },

    // 2. Optimize performance
    performance: {
      cacheResponses: true,
      deduplicateRequests: true,
      useProperHttpMethods: true,
      implementPagination: true
    },

    // 3. Maintain data consistency
    consistency: {
      validateResponses: true,
      handlePartialFailures: true,
      implementOptimisticUpdates: true,
      syncWithRealtime: true
    },

    // 4. Security considerations
    security: {
      validateInputs: true,
      sanitizeData: true,
      implementAuthentication: true,
      useHttpsOnly: true
    }
  };

  // Example implementation of validation
  validateApiResponse<T>(response: any, schema: any): T {
    // Implement schema validation
    if (!this.isValidResponse(response, schema)) {
      throw new Error('Invalid API response format');
    }
    return response as T;
  }

  private isValidResponse(response: any, schema: any): boolean {
    // Implement your validation logic
    return true;
  }

  // Example of optimistic updates
  optimisticUpdate<T>(
    item: T,
    updateFn: (item: T) => Observable<T>,
    rollbackFn: (error: any) => void
  ): Observable<T> {
    return updateFn(item).pipe(
      catchError(error => {
        rollbackFn(error);
        return throwError(error);
      })
    );
  }
}
```

### Performance Monitoring

```typescript
// api-performance.service.ts
@Injectable({
  providedIn: 'root'
})
export class ApiPerformanceService {
  private metrics = new Map<string, PerformanceMetric[]>();

  trackApiCall<T>(url: string, operation: Observable<T>): Observable<T> {
    const startTime = performance.now();
    
    return operation.pipe(
      tap(() => {
        const endTime = performance.now();
        this.recordMetric(url, endTime - startTime, 'success');
      }),
      catchError(error => {
        const endTime = performance.now();
        this.recordMetric(url, endTime - startTime, 'error');
        return throwError(error);
      })
    );
  }

  private recordMetric(url: string, duration: number, status: 'success' | 'error'): void {
    if (!this.metrics.has(url)) {
      this.metrics.set(url, []);
    }

    const metrics = this.metrics.get(url)!;
    metrics.push({
      timestamp: Date.now(),
      duration,
      status
    });

    // Keep only last 100 metrics per endpoint
    if (metrics.length > 100) {
      metrics.shift();
    }
  }

  getPerformanceReport(url: string): PerformanceReport {
    const metrics = this.metrics.get(url) || [];
    
    if (metrics.length === 0) {
      return { url, averageDuration: 0, successRate: 0, totalCalls: 0 };
    }

    const successfulCalls = metrics.filter(m => m.status === 'success');
    const averageDuration = metrics.reduce((sum, m) => sum + m.duration, 0) / metrics.length;
    const successRate = successfulCalls.length / metrics.length;

    return {
      url,
      averageDuration,
      successRate,
      totalCalls: metrics.length
    };
  }
}

interface PerformanceMetric {
  timestamp: number;
  duration: number;
  status: 'success' | 'error';
}

interface PerformanceReport {
  url: string;
  averageDuration: number;
  successRate: number;
  totalCalls: number;
}
```

## Summary

This lesson covered advanced API integration patterns that are essential for building robust, scalable Angular applications:

### Key Takeaways:
1. **Structured API Services**: Implement proper service architecture with loading states and error handling
2. **Advanced HTTP Patterns**: Handle concurrent requests, dependencies, and deduplication
3. **Caching Strategies**: Multi-level caching with TTL, LRU, and request deduplication
4. **Pagination & Infinite Scroll**: Efficient data loading with proper state management
5. **Error Handling & Resilience**: Comprehensive error strategies with circuit breakers and retries
6. **Real-time Synchronization**: WebSocket integration with fallback strategies
7. **Batch Operations**: Efficient bulk processing for better performance
8. **Performance Monitoring**: Track and optimize API call performance

### Next Steps:
- Practice implementing these patterns in your projects
- Monitor API performance and optimize based on metrics
- Implement proper error handling strategies
- Consider real-time requirements for your applications

In the next lesson, we'll explore **WebSockets with RxJS** for real-time communication patterns.
