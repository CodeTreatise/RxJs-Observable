# Filtering Operators

## Overview

Filtering operators are essential for controlling which values are emitted by an Observable stream. They allow you to selectively pass through values based on specific criteria, conditions, or timing requirements. These operators help you create more efficient and focused data streams by eliminating unwanted emissions.

## Learning Objectives

After completing this lesson, you will be able to:
- Filter Observable emissions based on various criteria
- Implement conditional logic in reactive streams
- Optimize performance by reducing unnecessary emissions
- Handle duplicate values and timing-based filtering
- Apply filtering operators in real-world Angular scenarios

## Core Filtering Operators

### 1. filter() - Conditional Filtering

The `filter()` operator emits only those values that pass a provided predicate function.

```typescript
import { of, fromEvent } from 'rxjs';
import { filter, map } from 'rxjs/operators';

// Basic filtering
const numbers$ = of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
const evenNumbers$ = numbers$.pipe(
  filter(n => n % 2 === 0)
);
evenNumbers$.subscribe(value => console.log(value));
// Output: 2, 4, 6, 8, 10

// Object filtering
interface User {
  id: number;
  name: string;
  isActive: boolean;
  role: 'admin' | 'user' | 'guest';
}

const users$ = of(
  { id: 1, name: 'Alice', isActive: true, role: 'admin' },
  { id: 2, name: 'Bob', isActive: false, role: 'user' },
  { id: 3, name: 'Charlie', isActive: true, role: 'user' },
  { id: 4, name: 'Diana', isActive: true, role: 'guest' }
) as Observable<User>;

const activeUsers$ = users$.pipe(
  filter(user => user.isActive)
);

const adminUsers$ = users$.pipe(
  filter(user => user.role === 'admin' && user.isActive)
);

// Angular form filtering
@Component({})
export class FormComponent {
  form = new FormGroup({
    email: new FormControl(''),
    age: new FormControl(0)
  });

  // Only emit valid email values
  validEmails$ = this.form.get('email')!.valueChanges.pipe(
    filter(email => this.isValidEmail(email))
  );

  // Only emit when age is 18 or older
  adultsOnly$ = this.form.get('age')!.valueChanges.pipe(
    filter(age => age >= 18)
  );

  private isValidEmail(email: string): boolean {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(email);
  }
}
```

**Marble Diagram:**
```
Input:  -1-2-3-4-5-6|
filter(x => x % 2 === 0)
Output: ---2---4---6|
```

### 2. take() - Take First N Values

The `take()` operator emits only the first `n` values from the source Observable.

```typescript
import { interval, fromEvent } from 'rxjs';
import { take, map } from 'rxjs/operators';

// Take first 5 emissions
const first5Numbers$ = interval(1000).pipe(
  take(5)
);
first5Numbers$.subscribe(value => console.log(value));
// Output: 0, 1, 2, 3, 4 (then completes)

// Take first 3 clicks
const button = document.getElementById('click-btn');
const first3Clicks$ = fromEvent(button, 'click').pipe(
  take(3),
  map(() => 'Click!')
);

// Angular onboarding flow
@Component({})
export class OnboardingComponent implements OnInit {
  private stepCompleted$ = new Subject<string>();
  
  onboardingSteps$ = this.stepCompleted$.pipe(
    take(3), // Only first 3 steps
    scan((steps, step) => [...steps, step], [] as string[])
  );

  ngOnInit() {
    this.onboardingSteps$.subscribe(steps => {
      if (steps.length === 3) {
        this.completeOnboarding();
      }
    });
  }

  completeStep(step: string) {
    this.stepCompleted$.next(step);
  }

  private completeOnboarding() {
    console.log('Onboarding completed!');
  }
}
```

**Marble Diagram:**
```
Input:  -1-2-3-4-5-6-7|
take(3)
Output: -1-2-3|
```

### 3. takeWhile() - Take While Condition is True

