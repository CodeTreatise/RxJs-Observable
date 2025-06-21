# RxJS Interview Questions & Scenarios

## Introduction

This lesson provides a comprehensive collection of RxJS interview questions, practical scenarios, and code challenges commonly encountered in technical interviews. From basic concepts to advanced patterns, we'll cover everything you need to confidently discuss RxJS in interviews.

## Learning Objectives

- Master essential RxJS concepts for technical interviews
- Practice common coding scenarios and problem-solving
- Understand how to explain reactive programming concepts clearly
- Learn to demonstrate RxJS knowledge through practical examples
- Prepare for both theoretical and hands-on interview questions

## Fundamental Concepts (Entry Level)

### 1. What is RxJS and Observable?

**Question**: Explain what RxJS is and what an Observable represents.

**Answer**:
```typescript
// ✅ Complete answer with example

// RxJS is a library for reactive programming using Observables
// Observable = A stream of data that can emit values over time

import { Observable } from 'rxjs';

// Creating a simple Observable
const myObservable$ = new Observable(subscriber => {
  subscriber.next('Hello');
  subscriber.next('World');
  subscriber.complete();
});

// Key points to mention:
// 1. Lazy evaluation - doesn't execute until subscribed
// 2. Can emit multiple values over time
// 3. Supports async operations
// 4. Provides operators for data transformation
// 5. Has built-in error handling and completion
```

### 2. Observable vs Promise

**Question**: What are the key differences between Observables and Promises?

**Answer**:
```typescript
// ✅ Key differences with examples

// Promise - Single value, eager execution
const promise = fetch('/api/data')
  .then(response => response.json())
  .then(data => console.log(data));
// Executes immediately when created

// Observable - Multiple values, lazy execution
const observable$ = this.http.get('/api/data').pipe(
  map(response => response.data)
);
// Only executes when subscribed

// Key differences:
// 1. Single vs Multiple values
// 2. Eager vs Lazy execution
// 3. No cancellation vs Cancellation support
// 4. Limited operators vs Rich operator library
// 5. Always async vs Can be sync or async

// Cancellation example
const subscription = observable$.subscribe(data => console.log(data));
subscription.unsubscribe(); // Can cancel

// Promise cannot be cancelled once started
```

### 3. Hot vs Cold Observables

**Question**: Explain the difference between hot and cold observables.

**Answer**:
```typescript
// ✅ Hot vs Cold explanation with examples

// COLD Observable - Each subscription creates new execution
const coldObservable$ = new Observable(subscriber => {
  console.log('Observable executed'); // Runs for each subscription
  subscriber.next(Math.random());
});

coldObservable$.subscribe(value => console.log('Sub 1:', value));
coldObservable$.subscribe(value => console.log('Sub 2:', value));
// Logs: "Observable executed" twice, different random numbers

// HOT Observable - Shares single execution
import { Subject } from 'rxjs';

const hotObservable$ = new Subject();
hotObservable$.subscribe(value => console.log('Sub 1:', value));
hotObservable$.subscribe(value => console.log('Sub 2:', value));

hotObservable$.next('Shared value'); // Both subscribers get same value

// Converting Cold to Hot
const sharedCold$ = coldObservable$.pipe(share());
```

## Core Operators (Intermediate Level)

### 4. map vs switchMap vs mergeMap

**Question**: Explain the differences between map, switchMap, and mergeMap operators.

**Answer**:
```typescript
// ✅ Operator differences with practical examples

import { of, interval } from 'rxjs';
import { map, switchMap, mergeMap, delay } from 'rxjs/operators';

// MAP - Transform values
const numbers$ = of(1, 2, 3);
const doubled$ = numbers$.pipe(
  map(x => x * 2) // 2, 4, 6
);

// SWITCHMAP - Cancel previous, switch to new
const searchTerm$ = new Subject<string>();
const searchResults$ = searchTerm$.pipe(
  switchMap(term => 
    this.searchService.search(term) // Cancels previous search
  )
);
// Use case: Search autocomplete, navigation

// MERGEMAP - Run all in parallel
const userIds$ = of(1, 2, 3);
const userDetails$ = userIds$.pipe(
  mergeMap(id => 
    this.userService.getUser(id) // All requests run simultaneously
  )
);
// Use case: Parallel API calls, file uploads

// Key differences:
// map: 1-to-1 transformation
// switchMap: 1-to-many, cancel previous
// mergeMap: 1-to-many, run all parallel
```

