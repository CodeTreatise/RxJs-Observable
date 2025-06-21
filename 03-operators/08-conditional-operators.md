# Conditional Operators

## Overview

Conditional operators provide powerful ways to implement branching logic and conditional behavior in reactive streams. These operators allow you to make decisions based on emitted values, stream states, or external conditions, enabling sophisticated control flow in your Observable pipelines without breaking the reactive paradigm.

## Learning Objectives

After completing this lesson, you will be able to:
- Implement conditional logic in reactive streams
- Control Observable flow based on dynamic conditions
- Handle default values and empty streams gracefully
- Apply conditional operators in complex Angular scenarios
- Build adaptive and responsive reactive applications

## Core Conditional Operators

### 1. iif() - Conditional Observable Creation

The `iif()` operator subscribes to one of two Observables based on a condition evaluated at subscription time.

```typescript
import { iif, of, timer } from 'rxjs';
import { map } from 'rxjs/operators';

// Basic conditional creation
const condition = Math.random() > 0.5;
const conditional$ = iif(
  () => condition,
  of('Condition was true'),
  of('Condition was false')
);

// Angular feature flag implementation
@Injectable()
export class FeatureFlagService {
  constructor(
    private configService: ConfigService,
    private userService: UserService
  ) {}

  getFeatureBasedContent(): Observable<Content> {
    return iif(
      () => this.configService.isFeatureEnabled('newUI'),
      this.getNewUIContent(),
      this.getLegacyUIContent()
    );
  }

  getUserSpecificData(): Observable<UserData> {
    return iif(
      () => this.userService.isPremiumUser(),
      this.getPremiumUserData(),
      this.getBasicUserData()
    );
  }

  private getNewUIContent(): Observable<Content> {
    return this.http.get<Content>('/api/content/new-ui');
  }

  private getLegacyUIContent(): Observable<Content> {
    return this.http.get<Content>('/api/content/legacy');
  }

  private getPremiumUserData(): Observable<UserData> {
    return this.http.get<UserData>('/api/premium-data');
  }

  private getBasicUserData(): Observable<UserData> {
    return this.http.get<UserData>('/api/basic-data');
  }
}

// Environment-based service selection
@Component({})
export class AdaptiveComponent implements OnInit {
  data$!: Observable<AppData>;

  ngOnInit() {
    this.data$ = iif(
      () => environment.production,
      this.productionDataService.getData(),
      this.developmentDataService.getData()
    ).pipe(
      catchError(error => {
        console.error('Data loading failed:', error);
        return of(this.getDefaultData());
      })
    );
  }

  private getDefaultData(): AppData {
    return { message: 'Default data loaded', items: [] };
  }
}
```

**Marble Diagram:**
```
Condition true:
iif(() => true, A$, B$)
A$: -1-2-3|
Output: -1-2-3|

Condition false:
iif(() => false, A$, B$)
B$: -a-b-c|
Output: -a-b-c|
```

### 2. defaultIfEmpty() - Provide Default Values

The `defaultIfEmpty()` operator emits a default value if the source Observable completes without emitting any values.