The `takeWhile()` operator emits values as long as a provided predicate returns true.

```typescript
import { interval } from 'rxjs';
import { takeWhile, map } from 'rxjs/operators';

// Take while values are less than 5
const lessThan5$ = interval(1000).pipe(
  takeWhile(value => value < 5)
);

// Include the last value that made condition false
const lessThan5Inclusive$ = interval(1000).pipe(
  takeWhile(value => value < 5, true) // inclusive: true
);

// Real-world example: Loading progress
@Component({
  template: `
    <div class="progress-bar">
      <div class="progress" [style.width.%]="progress$ | async"></div>
    </div>
    <p>{{ (isCompleted$ | async) ? 'Completed!' : 'Loading...' }}</p>
  `
})
export class ProgressComponent {
  progress$ = interval(100).pipe(
    map(i => i * 2), // 2% every 100ms
    takeWhile(progress => progress <= 100, true)
  );

  isCompleted$ = this.progress$.pipe(
    map(progress => progress >= 100),
    filter(completed => completed),
    take(1)
  );
}

// Gaming example: Health monitoring
@Injectable()
export class GameService {
  private healthSubject = new BehaviorSubject(100);
  
  health$ = this.healthSubject.asObservable();
  
  gameActive$ = this.health$.pipe(
    takeWhile(health => health > 0)
  );

  gameOver$ = this.health$.pipe(
    filter(health => health <= 0),
    take(1)
  );

  takeDamage(amount: number) {
    const currentHealth = this.healthSubject.value;
    this.healthSubject.next(Math.max(0, currentHealth - amount));
  }
}
```

### 4. skip() - Skip First N Values

The `skip()` operator skips the first `n` values from the source Observable.

```typescript
import { interval, of } from 'rxjs';
import { skip } from 'rxjs/operators';

// Skip first 3 values
const skipFirst3$ = interval(1000).pipe(
  skip(3),
  take(5)
);
skipFirst3$.subscribe(value => console.log(value));
// Output: 3, 4, 5, 6, 7

// Skip initial loading state
@Component({})
export class DataComponent {
  private dataSubject = new BehaviorSubject<Data | null>(null);
  
  // Skip the initial null value
  data$ = this.dataSubject.pipe(
    skip(1), // Skip initial null
    filter(data => data !== null)
  );

  ngOnInit() {
    this.loadData();
  }

  private loadData() {
    this.http.get<Data>('/api/data').subscribe(
      data => this.dataSubject.next(data)
    );
  }
}
```

**Marble Diagram:**
```
Input:  -1-2-3-4-5-6|
skip(2)
Output: -----3-4-5-6|
```

### 5. skipWhile() - Skip While Condition is True

The `skipWhile()` operator skips values while the predicate returns true, then emits all subsequent values.

```typescript
import { of } from 'rxjs';
import { skipWhile } from 'rxjs/operators';

// Skip negative numbers
const numbers$ = of(-3, -2, -1, 0, 1, 2, 3);
const positiveAndZero$ = numbers$.pipe(
  skipWhile(n => n < 0)
);
positiveAndZero$.subscribe(value => console.log(value));
// Output: 0, 1, 2, 3

// Skip invalid states
@Component({})
export class AuthComponent {
  private authState$ = new BehaviorSubject<AuthState>('initializing');
  
  validAuthStates$ = this.authState$.pipe(
    skipWhile(state => state === 'initializing'),
    distinctUntilChanged()
  );

  authenticatedUser$ = this.validAuthStates$.pipe(
    filter(state => state === 'authenticated'),
    switchMap(() => this.userService.getCurrentUser())
  );
}
```

### 6. distinct() - Emit Only Unique Values

The `distinct()` operator emits only values that haven't been emitted before.