### 5. Subject Types

**Question**: What are the different types of Subjects in RxJS?

**Answer**:
```typescript
// ✅ Subject types with use cases

import { Subject, BehaviorSubject, ReplaySubject, AsyncSubject } from 'rxjs';

// 1. SUBJECT - Basic multicast, no initial value
const basicSubject$ = new Subject<string>();
basicSubject$.subscribe(console.log);
basicSubject$.next('Hello'); // Logs: Hello
// Late subscribers don't get previous values

// 2. BEHAVIORSUBJECT - Has current value
const behaviorSubject$ = new BehaviorSubject<string>('Initial');
behaviorSubject$.subscribe(console.log); // Logs: Initial immediately
behaviorSubject$.next('Updated');
// Use case: Current state, user authentication status

// 3. REPLAYSUBJECT - Buffers previous values
const replaySubject$ = new ReplaySubject<string>(2); // Buffer last 2
replaySubject$.next('First');
replaySubject$.next('Second');
replaySubject$.subscribe(console.log); // Logs: First, Second
// Use case: Event history, chat messages

// 4. ASYNCSUBJECT - Only emits last value on completion
const asyncSubject$ = new AsyncSubject<string>();
asyncSubject$.next('First');
asyncSubject$.next('Last');
asyncSubject$.complete(); // Only now subscribers get 'Last'
// Use case: Single async result
```

## Practical Scenarios

### 6. Implementing Search with Debouncing

**Question**: How would you implement a search feature with debouncing to avoid excessive API calls?

**Answer**:
```typescript
// ✅ Complete search implementation

import { fromEvent } from 'rxjs';
import { debounceTime, distinctUntilChanged, switchMap, filter } from 'rxjs/operators';

@Component({
  template: `
    <input #searchInput placeholder="Search users...">
    <div *ngFor="let user of searchResults$ | async">{{ user.name }}</div>
  `
})
export class SearchComponent {
  @ViewChild('searchInput') searchInput!: ElementRef;
  
  searchResults$ = fromEvent(this.searchInput.nativeElement, 'input').pipe(
    map((event: any) => event.target.value),
    filter(term => term.length >= 2), // Minimum 2 characters
    debounceTime(300), // Wait 300ms after user stops typing
    distinctUntilChanged(), // Only if search term changed
    switchMap(term => 
      this.searchService.searchUsers(term).pipe(
        catchError(error => {
          console.error('Search failed:', error);
          return of([]); // Return empty array on error
        })
      )
    )
  );

  constructor(private searchService: SearchService) {}
}

// Why this works:
// 1. debounceTime prevents API spam
// 2. distinctUntilChanged avoids duplicate searches
// 3. switchMap cancels previous requests
// 4. filter ensures minimum search criteria
// 5. catchError provides fallback behavior
```

### 7. Managing Component Subscriptions

**Question**: How do you properly manage subscriptions in Angular components to prevent memory leaks?

**Answer**:
```typescript
// ✅ Multiple approaches to subscription management

// APPROACH 1: takeUntil pattern (Recommended)
@Component({...})
export class MyComponent implements OnDestroy {
  private destroy$ = new Subject<void>();
  
  ngOnInit() {
    this.dataService.getData().pipe(
      takeUntil(this.destroy$)
    ).subscribe(data => this.processData(data));
    
    this.userService.getCurrentUser().pipe(
      takeUntil(this.destroy$)
    ).subscribe(user => this.currentUser = user);
  }
  
  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// APPROACH 2: async pipe (Best for templates)
@Component({
  template: `
    <div *ngIf="loading$ | async">Loading...</div>
    <div *ngFor="let item of data$ | async">{{ item.name }}</div>
  `
})
export class AsyncPipeComponent {
  data$ = this.dataService.getData();
  loading$ = this.loadingService.isLoading$;
  // No manual subscription management needed
}

// APPROACH 3: Manual subscription tracking
@Component({...})
export class ManualComponent implements OnDestroy {
  private subscriptions = new Subscription();
  
  ngOnInit() {
    this.subscriptions.add(
      this.dataService.getData().subscribe(data => this.processData(data))
    );
  }
  
  ngOnDestroy() {
    this.subscriptions.unsubscribe();
  }
}
```

### 8. Error Handling Strategies

**Question**: How do you implement comprehensive error handling in RxJS?

