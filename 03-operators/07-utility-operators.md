# Utility Operators

## Overview

Utility operators are essential tools that provide debugging capabilities, side effects management, and stream manipulation utilities. These operators don't transform the data itself but help you monitor, debug, and manage the behavior of Observable streams. They are crucial for development, monitoring, and creating robust reactive applications.

## Learning Objectives

After completing this lesson, you will be able to:
- Debug Observable streams effectively using utility operators
- Implement side effects without disrupting the main data flow
- Monitor and log Observable behavior for development and production
- Control subscription timing and lifecycle
- Apply utility operators for performance monitoring and optimization

## Core Utility Operators

### 1. tap() - Perform Side Effects

The `tap()` operator allows you to perform side effects for notifications from the source Observable without affecting the stream.

```typescript
import { of, interval } from 'rxjs';
import { tap, map, filter } from 'rxjs/operators';

// Basic side effects
const numbers$ = of(1, 2, 3, 4, 5);
const withLogging$ = numbers$.pipe(
  tap(value => console.log('Processing:', value)),
  map(x => x * 2),
  tap(value => console.log('After doubling:', value)),
  filter(x => x > 4),
  tap(value => console.log('After filtering:', value))
);

withLogging$.subscribe(value => console.log('Final:', value));

// Angular HTTP request with logging
@Injectable()
export class LoggingApiService {
  constructor(
    private http: HttpClient,
    private logger: LoggerService,
    private analytics: AnalyticsService
  ) {}

  getUser(id: string): Observable<User> {
    const startTime = Date.now();
    
    return this.http.get<User>(`/api/users/${id}`).pipe(
      tap({
        next: user => {
          const duration = Date.now() - startTime;
          this.logger.info(`User ${id} loaded in ${duration}ms`);
          this.analytics.track('user_loaded', { userId: id, duration });
        },
        error: error => {
          const duration = Date.now() - startTime;
          this.logger.error(`Failed to load user ${id} after ${duration}ms:`, error);
          this.analytics.track('user_load_error', { userId: id, error: error.message });
        },
        complete: () => {
          this.logger.debug(`User ${id} request completed`);
        }
      })
    );
  }

  // Progress tracking for uploads
  uploadFile(file: File): Observable<UploadProgress> {
    return this.http.post<UploadProgress>('/api/upload', this.createFormData(file)).pipe(
      tap(progress => {
        console.log(`Upload progress: ${progress.percentage}%`);
        
        // Update UI indicators
        this.updateProgressBar(progress.percentage);
        
        // Analytics tracking
        if (progress.percentage === 100) {
          this.analytics.track('file_upload_completed', {
            fileName: file.name,
            fileSize: file.size
          });
        }
      })
    );
  }

  private createFormData(file: File): FormData {
    const formData = new FormData();
    formData.append('file', file);
    return formData;
  }

  private updateProgressBar(percentage: number): void {
    // Update progress bar component
  }
}
```

**Marble Diagram:**
```
Input:  -1-2-3|
tap(x => console.log(x))
Output: -1-2-3| (same values, but console.log side effect)
```

### 2. finalize() - Cleanup on Complete or Error

The `finalize()` operator executes a callback when the Observable completes, errors, or is unsubscribed.