```typescript
import { of } from 'rxjs';
import { distinct } from 'rxjs/operators';

// Remove duplicates
const numbers$ = of(1, 2, 2, 3, 1, 4, 3, 5);
const uniqueNumbers$ = numbers$.pipe(
  distinct()
);
uniqueNumbers$.subscribe(value => console.log(value));
// Output: 1, 2, 3, 4, 5

// Distinct by key
interface Product {
  id: number;
  name: string;
  category: string;
}

const products$ = of(
  { id: 1, name: 'Laptop', category: 'Electronics' },
  { id: 2, name: 'Phone', category: 'Electronics' },
  { id: 3, name: 'Book', category: 'Media' },
  { id: 4, name: 'Tablet', category: 'Electronics' }
);

const uniqueCategories$ = products$.pipe(
  distinct(product => product.category),
  map(product => product.category)
);

// Angular search history
@Injectable()
export class SearchService {
  private searchHistory$ = new BehaviorSubject<string[]>([]);
  
  addToHistory(term: string) {
    const currentHistory = this.searchHistory$.value;
    const updatedHistory = [term, ...currentHistory].slice(0, 10); // Keep last 10
    
    const uniqueHistory$ = from(updatedHistory).pipe(
      distinct(),
      toArray()
    );
    
    uniqueHistory$.subscribe(history => 
      this.searchHistory$.next(history)
    );
  }

  getHistory() {
    return this.searchHistory$.asObservable();
  }
}
```

### 7. distinctUntilChanged() - Emit When Changed

The `distinctUntilChanged()` operator emits only when the current value is different from the previous value.

```typescript
import { of, fromEvent } from 'rxjs';
import { distinctUntilChanged, map } from 'rxjs/operators';

// Remove consecutive duplicates
const values$ = of(1, 1, 2, 2, 2, 3, 1, 1);
const changedValues$ = values$.pipe(
  distinctUntilChanged()
);
changedValues$.subscribe(value => console.log(value));
// Output: 1, 2, 3, 1

// Form validation optimization
@Component({})
export class OptimizedFormComponent {
  form = new FormGroup({
    username: new FormControl('')
  });

  // Only validate when username actually changes
  usernameValidation$ = this.form.get('username')!.valueChanges.pipe(
    distinctUntilChanged(),
    debounceTime(300),
    switchMap(username => 
      username ? this.validateUsername(username) : of(null)
    )
  );

  // Window resize optimization
  @HostListener('window:resize', ['$event'])
  onResize(event: Event) {
    fromEvent(window, 'resize').pipe(
      map(() => ({ width: window.innerWidth, height: window.innerHeight })),
      distinctUntilChanged((prev, curr) => 
        prev.width === curr.width && prev.height === curr.height
      ),
      debounceTime(250)
    ).subscribe(size => {
      this.handleResize(size);
    });
  }

  private validateUsername(username: string): Observable<ValidationResult> {
    return this.http.post<ValidationResult>('/api/validate-username', { username });
  }

  private handleResize(size: { width: number; height: number }) {
    console.log('Window resized:', size);
  }
}
```

### 8. first() - Emit First Value or First Matching

The `first()` operator emits the first value, or the first value that matches a predicate.

```typescript
import { of, fromEvent } from 'rxjs';
import { first, filter } from 'rxjs/operators';

// First value
const numbers$ = of(1, 2, 3, 4, 5);
const firstNumber$ = numbers$.pipe(first());
firstNumber$.subscribe(value => console.log(value));
// Output: 1

// First matching value
const firstEven$ = numbers$.pipe(
  first(n => n % 2 === 0)
);
firstEven$.subscribe(value => console.log(value));
// Output: 2

// First click on specific element
const button = document.getElementById('special-btn');
const firstSpecialClick$ = fromEvent(document, 'click').pipe(
  first(event => event.target === button)
);

// Angular route resolution
@Injectable()
export class DataResolver implements Resolve<Data> {
  constructor(
    private dataService: DataService,
    private router: Router
  ) {}

  resolve(route: ActivatedRouteSnapshot): Observable<Data> {
    const id = route.params['id'];
    
    return this.dataService.getData(id).pipe(
      first(), // Complete after first emission
      catchError(error => {
        this.router.navigate(['/not-found']);
        return EMPTY;
      })
    );
  }
}
```

