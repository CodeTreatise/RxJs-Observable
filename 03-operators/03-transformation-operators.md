# Transformation Operators

## Overview

Transformation operators are among the most frequently used operators in RxJS. They take an Observable as input and return a new Observable with transformed data. These operators allow you to modify, reshape, and project the values emitted by Observable streams without changing the original stream.

## Learning Objectives

After completing this lesson, you will be able to:
- Transform data streams using various projection operators
- Understand the difference between flattening strategies
- Handle nested Observables effectively
- Apply transformation operators in Angular applications
- Choose the appropriate transformation operator for different scenarios

## Core Transformation Operators

### 1. map() - Transform Each Value

The `map()` operator applies a function to each value emitted by the source Observable and emits the transformed values.

```typescript
import { of } from 'rxjs';
import { map } from 'rxjs/operators';

// Basic transformation
const numbers$ = of(1, 2, 3, 4, 5);
const doubled$ = numbers$.pipe(
  map(x => x * 2)
);
doubled$.subscribe(value => console.log(value));
// Output: 2, 4, 6, 8, 10

// Object transformation
interface User {
  id: number;
  firstName: string;
  lastName: string;
}

const users$ = of(
  { id: 1, firstName: 'John', lastName: 'Doe' },
  { id: 2, firstName: 'Jane', lastName: 'Smith' }
);

const userDisplayNames$ = users$.pipe(
  map(user => ({
    id: user.id,
    displayName: `${user.firstName} ${user.lastName}`,
    initials: `${user.firstName[0]}${user.lastName[0]}`
  }))
);

// Angular HTTP response transformation
@Injectable()
export class UserService {
  constructor(private http: HttpClient) {}

  getUsers(): Observable<UserDisplay[]> {
    return this.http.get<User[]>('/api/users').pipe(
      map(users => users.map(user => ({
        id: user.id,
        displayName: `${user.firstName} ${user.lastName}`,
        avatar: user.avatar || '/assets/default-avatar.png'
      })))
    );
  }
}
```

**Marble Diagram:**
```
Input:  -1-2-3-4-5|
map(x => x * 2)
Output: -2-4-6-8-10|
```

### 2. mapTo() - Transform to Constant Value

The `mapTo()` operator maps every emission to the same constant value.

```typescript
import { fromEvent } from 'rxjs';
import { mapTo, scan } from 'rxjs/operators';

// Convert clicks to increments
const button = document.getElementById('counter-btn');
const clicks$ = fromEvent(button, 'click');

const clickCounter$ = clicks$.pipe(
  mapTo(1), // Each click becomes the number 1
  scan((acc, value) => acc + value, 0) // Accumulate the count
);

// Angular example: Toggle state
@Component({
  template: `
    <button (click)="toggle()">Toggle: {{ isToggled$ | async }}</button>
  `
})
export class ToggleComponent {
  private toggleSubject = new Subject<void>();
  
  isToggled$ = this.toggleSubject.pipe(
    mapTo(true), // Each toggle event becomes true
    scan((current) => !current, false) // Toggle the boolean
  );

  toggle() {
    this.toggleSubject.next();
  }
}
```

### 3. pluck() - Extract Property

The `pluck()` operator extracts a nested property from each emitted object.

```typescript
import { of } from 'rxjs';
import { pluck } from 'rxjs/operators';

const users$ = of(
  { id: 1, profile: { name: 'John', email: 'john@example.com' } },
  { id: 2, profile: { name: 'Jane', email: 'jane@example.com' } }
);

// Extract nested property
const names$ = users$.pipe(
  pluck('profile', 'name')
);
names$.subscribe(name => console.log(name));
// Output: 'John', 'Jane'

// Angular form example
@Component({})
export class FormComponent {
  form = new FormGroup({
    user: new FormGroup({
      email: new FormControl(''),
      profile: new FormGroup({
        firstName: new FormControl(''),
        lastName: new FormControl('')
      })
    })
  });

  firstName$ = this.form.valueChanges.pipe(
    pluck('user', 'profile', 'firstName')
  );
}
```

**Note:** `pluck()` is deprecated in RxJS 7+. Use `map()` with property access instead:

```typescript
// Modern approach (RxJS 7+)
const names$ = users$.pipe(
  map(user => user.profile.name)
);
```

### 4. switchMap() - Switch to New Observable

The `switchMap()` operator maps each value to an Observable, then flattens the result by switching to the latest inner Observable and canceling previous ones.

```typescript
import { fromEvent, of, timer } from 'rxjs';
import { switchMap, map } from 'rxjs/operators';

// Basic example
const button = document.getElementById('search-btn');
const clicks$ = fromEvent(button, 'click');

const searchResults$ = clicks$.pipe(
  switchMap(() => 
    timer(0, 1000).pipe(
      map(i => `Search result ${i}`)
    )
  )
);

// Angular search implementation
@Component({
  template: `
    <input [(ngModel)]="searchTerm" placeholder="Search...">
    <div *ngFor="let result of searchResults$ | async">
      {{ result.title }}
    </div>
  `
})
export class SearchComponent {
  searchTerm = '';
  
  searchResults$ = this.searchTermChanges$.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap(term => 
      term ? this.searchService.search(term) : of([])
    )
  );

  private searchTermChanges$ = new BehaviorSubject('');

  ngOnInit() {
    // Monitor search term changes
    this.searchTermChanges$.next(this.searchTerm);
  }

  onSearchTermChange(term: string) {
    this.searchTermChanges$.next(term);
  }
}

@Injectable()
export class SearchService {
  constructor(private http: HttpClient) {}

  search(term: string): Observable<SearchResult[]> {
    return this.http.get<SearchResult[]>(`/api/search?q=${term}`);
  }
}
```

**Marble Diagram:**
```
Input:    -a-----b--c----|
switchMap(x => timer(0, 10).pipe(take(3)))
Inner a:   0-1-2|
Inner b:         0-1-2|
Inner c:            0-1-2|
Output:   -0-1-2-0-1-0-1-2|
                  ^     ^ cancellation
```

### 5. mergeMap() - Merge Multiple Observables

The `mergeMap()` operator maps each value to an Observable, then merges all inner Observables concurrently.

```typescript
import { of, interval } from 'rxjs';
import { mergeMap, map, take } from 'rxjs/operators';

// Process multiple requests concurrently
const userIds$ = of(1, 2, 3);

const userDetails$ = userIds$.pipe(
  mergeMap(id => 
    this.http.get<User>(`/api/users/${id}`)
  )
);

// File upload example
@Component({})
export class FileUploadComponent {
  uploadFiles(files: FileList): Observable<UploadResult[]> {
    return from(Array.from(files)).pipe(
      mergeMap(file => this.uploadFile(file)),
      toArray() // Collect all results
    );
  }

  private uploadFile(file: File): Observable<UploadResult> {
    const formData = new FormData();
    formData.append('file', file);
    
    return this.http.post<UploadResult>('/api/upload', formData);
  }
}

// Rate limiting with mergeMap
const rateLimitedRequests$ = requests$.pipe(
  mergeMap(request => 
    this.processRequest(request),
    3 // Maximum 3 concurrent requests
  )
);
```

**Marble Diagram:**
```
Input:   -a-b-c----|
mergeMap(x => timer(0, 10).pipe(take(3)))
Inner a:  0-1-2|
Inner b:   0-1-2|
Inner c:    0-1-2|
Output:  -001123-2|
```

### 6. concatMap() - Concatenate Observables

The `concatMap()` operator maps each value to an Observable and concatenates them sequentially, waiting for each inner Observable to complete before starting the next.

```typescript
import { of } from 'rxjs';
import { concatMap, delay } from 'rxjs/operators';

// Sequential processing
const tasks$ = of('Task 1', 'Task 2', 'Task 3');

const sequentialExecution$ = tasks$.pipe(
  concatMap(task => 
    of(`Processing ${task}`).pipe(
      delay(1000) // Simulate async work
    )
  )
);

// Angular form submission queue
@Component({})
export class FormSubmissionComponent {
  private submissionQueue$ = new Subject<FormData>();

  constructor() {
    // Process submissions sequentially
    this.submissionQueue$.pipe(
      concatMap(formData => this.submitForm(formData))
    ).subscribe(
      result => console.log('Form submitted:', result),
      error => console.error('Submission failed:', error)
    );
  }

  submitForm(formData: FormData): Observable<SubmissionResult> {
    return this.http.post<SubmissionResult>('/api/submit', formData);
  }

  onSubmit(formData: FormData) {
    this.submissionQueue$.next(formData);
  }
}
```