```typescript
import { EMPTY, of } from 'rxjs';
import { defaultIfEmpty, filter } from 'rxjs/operators';

// Basic default value
const empty$ = EMPTY.pipe(
  defaultIfEmpty('No data available')
);

const filtered$ = of(1, 2, 3, 4, 5).pipe(
  filter(x => x > 10), // No values pass the filter
  defaultIfEmpty('No values matched the criteria')
);

// Angular search component with fallbacks
@Component({
  template: `
    <div class="search-container">
      <input 
        [(ngModel)]="searchTerm" 
        (input)="onSearchChange()"
        placeholder="Search products...">
      
      <div class="search-results">
        <div *ngFor="let product of searchResults$ | async" class="product-item">
          <h3>{{ product.name }}</h3>
          <p>{{ product.description }}</p>
          <span class="price">{{ product.price | currency }}</span>
        </div>
      </div>

      <div *ngIf="(searchResults$ | async)?.length === 0" class="no-results">
        {{ noResultsMessage$ | async }}
      </div>
    </div>
  `
})
export class SearchWithFallbackComponent {
  searchTerm = '';
  private searchSubject = new Subject<string>();

  searchResults$ = this.searchSubject.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap(term => 
      term.length > 0 
        ? this.searchService.search(term)
        : of([])
    ),
    defaultIfEmpty([]) // Ensure we always have an array
  );

  noResultsMessage$ = this.searchSubject.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    filter(term => term.length > 0),
    switchMap(term => 
      this.searchService.search(term).pipe(
        map(results => results.length === 0 ? `No results found for "${term}"` : ''),
        defaultIfEmpty('Search service temporarily unavailable')
      )
    )
  );

  onSearchChange() {
    this.searchSubject.next(this.searchTerm);
  }

  constructor(private searchService: SearchService) {}
}

// User preferences with defaults
@Injectable()
export class UserPreferencesService {
  constructor(
    private http: HttpClient,
    private localStorageService: LocalStorageService
  ) {}

  getUserPreferences(): Observable<UserPreferences> {
    return this.http.get<UserPreferences>('/api/user/preferences').pipe(
      defaultIfEmpty(this.getDefaultPreferences()),
      catchError(() => {
        // Fallback to local storage
        return this.localStorageService.get<UserPreferences>('userPreferences').pipe(
          defaultIfEmpty(this.getDefaultPreferences())
        );
      })
    );
  }

  private getDefaultPreferences(): UserPreferences {
    return {
      theme: 'light',
      language: 'en',
      notifications: true,
      autoSave: true,
      itemsPerPage: 10
    };
  }
}
```

### 3. every() - Test All Values

The `every()` operator returns an Observable that emits whether every value emitted by the source Observable satisfies a condition.

```typescript
import { of } from 'rxjs';
import { every } from 'rxjs/operators';

// Basic validation
const numbers$ = of(2, 4, 6, 8, 10);
const allEven$ = numbers$.pipe(
  every(n => n % 2 === 0)
);
allEven$.subscribe(result => console.log('All even:', result)); // true

// Angular form validation
@Component({
  template: `
    <form [formGroup]="passwordForm">
      <div *ngFor="let field of passwordFields; let i = index" class="form-group">
        <label>{{ field.label }}</label>
        <input 
          [formControlName]="field.name"
          type="password"
          [placeholder]="field.placeholder">
        <div *ngIf="field.validation$ | async as validation" 
             [class]="validation.isValid ? 'valid' : 'invalid'">
          {{ validation.message }}
        </div>
      </div>

      <div class="password-strength">
        <div *ngIf="overallValidation$ | async as validation">
          Password Strength: 
          <span [class]="validation.isValid ? 'strong' : 'weak'">
            {{ validation.isValid ? 'Strong' : 'Weak' }}
          </span>
        </div>
      </div>

      <button 
        type="submit" 
        [disabled]="!(isFormValid$ | async)">
        Create Password
      </button>
    </form>
  `
})
export class PasswordValidationComponent {
  passwordForm = new FormGroup({
    password: new FormControl(''),
    confirmPassword: new FormControl('')
  });

  passwordFields = [
    { name: 'password', label: 'Password', placeholder: 'Enter password' },
    { name: 'confirmPassword', label: 'Confirm Password', placeholder: 'Confirm password' }
  ];

  // Validate all password criteria
  passwordCriteria$ = this.passwordForm.get('password')!.valueChanges.pipe(
    map(password => [
      { rule: 'minLength', test: password.length >= 8, message: 'At least 8 characters' },
      { rule: 'hasUpper', test: /[A-Z]/.test(password), message: 'Contains uppercase letter' },
      { rule: 'hasLower', test: /[a-z]/.test(password), message: 'Contains lowercase letter' },
      { rule: 'hasNumber', test: /\d/.test(password), message: 'Contains number' },
      { rule: 'hasSpecial', test: /[!@#$%^&*]/.test(password), message: 'Contains special character' }
    ])
  );

  overallValidation$ = this.passwordCriteria$.pipe(
    map(criteria => criteria.map(c => c.test)),
    mergeMap(tests => from(tests)),
    every(test => test),
    map(isValid => ({
      isValid,
      message: isValid ? 'All criteria met' : 'Some criteria not met'
    }))
  );

  isFormValid$ = combineLatest([
    this.overallValidation$,
    this.passwordForm.get('confirmPassword')!.valueChanges.pipe(startWith(''))
  ]).pipe(
    map(([passwordValid, confirmPassword]) => {
      const password = this.passwordForm.get('password')!.value;
      return passwordValid.isValid && password === confirmPassword;
    })
  );
}

// Data integrity validation
@Injectable()
export class DataIntegrityService {
  validateDataBatch<T>(
    data: T[], 
    validator: (item: T) => boolean
  ): Observable<ValidationResult> {
    return from(data).pipe(
      every(validator),
      map(isValid => ({
        isValid,
        totalItems: data.length,
        message: isValid ? 'All items are valid' : 'Some items failed validation'
      }))
    );
  }

  validateApiResponses(): Observable<boolean> {
    const apiCalls = [
      this.http.get('/api/health'),
      this.http.get('/api/status'),
      this.http.get('/api/version')
    ];

    return forkJoin(apiCalls).pipe(
      mergeMap(responses => from(responses)),
      every(response => response.status === 'ok'),
      tap(allHealthy => {
        if (allHealthy) {
          console.log('All APIs are healthy');
        } else {
          console.warn('Some APIs are not responding correctly');
        }
      })
    );
  }
}
```