### 9. last() - Emit Last Value

The `last()` operator emits the last value from the source Observable.

```typescript
import { of, interval } from 'rxjs';
import { last, take } from 'rxjs/operators';

// Last value
const numbers$ = of(1, 2, 3, 4, 5);
const lastNumber$ = numbers$.pipe(last());
lastNumber$.subscribe(value => console.log(value));
// Output: 5

// Last value matching predicate
const lastEven$ = numbers$.pipe(
  last(n => n % 2 === 0)
);
lastEven$.subscribe(value => console.log(value));
// Output: 4

// Final status update
@Component({})
export class BatchProcessComponent {
  private statusUpdates$ = new Subject<string>();
  
  finalStatus$ = this.statusUpdates$.pipe(
    last(),
    catchError(() => of('Process failed'))
  );

  processBatch(items: any[]) {
    from(items).pipe(
      concatMap(item => this.processItem(item)),
      tap(result => this.statusUpdates$.next(result.status)),
      last(),
      finalize(() => this.statusUpdates$.complete())
    ).subscribe(
      finalResult => console.log('Batch completed:', finalResult),
      error => console.error('Batch failed:', error)
    );
  }

  private processItem(item: any): Observable<ProcessResult> {
    return this.http.post<ProcessResult>('/api/process', item);
  }
}
```

### 10. single() - Emit Single Value or Error

The `single()` operator emits the single value that matches a predicate, or errors if zero or multiple values match.

```typescript
import { of, throwError } from 'rxjs';
import { single, catchError } from 'rxjs/operators';

// Single matching value
const numbers$ = of(1, 2, 3, 4, 5);
const singleEven$ = numbers$.pipe(
  single(n => n === 3)
);

// Error handling for multiple matches
const multipleEvens$ = numbers$.pipe(
  single(n => n % 2 === 0), // Will error: multiple even numbers
  catchError(error => {
    console.error('Expected single value, got multiple');
    return of(null);
  })
);

// Find unique user
@Injectable()
export class UserService {
  findUserByEmail(email: string): Observable<User> {
    return this.http.get<User[]>(`/api/users?email=${email}`).pipe(
      mergeMap(users => from(users)),
      single(), // Ensure exactly one user found
      catchError(error => {
        if (error.name === 'EmptyError') {
          return throwError('User not found');
        } else if (error.name === 'SequenceError') {
          return throwError('Multiple users found with same email');
        }
        return throwError(error);
      })
    );
  }
}
```

## Advanced Filtering Patterns

### 1. Time-Based Filtering

```typescript
import { interval, fromEvent } from 'rxjs';
import { filter, timestamp, map } from 'rxjs/operators';

// Filter based on time
const businessHoursOnly$ = interval(1000).pipe(
  timestamp(),
  filter(({ timestamp }) => {
    const hour = new Date(timestamp).getHours();
    return hour >= 9 && hour <= 17; // 9 AM to 5 PM
  }),
  map(({ value }) => value)
);

// Rate limiting clicks
const rateLimitedClicks$ = fromEvent(button, 'click').pipe(
  timestamp(),
  filter(({ timestamp }, index, array) => {
    if (index === 0) return true;
    const prevTimestamp = array[index - 1].timestamp;
    return timestamp - prevTimestamp > 1000; // Min 1 second between clicks
  })
);
```

### 2. Complex Conditional Filtering