**Answer**:
```typescript
// ✅ Comprehensive error handling strategies

// 1. CATCH AND REPLACE
const dataWithFallback$ = this.apiService.getData().pipe(
  catchError(error => {
    console.error('API failed, using fallback:', error);
    return of([]); // Return empty array as fallback
  })
);

// 2. CATCH AND RETHROW
const dataWithLogging$ = this.apiService.getData().pipe(
  catchError(error => {
    this.logError(error);
    this.showUserNotification('Failed to load data');
    return throwError(() => error); // Re-throw for upstream handling
  })
);

// 3. RETRY WITH BACKOFF
const resilientData$ = this.apiService.getData().pipe(
  retryWhen(errors => 
    errors.pipe(
      scan((retryCount, error) => {
        if (retryCount >= 3) throw error;
        return retryCount + 1;
      }, 0),
      delayWhen(retryCount => timer(1000 * retryCount)) // Exponential backoff
    )
  ),
  catchError(error => {
    this.handleFinalError(error);
    return of(null);
  })
);

// 4. GLOBAL ERROR HANDLING
@Injectable()
export class ErrorHandlingService {
  handleApiError(error: any): Observable<never> {
    let errorMessage = 'An error occurred';
    
    if (error.status === 401) {
      errorMessage = 'Authentication required';
      this.router.navigate(['/login']);
    } else if (error.status === 403) {
      errorMessage = 'Access denied';
    } else if (error.status >= 500) {
      errorMessage = 'Server error. Please try again later.';
    }
    
    this.notificationService.showError(errorMessage);
    return throwError(() => new Error(errorMessage));
  }
}
```

## Advanced Patterns

### 9. Creating Custom Operators

**Question**: How would you create a custom RxJS operator?

**Answer**:
```typescript
// ✅ Custom operator creation patterns

// 1. SIMPLE CUSTOM OPERATOR
function multiplyBy(multiplier: number) {
  return <T extends number>(source: Observable<T>): Observable<T> => {
    return source.pipe(
      map(value => (value * multiplier) as T)
    );
  };
}

// Usage
const numbers$ = of(1, 2, 3).pipe(
  multiplyBy(10) // 10, 20, 30
);

// 2. COMPLEX CUSTOM OPERATOR WITH STATE
function bufferUntilCount<T>(count: number) {
  return (source: Observable<T>): Observable<T[]> => {
    return new Observable<T[]>(subscriber => {
      let buffer: T[] = [];
      
      const subscription = source.subscribe({
        next: (value) => {
          buffer.push(value);
          if (buffer.length >= count) {
            subscriber.next([...buffer]);
            buffer = [];
          }
        },
        error: (error) => subscriber.error(error),
        complete: () => {
          if (buffer.length > 0) {
            subscriber.next(buffer);
          }
          subscriber.complete();
        }
      });
      
      return () => subscription.unsubscribe();
    });
  };
}

// 3. CUSTOM OPERATOR WITH CONFIGURATION
interface RetryConfig {
  maxRetries: number;
  delay: number;
  backoff?: boolean;
}

function customRetry<T>(config: RetryConfig) {
  return (source: Observable<T>): Observable<T> => {
    return source.pipe(
      retryWhen(errors => 
        errors.pipe(
          scan((retryCount, error) => {
            if (retryCount >= config.maxRetries) {
              throw error;
            }
            return retryCount + 1;
          }, 0),
          delayWhen(retryCount => {
            const delay = config.backoff 
              ? config.delay * Math.pow(2, retryCount - 1)
              : config.delay;
            return timer(delay);
          })
        )
      )
    );
  };
}
```

### 10. State Management Pattern

**Question**: How would you implement a simple state management solution using RxJS?

**Answer**:
```typescript
// ✅ Simple state management with RxJS

interface AppState {
  users: User[];
  selectedUser: User | null;
  loading: boolean;
}

@Injectable({ providedIn: 'root' })
export class StateService {
  private state$ = new BehaviorSubject<AppState>({
    users: [],
    selectedUser: null,
    loading: false
  });

  // Selectors
  readonly users$ = this.state$.pipe(
    map(state => state.users),
    distinctUntilChanged()
  );

  readonly selectedUser$ = this.state$.pipe(
    map(state => state.selectedUser),
    distinctUntilChanged()
  );

  readonly loading$ = this.state$.pipe(
    map(state => state.loading),
    distinctUntilChanged()
  );

  // Actions
  setUsers(users: User[]): void {
    this.updateState(state => ({ ...state, users }));
  }

  selectUser(user: User): void {
    this.updateState(state => ({ ...state, selectedUser: user }));
  }

  setLoading(loading: boolean): void {
    this.updateState(state => ({ ...state, loading }));
  }

  // Private helper
  private updateState(updateFn: (state: AppState) => AppState): void {
    const currentState = this.state$.value;
    const newState = updateFn(currentState);
    this.state$.next(newState);
  }

  // Complex action example
  loadUsers(): Observable<User[]> {
    this.setLoading(true);
    
    return this.apiService.getUsers().pipe(
      tap(users => {
        this.setUsers(users);
        this.setLoading(false);
      }),
      catchError(error => {
        this.setLoading(false);
        return throwError(() => error);
      })
    );
  }
}
```