### 4. find() - Find First Matching Value

The `find()` operator emits the first value that matches a specified condition, then completes.

```typescript
import { of, from } from 'rxjs';
import { find } from 'rxjs/operators';

// Basic find operation
const numbers$ = of(1, 3, 5, 8, 9, 12);
const firstEven$ = numbers$.pipe(
  find(n => n % 2 === 0)
);
firstEven$.subscribe(result => console.log('First even:', result)); // 8

// Angular user management
@Injectable()
export class UserManagementService {
  constructor(private http: HttpClient) {}

  findUserByEmail(email: string): Observable<User | undefined> {
    return this.http.get<User[]>('/api/users').pipe(
      mergeMap(users => from(users)),
      find(user => user.email === email)
    );
  }

  findAvailableSlot(date: Date): Observable<TimeSlot | undefined> {
    return this.http.get<TimeSlot[]>(`/api/slots/${date.toISOString()}`).pipe(
      mergeMap(slots => from(slots)),
      find(slot => slot.isAvailable && !slot.isBooked)
    );
  }

  findCompatibleVersion(requirements: VersionRequirements): Observable<Version | undefined> {
    return this.http.get<Version[]>('/api/versions').pipe(
      mergeMap(versions => from(versions)),
      find(version => this.isVersionCompatible(version, requirements))
    );
  }

  private isVersionCompatible(version: Version, requirements: VersionRequirements): boolean {
    return version.major >= requirements.minMajor &&
           version.minor >= requirements.minMinor &&
           version.features.every(feature => requirements.requiredFeatures.includes(feature));
  }
}

// Dynamic component loading
@Component({})
export class DynamicLoaderComponent implements OnInit {
  private availableComponents$ = of([
    { name: 'TableComponent', priority: 1, supports: ['data-display'] },
    { name: 'ChartComponent', priority: 2, supports: ['data-visualization'] },
    { name: 'MapComponent', priority: 3, supports: ['geo-data'] }
  ]);

  ngOnInit() {
    this.loadBestComponent('data-visualization');
  }

  loadBestComponent(requirement: string) {
    this.availableComponents$.pipe(
      mergeMap(components => from(components)),
      find(component => component.supports.includes(requirement))
    ).subscribe(component => {
      if (component) {
        console.log(`Loading ${component.name} for ${requirement}`);
        this.loadComponent(component.name);
      } else {
        console.warn(`No component found for requirement: ${requirement}`);
      }
    });
  }

  private loadComponent(componentName: string) {
    // Dynamic component loading logic
  }
}
```