```typescript
// Multi-condition filtering
const validTransactions$ = transactions$.pipe(
  filter(transaction => 
    transaction.amount > 0 &&
    transaction.status === 'pending' &&
    transaction.userId === currentUserId &&
    this.isValidCurrency(transaction.currency)
  )
);

// State-based filtering
@Component({})
export class ConditionalFilterComponent {
  private isLoggedIn$ = new BehaviorSubject(false);
  private userRole$ = new BehaviorSubject<string>('guest');

  adminActions$ = actions$.pipe(
    withLatestFrom(this.isLoggedIn$, this.userRole$),
    filter(([action, isLoggedIn, role]) => 
      isLoggedIn && role === 'admin'
    ),
    map(([action]) => action)
  );
}
```

### 3. Performance-Optimized Filtering

```typescript
// Memoized filtering
class OptimizedFilterService {
  private filterCache = new Map<string, (item: any) => boolean>();

  createOptimizedFilter(predicate: string): OperatorFunction<any, any> {
    if (!this.filterCache.has(predicate)) {
      this.filterCache.set(predicate, new Function('item', `return ${predicate}`));
    }
    
    const filterFn = this.filterCache.get(predicate)!;
    return filter(filterFn);
  }
}

// Batch filtering
const batchFilter$ = source$.pipe(
  bufferTime(1000),
  map(batch => batch.filter(item => this.expensiveFilterCheck(item))),
  mergeMap(filteredBatch => from(filteredBatch))
);
```

## Real-World Angular Examples

### 1. Smart Search Component

```typescript
@Component({
  template: `
    <input 
      #searchInput 
      placeholder="Search products..."
      [value]="searchTerm$ | async">
    
    <div *ngIf="isSearching$ | async" class="loading">Searching...</div>
    
    <div class="results">
      <div *ngFor="let product of searchResults$ | async" class="product-card">
        <h3>{{ product.name }}</h3>
        <p>{{ product.description }}</p>
        <span class="price">{{ product.price | currency }}</span>
      </div>
    </div>
    
    <div *ngIf="noResults$ | async" class="no-results">
      No products found for "{{ searchTerm$ | async }}"
    </div>
  `
})
export class SmartSearchComponent implements AfterViewInit {
  @ViewChild('searchInput') searchInput!: ElementRef<HTMLInputElement>;

  searchTerm$!: Observable<string>;
  searchResults$!: Observable<Product[]>;
  isSearching$!: Observable<boolean>;
  noResults$!: Observable<boolean>;

  ngAfterViewInit() {
    this.searchTerm$ = fromEvent(this.searchInput.nativeElement, 'input').pipe(
      map((event: Event) => (event.target as HTMLInputElement).value),
      distinctUntilChanged(),
      debounceTime(300),
      shareReplay(1)
    );

    const searchWithResults$ = this.searchTerm$.pipe(
      filter(term => term.length >= 2), // Only search with 2+ characters
      switchMap(term => 
        this.searchService.search(term).pipe(
          map(results => ({ term, results })),
          startWith({ term, results: null }) // Loading state
        )
      ),
      share()
    );

    this.searchResults$ = searchWithResults$.pipe(
      filter(({ results }) => results !== null),
      map(({ results }) => results!),
      startWith([])
    );

    this.isSearching$ = searchWithResults$.pipe(
      map(({ results }) => results === null),
      startWith(false)
    );

    this.noResults$ = combineLatest([
      this.searchResults$,
      this.searchTerm$,
      this.isSearching$
    ]).pipe(
      map(([results, term, isSearching]) => 
        !isSearching && term.length >= 2 && results.length === 0
      )
    );
  }
}
```

### 2. Permission-Based Content Filter