**Marble Diagram:**
```
Input:    -a-b-c----|
concatMap(x => timer(0, 10).pipe(take(3)))
Inner a:   0-1-2|
Inner b:         0-1-2|
Inner c:               0-1-2|
Output:   -0-1-2-0-1-2-0-1-2|
```

### 7. exhaustMap() - Ignore New Values While Processing

The `exhaustMap()` operator maps to an inner Observable and ignores new values while the current inner Observable is still active.

```typescript
import { fromEvent } from 'rxjs';
import { exhaustMap } from 'rxjs/operators';

// Prevent multiple API calls
const button = document.getElementById('save-btn');
const clicks$ = fromEvent(button, 'click');

const saveOperation$ = clicks$.pipe(
  exhaustMap(() => 
    this.http.post('/api/save', this.formData).pipe(
      finalize(() => console.log('Save operation completed'))
    )
  )
);

// Login form example
@Component({
  template: `
    <form (ngSubmit)="onLogin()">
      <input [(ngModel)]="credentials.username" placeholder="Username">
      <input [(ngModel)]="credentials.password" type="password" placeholder="Password">
      <button type="submit" [disabled]="isLoggingIn$ | async">
        {{ (isLoggingIn$ | async) ? 'Logging in...' : 'Login' }}
      </button>
    </form>
  `
})
export class LoginComponent {
  private loginAttempts$ = new Subject<LoginCredentials>();
  credentials = { username: '', password: '' };

  loginResult$ = this.loginAttempts$.pipe(
    exhaustMap(creds => 
      this.authService.login(creds).pipe(
        catchError(error => of({ success: false, error }))
      )
    )
  );

  isLoggingIn$ = this.loginResult$.pipe(
    map(result => result === null), // null while in progress
    startWith(false)
  );

  onLogin() {
    this.loginAttempts$.next(this.credentials);
  }
}
```

**Marble Diagram:**
```
Input:     -a-b-c-d-e----|
exhaustMap(x => timer(0, 15).pipe(take(3)))
Inner a:    0--1--2|
Ignored:     b c d
Inner e:              0--1--2|
Output:    -0--1--2------0--1--2|
```

### 8. expand() - Recursive Observable Expansion

The `expand()` operator recursively applies a projection function to each emitted value and merges the results.

```typescript
import { of } from 'rxjs';
import { expand, map, take, takeWhile } from 'rxjs/operators';

// Recursive data fetching (pagination)
const fetchAllPages$ = of(1).pipe(
  expand(page => 
    this.http.get<PageData>(`/api/data?page=${page}`).pipe(
      map(response => response.nextPage),
      takeWhile(nextPage => nextPage !== null)
    )
  ),
  take(10) // Prevent infinite recursion
);

// File system traversal
interface Directory {
  name: string;
  subdirectories: string[];
}

const traverseDirectory$ = of('/root').pipe(
  expand(path => 
    this.fileService.getDirectory(path).pipe(
      mergeMap(dir => from(dir.subdirectories))
    )
  ),
  take(100) // Limit depth
);

// Angular infinite scroll
@Component({})
export class InfiniteScrollComponent {
  private loadMore$ = new Subject<number>();
  
  allItems$ = this.loadMore$.pipe(
    startWith(1), // Start with page 1
    expand(page => 
      this.dataService.getPage(page).pipe(
        map(response => response.hasMore ? page + 1 : EMPTY),
        takeWhile(nextPage => nextPage !== EMPTY)
      )
    ),
    mergeMap(page => this.dataService.getPage(page)),
    scan((allItems, newItems) => [...allItems, ...newItems], [])
  );

  loadMoreItems() {
    this.loadMore$.next();
  }
}
```

## Advanced Transformation Patterns