### 5. findIndex() - Find Index of First Match

The `findIndex()` operator emits the index of the first value that matches a specified condition.

```typescript
import { of, from } from 'rxjs';
import { findIndex } from 'rxjs/operators';

// Basic find index
const items$ = of('apple', 'banana', 'cherry', 'date');
const bananaIndex$ = items$.pipe(
  findIndex(item => item === 'banana')
);
bananaIndex$.subscribe(index => console.log('Banana index:', index)); // 1

// Angular list management
@Component({
  template: `
    <div class="list-container">
      <div class="search-bar">
        <input 
          [(ngModel)]="searchTerm"
          (input)="onSearch()"
          placeholder="Search items...">
        <span *ngIf="foundIndex$ | async as index">
          Found at position: {{ index + 1 }}
        </span>
      </div>

      <div class="item-list">
        <div 
          *ngFor="let item of items; let i = index" 
          class="item"
          [class.highlighted]="(foundIndex$ | async) === i">
          {{ item.name }}
        </div>
      </div>

      <div class="navigation">
        <button (click)="scrollToFound()" [disabled]="(foundIndex$ | async) === -1">
          Scroll to Found Item
        </button>
      </div>
    </div>
  `
})
export class SearchableListComponent {
  items = [
    { name: 'Item 1', category: 'A' },
    { name: 'Item 2', category: 'B' },
    { name: 'Special Item', category: 'A' },
    { name: 'Item 4', category: 'C' }
  ];

  searchTerm = '';
  private searchSubject = new Subject<string>();

  foundIndex$ = this.searchSubject.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap(term => {
      if (!term) return of(-1);
      
      return from(this.items).pipe(
        findIndex(item => 
          item.name.toLowerCase().includes(term.toLowerCase())
        )
      );
    })
  );

  onSearch() {
    this.searchSubject.next(this.searchTerm);
  }

  scrollToFound() {
    this.foundIndex$.pipe(take(1)).subscribe(index => {
      if (index >= 0) {
        const element = document.querySelector(`.item:nth-child(${index + 1})`);
        element?.scrollIntoView({ behavior: 'smooth' });
      }
    });
  }
}

// Error location service
@Injectable()
export class ErrorLocationService {
  findErrorInLog(errorSignature: string): Observable<number> {
    return this.http.get<LogEntry[]>('/api/logs').pipe(
      mergeMap(logs => from(logs)),
      findIndex(log => log.message.includes(errorSignature))
    );
  }

  findFirstFailureInBatch(batchId: string): Observable<number> {
    return this.http.get<BatchItem[]>(`/api/batch/${batchId}/items`).pipe(
      mergeMap(items => from(items)),
      findIndex(item => item.status === 'failed')
    );
  }
}
```

### 6. isEmpty() - Check if Observable is Empty

The `isEmpty()` operator emits true if the source Observable completes without emitting any values, false otherwise.

