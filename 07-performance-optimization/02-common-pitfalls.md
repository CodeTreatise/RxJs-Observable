# Common RxJS Pitfalls & Solutions

## Table of Contents
1. [Introduction to RxJS Pitfalls](#introduction)
2. [Memory Leak Pitfalls](#memory-leaks)
3. [Subscription Management Issues](#subscription-issues)
4. [Operator Misuse Patterns](#operator-misuse)
5. [Error Handling Pitfalls](#error-handling)
6. [Performance Anti-patterns](#performance-antipatterns)
7. [Angular Integration Pitfalls](#angular-pitfalls)
8. [Testing & Debugging Issues](#testing-issues)
9. [Prevention Strategies](#prevention)
10. [Exercises](#exercises)

## Introduction to RxJS Pitfalls {#introduction}

Understanding common RxJS pitfalls is crucial for building robust Angular applications. These patterns can lead to memory leaks, performance issues, and unexpected behavior.

### Most Common RxJS Mistakes

```typescript
// Top 10 RxJS Pitfalls Ranking
const COMMON_RXJS_PITFALLS = {
  1: 'Memory leaks from unmanaged subscriptions',
  2: 'Nested subscriptions (Callback hell)',
  3: 'Incorrect error handling',
  4: 'Misusing switchMap vs mergeMap vs concatMap',
  5: 'Not using shareReplay() for expensive operations',
  6: 'Creating observables in templates',
  7: 'Forgetting to handle unsubscription in components',
  8: 'Overusing subjects instead of operators',
  9: 'Not debouncing user input',
  10: 'Mixing promises and observables incorrectly'
};

// Impact Assessment
interface PitfallImpact {
  severity: 'low' | 'medium' | 'high' | 'critical';
  frequency: number; // How often it occurs (1-10)
  detectability: 'easy' | 'medium' | 'hard';
  solutions: string[];
}
```

## Memory Leak Pitfalls {#memory-leaks}

### Pitfall 1: Unmanaged Subscriptions

```typescript
// ❌ PITFALL: Memory leak from unmanaged subscriptions
@Component({
  selector: 'app-leaky',
  template: `<div>{{ data }}</div>`
})
export class LeakyComponent implements OnInit {
  data: any;

  constructor(private dataService: DataService) {}

  ngOnInit() {
    // ❌ Subscription never cleaned up
    this.dataService.getData().subscribe(data => {
      this.data = data;
    });

    // ❌ Multiple subscriptions without cleanup
    this.dataService.getUser().subscribe(user => {
      this.dataService.getUserPreferences(user.id).subscribe(prefs => {
        // Nested subscriptions create multiple leaks
      });
    });

    // ❌ Interval subscription without cleanup
    interval(1000).subscribe(tick => {
      console.log('Tick:', tick);
    });
  }
}

// ✅ SOLUTION: Proper subscription management
@Component({
  selector: 'app-fixed',
  template: `<div>{{ data$ | async }}</div>`
})
export class FixedComponent implements OnInit, OnDestroy {
  data$: Observable<any>;
  private destroy$ = new Subject<void>();

  constructor(private dataService: DataService) {}

  ngOnInit() {
    // ✅ Use async pipe to auto-manage subscriptions
    this.data$ = this.dataService.getData();

    // ✅ Manual subscription with proper cleanup
    this.dataService.getUser().pipe(
      switchMap(user => this.dataService.getUserPreferences(user.id)),
      takeUntil(this.destroy$)
    ).subscribe(prefs => {
      // Handle preferences
    });

    // ✅ Interval with cleanup
    interval(1000).pipe(
      takeUntil(this.destroy$)
    ).subscribe(tick => {
      console.log('Tick:', tick);
    });
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### Pitfall 2: Event Listener Leaks

```typescript
// ❌ PITFALL: Event listeners not properly cleaned up
@Component({
  selector: 'app-event-leak',
  template: `<div>Event handling component</div>`
})
export class EventLeakComponent implements OnInit {
  ngOnInit() {
    // ❌ Direct event listeners without cleanup
    fromEvent(document, 'click').subscribe(event => {
      console.log('Document clicked');
    });

    fromEvent(window, 'resize').subscribe(event => {
      console.log('Window resized');
    });
  }
}

// ✅ SOLUTION: Proper event listener management
@Component({
  selector: 'app-event-fixed',
  template: `<div>Event handling component</div>`
})
export class EventFixedComponent extends OptimizedComponent {
  protected initializeSubscriptions(): void {
    // ✅ Event listeners with proper cleanup
    this.subscribe(
      fromEvent(document, 'click'),
      event => console.log('Document clicked')
    );

    this.subscribe(
      fromEvent(window, 'resize').pipe(debounceTime(250)),
      event => console.log('Window resized')
    );
  }
}
```

### Pitfall 3: Subject Memory Leaks

```typescript
// ❌ PITFALL: Subjects not properly completed
class LeakyService {
  private dataSubject$ = new BehaviorSubject(null);
  
  getData() {
    return this.dataSubject$.asObservable();
  }
  
  updateData(data: any) {
    this.dataSubject$.next(data);
  }
  
  // ❌ No cleanup method - subjects never completed
}

// ✅ SOLUTION: Proper subject lifecycle management
@Injectable({
  providedIn: 'root'
})
export class FixedService implements OnDestroy {
  private dataSubject$ = new BehaviorSubject(null);
  private destroy$ = new Subject<void>();
  
  getData() {
    return this.dataSubject$.asObservable().pipe(
      takeUntil(this.destroy$)
    );
  }
  
  updateData(data: any) {
    this.dataSubject$.next(data);
  }
  
  ngOnDestroy() {
    this.dataSubject$.complete();
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## Subscription Management Issues {#subscription-issues}

### Pitfall 4: Nested Subscriptions (Callback Hell)

```typescript
// ❌ PITFALL: Nested subscriptions create callback hell
class NestedSubscriptionsComponent {
  loadUserData(userId: string) {
    this.userService.getUser(userId).subscribe(user => {
      this.userService.getUserProfile(user.id).subscribe(profile => {
        this.userService.getUserPreferences(user.id).subscribe(preferences => {
          this.userService.getUserActivity(user.id).subscribe(activity => {
            // Deeply nested callbacks
            this.processUserData(user, profile, preferences, activity);
          });
        });
      });
    });
  }
}

// ✅ SOLUTION: Use flattening operators
class FlattenedSubscriptionsComponent {
  loadUserData(userId: string) {
    this.userService.getUser(userId).pipe(
      switchMap(user => 
        forkJoin({
          user: of(user),
          profile: this.userService.getUserProfile(user.id),
          preferences: this.userService.getUserPreferences(user.id),
          activity: this.userService.getUserActivity(user.id)
        })
      ),
      takeUntil(this.destroy$)
    ).subscribe(({ user, profile, preferences, activity }) => {
      this.processUserData(user, profile, preferences, activity);
    });
  }
}
```

### Pitfall 5: Multiple Manual Subscriptions

```typescript
// ❌ PITFALL: Too many manual subscriptions
class TooManySubscriptionsComponent implements OnInit, OnDestroy {
  private subscriptions: Subscription[] = [];

  ngOnInit() {
    // ❌ Managing many subscriptions manually
    this.subscriptions.push(
      this.service1.getData().subscribe(data1 => this.data1 = data1)
    );
    this.subscriptions.push(
      this.service2.getData().subscribe(data2 => this.data2 = data2)
    );
    this.subscriptions.push(
      this.service3.getData().subscribe(data3 => this.data3 = data3)
    );
    // ... many more subscriptions
  }

  ngOnDestroy() {
    this.subscriptions.forEach(sub => sub.unsubscribe());
  }
}

// ✅ SOLUTION: Use view model pattern with async pipe
class ViewModelComponent implements OnInit {
  viewModel$: Observable<ViewModel>;

  ngOnInit() {
    // ✅ Single subscription via async pipe
    this.viewModel$ = combineLatest([
      this.service1.getData(),
      this.service2.getData(),
      this.service3.getData()
    ]).pipe(
      map(([data1, data2, data3]) => ({
        data1,
        data2,
        data3,
        combinedData: this.combineData(data1, data2, data3)
      }))
    );
  }
}
```

## Operator Misuse Patterns {#operator-misuse}

### Pitfall 6: Incorrect Flattening Operators

```typescript
// ❌ PITFALL: Wrong flattening operator choice
class WrongFlatteningComponent {
  // ❌ Using mergeMap for search - creates race conditions
  searchWithMergeMap(searchTerm: string) {
    return this.searchInput$.pipe(
      mergeMap(term => this.searchService.search(term))
    );
    // Problem: Old search results can override newer ones
  }

  // ❌ Using switchMap for saving data - can lose data
  saveDataWithSwitchMap(data: any) {
    return this.saveButton$.pipe(
      switchMap(() => this.dataService.save(data))
    );
    // Problem: Previous save operations get cancelled
  }

  // ❌ Using concatMap for independent operations - creates backlog
  processItemsWithConcatMap(items: any[]) {
    return from(items).pipe(
      concatMap(item => this.processItem(item))
    );
    // Problem: Items processed sequentially even when independent
  }
}

// ✅ SOLUTION: Choose correct flattening operators
class CorrectFlatteningComponent {
  // ✅ Use switchMap for search - cancel previous searches
  searchWithSwitchMap(searchTerm: string) {
    return this.searchInput$.pipe(
      debounceTime(300),
      switchMap(term => this.searchService.search(term))
    );
  }

  // ✅ Use concatMap for saving - ensure all saves complete
  saveDataWithConcatMap(data: any) {
    return this.saveButton$.pipe(
      concatMap(() => this.dataService.save(data))
    );
  }

  // ✅ Use mergeMap for independent operations - process in parallel
  processItemsWithMergeMap(items: any[], concurrency = 3) {
    return from(items).pipe(
      mergeMap(item => this.processItem(item), concurrency)
    );
  }

  // ✅ Use exhaustMap for preventing duplicate operations
  preventDuplicateRequests() {
    return this.button$.pipe(
      exhaustMap(() => this.expensiveOperation())
    );
  }
}
```

### Pitfall 7: Overusing Subjects

```typescript
// ❌ PITFALL: Using subjects when operators would be better
class SubjectOveruseService {
  private dataSubject$ = new BehaviorSubject([]);
  private filteredDataSubject$ = new BehaviorSubject([]);
  private searchTermSubject$ = new BehaviorSubject('');

  getData() {
    return this.dataSubject$.asObservable();
  }

  getFilteredData() {
    return this.filteredDataSubject$.asObservable();
  }

  updateData(data: any[]) {
    this.dataSubject$.next(data);
    this.filterData(); // Manual coordination
  }

  updateSearchTerm(term: string) {
    this.searchTermSubject$.next(term);
    this.filterData(); // Manual coordination
  }

  private filterData() {
    // ❌ Manual filtering logic with multiple subjects
    const data = this.dataSubject$.value;
    const term = this.searchTermSubject$.value;
    const filtered = data.filter(item => 
      item.name.toLowerCase().includes(term.toLowerCase())
    );
    this.filteredDataSubject$.next(filtered);
  }
}

// ✅ SOLUTION: Use operators for data transformation
class OperatorBasedService {
  private dataSubject$ = new BehaviorSubject([]);
  private searchTermSubject$ = new BehaviorSubject('');

  // ✅ Declarative data transformation
  filteredData$ = combineLatest([
    this.dataSubject$.asObservable(),
    this.searchTermSubject$.pipe(
      debounceTime(300),
      distinctUntilChanged()
    )
  ]).pipe(
    map(([data, searchTerm]) => 
      data.filter(item => 
        item.name.toLowerCase().includes(searchTerm.toLowerCase())
      )
    ),
    shareReplay(1)
  );

  updateData(data: any[]) {
    this.dataSubject$.next(data);
    // No manual coordination needed
  }

  updateSearchTerm(term: string) {
    this.searchTermSubject$.next(term);
    // No manual coordination needed
  }
}
```

## Error Handling Pitfalls {#error-handling}

### Pitfall 8: Inadequate Error Handling

```typescript
// ❌ PITFALL: No error handling or stream termination
class PoorErrorHandlingComponent {
  loadData() {
    // ❌ No error handling - stream terminates on first error
    this.dataService.getData().subscribe(data => {
      this.processData(data);
    });

    // ❌ Generic error handling without recovery
    this.dataService.getData().subscribe({
      next: data => this.processData(data),
      error: error => console.error(error) // Stream still terminates
    });
  }
}

// ✅ SOLUTION: Comprehensive error handling with recovery
class RobustErrorHandlingComponent {
  loadData() {
    // ✅ Error handling with retry and fallback
    this.dataService.getData().pipe(
      retry({
        count: 3,
        delay: (error, retryCount) => timer(1000 * retryCount)
      }),
      catchError(error => {
        console.error('Failed to load data after retries:', error);
        // Return fallback data to keep stream alive
        return of(this.getFallbackData());
      }),
      takeUntil(this.destroy$)
    ).subscribe(data => {
      this.processData(data);
    });
  }

  // ✅ Error handling with user notification
  loadDataWithUserFeedback() {
    this.dataService.getData().pipe(
      retryWhen(errors => 
        errors.pipe(
          scan((retryCount, error) => {
            if (retryCount >= 3) {
              throw error; // Stop retrying after 3 attempts
            }
            return retryCount + 1;
          }, 0),
          delay(1000)
        )
      ),
      catchError(error => {
        this.notificationService.showError('Failed to load data');
        return EMPTY; // Complete the stream gracefully
      }),
      takeUntil(this.destroy$)
    ).subscribe(data => {
      this.processData(data);
    });
  }

  private getFallbackData() {
    return { message: 'No data available', items: [] };
  }
}
```

### Pitfall 9: Error Propagation Issues

```typescript
// ❌ PITFALL: Errors in one stream break entire chain
class ErrorPropagationIssue {
  loadMultipleDataSources() {
    // ❌ If any stream errors, entire observable fails
    return forkJoin({
      users: this.userService.getUsers(),
      products: this.productService.getProducts(), // If this fails...
      orders: this.orderService.getOrders() // ...these won't be fetched
    });
  }
}

// ✅ SOLUTION: Handle errors at appropriate levels
class IsolatedErrorHandling {
  loadMultipleDataSources() {
    // ✅ Each stream handles its own errors
    return forkJoin({
      users: this.userService.getUsers().pipe(
        catchError(error => {
          console.error('Users failed to load:', error);
          return of([]); // Return empty array as fallback
        })
      ),
      products: this.productService.getProducts().pipe(
        catchError(error => {
          console.error('Products failed to load:', error);
          return of([]);
        })
      ),
      orders: this.orderService.getOrders().pipe(
        catchError(error => {
          console.error('Orders failed to load:', error);
          return of([]);
        })
      )
    });
  }

  // ✅ Alternative: Use combineLatest with individual error handling
  loadDataWithPartialFailure() {
    const users$ = this.userService.getUsers().pipe(
      map(users => ({ users, error: null })),
      catchError(error => of({ users: [], error: error.message }))
    );

    const products$ = this.productService.getProducts().pipe(
      map(products => ({ products, error: null })),
      catchError(error => of({ products: [], error: error.message }))
    );

    return combineLatest([users$, products$]).pipe(
      map(([usersResult, productsResult]) => ({
        users: usersResult.users,
        products: productsResult.products,
        errors: [usersResult.error, productsResult.error].filter(Boolean)
      }))
    );
  }
}
```

## Performance Anti-patterns {#performance-antipatterns}

### Pitfall 10: Creating Observables in Templates

```typescript
// ❌ PITFALL: Creating observables in templates
@Component({
  template: `
    <!-- ❌ Creates new observable on every change detection -->
    <div *ngFor="let item of getItems() | async">{{ item.name }}</div>
    
    <!-- ❌ Multiple async pipes for same data -->
    <div>Total: {{ (getItems() | async)?.length }}</div>
    <div>First: {{ (getItems() | async)?.[0]?.name }}</div>
  `
})
class TemplateObservableAntipattern {
  getItems(): Observable<Item[]> {
    // ❌ Creates new HTTP request on every call
    return this.http.get<Item[]>('/api/items');
  }
}

// ✅ SOLUTION: Create observables in component initialization
@Component({
  template: `
    <!-- ✅ Single subscription shared via async pipe -->
    <ng-container *ngIf="viewModel$ | async as vm">
      <div *ngFor="let item of vm.items">{{ item.name }}</div>
      <div>Total: {{ vm.items.length }}</div>
      <div>First: {{ vm.items[0]?.name }}</div>
    </ng-container>
  `
})
class OptimizedTemplateComponent implements OnInit {
  viewModel$: Observable<ViewModel>;

  ngOnInit() {
    // ✅ Create observable once during initialization
    const items$ = this.http.get<Item[]>('/api/items').pipe(
      shareReplay(1),
      catchError(() => of([]))
    );

    this.viewModel$ = items$.pipe(
      map(items => ({
        items,
        total: items.length,
        firstItem: items[0] || null
      }))
    );
  }
}
```

### Pitfall 11: Not Debouncing User Input

```typescript
// ❌ PITFALL: Processing every keystroke
@Component({
  template: `
    <input (input)="onSearch($event.target.value)">
    <div *ngFor="let result of searchResults">{{ result }}</div>
  `
})
class UnoptimizedSearchComponent {
  searchResults: any[] = [];

  onSearch(term: string) {
    // ❌ Makes API call on every keystroke
    this.searchService.search(term).subscribe(results => {
      this.searchResults = results;
    });
  }
}

// ✅ SOLUTION: Debounce user input
@Component({
  template: `
    <input #searchInput>
    <div *ngFor="let result of searchResults$ | async">{{ result }}</div>
  `
})
class OptimizedSearchComponent implements OnInit, AfterViewInit {
  @ViewChild('searchInput') searchInput!: ElementRef;
  searchResults$: Observable<any[]>;

  ngAfterViewInit() {
    // ✅ Debounce search input
    this.searchResults$ = fromEvent(this.searchInput.nativeElement, 'input').pipe(
      map((event: any) => event.target.value),
      debounceTime(300),
      distinctUntilChanged(),
      filter(term => term.length >= 2),
      switchMap(term => 
        this.searchService.search(term).pipe(
          catchError(() => of([]))
        )
      ),
      shareReplay(1)
    );
  }
}
```

## Angular Integration Pitfalls {#angular-pitfalls}

### Pitfall 12: Change Detection Issues

```typescript
// ❌ PITFALL: Not triggering change detection
@Component({
  template: `<div>{{ data }}</div>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
class ChangeDetectionIssue implements OnInit {
  data: any;

  ngOnInit() {
    // ❌ With OnPush, manual subscription won't trigger change detection
    this.dataService.getData().subscribe(data => {
      this.data = data; // Component won't update
    });
  }
}

// ✅ SOLUTION: Use async pipe or trigger change detection manually
@Component({
  template: `<div>{{ data$ | async }}</div>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
class FixedChangeDetection implements OnInit {
  data$: Observable<any>;

  ngOnInit() {
    // ✅ Async pipe automatically triggers change detection
    this.data$ = this.dataService.getData();
  }
}

// Alternative solution with manual change detection
@Component({
  template: `<div>{{ data }}</div>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
class ManualChangeDetection implements OnInit {
  data: any;

  constructor(private cdr: ChangeDetectorRef) {}

  ngOnInit() {
    this.dataService.getData().pipe(
      takeUntil(this.destroy$)
    ).subscribe(data => {
      this.data = data;
      this.cdr.markForCheck(); // ✅ Manually trigger change detection
    });
  }
}
```

### Pitfall 13: Form Control Integration Issues

```typescript
// ❌ PITFALL: Not properly integrating observables with reactive forms
class PoorFormIntegration {
  form = this.fb.group({
    username: [''],
    email: ['']
  });

  ngOnInit() {
    // ❌ Manual subscription without proper cleanup
    this.form.get('username')?.valueChanges.subscribe(value => {
      this.validateUsername(value);
    });
  }

  private validateUsername(username: string) {
    // Validation logic
  }
}

// ✅ SOLUTION: Proper form observable integration
class ImprovedFormIntegration extends OptimizedComponent {
  form = this.fb.group({
    username: [''],
    email: ['']
  });

  protected initializeSubscriptions(): void {
    // ✅ Debounced validation with proper cleanup
    this.subscribe(
      this.form.get('username')!.valueChanges.pipe(
        debounceTime(300),
        distinctUntilChanged(),
        filter(value => value && value.length >= 3),
        switchMap(username => this.validateUsernameAsync(username))
      ),
      validationResult => {
        this.handleValidationResult(validationResult);
      }
    );

    // ✅ Cross-field validation
    this.subscribe(
      this.form.valueChanges.pipe(
        debounceTime(500),
        map(formValue => this.crossFieldValidation(formValue))
      ),
      validationErrors => {
        this.updateFormErrors(validationErrors);
      }
    );
  }

  private validateUsernameAsync(username: string): Observable<ValidationResult> {
    return this.userService.checkUsernameAvailability(username);
  }
}
```

## Testing & Debugging Issues {#testing-issues}

### Pitfall 14: Testing Asynchronous Code Incorrectly

```typescript
// ❌ PITFALL: Testing observables without proper setup
describe('DataService', () => {
  it('should fetch data', () => {
    const service = TestBed.inject(DataService);
    
    // ❌ No marble testing or proper async handling
    service.getData().subscribe(data => {
      expect(data).toBeDefined();
    });
    // Test completes before subscription executes
  });
});

// ✅ SOLUTION: Proper observable testing
describe('DataService', () => {
  let service: DataService;
  let httpMock: HttpTestingController;
  let scheduler: TestScheduler;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [DataService]
    });
    
    service = TestBed.inject(DataService);
    httpMock = TestBed.inject(HttpTestingController);
    scheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should fetch data with marble testing', () => {
    scheduler.run(({ cold, expectObservable }) => {
      const mockData = [{ id: 1, name: 'Test' }];
      
      // Mock HTTP response
      service.getData().subscribe();
      const req = httpMock.expectOne('/api/data');
      req.flush(mockData);

      // ✅ Use marble testing for timing
      const expected = cold('(a|)', { a: mockData });
      expectObservable(service.getData()).toBe('(a|)', { a: mockData });
    });
  });

  it('should handle errors properly', fakeAsync(() => {
    const errorMessage = 'Server error';
    
    service.getData().subscribe({
      next: () => fail('Should have failed'),
      error: (error) => {
        expect(error.message).toContain(errorMessage);
      }
    });

    const req = httpMock.expectOne('/api/data');
    req.error(new ErrorEvent('Network error'), { status: 500 });
    
    tick();
  }));
});
```

## Prevention Strategies {#prevention}

### Automated Detection Tools

```typescript
// ✅ Custom linting rules for RxJS best practices
class RxJSLintRules {
  // Rule: Detect unmanaged subscriptions
  static detectUnmanagedSubscriptions(sourceFile: any): LintIssue[] {
    const issues: LintIssue[] = [];
    
    // Look for .subscribe() calls without takeUntil or async pipe
    // Implementation would use TypeScript compiler API
    
    return issues;
  }

  // Rule: Detect nested subscriptions
  static detectNestedSubscriptions(sourceFile: any): LintIssue[] {
    // Implementation to detect subscription inside subscription
    return [];
  }

  // Rule: Detect observables created in templates
  static detectTemplateObservables(templateFile: string): LintIssue[] {
    // Implementation to detect method calls in templates that return observables
    return [];
  }
}

// ✅ Runtime monitoring for development
@Injectable()
export class RxJSDevMonitor {
  private subscriptionCount = 0;
  private subscriptions = new Map<string, SubscriptionInfo>();

  monitorSubscription(label: string): void {
    this.subscriptionCount++;
    this.subscriptions.set(label, {
      created: Date.now(),
      active: true
    });

    if (this.subscriptionCount > 100) {
      console.warn('⚠️ High subscription count detected:', this.subscriptionCount);
      this.printSubscriptionReport();
    }
  }

  private printSubscriptionReport(): void {
    const activeSubscriptions = Array.from(this.subscriptions.entries())
      .filter(([_, info]) => info.active)
      .map(([label, info]) => ({
        label,
        age: Date.now() - info.created
      }))
      .sort((a, b) => b.age - a.age);

    console.table(activeSubscriptions);
  }
}
```

### Best Practice Checklist

```typescript
const RXJS_PITFALL_PREVENTION_CHECKLIST = {
  subscription_management: [
    '✅ Use async pipe whenever possible',
    '✅ Implement takeUntil pattern for manual subscriptions',
    '✅ Complete subjects in ngOnDestroy',
    '✅ Use subscription pooling for similar streams'
  ],
  
  operator_usage: [
    '✅ Choose correct flattening operators (switchMap vs mergeMap vs concatMap)',
    '✅ Use shareReplay for expensive operations',
    '✅ Debounce user input streams',
    '✅ Prefer operators over manual subject management'
  ],
  
  error_handling: [
    '✅ Handle errors at appropriate levels',
    '✅ Implement retry logic for recoverable errors',
    '✅ Provide fallback values to keep streams alive',
    '✅ Use catchError to prevent stream termination'
  ],
  
  performance: [
    '✅ Create observables in component initialization, not templates',
    '✅ Use OnPush change detection with observables',
    '✅ Implement virtual scrolling for large lists',
    '✅ Monitor memory usage and subscription counts'
  ],
  
  testing: [
    '✅ Use marble testing for complex observable logic',
    '✅ Mock HTTP calls in tests',
    '✅ Test error scenarios',
    '✅ Use fakeAsync for time-based operations'
  ]
};
```

### Code Review Guidelines

```typescript
interface CodeReviewChecklist {
  subscriptions: {
    hasProperCleanup: boolean;
    usesAsyncPipe: boolean;
    avoidsNestedSubscriptions: boolean;
  };
  
  operators: {
    correctFlatteningOperators: boolean;
    appropriateErrorHandling: boolean;
    efficientFiltering: boolean;
  };
  
  performance: {
    avoidsTemplateObservables: boolean;
    usesOnPushDetection: boolean;
    implementsDebouncing: boolean;
  };
}

// Example code review automation
function reviewRxJSCode(codeFile: string): CodeReviewChecklist {
  return {
    subscriptions: {
      hasProperCleanup: checkForTakeUntil(codeFile),
      usesAsyncPipe: checkAsyncPipeUsage(codeFile),
      avoidsNestedSubscriptions: !hasNestedSubscriptions(codeFile)
    },
    operators: {
      correctFlatteningOperators: validateFlatteningOperators(codeFile),
      appropriateErrorHandling: hasErrorHandling(codeFile),
      efficientFiltering: checkFilteringPatterns(codeFile)
    },
    performance: {
      avoidsTemplateObservables: !hasTemplateObservables(codeFile),
      usesOnPushDetection: hasOnPushStrategy(codeFile),
      implementsDebouncing: hasDebouncing(codeFile)
    }
  };
}
```

## Exercises {#exercises}

### Exercise 1: Pitfall Detection Tool

Create a tool that automatically detects common RxJS pitfalls:
- Scan TypeScript files for unmanaged subscriptions
- Identify nested subscription patterns
- Flag missing error handling
- Report performance anti-patterns

### Exercise 2: Memory Leak Simulator

Build a component that demonstrates memory leaks:
- Create various types of memory leaks
- Show how they impact application performance
- Implement fixes and measure improvements
- Add monitoring to detect leaks in real-time

### Exercise 3: Operator Confusion Examples

Create examples that show the differences between flattening operators:
- Demonstrate race conditions with wrong operator choice
- Show data loss scenarios
- Provide interactive examples to understand timing
- Build a decision tree for operator selection

### Exercise 4: Error Handling Scenarios

Implement comprehensive error handling examples:
- Show different error propagation patterns
- Demonstrate retry strategies
- Implement graceful degradation
- Create user-friendly error recovery flows

---

**Next Steps:**
- Learn about memory leak identification and prevention
- Explore bundle size optimization techniques
- Master browser compatibility considerations
- Implement automated pitfall detection systems