```typescript
import { of, throwError, timer } from 'rxjs';
import { finalize, mergeMap } from 'rxjs/operators';

// Basic cleanup
const withCleanup$ = of(1, 2, 3).pipe(
  finalize(() => console.log('Stream completed or terminated'))
);

// Angular component with loading states
@Component({
  template: `
    <div class="data-container">
      <div *ngIf="isLoading" class="loading-spinner">Loading...</div>
      <div *ngFor="let item of data$ | async" class="data-item">
        {{ item.name }}
      </div>
      <button (click)="refresh()" [disabled]="isLoading">Refresh</button>
    </div>
  `
})
export class DataDisplayComponent implements OnInit {
  data$!: Observable<DataItem[]>;
  isLoading = false;
  private refreshTrigger$ = new Subject<void>();

  ngOnInit() {
    this.data$ = this.refreshTrigger$.pipe(
      startWith(null), // Initial load
      tap(() => this.setLoading(true)),
      switchMap(() => this.dataService.getData()),
      finalize(() => this.setLoading(false)), // Always cleanup loading state
      catchError(error => {
        this.handleError(error);
        return of([]); // Return empty array on error
      })
    );
  }

  refresh() {
    if (!this.isLoading) {
      this.refreshTrigger$.next();
    }
  }

  private setLoading(loading: boolean): void {
    this.isLoading = loading;
  }

  private handleError(error: any): void {
    console.error('Data loading failed:', error);
    this.notificationService.showError('Failed to load data');
  }
}

// Resource cleanup in services
@Injectable()
export class ResourceService {
  private websocketConnection: WebSocket | null = null;

  connectToWebSocket(): Observable<MessageEvent> {
    return new Observable(observer => {
      this.websocketConnection = new WebSocket('wss://api.example.com/ws');
      
      this.websocketConnection.onmessage = event => observer.next(event);
      this.websocketConnection.onerror = error => observer.error(error);
      this.websocketConnection.onclose = () => observer.complete();
      
      return () => {
        if (this.websocketConnection) {
          this.websocketConnection.close();
          this.websocketConnection = null;
        }
      };
    }).pipe(
      finalize(() => {
        console.log('WebSocket connection cleaned up');
        this.websocketConnection = null;
      })
    );
  }

  // Database transaction with cleanup
  performTransaction<T>(operation: () => Observable<T>): Observable<T> {
    let transaction: Transaction;
    
    return this.databaseService.beginTransaction().pipe(
      tap(tx => transaction = tx),
      mergeMap(() => operation()),
      tap(() => transaction.commit()),
      finalize(() => {
        if (transaction && !transaction.isCompleted) {
          transaction.rollback();
          console.log('Transaction rolled back in finalize');
        }
      })
    );
  }
}
```

### 3. delay() - Delay Emissions

The `delay()` operator delays the emission of items from the source Observable by a specified time.

```typescript
import { of, fromEvent } from 'rxjs';
import { delay, mergeMap } from 'rxjs/operators';

// Basic delay
const delayed$ = of('Hello', 'World').pipe(
  delay(1000) // Delay by 1 second
);

// Angular notification system with staggered animations
@Injectable()
export class NotificationService {
  private notifications$ = new Subject<Notification>();

  notifications = this.notifications$.pipe(
    // Stagger notifications to avoid overwhelming UI
    concatMap((notification, index) => 
      of(notification).pipe(
        delay(index * 200) // 200ms delay between notifications
      )
    )
  );

  showNotification(message: string, type: 'info' | 'success' | 'error' = 'info') {
    this.notifications$.next({
      id: this.generateId(),
      message,
      type,
      timestamp: Date.now()
    });
  }

  private generateId(): string {
    return Math.random().toString(36).substr(2, 9);
  }
}

// Simulated typing effect
@Component({
  template: `
    <div class="typewriter">
      <span>{{ displayText$ | async }}</span>
      <span class="cursor">|</span>
    </div>
  `
})
export class TypewriterComponent implements OnInit {
  private text = "Welcome to our application!";
  displayText$!: Observable<string>;

  ngOnInit() {
    this.displayText$ = from(this.text.split('')).pipe(
      concatMap((char, index) => 
        of(char).pipe(
          delay(index * 100) // 100ms delay per character
        )
      ),
      scan((acc, char) => acc + char, ''), // Accumulate characters
      startWith('') // Start with empty string
    );
  }
}

// Debounced search with delay
@Component({})
export class SearchComponent {
  searchControl = new FormControl('');

  searchResults$ = this.searchControl.valueChanges.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    tap(() => console.log('Search triggered')),
    switchMap(query => 
      query ? 
        this.searchService.search(query).pipe(
          delay(100) // Small delay to show loading state
        ) : 
        of([])
    ),
    tap(results => console.log(`Found ${results.length} results`))
  );
}
```

### 4. delayWhen() - Dynamic Delay

The `delayWhen()` operator delays emissions based on a dynamic condition or timing.