```typescript
import { EMPTY, of } from 'rxjs';
import { isEmpty, filter } from 'rxjs/operators';

// Basic empty check
const empty$ = EMPTY.pipe(isEmpty());
empty$.subscribe(result => console.log('Is empty:', result)); // true

const nonEmpty$ = of(1, 2, 3).pipe(isEmpty());
nonEmpty$.subscribe(result => console.log('Is empty:', result)); // false

// Angular data loading with empty state handling
@Component({
  template: `
    <div class="data-container">
      <div *ngIf="isLoading$ | async" class="loading">
        Loading data...
      </div>

      <div *ngIf="isEmpty$ | async" class="empty-state">
        <h3>No Data Available</h3>
        <p>{{ emptyMessage$ | async }}</p>
        <button (click)="refresh()">Try Again</button>
      </div>

      <div *ngIf="hasData$ | async" class="data-list">
        <div *ngFor="let item of data$ | async" class="data-item">
          {{ item.name }}
        </div>
      </div>
    </div>
  `
})
export class DataWithEmptyStateComponent implements OnInit {
  private refreshTrigger$ = new Subject<void>();
  
  data$ = this.refreshTrigger$.pipe(
    startWith(null),
    switchMap(() => this.dataService.getData()),
    shareReplay(1)
  );

  isEmpty$ = this.data$.pipe(
    isEmpty(),
    catchError(() => of(false)) // Assume not empty on error
  );

  hasData$ = this.isEmpty$.pipe(
    map(empty => !empty)
  );

  isLoading$ = new BehaviorSubject<boolean>(false);

  emptyMessage$ = this.isEmpty$.pipe(
    filter(empty => empty),
    map(() => this.getEmptyMessage())
  );

  ngOnInit() {
    this.refresh();
  }

  refresh() {
    this.isLoading$.next(true);
    this.refreshTrigger$.next();
    
    // Reset loading state after data loads
    this.data$.pipe(take(1)).subscribe(() => {
      this.isLoading$.next(false);
    });
  }

  private getEmptyMessage(): string {
    const messages = [
      "It looks like there's no data to show right now.",
      "No items found. Try refreshing or check back later.",
      "The data source appears to be empty."
    ];
    return messages[Math.floor(Math.random() * messages.length)];
  }
}

// Shopping cart empty state
@Injectable()
export class ShoppingCartService {
  private cartItems$ = new BehaviorSubject<CartItem[]>([]);

  getCartItems(): Observable<CartItem[]> {
    return this.cartItems$.asObservable();
  }

  isCartEmpty(): Observable<boolean> {
    return this.cartItems$.pipe(
      isEmpty(),
      startWith(true) // Start with empty assumption
    );
  }

  getCartStatus(): Observable<CartStatus> {
    return combineLatest([
      this.cartItems$,
      this.isCartEmpty()
    ]).pipe(
      map(([items, isEmpty]) => ({
        isEmpty,
        itemCount: items.length,
        totalValue: items.reduce((sum, item) => sum + item.price * item.quantity, 0),
        message: isEmpty ? 'Your cart is empty' : `${items.length} items in cart`
      }))
    );
  }

  addItem(item: CartItem) {
    const currentItems = this.cartItems$.value;
    this.cartItems$.next([...currentItems, item]);
  }

  clearCart() {
    this.cartItems$.next([]);
  }
}
```

## Advanced Conditional Patterns

### 1. Complex Business Logic

```typescript
// Advanced conditional workflows
@Injectable()
export class OrderProcessingService {
  processOrder(order: Order): Observable<ProcessingResult> {
    return iif(
      () => this.isExpressOrder(order),
      this.processExpressOrder(order),
      iif(
        () => this.isBulkOrder(order),
        this.processBulkOrder(order),
        this.processStandardOrder(order)
      )
    ).pipe(
      mergeMap(result => this.validateProcessingResult(result)),
      defaultIfEmpty(this.createFailureResult('No processing result'))
    );
  }

  private isExpressOrder(order: Order): boolean {
    return order.priority === 'express' && order.total > 100;
  }

  private isBulkOrder(order: Order): boolean {
    return order.items.length > 10 || order.total > 1000;
  }

  private processExpressOrder(order: Order): Observable<ProcessingResult> {
    return this.http.post<ProcessingResult>('/api/orders/express', order);
  }

  private processBulkOrder(order: Order): Observable<ProcessingResult> {
    return this.http.post<ProcessingResult>('/api/orders/bulk', order);
  }

  private processStandardOrder(order: Order): Observable<ProcessingResult> {
    return this.http.post<ProcessingResult>('/api/orders/standard', order);
  }

  private validateProcessingResult(result: ProcessingResult): Observable<ProcessingResult> {
    return iif(
      () => result.isValid && result.confirmationNumber,
      of(result),
      throwError('Invalid processing result')
    );
  }

  private createFailureResult(reason: string): ProcessingResult {
    return {
      isValid: false,
      confirmationNumber: '',
      error: reason,
      timestamp: new Date()
    };
  }
}
```

### 2. Dynamic Content Loading