```typescript
@Injectable()
export class PermissionService {
  private permissions$ = new BehaviorSubject<string[]>([]);

  hasPermission(permission: string): Observable<boolean> {
    return this.permissions$.pipe(
      map(permissions => permissions.includes(permission)),
      distinctUntilChanged()
    );
  }

  hasAnyPermission(requiredPermissions: string[]): Observable<boolean> {
    return this.permissions$.pipe(
      map(permissions => 
        requiredPermissions.some(req => permissions.includes(req))
      ),
      distinctUntilChanged()
    );
  }
}

@Component({
  template: `
    <div *ngFor="let item of filteredContent$ | async" class="content-item">
      <h3>{{ item.title }}</h3>
      <p>{{ item.description }}</p>
      <button *ngIf="canEdit$ | async" (click)="editItem(item)">Edit</button>
      <button *ngIf="canDelete$ | async" (click)="deleteItem(item)">Delete</button>
    </div>
  `
})
export class ContentComponent {
  content$ = this.contentService.getContent();
  
  filteredContent$ = combineLatest([
    this.content$,
    this.permissionService.permissions$
  ]).pipe(
    map(([content, permissions]) => 
      content.filter(item => 
        this.hasRequiredPermission(item.requiredPermissions, permissions)
      )
    )
  );

  canEdit$ = this.permissionService.hasPermission('content.edit');
  canDelete$ = this.permissionService.hasPermission('content.delete');

  private hasRequiredPermission(required: string[], userPermissions: string[]): boolean {
    return required.every(perm => userPermissions.includes(perm));
  }
}
```

### 3. Real-time Data Filtering Dashboard

```typescript
@Component({
  template: `
    <div class="filters">
      <select [(ngModel)]="selectedCategory">
        <option value="">All Categories</option>
        <option *ngFor="let cat of categories$ | async" [value]="cat">
          {{ cat }}
        </option>
      </select>
      
      <input 
        type="range" 
        min="0" 
        max="1000" 
        [(ngModel)]="maxPrice"
        (input)="onPriceChange($event)">
      <span>Max Price: {{ maxPrice | currency }}</span>
    </div>

    <div class="metrics">
      <div class="metric">
        <h4>Total Items</h4>
        <span>{{ totalCount$ | async }}</span>
      </div>
      <div class="metric">
        <h4>Filtered Items</h4>
        <span>{{ filteredCount$ | async }}</span>
      </div>
      <div class="metric">
        <h4>Average Price</h4>
        <span>{{ averagePrice$ | async | currency }}</span>
      </div>
    </div>

    <div class="items">
      <div *ngFor="let item of filteredItems$ | async" class="item-card">
        <h3>{{ item.name }}</h3>
        <p>Category: {{ item.category }}</p>
        <p>Price: {{ item.price | currency }}</p>
      </div>
    </div>
  `
})
export class FilteredDashboardComponent {
  selectedCategory = '';
  maxPrice = 1000;

  private categoryFilter$ = new BehaviorSubject<string>('');
  private priceFilter$ = new BehaviorSubject<number>(1000);

  // Real-time data stream
  allItems$ = timer(0, 5000).pipe( // Update every 5 seconds
    switchMap(() => this.dataService.getItems()),
    shareReplay(1)
  );

  categories$ = this.allItems$.pipe(
    map(items => [...new Set(items.map(item => item.category))]),
    distinctUntilChanged()
  );

  filteredItems$ = combineLatest([
    this.allItems$,
    this.categoryFilter$,
    this.priceFilter$
  ]).pipe(
    map(([items, category, maxPrice]) => items.filter(item => 
      (category === '' || item.category === category) &&
      item.price <= maxPrice
    )),
    shareReplay(1)
  );

  totalCount$ = this.allItems$.pipe(
    map(items => items.length)
  );

  filteredCount$ = this.filteredItems$.pipe(
    map(items => items.length)
  );

  averagePrice$ = this.filteredItems$.pipe(
    map(items => items.length > 0 
      ? items.reduce((sum, item) => sum + item.price, 0) / items.length 
      : 0
    )
  );

  onCategoryChange(category: string) {
    this.categoryFilter$.next(category);
  }

  onPriceChange(event: Event) {
    const price = +(event.target as HTMLInputElement).value;
    this.priceFilter$.next(price);
  }
}
```