```typescript
import { timer, of } from 'rxjs';
import { delayWhen, mergeMap } from 'rxjs/operators';

// Dynamic delay based on value
const dynamicDelay$ = of(1, 2, 3, 4, 5).pipe(
  delayWhen(value => timer(value * 1000)) // Delay each value by its own value in seconds
);

// Angular rate limiting service
@Injectable()
export class RateLimitedService {
  private lastRequestTime = 0;
  private minDelay = 1000; // Minimum 1 second between requests

  makeRequest<T>(request: () => Observable<T>): Observable<T> {
    return of(null).pipe(
      delayWhen(() => {
        const now = Date.now();
        const timeSinceLastRequest = now - this.lastRequestTime;
        const delayNeeded = Math.max(0, this.minDelay - timeSinceLastRequest);
        
        this.lastRequestTime = now + delayNeeded;
        return timer(delayNeeded);
      }),
      mergeMap(() => request())
    );
  }
}

// Priority-based message processing
@Injectable()
export class MessageProcessor {
  processMessage(message: Message): Observable<ProcessedMessage> {
    return of(message).pipe(
      delayWhen(msg => {
        // High priority messages processed immediately
        if (msg.priority === 'high') {
          return of(null);
        }
        // Medium priority delayed by 1 second
        if (msg.priority === 'medium') {
          return timer(1000);
        }
        // Low priority delayed by 5 seconds
        return timer(5000);
      }),
      mergeMap(msg => this.processMessageInternal(msg))
    );
  }

  private processMessageInternal(message: Message): Observable<ProcessedMessage> {
    return this.http.post<ProcessedMessage>('/api/process-message', message);
  }
}
```

### 5. timeout() - Add Timeout to Operations

The `timeout()` operator adds a timeout to Observable operations, emitting an error if the timeout is exceeded.

```typescript
import { timer, of } from 'rxjs';
import { timeout, catchError } from 'rxjs/operators';

// Basic timeout
const withTimeout$ = timer(5000).pipe(
  timeout(3000), // 3 second timeout
  catchError(error => of('Timeout occurred'))
);

// Angular HTTP with timeout
@Injectable()
export class TimeoutApiService {
  constructor(private http: HttpClient) {}

  getData(timeoutMs: number = 10000): Observable<Data> {
    return this.http.get<Data>('/api/data').pipe(
      timeout(timeoutMs),
      catchError(error => {
        if (error.name === 'TimeoutError') {
          console.error('Request timed out');
          return throwError('Request took too long. Please try again.');
        }
        return throwError(error);
      })
    );
  }

  // Different timeouts for different operations
  getCriticalData(): Observable<CriticalData> {
    return this.http.get<CriticalData>('/api/critical').pipe(
      timeout(5000), // Short timeout for critical data
      retry(2),
      catchError(error => throwError('Critical data unavailable'))
    );
  }

  getLargeData(): Observable<LargeData> {
    return this.http.get<LargeData>('/api/large-dataset').pipe(
      timeout(30000), // Longer timeout for large data
      catchError(error => {
        if (error.name === 'TimeoutError') {
          return throwError('Large dataset download timed out');
        }
        return throwError(error);
      })
    );
  }
}

// User interaction timeout
@Component({})
export class UserInteractionComponent {
  private userActions$ = new Subject<UserAction>();

  // Timeout user sessions
  sessionTimeout$ = this.userActions$.pipe(
    timeout(30 * 60 * 1000), // 30 minutes of inactivity
    catchError(error => {
      if (error.name === 'TimeoutError') {
        this.authService.logout();
        this.router.navigate(['/login']);
        return of('Session timed out');
      }
      return throwError(error);
    })
  );

  onUserAction(action: UserAction) {
    this.userActions$.next(action);
  }
}
```

### 6. timestamp() - Add Timestamps

The `timestamp()` operator attaches a timestamp to each emitted value.

```typescript
import { interval } from 'rxjs';
import { timestamp, map } from 'rxjs/operators';

// Basic timestamp
const withTimestamp$ = interval(1000).pipe(
  timestamp(),
  map(({ value, timestamp }) => `Value ${value} at ${new Date(timestamp)}`)
);

// Angular performance monitoring
@Injectable()
export class PerformanceMonitoringService {
  private userActions$ = new Subject<UserAction>();

  // Monitor user interaction timing
  monitoredActions$ = this.userActions$.pipe(
    timestamp(),
    map(({ value: action, timestamp }) => ({
      ...action,
      timestamp,
      formattedTime: new Date(timestamp).toISOString()
    })),
    tap(action => this.logUserAction(action))
  );

  trackAction(action: UserAction) {
    this.userActions$.next(action);
  }

  private logUserAction(action: UserActionWithTimestamp): void {
    console.log(`Action: ${action.type} at ${action.formattedTime}`);
    
    // Send to analytics service
    this.analytics.track(action.type, {
      timestamp: action.timestamp,
      ...action.data
    });
  }
}

// API response time monitoring
@Injectable()
export class ApiMonitoringService {
  constructor(private http: HttpClient) {}

  monitoredRequest<T>(url: string): Observable<T> {
    const startTime = Date.now();
    
    return this.http.get<T>(url).pipe(
      timestamp(),
      tap(({ timestamp }) => {
        const responseTime = timestamp - startTime;
        this.recordResponseTime(url, responseTime);
      }),
      map(({ value }) => value)
    );
  }

  private recordResponseTime(url: string, responseTime: number): void {
    console.log(`${url} responded in ${responseTime}ms`);
    
    // Record performance metrics
    this.metricsService.recordResponseTime(url, responseTime);
    
    // Alert if response time is too high
    if (responseTime > 5000) {
      this.alertService.slowApiResponse(url, responseTime);
    }
  }
}
```