## Common Pitfalls and Solutions

### 11. Memory Leaks

**Question**: What are common causes of memory leaks in RxJS and how do you prevent them?

**Answer**:
```typescript
// ✅ Common memory leak patterns and solutions

// PROBLEM 1: Forgotten subscriptions
// ❌ Bad
export class BadComponent {
  ngOnInit() {
    this.dataService.getData().subscribe(data => {
      this.data = data;
    }); // Never unsubscribed - MEMORY LEAK
  }
}

// ✅ Good
export class GoodComponent implements OnDestroy {
  private destroy$ = new Subject<void>();
  
  ngOnInit() {
    this.dataService.getData().pipe(
      takeUntil(this.destroy$)
    ).subscribe(data => this.data = data);
  }
  
  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// PROBLEM 2: Infinite observables
// ❌ Bad
const leak$ = interval(1000).subscribe(); // Never completes

// ✅ Good
const controlled$ = interval(1000).pipe(
  take(10), // Limit emissions
  takeUntil(componentDestroy$)
).subscribe();

// PROBLEM 3: Event listeners
// ❌ Bad
fromEvent(window, 'resize').subscribe(); // Never cleaned up

// ✅ Good
fromEvent(window, 'resize').pipe(
  takeUntil(this.destroy$)
).subscribe();

// PROBLEM 4: Shared state not cleaned up
// ❌ Bad
const sharedState$ = new BehaviorSubject(initialState);
// State persists even when components are destroyed

// ✅ Good - Use finite lifecycle
const componentState$ = new BehaviorSubject(initialState);
// Clean up in ngOnDestroy
ngOnDestroy() {
  this.componentState$.complete();
}
```

### 12. Performance Issues

**Question**: How do you identify and fix performance issues in RxJS streams?

**Answer**:
```typescript
// ✅ Performance optimization techniques

// PROBLEM 1: Unnecessary subscriptions in templates
// ❌ Bad
@Component({
  template: `
    <div>{{ (data$ | async)?.name }}</div>
    <div>{{ (data$ | async)?.email }}</div>
    <div>{{ (data$ | async)?.phone }}</div>
  ` // Creates 3 subscriptions
})

// ✅ Good
@Component({
  template: `
    <div *ngIf="data$ | async as data">
      <div>{{ data.name }}</div>
      <div>{{ data.email }}</div>
      <div>{{ data.phone }}</div>
    </div>
  ` // Single subscription
})

// PROBLEM 2: No shareReplay for expensive operations
// ❌ Bad
getExpensiveData() {
  return this.http.get('/expensive').pipe(
    map(data => this.heavyProcessing(data))
  ); // Executed on every subscription
}

// ✅ Good
getExpensiveData() {
  return this.http.get('/expensive').pipe(
    map(data => this.heavyProcessing(data)),
    shareReplay(1) // Cache result
  );
}

// PROBLEM 3: Inefficient operator chains
// ❌ Bad
const inefficient$ = source$.pipe(
  map(x => x),
  filter(() => true),
  map(x => x * 2),
  map(x => x + 1)
);

// ✅ Good
const efficient$ = source$.pipe(
  map(x => x * 2 + 1) // Combine operations
);

// PERFORMANCE MONITORING
function measurePerformance<T>(label: string) {
  return (source: Observable<T>): Observable<T> => {
    return new Observable<T>(subscriber => {
      const start = performance.now();
      let count = 0;
      
      return source.subscribe({
        next: value => {
          count++;
          subscriber.next(value);
        },
        error: error => {
          console.log(`${label} failed after ${performance.now() - start}ms, ${count} items`);
          subscriber.error(error);
        },
        complete: () => {
          console.log(`${label} completed in ${performance.now() - start}ms, ${count} items`);
          subscriber.complete();
        }
      });
    });
  };
}
```