### 1. Conditional Transformation

```typescript
import { of } from 'rxjs';
import { map, switchMap } from 'rxjs/operators';

// Transform based on conditions
const processUser$ = user$.pipe(
  switchMap(user => {
    if (user.isAdmin) {
      return this.getAdminData(user.id);
    } else if (user.isPremium) {
      return this.getPremiumData(user.id);
    } else {
      return of(this.getBasicData(user));
    }
  })
);

// Using mergeMap with conditional logic
const conditionalTransform$ = source$.pipe(
  mergeMap(value => 
    value > 10 
      ? of(value).pipe(map(x => x * 2))
      : of(value).pipe(map(x => x + 1))
  )
);
```

### 2. Error Recovery in Transformations

```typescript
// Graceful error handling in transformations
const robustTransformation$ = source$.pipe(
  mergeMap(item => 
    this.processItem(item).pipe(
      catchError(error => {
        console.error(`Failed to process ${item}:`, error);
        return of(this.getDefaultValue(item));
      })
    )
  )
);

// Retry with exponential backoff
const retryableTransformation$ = source$.pipe(
  mergeMap(item => 
    this.apiCall(item).pipe(
      retryWhen(errors => 
        errors.pipe(
          scan((retryCount, error) => {
            if (retryCount >= 3) throw error;
            return retryCount + 1;
          }, 0),
          delay(1000)
        )
      )
    )
  )
);
```

### 3. Performance Optimization

```typescript
// Debounced transformation
const optimizedSearch$ = searchTerm$.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(term => 
    term.length > 2 
      ? this.searchService.search(term)
      : of([])
  )
);

// Cached transformations
@Injectable()
export class CachedDataService {
  private cache = new Map<string, Observable<any>>();

  getData(id: string): Observable<Data> {
    if (!this.cache.has(id)) {
      const data$ = this.http.get<Data>(`/api/data/${id}`).pipe(
        shareReplay(1) // Cache the result
      );
      this.cache.set(id, data$);
    }
    return this.cache.get(id)!;
  }
}
```

## Real-World Angular Examples

### 1. Dynamic Form Validation

```typescript
@Component({
  template: `
    <form [formGroup]="userForm">
      <input formControlName="email" placeholder="Email">
      <div *ngIf="emailValidation$ | async as validation">
        <span [class]="validation.class">{{ validation.message }}</span>
      </div>
    </form>
  `
})
export class DynamicValidationComponent {
  userForm = new FormGroup({
    email: new FormControl('')
  });

  emailValidation$ = this.userForm.get('email')!.valueChanges.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap(email => 
      email ? this.validateEmail(email) : of(null)
    ),
    map(result => result ? {
      class: result.isValid ? 'text-success' : 'text-error',
      message: result.message
    } : null)
  );

  private validateEmail(email: string): Observable<ValidationResult> {
    return this.http.post<ValidationResult>('/api/validate-email', { email });
  }
}
```

### 2. Real-time Data Dashboard

```typescript
@Component({
  template: `
    <div class="dashboard">
      <div *ngFor="let metric of metrics$ | async" class="metric-card">
        <h3>{{ metric.name }}</h3>
        <p>{{ metric.value | number }}</p>
        <small>{{ metric.trend }}</small>
      </div>
    </div>
  `
})
export class DashboardComponent {
  private refreshTrigger$ = interval(30000); // Refresh every 30 seconds

  metrics$ = this.refreshTrigger$.pipe(
    startWith(0), // Initial load
    switchMap(() => this.dataService.getMetrics()),
    map(rawMetrics => rawMetrics.map(metric => ({
      ...metric,
      trend: this.calculateTrend(metric.current, metric.previous)
    }))),
    catchError(error => {
      console.error('Failed to load metrics:', error);
      return of([]);
    })
  );

  private calculateTrend(current: number, previous: number): string {
    const change = ((current - previous) / previous) * 100;
    return change > 0 ? `↑ ${change.toFixed(1)}%` : `↓ ${Math.abs(change).toFixed(1)}%`;
  }
}
```

### 3. Progressive Data Loading