### 7. timeInterval() - Time Between Emissions

The `timeInterval()` operator emits an object containing the current value and the time elapsed since the previous emission.

```typescript
import { fromEvent, interval } from 'rxjs';
import { timeInterval, map } from 'rxjs/operators';

// Monitor click intervals
const clickIntervals$ = fromEvent(document, 'click').pipe(
  timeInterval(),
  map(({ value, interval }) => `Click after ${interval}ms`)
);

// Angular user behavior analytics
@Component({})
export class UserBehaviorComponent {
  private userEvents$ = new Subject<UserEvent>();

  // Analyze user interaction patterns
  interactionAnalysis$ = this.userEvents$.pipe(
    timeInterval(),
    map(({ value: event, interval }) => ({
      event,
      timeSinceLast: interval,
      isRapidInteraction: interval < 1000
    })),
    tap(analysis => this.analyzeUserBehavior(analysis))
  );

  onUserEvent(event: UserEvent) {
    this.userEvents$.next(event);
  }

  private analyzeUserBehavior(analysis: UserInteractionAnalysis): void {
    if (analysis.isRapidInteraction) {
      console.log('Rapid user interaction detected');
    }
    
    // Track interaction patterns
    this.behaviorService.recordInteraction(analysis);
  }
}

// Performance monitoring for data processing
@Injectable()
export class DataProcessingMonitor {
  monitorProcessing<T>(source$: Observable<T>): Observable<T> {
    return source$.pipe(
      timeInterval(),
      tap(({ interval }) => {
        if (interval > 1000) {
          console.warn(`Slow processing detected: ${interval}ms`);
        }
      }),
      map(({ value }) => value)
    );
  }

  // Monitor stream throughput
  monitorThroughput<T>(source$: Observable<T>, windowSize: number = 10): Observable<T> {
    return source$.pipe(
      timeInterval(),
      scan((acc, { value, interval }) => {
        acc.intervals.push(interval);
        if (acc.intervals.length > windowSize) {
          acc.intervals.shift();
        }
        
        const avgInterval = acc.intervals.reduce((sum, i) => sum + i, 0) / acc.intervals.length;
        const throughput = 1000 / avgInterval; // items per second
        
        console.log(`Throughput: ${throughput.toFixed(2)} items/sec`);
        
        return { intervals: acc.intervals, value };
      }, { intervals: [] as number[], value: null as T }),
      map(({ value }) => value!)
    );
  }
}
```

### 8. share() - Share Subscription

The `share()` operator makes a cold Observable hot by sharing the subscription among multiple observers.

```typescript
import { interval, timer } from 'rxjs';
import { share, take } from 'rxjs/operators';

// Share expensive operations
const expensiveOperation$ = timer(0, 1000).pipe(
  map(() => {
    console.log('Expensive calculation performed');
    return Math.random();
  }),
  share() // Share the subscription
);

// Multiple subscribers share the same calculation
expensiveOperation$.subscribe(value => console.log('Subscriber 1:', value));
expensiveOperation$.subscribe(value => console.log('Subscriber 2:', value));

// Angular shared data service
@Injectable({ providedIn: 'root' })
export class SharedDataService {
  // Shared user data across components
  private userData$ = this.loadUserData().pipe(
    share() // Share among all components
  );

  getCurrentUser(): Observable<User> {
    return this.userData$;
  }

  private loadUserData(): Observable<User> {
    console.log('Loading user data...'); // Only logged once
    return this.http.get<User>('/api/current-user');
  }

  // Shared real-time updates
  private notifications$ = this.websocketService.connect().pipe(
    map(event => this.parseNotification(event.data)),
    share() // Share notifications across the app
  );

  getNotifications(): Observable<Notification> {
    return this.notifications$;
  }

  private parseNotification(data: string): Notification {
    return JSON.parse(data);
  }
}

// Component using shared data
@Component({})
export class UserProfileComponent {
  user$ = this.sharedDataService.getCurrentUser(); // Shares the HTTP request
  notifications$ = this.sharedDataService.getNotifications(); // Shares WebSocket connection

  constructor(private sharedDataService: SharedDataService) {}
}
```