## Behavioral Questions

### 13. When to Use RxJS

**Question**: When would you choose RxJS over other approaches for handling async operations?

**Answer**:
```typescript
// ✅ RxJS use cases vs alternatives

// USE RXJS WHEN:

// 1. Multiple async values over time
const realTimeData$ = webSocketService.connect().pipe(
  map(message => JSON.parse(message)),
  filter(data => data.type === 'UPDATE')
);

// 2. Complex data transformations
const processedData$ = rawData$.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(query => apiCall(query)),
  map(response => response.data),
  filter(data => data.isValid)
);

// 3. Coordinating multiple async operations
const combinedData$ = combineLatest([
  userService.getCurrentUser(),
  settingsService.getSettings(),
  notificationService.getNotifications()
]).pipe(
  map(([user, settings, notifications]) => ({
    user, settings, notifications
  }))
);

// DON'T USE RXJS WHEN:

// 1. Simple single async operation
// Use Promise instead
async function getUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
}

// 2. Simple event handling
// Use regular event listeners
button.addEventListener('click', () => {
  console.log('Button clicked');
});

// 3. Synchronous operations
// Use regular functions
function calculateTotal(items: Item[]): number {
  return items.reduce((sum, item) => sum + item.price, 0);
}
```

### 14. Debugging RxJS

**Question**: How do you debug complex RxJS streams?

**Answer**:
```typescript
// ✅ RxJS debugging techniques

// 1. TAP OPERATOR FOR LOGGING
const debuggedStream$ = source$.pipe(
  tap(value => console.log('Before map:', value)),
  map(x => x * 2),
  tap(value => console.log('After map:', value)),
  filter(x => x > 10),
  tap(value => console.log('After filter:', value))
);

// 2. CUSTOM DEBUG OPERATOR
function debug<T>(label: string) {
  return (source: Observable<T>): Observable<T> => {
    return source.pipe(
      tap({
        next: value => console.log(`${label} - Next:`, value),
        error: error => console.log(`${label} - Error:`, error),
        complete: () => console.log(`${label} - Complete`)
      })
    );
  };
}

// Usage
const stream$ = source$.pipe(
  debug('Input'),
  map(x => x * 2),
  debug('After map'),
  filter(x => x > 10),
  debug('After filter')
);

// 3. RXJS DEV TOOLS
// Install: npm install @ngrx/store-devtools
import { tap } from 'rxjs/operators';

const trackedStream$ = source$.pipe(
  tap(value => {
    if (window['__REDUX_DEVTOOLS_EXTENSION__']) {
      window['__REDUX_DEVTOOLS_EXTENSION__'].send('STREAM_VALUE', value);
    }
  })
);

// 4. MARBLE TESTING FOR DEBUGGING
import { TestScheduler } from 'rxjs/testing';

const testScheduler = new TestScheduler((actual, expected) => {
  console.log('Expected:', expected);
  console.log('Actual:', actual);
  expect(actual).toEqual(expected);
});

testScheduler.run(({ cold, hot, expectObservable }) => {
  const input$ = cold(' a-b-c|');
  const expected =     'x-y-z|';
  
  const result$ = input$.pipe(
    map(value => value.toUpperCase())
  );
  
  expectObservable(result$).toBe(expected, {
    x: 'A', y: 'B', z: 'C'
  });
});
```

## Code Challenges

### 15. Rate Limiting

**Question**: Implement a rate limiter that allows maximum 5 API calls per second.

**Answer**:
```typescript
// ✅ Rate limiting implementation

function rateLimit<T>(maxCalls: number, timeWindow: number) {
  return (source: Observable<T>): Observable<T> => {
    return new Observable<T>(subscriber => {
      const timestamps: number[] = [];
      
      return source.subscribe({
        next: (value) => {
          const now = Date.now();
          
          // Remove timestamps outside time window
          while (timestamps.length > 0 && now - timestamps[0] > timeWindow) {
            timestamps.shift();
          }
          
          // Check if we can make the call
          if (timestamps.length < maxCalls) {
            timestamps.push(now);
            subscriber.next(value);
          } else {
            // Calculate delay needed
            const oldestCall = timestamps[0];
            const delay = timeWindow - (now - oldestCall);
            
            setTimeout(() => {
              timestamps.shift();
              timestamps.push(Date.now());
              subscriber.next(value);
            }, delay);
          }
        },
        error: error => subscriber.error(error),
        complete: () => subscriber.complete()
      });
    });
  };
}

// Usage
const apiCalls$ = source$.pipe(
  rateLimit(5, 1000), // 5 calls per second
  switchMap(data => this.apiService.processData(data))
);
```