```typescript
@Component({})
export class ProgressiveLoadingComponent {
  // Load data progressively
  userData$ = this.route.params.pipe(
    map(params => params['userId']),
    switchMap(userId => 
      // Load basic user info first
      this.userService.getUserBasic(userId).pipe(
        // Then load additional details
        mergeMap(basicUser => 
          forkJoin({
            user: of(basicUser),
            profile: this.userService.getUserProfile(userId),
            preferences: this.userService.getUserPreferences(userId),
            activity: this.userService.getUserActivity(userId)
          })
        )
      )
    ),
    shareReplay(1)
  );

  // Extract specific data streams
  basicInfo$ = this.userData$.pipe(map(data => data.user));
  profile$ = this.userData$.pipe(map(data => data.profile));
  preferences$ = this.userData$.pipe(map(data => data.preferences));
  activity$ = this.userData$.pipe(map(data => data.activity));
}
```

## Best Practices

### 1. Choose the Right Flattening Strategy

```typescript
// ✅ Use switchMap for: Search, navigation, user interactions
searchResults$ = searchTerm$.pipe(
  switchMap(term => this.search(term))
);

// ✅ Use mergeMap for: Independent parallel operations
uploadResults$ = files$.pipe(
  mergeMap(file => this.upload(file))
);

// ✅ Use concatMap for: Sequential operations, maintaining order
processQueue$ = tasks$.pipe(
  concatMap(task => this.process(task))
);

// ✅ Use exhaustMap for: Preventing duplicate operations
saveOperation$ = saveClicks$.pipe(
  exhaustMap(() => this.save())
);
```

### 2. Handle Errors Gracefully

```typescript
// ✅ Good: Handle errors at the right level
const robustOperation$ = source$.pipe(
  mergeMap(item => 
    this.processItem(item).pipe(
      catchError(error => of(this.fallbackValue(item)))
    )
  )
);

// ❌ Bad: Letting errors kill the stream
const fragileOperation$ = source$.pipe(
  mergeMap(item => this.processItem(item)) // Uncaught errors will terminate
);
```

### 3. Optimize Performance

```typescript
// ✅ Good: Use shareReplay for expensive operations
const expensiveData$ = trigger$.pipe(
  switchMap(() => this.expensiveOperation()),
  shareReplay(1)
);

// ✅ Good: Debounce user input
const searchResults$ = searchInput$.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(term => this.search(term))
);
```

## Common Pitfalls

### 1. Memory Leaks

```typescript
// ❌ Bad: Not unsubscribing from long-running operations
@Component({})
export class LeakyComponent {
  ngOnInit() {
    interval(1000).pipe(
      switchMap(() => this.getData())
    ).subscribe(); // Never unsubscribed!
  }
}

// ✅ Good: Proper subscription management
@Component({})
export class ProperComponent implements OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit() {
    interval(1000).pipe(
      switchMap(() => this.getData()),
      takeUntil(this.destroy$)
    ).subscribe();
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### 2. Nested Subscriptions

```typescript
// ❌ Bad: Nested subscriptions
source$.subscribe(value => {
  this.processValue(value).subscribe(result => {
    console.log(result);
  });
});

// ✅ Good: Use flattening operators
source$.pipe(
  switchMap(value => this.processValue(value))
).subscribe(result => {
  console.log(result);
});
```

## Exercises

### Exercise 1: User Search with Caching
Implement a user search feature that caches results and handles loading states.

### Exercise 2: File Upload Queue
Create a file upload system that processes files sequentially with progress tracking.

### Exercise 3: Real-time Chat Transformation
Build a chat message processor that handles different message types and user mentions.

## Summary

Transformation operators are essential for reactive programming:

- **map()**: Transform individual values
- **switchMap()**: Switch to new Observable, cancel previous
- **mergeMap()**: Merge multiple Observables concurrently
- **concatMap()**: Process Observables sequentially
- **exhaustMap()**: Ignore new values while processing
- **expand()**: Recursive Observable expansion

Choose the right operator based on your concurrency requirements and error handling needs.

## Next Steps

In the next lesson, we'll explore **Filtering Operators**, which allow you to selectively emit values based on specific criteria.