### 9. shareReplay() - Share and Replay

The `shareReplay()` operator shares the subscription and replays the specified number of values to new subscribers.

```typescript
import { shareReplay } from 'rxjs/operators';

// Cache API responses
@Injectable()
export class CachedApiService {
  // Cache configuration data
  private config$ = this.http.get<AppConfig>('/api/config').pipe(
    shareReplay(1) // Cache the latest value
  );

  getConfig(): Observable<AppConfig> {
    return this.config$;
  }

  // Cache user permissions
  private permissions$ = this.http.get<Permission[]>('/api/permissions').pipe(
    shareReplay(1), // Cache permissions
    tap(permissions => console.log('Permissions loaded'))
  );

  getPermissions(): Observable<Permission[]> {
    return this.permissions$;
  }

  // Refresh cached data
  refreshConfig(): void {
    this.config$ = this.http.get<AppConfig>('/api/config').pipe(
      shareReplay(1)
    );
  }
}

// Real-time data with replay
@Injectable()
export class RealTimeDataService {
  // Keep last 10 values for new subscribers
  private sensorData$ = this.websocketService.getSensorData().pipe(
    shareReplay(10)
  );

  getSensorData(): Observable<SensorReading> {
    return this.sensorData$;
  }

  // Chat messages with history
  private chatMessages$ = this.websocketService.getChatMessages().pipe(
    shareReplay(50) // Keep last 50 messages
  );

  getChatMessages(): Observable<ChatMessage> {
    return this.chatMessages$;
  }
}
```

## Advanced Utility Patterns

### 1. Debugging Pipeline

```typescript
// Comprehensive debugging utility
@Injectable()
export class DebugUtilityService {
  debugPipeline<T>(label: string, logLevel: 'info' | 'debug' | 'warn' = 'info') {
    return (source$: Observable<T>) => source$.pipe(
      tap({
        next: value => this.log(`[${label}] Next:`, value, logLevel),
        error: error => this.log(`[${label}] Error:`, error, 'warn'),
        complete: () => this.log(`[${label}] Complete`, '', logLevel)
      }),
      timestamp(),
      tap(({ timestamp }) => 
        this.log(`[${label}] Timestamp:`, new Date(timestamp).toISOString(), 'debug')
      ),
      map(({ value }) => value)
    );
  }

  private log(message: string, data: any, level: string): void {
    if (environment.production && level === 'debug') return;
    
    console[level](message, data);
  }
}

// Usage in components
@Component({})
export class DebuggableComponent {
  data$ = this.dataService.getData().pipe(
    this.debugUtils.debugPipeline('UserData'),
    map(data => this.processData(data)),
    this.debugUtils.debugPipeline('ProcessedData')
  );

  constructor(private debugUtils: DebugUtilityService) {}
}
```

### 2. Performance Monitoring

```typescript
// Performance monitoring utilities
@Injectable()
export class PerformanceUtilities {
  measurePerformance<T>(operationName: string) {
    return (source$: Observable<T>) => {
      const startTime = performance.now();
      let itemCount = 0;
      
      return source$.pipe(
        tap(() => itemCount++),
        finalize(() => {
          const duration = performance.now() - startTime;
          const throughput = itemCount / (duration / 1000);
          
          console.log(`[${operationName}] Performance:`, {
            duration: `${duration.toFixed(2)}ms`,
            itemCount,
            throughput: `${throughput.toFixed(2)} items/sec`
          });
          
          // Send metrics to monitoring service
          this.metricsService.recordPerformance(operationName, {
            duration,
            itemCount,
            throughput
          });
        })
      );
    };
  }

  // Memory usage monitoring
  monitorMemory<T>(checkpointName: string) {
    return (source$: Observable<T>) => source$.pipe(
      tap(() => {
        if ('memory' in performance) {
          const memory = (performance as any).memory;
          console.log(`[${checkpointName}] Memory:`, {
            used: `${(memory.usedJSHeapSize / 1024 / 1024).toFixed(2)}MB`,
            total: `${(memory.totalJSHeapSize / 1024 / 1024).toFixed(2)}MB`,
            limit: `${(memory.jsHeapSizeLimit / 1024 / 1024).toFixed(2)}MB`
          });
        }
      })
    );
  }
}
```