```typescript
// Content adaptation based on user context
@Component({})
export class AdaptiveContentComponent implements OnInit {
  content$!: Observable<ContentBlock[]>;

  ngOnInit() {
    this.content$ = combineLatest([
      this.userService.getCurrentUser(),
      this.deviceService.getDeviceInfo(),
      this.featureService.getEnabledFeatures()
    ]).pipe(
      switchMap(([user, device, features]) => 
        this.getAdaptedContent(user, device, features)
      )
    );
  }

  private getAdaptedContent(
    user: User, 
    device: DeviceInfo, 
    features: string[]
  ): Observable<ContentBlock[]> {
    return iif(
      () => device.isMobile,
      this.getMobileContent(user, features),
      iif(
        () => device.isTablet,
        this.getTabletContent(user, features),
        this.getDesktopContent(user, features)
      )
    ).pipe(
      mergeMap(content => from(content)),
      filter(block => this.isContentApplicable(block, user, features)),
      defaultIfEmpty(this.getDefaultContent()),
      toArray()
    );
  }

  private isContentApplicable(
    block: ContentBlock, 
    user: User, 
    features: string[]
  ): boolean {
    return block.requiredFeatures.every(feature => features.includes(feature)) &&
           (block.userRoles.length === 0 || block.userRoles.includes(user.role));
  }

  private getMobileContent(user: User, features: string[]): Observable<ContentBlock[]> {
    return this.http.get<ContentBlock[]>('/api/content/mobile');
  }

  private getTabletContent(user: User, features: string[]): Observable<ContentBlock[]> {
    return this.http.get<ContentBlock[]>('/api/content/tablet');
  }

  private getDesktopContent(user: User, features: string[]): Observable<ContentBlock[]> {
    return this.http.get<ContentBlock[]>('/api/content/desktop');
  }

  private getDefaultContent(): ContentBlock {
    return {
      id: 'default',
      type: 'message',
      content: 'Welcome! Content is loading...',
      requiredFeatures: [],
      userRoles: []
    };
  }
}
```

### 3. Validation Pipelines

```typescript
// Multi-stage validation with conditional logic
@Injectable()
export class ValidationPipelineService {
  validateUserData(userData: UserData): Observable<ValidationResult> {
    return of(userData).pipe(
      // Stage 1: Basic validation
      mergeMap(data => this.basicValidation(data)),
      
      // Stage 2: Conditional advanced validation
      mergeMap(result => 
        iif(
          () => result.isValid && result.data.requiresAdvancedValidation,
          this.advancedValidation(result.data),
          of(result)
        )
      ),
      
      // Stage 3: External validation if needed
      mergeMap(result =>
        iif(
          () => result.isValid && result.data.requiresExternalValidation,
          this.externalValidation(result.data),
          of(result)
        )
      ),
      
      // Final validation summary
      map(result => this.createFinalResult(result)),
      defaultIfEmpty(this.createFailureResult('Validation pipeline failed'))
    );
  }

  private basicValidation(data: UserData): Observable<ValidationResult> {
    const errors: string[] = [];
    
    if (!data.email || !this.isValidEmail(data.email)) {
      errors.push('Invalid email format');
    }
    
    if (!data.username || data.username.length < 3) {
      errors.push('Username must be at least 3 characters');
    }

    return of({
      isValid: errors.length === 0,
      errors,
      data,
      stage: 'basic'
    });
  }

  private advancedValidation(data: UserData): Observable<ValidationResult> {
    return this.http.post<ValidationResponse>('/api/validate/advanced', data).pipe(
      map(response => ({
        isValid: response.isValid,
        errors: response.errors || [],
        data,
        stage: 'advanced'
      }))
    );
  }

  private externalValidation(data: UserData): Observable<ValidationResult> {
    return this.http.post<ValidationResponse>('/api/validate/external', data).pipe(
      map(response => ({
        isValid: response.isValid,
        errors: response.errors || [],
        data,
        stage: 'external'
      }))
    );
  }

  private createFinalResult(result: ValidationResult): ValidationResult {
    return {
      ...result,
      summary: `Validation ${result.isValid ? 'passed' : 'failed'} at ${result.stage} stage`,
      timestamp: new Date()
    };
  }

  private createFailureResult(reason: string): ValidationResult {
    return {
      isValid: false,
      errors: [reason],
      data: null,
      stage: 'pipeline',
      timestamp: new Date()
    };
  }

  private isValidEmail(email: string): boolean {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(email);
  }
}
```