### 16. Cache with Expiration

**Question**: Implement a caching mechanism that expires after a certain time.

**Answer**:
```typescript
// ✅ Cache with expiration implementation

interface CacheEntry<T> {
  value: T;
  timestamp: number;
}

class ExpiringCache<T> {
  private cache = new Map<string, CacheEntry<T>>();
  
  constructor(private expirationTime: number) {}
  
  get(key: string): T | null {
    const entry = this.cache.get(key);
    
    if (!entry) return null;
    
    if (Date.now() - entry.timestamp > this.expirationTime) {
      this.cache.delete(key);
      return null;
    }
    
    return entry.value;
  }
  
  set(key: string, value: T): void {
    this.cache.set(key, {
      value,
      timestamp: Date.now()
    });
  }
  
  clear(): void {
    this.cache.clear();
  }
}

// Observable wrapper
function cachedObservable<T>(
  cache: ExpiringCache<T>,
  key: string,
  factory: () => Observable<T>
): Observable<T> {
  const cached = cache.get(key);
  
  if (cached !== null) {
    return of(cached);
  }
  
  return factory().pipe(
    tap(value => cache.set(key, value))
  );
}

// Usage
const cache = new ExpiringCache<User[]>(5 * 60 * 1000); // 5 minutes

function getUsers(): Observable<User[]> {
  return cachedObservable(
    cache,
    'users',
    () => this.http.get<User[]>('/api/users')
  );
}
```

## Tips for Success

### Interview Best Practices

```typescript
// ✅ How to succeed in RxJS interviews

// 1. START SIMPLE, BUILD COMPLEXITY
// Begin with basic Observable creation
const simple$ = of(1, 2, 3);

// Then add operators
const complex$ = simple$.pipe(
  map(x => x * 2),
  filter(x => x > 2),
  take(2)
);

// 2. EXPLAIN YOUR THINKING
// "I'm using switchMap here because we want to cancel 
// previous API requests when a new search term comes in"

// 3. CONSIDER EDGE CASES
// "We should handle errors with catchError"
// "We need to unsubscribe to prevent memory leaks"
// "Let's add debouncing to avoid excessive API calls"

// 4. DEMONSTRATE MARBLE DIAGRAMS
// Draw or explain timing: "Input: a-b-c, Output: x-y-z"

// 5. MENTION REAL-WORLD USAGE
// "This pattern is common in search autocomplete"
// "We use this for real-time notifications"
// "This helps with form validation"
```

### Common Mistakes to Avoid

```typescript
// ❌ Common interview mistakes

// 1. Forgetting to unsubscribe
component.ngOnInit() {
  this.data$.subscribe(); // Missing cleanup
}

// 2. Nested subscriptions
getData() {
  this.service1.getData().subscribe(data1 => {
    this.service2.getData().subscribe(data2 => {
      // Use operators instead!
    });
  });
}

// 3. Not handling errors
this.api.getData().subscribe(data => {
  // What if API fails?
});

// 4. Overcomplicating simple scenarios
// Don't use RxJS for everything!

// 5. Not explaining operator choice
// Always explain WHY you chose an operator
```

## Summary

### Key Interview Topics
1. **Fundamentals**: Observable, Observer, Subscription
2. **Operators**: map, filter, switchMap, mergeMap, concatMap
3. **Subjects**: Different types and use cases
4. **Memory Management**: Unsubscription strategies
5. **Error Handling**: catchError, retry patterns
6. **Performance**: shareReplay, debouncing, distinctUntilChanged
7. **Testing**: Marble testing basics
8. **Real-world Patterns**: Search, state management, API coordination

### Practice Strategy
1. **Code Examples**: Prepare working examples for each concept
2. **Explain Simply**: Practice explaining complex concepts in simple terms
3. **Draw Diagrams**: Use marble diagrams to illustrate timing
4. **Think Aloud**: Verbalize your thought process
5. **Ask Questions**: Clarify requirements before coding

### Final Tips
- Practice with online coding platforms
- Build a portfolio project using RxJS
- Stay updated with latest RxJS versions
- Join RxJS community discussions
- Practice explaining concepts to others

Good luck with your RxJS interviews! Remember, understanding the fundamentals and being able to explain your reasoning is more important than memorizing every operator.