### 3. Error Recovery with Utilities

```typescript
// Error recovery with utility operators
@Injectable()
export class ErrorRecoveryService {
  withErrorRecovery<T>(
    fallbackValue: T,
    retryCount: number = 3,
    operationName: string = 'Operation'
  ) {
    return (source$: Observable<T>) => source$.pipe(
      retry(retryCount),
      tap(() => console.log(`${operationName} succeeded`)),
      catchError(error => {
        console.error(`${operationName} failed after ${retryCount} retries:`, error);
        
        // Log to error tracking service
        this.errorTracker.captureError(error, {
          operation: operationName,
          retryCount
        });
        
        // Notify user
        this.notificationService.showWarning(
          `${operationName} failed. Using fallback data.`
        );
        
        return of(fallbackValue);
      }),
      finalize(() => console.log(`${operationName} completed`))
    );
  }
}

// Usage
@Component({})
export class ResilientComponent {
  data$ = this.apiService.getData().pipe(
    this.errorRecovery.withErrorRecovery([], 3, 'Data Loading')
  );
}
```

## Best Practices

### 1. Effective Debugging

```typescript
// ✅ Use tap() for side effects without disrupting flow
const debuggedStream$ = source$.pipe(
  tap(value => console.log('Before processing:', value)),
  map(processData),
  tap(value => console.log('After processing:', value)),
  filter(isValid),
  tap(value => console.log('After filtering:', value))
);

// ✅ Use finalize() for guaranteed cleanup
const withCleanup$ = source$.pipe(
  tap(() => this.startLoading()),
  mergeMap(processData),
  finalize(() => this.stopLoading()) // Always called
);
```

### 2. Performance Optimization

```typescript
// ✅ Use shareReplay() for expensive operations
const expensiveData$ = this.expensiveOperation().pipe(
  shareReplay(1) // Cache result for multiple subscribers
);

// ✅ Monitor performance in development
const monitored$ = source$.pipe(
  this.performanceUtils.measurePerformance('DataProcessing'),
  map(processData)
);
```

### 3. Production Considerations

```typescript
// ✅ Conditional debugging based on environment
const conditionalDebug$ = source$.pipe(
  // Only debug in development
  ...(environment.production ? [] : [
    tap(value => console.log('Debug:', value))
  ]),
  map(processData)
);

// ✅ Proper timeout handling
const withTimeout$ = source$.pipe(
  timeout(10000),
  catchError(error => {
    if (error.name === 'TimeoutError') {
      return of(fallbackValue);
    }
    return throwError(error);
  })
);
```

## Common Pitfalls

### 1. Overusing tap()

```typescript
// ❌ Too many tap operators
const over_tapped$ = source$.pipe(
  tap(x => console.log('1:', x)),
  map(x => x + 1),
  tap(x => console.log('2:', x)),
  filter(x => x > 0),
  tap(x => console.log('3:', x))
);

// ✅ Strategic use of tap
const better$ = source$.pipe(
  tap(x => console.log('Input:', x)),
  map(x => x + 1),
  filter(x => x > 0),
  tap(x => console.log('Output:', x))
);
```

### 2. Memory Leaks with shareReplay()

```typescript
// ❌ Unbounded shareReplay
const problematic$ = source$.pipe(
  shareReplay() // Keeps all values in memory
);

// ✅ Bounded shareReplay
const safe$ = source$.pipe(
  shareReplay(1) // Only keep latest value
);
```

## Exercises

### Exercise 1: Debug Dashboard
Create a debugging dashboard that monitors Observable streams in real-time, showing emission rates, error rates, and performance metrics.

### Exercise 2: Performance Monitor
Build a performance monitoring service that tracks the execution time and memory usage of different Observable operations.

### Exercise 3: Error Recovery System
Implement a comprehensive error recovery system that logs errors, provides fallbacks, and monitors system health.

## Summary

Utility operators are essential for building robust, maintainable reactive applications:

- **tap()**: Perform side effects without affecting the stream
- **finalize()**: Guaranteed cleanup on completion or error
- **delay()**: Add timing control to emissions
- **timeout()**: Add timeout constraints to operations
- **timestamp()**: Add timing information to values
- **share()/shareReplay()**: Optimize subscription sharing and caching

These operators help with debugging, monitoring, performance optimization, and building resilient applications.

## Next Steps

In the next lesson, we'll explore **Conditional Operators**, which provide powerful tools for implementing conditional logic in reactive streams.