## Best Practices

### 1. Effective Conditional Logic

```typescript
// ✅ Use iif() for subscription-time decisions
const adaptiveContent$ = iif(
  () => this.userService.isPremiumUser(),
  this.getPremiumContent(),
  this.getBasicContent()
);

// ✅ Provide meaningful defaults
const safeData$ = apiCall$.pipe(
  defaultIfEmpty([]),
  catchError(() => of([]))
);

// ✅ Combine conditions effectively
const complexValidation$ = formData$.pipe(
  mergeMap(data => from([
    this.validateRequired(data),
    this.validateFormat(data),
    this.validateBusiness(data)
  ])),
  every(result => result.isValid)
);
```

### 2. Performance Optimization

```typescript
// ✅ Cache conditional results
const cachedFeatureCheck$ = this.featureFlags$.pipe(
  map(flags => flags.includes('newFeature')),
  distinctUntilChanged(),
  shareReplay(1)
);

// ✅ Use efficient finding
const efficientSearch$ = largeDataSet$.pipe(
  find(item => item.id === targetId),
  defaultIfEmpty(null)
);
```

### 3. Error Handling

```typescript
// ✅ Handle empty states gracefully
const robustData$ = source$.pipe(
  defaultIfEmpty([]),
  catchError(error => {
    console.error('Data loading failed:', error);
    return of(this.getFallbackData());
  })
);

// ✅ Validate conditions safely
const safeConditional$ = iif(
  () => {
    try {
      return this.complexCondition();
    } catch (error) {
      console.error('Condition evaluation failed:', error);
      return false;
    }
  },
  this.successPath(),
  this.fallbackPath()
);
```

## Common Pitfalls

### 1. Complex Nested Conditions

```typescript
// ❌ Too many nested iif operators
const overly_complex$ = iif(
  () => condition1,
  iif(() => condition2, 
    iif(() => condition3, stream1$, stream2$),
    stream3$
  ),
  stream4$
);

// ✅ Use helper functions for clarity
const clear$ = this.getStreamBasedOnConditions();

private getStreamBasedOnConditions(): Observable<any> {
  if (condition1 && condition2 && condition3) return stream1$;
  if (condition1 && condition2) return stream2$;
  if (condition1) return stream3$;
  return stream4$;
}
```

### 2. Not Handling Empty Cases

```typescript
// ❌ Not providing defaults
const risky$ = source$.pipe(
  filter(item => item.isValid)
  // Could emit nothing if no items are valid
);

// ✅ Always provide fallbacks
const safe$ = source$.pipe(
  filter(item => item.isValid),
  defaultIfEmpty([])
);
```

## Exercises

### Exercise 1: Smart Content Router
Create a content routing system that dynamically selects content based on user preferences, device capabilities, and feature flags.

### Exercise 2: Advanced Form Validator
Build a multi-stage form validation system that conditionally applies different validation rules based on user input.

### Exercise 3: Adaptive Data Loader
Implement a data loading system that adapts its strategy based on network conditions, user preferences, and data availability.

## Summary

Conditional operators provide powerful tools for implementing branching logic in reactive streams:

- **iif()**: Choose between Observables at subscription time
- **defaultIfEmpty()**: Provide fallback values for empty streams
- **every()**: Test if all values meet a condition
- **find()**: Locate the first matching value
- **findIndex()**: Get the index of the first match
- **isEmpty()**: Check if a stream is empty

These operators enable sophisticated control flow while maintaining the reactive paradigm, making your applications more adaptive and robust.

## Next Steps

In the next lesson, we'll explore **Mathematical Operators**, which provide aggregation, calculation, and statistical operations for reactive streams.