## Best Practices

### 1. Choose the Right Filtering Operator

```typescript
// ✅ Use filter() for conditional logic
const validItems$ = items$.pipe(
  filter(item => item.isValid && item.price > 0)
);

// ✅ Use distinctUntilChanged() to prevent unnecessary emissions
const optimizedValues$ = source$.pipe(
  distinctUntilChanged(),
  // ... other operators
);

// ✅ Use take() for one-time operations
const initialValue$ = source$.pipe(
  take(1)
);

// ✅ Use first() when you expect exactly one result
const singleResult$ = results$.pipe(
  first(result => result.id === targetId)
);
```

### 2. Performance Optimization

```typescript
// ✅ Filter early in the pipeline
const processedData$ = source$.pipe(
  filter(item => item.isActive), // Filter first
  map(item => this.expensiveTransform(item)), // Then transform
  // ... other operations
);

// ✅ Use shareReplay() for expensive filtering
const expensiveFilter$ = source$.pipe(
  filter(item => this.costlyValidation(item)),
  shareReplay(1) // Cache the result
);

// ❌ Avoid filtering after expensive operations
const inefficient$ = source$.pipe(
  map(item => this.expensiveTransform(item)), // Expensive operation first
  filter(item => item.isValid) // Filter after transformation
);
```

### 3. Error Handling

```typescript
// ✅ Handle errors in filtering predicates
const safeFilter$ = source$.pipe(
  filter(item => {
    try {
      return this.complexValidation(item);
    } catch (error) {
      console.error('Validation error:', error);
      return false; // Exclude invalid items
    }
  })
);

// ✅ Provide fallbacks for empty streams
const withFallback$ = source$.pipe(
  filter(item => item.isValid),
  defaultIfEmpty([]) // Provide empty array if no items pass filter
);
```

## Common Pitfalls

### 1. Over-filtering

```typescript
// ❌ Multiple filters that could be combined
const inefficientFiltering$ = source$.pipe(
  filter(item => item.isActive),
  filter(item => item.price > 0),
  filter(item => item.category === 'electronics')
);

// ✅ Combine filters for better performance
const efficientFiltering$ = source$.pipe(
  filter(item => 
    item.isActive && 
    item.price > 0 && 
    item.category === 'electronics'
  )
);
```

### 2. Memory Leaks with Distinct

```typescript
// ❌ distinct() without cleanup can cause memory leaks
const problematic$ = infiniteStream$.pipe(
  distinct() // Keeps track of all seen values forever
);

// ✅ Use distinctUntilChanged() for streams
const better$ = infiniteStream$.pipe(
  distinctUntilChanged() // Only compares with previous value
);

// ✅ Or use distinct() with a key selector and cleanup
const manageable$ = infiniteStream$.pipe(
  distinct(item => item.id),
  take(1000) // Limit the stream
);
```

## Exercises

### Exercise 1: Smart Form Validation
Create a reactive form that validates email uniqueness in real-time, but only after the user stops typing for 500ms and only if the email format is valid.

### Exercise 2: Data Stream Filter
Build a data filtering system that allows users to filter a real-time stream of products by multiple criteria (category, price range, availability) with performance optimization.

### Exercise 3: Permission-Based Navigation
Implement a navigation system that filters menu items based on user permissions and shows/hides routes dynamically.

## Summary

Filtering operators are essential for creating efficient, focused data streams:

- **filter()**: Conditional filtering with predicates
- **take()**: Limit number of emissions
- **takeWhile()**: Emit while condition is true
- **skip()**: Skip first n values
- **distinct()**: Remove duplicates
- **distinctUntilChanged()**: Emit only when changed
- **first()**: Get first matching value
- **last()**: Get last value

Choose the right filtering operator based on your specific use case and always consider performance implications.

## Next Steps

In the next lesson, we'll explore **Combination Operators**, which allow you to combine multiple Observable streams in various ways.
