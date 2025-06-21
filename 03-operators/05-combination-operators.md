# Combination Operators

## Overview

Combination operators are powerful tools that allow you to combine multiple Observable streams into a single stream. These operators enable you to merge, join, and coordinate data from different sources, making them essential for complex reactive applications where you need to work with multiple data streams simultaneously.

## Learning Objectives

After completing this lesson, you will be able to:
- Combine multiple Observable streams using various strategies
- Understand the timing and behavior differences between combination operators
- Handle complex data synchronization scenarios
- Apply combination operators in real-world Angular applications
- Choose the appropriate combination strategy for different use cases

## Core Combination Operators

### 1. combineLatest() - Combine Latest Values

The `combineLatest()` operator combines the latest values from multiple Observables whenever any of them emits.

```typescript
import { combineLatest, of, interval } from 'rxjs';
import { map, startWith } from 'rxjs/operators';

// Basic combination
const numbers$ = interval(1000);
const letters$ = of('A', 'B', 'C');

const combined$ = combineLatest([numbers$, letters$]);
combined$.subscribe(([num, letter]) => console.log(`${num}-${letter}`));
// Output: [0, 'C'], [1, 'C'], [2, 'C'], ...

// Angular form example
@Component({
  template: `
    <form [formGroup]="userForm">
      <input formControlName="firstName" placeholder="First Name">
      <input formControlName="lastName" placeholder="Last Name">
      <input formControlName="email" placeholder="Email">
    </form>
    
    <div *ngIf="formSummary$ | async as summary">
      <h3>Form Summary</h3>
      <p>Full Name: {{ summary.fullName }}</p>
      <p>Email: {{ summary.email }}</p>
      <p>Is Valid: {{ summary.isValid ? 'Yes' : 'No' }}</p>
    </div>
  `
})
export class CombinedFormComponent {
  userForm = new FormGroup({
    firstName: new FormControl(''),
    lastName: new FormControl(''),
    email: new FormControl('')
  });

  formSummary$ = combineLatest([
    this.userForm.get('firstName')!.valueChanges.pipe(startWith('')),
    this.userForm.get('lastName')!.valueChanges.pipe(startWith('')),
    this.userForm.get('email')!.valueChanges.pipe(startWith('')),
    this.userForm.statusChanges.pipe(startWith('INVALID'))
  ]).pipe(
    map(([firstName, lastName, email, status]) => ({
      fullName: `${firstName} ${lastName}`.trim(),
      email,
      isValid: status === 'VALID'
    }))
  );
}
```

**Marble Diagram:**
```
A:      -1---2---3---4|
B:      --a---b---c--|
combineLatest([A, B])
Output: --[1,a][2,a][2,b][3,b][3,c][4,c]|
```

### 2. zip() - Pair Values by Index

The `zip()` operator combines values from multiple Observables by pairing them based on their emission index.

```typescript
import { zip, of, interval } from 'rxjs';
import { map, take } from 'rxjs/operators';

// Basic zipping
const numbers$ = of(1, 2, 3, 4);
const letters$ = of('A', 'B', 'C');

const zipped$ = zip(numbers$, letters$);
zipped$.subscribe(([num, letter]) => console.log(`${num}-${letter}`));
// Output: [1,'A'], [2,'B'], [3,'C']

// Different timing
const slow$ = interval(1000).pipe(take(3));
const fast$ = interval(300).pipe(take(3));

const timed$ = zip(slow$, fast$);
// Waits for both to emit at each index

// Angular data combination
@Injectable()
export class DataCombinationService {
  
  // Combine user profile with settings and preferences
  getUserCompleteData(userId: string): Observable<CompleteUserData> {
    return zip(
      this.http.get<UserProfile>(`/api/users/${userId}`),
      this.http.get<UserSettings>(`/api/users/${userId}/settings`),
      this.http.get<UserPreferences>(`/api/users/${userId}/preferences`)
    ).pipe(
      map(([profile, settings, preferences]) => ({
        profile,
        settings,
        preferences,
        lastUpdated: new Date()
      }))
    );
  }

  // Process parallel uploads
  uploadMultipleFiles(files: File[]): Observable<UploadResult[]> {
    const uploadObservables = files.map(file => this.uploadFile(file));
    
    return zip(...uploadObservables).pipe(
      map(results => results.map((result, index) => ({
        ...result,
        fileName: files[index].name,
        fileSize: files[index].size
      })))
    );
  }

  private uploadFile(file: File): Observable<UploadResult> {
    const formData = new FormData();
    formData.append('file', file);
    return this.http.post<UploadResult>('/api/upload', formData);
  }
}
```

**Marble Diagram:**
```
A:     -1---2---3|
B:     --a---b---c|
zip(A, B)
Output: --[1,a]--[2,b]--[3,c]|
```

### 3. merge() - Merge All Emissions

The `merge()` operator merges multiple Observables into a single Observable that emits all values from all sources.

```typescript
import { merge, fromEvent, interval } from 'rxjs';
import { map } from 'rxjs/operators';

// Basic merging
const clicks$ = fromEvent(document, 'click').pipe(map(() => 'click'));
const timer$ = interval(1000).pipe(map(() => 'timer'));

const merged$ = merge(clicks$, timer$);
merged$.subscribe(value => console.log(value));
// Output: mix of 'click' and 'timer' events

// Angular notification system
@Injectable()
export class NotificationService {
  private errorNotifications$ = new Subject<Notification>();
  private successNotifications$ = new Subject<Notification>();
  private infoNotifications$ = new Subject<Notification>();

  // Merge all notification types
  allNotifications$ = merge(
    this.errorNotifications$.pipe(map(n => ({ ...n, type: 'error' }))),
    this.successNotifications$.pipe(map(n => ({ ...n, type: 'success' }))),
    this.infoNotifications$.pipe(map(n => ({ ...n, type: 'info' })))
  );

  showError(message: string) {
    this.errorNotifications$.next({ message, timestamp: Date.now() });
  }

  showSuccess(message: string) {
    this.successNotifications$.next({ message, timestamp: Date.now() });
  }

  showInfo(message: string) {
    this.infoNotifications$.next({ message, timestamp: Date.now() });
  }
}

@Component({
  template: `
    <div class="notifications">
      <div 
        *ngFor="let notification of notifications$ | async" 
        [class]="'notification notification-' + notification.type">
        {{ notification.message }}
      </div>
    </div>
  `
})
export class NotificationComponent {
  notifications$ = this.notificationService.allNotifications$;

  constructor(private notificationService: NotificationService) {}
}
```

**Marble Diagram:**
```
A:     -1---3---5|
B:     --2---4---6|
merge(A, B)
Output: -1-2-3-4-5-6|
```

### 4. concat() - Sequential Concatenation

The `concat()` operator concatenates multiple Observables sequentially, waiting for each to complete before subscribing to the next.

```typescript
import { concat, of, timer } from 'rxjs';
import { map } from 'rxjs/operators';

// Basic concatenation
const first$ = of(1, 2, 3);
const second$ = of(4, 5, 6);
const third$ = of(7, 8, 9);

const concatenated$ = concat(first$, second$, third$);
concatenated$.subscribe(value => console.log(value));
// Output: 1, 2, 3, 4, 5, 6, 7, 8, 9

// Sequential API calls
@Injectable()
export class SequentialDataService {
  
  // Load data in sequence for dependency reasons
  loadApplicationData(): Observable<AppData> {
    return concat(
      this.loadUserData().pipe(
        tap(userData => console.log('User data loaded'))
      ),
      this.loadUserPreferences().pipe(
        tap(preferences => console.log('Preferences loaded'))
      ),
      this.loadUserDashboard().pipe(
        tap(dashboard => console.log('Dashboard loaded'))
      )
    ).pipe(
      scan((acc, data) => ({ ...acc, ...data }), {} as AppData),
      last() // Only emit the final accumulated result
    );
  }

  // Sequential form steps
  processMultiStepForm(formData: MultiStepFormData): Observable<ProcessResult> {
    return concat(
      this.validateStep1(formData.step1),
      this.validateStep2(formData.step2),
      this.validateStep3(formData.step3),
      this.submitForm(formData)
    ).pipe(
      scan((results, result) => [...results, result], [] as ProcessResult[]),
      last(),
      map(results => results[results.length - 1]) // Return final result
    );
  }

  private loadUserData(): Observable<UserData> {
    return this.http.get<UserData>('/api/user');
  }

  private loadUserPreferences(): Observable<UserPreferences> {
    return this.http.get<UserPreferences>('/api/user/preferences');
  }

  private loadUserDashboard(): Observable<DashboardData> {
    return this.http.get<DashboardData>('/api/user/dashboard');
  }
}
```

### 5. forkJoin() - Wait for All to Complete

The `forkJoin()` operator waits for all provided Observables to complete, then emits the last value from each as an array.

```typescript
import { forkJoin, of } from 'rxjs';
import { map, delay } from 'rxjs/operators';

// Basic fork join
const obs1$ = of('A').pipe(delay(1000));
const obs2$ = of('B').pipe(delay(2000));
const obs3$ = of('C').pipe(delay(1500));

const forked$ = forkJoin([obs1$, obs2$, obs3$]);
forked$.subscribe(result => console.log(result));
// Output after 2 seconds: ['A', 'B', 'C']

// Angular dashboard data loading
@Component({
  template: `
    <div *ngIf="isLoading$ | async" class="loading">
      Loading dashboard data...
    </div>

    <div *ngIf="dashboardData$ | async as data" class="dashboard">
      <div class="user-info">
        <h2>{{ data.user.name }}</h2>
        <p>{{ data.user.email }}</p>
      </div>

      <div class="stats">
        <div *ngFor="let stat of data.statistics" class="stat-card">
          <h3>{{ stat.label }}</h3>
          <p>{{ stat.value }}</p>
        </div>
      </div>

      <div class="recent-activities">
        <h3>Recent Activities</h3>
        <ul>
          <li *ngFor="let activity of data.activities">
            {{ activity.description }} - {{ activity.date | date }}
          </li>
        </ul>
      </div>
    </div>
  `
})
export class DashboardComponent implements OnInit {
  dashboardData$!: Observable<DashboardData>;
  isLoading$ = new BehaviorSubject<boolean>(true);

  constructor(
    private userService: UserService,
    private statisticsService: StatisticsService,
    private activityService: ActivityService
  ) {}

  ngOnInit() {
    this.dashboardData$ = forkJoin({
      user: this.userService.getCurrentUser(),
      statistics: this.statisticsService.getUserStatistics(),
      activities: this.activityService.getRecentActivities(),
      preferences: this.userService.getUserPreferences()
    }).pipe(
      map(data => ({
        ...data,
        lastUpdated: new Date()
      })),
      tap(() => this.isLoading$.next(false)),
      catchError(error => {
        console.error('Failed to load dashboard:', error);
        this.isLoading$.next(false);
        return of(null);
      })
    );
  }
}

// Parallel API calls with error handling
@Injectable()
export class ParallelDataService {
  
  loadProductDetails(productId: string): Observable<ProductDetails> {
    return forkJoin({
      product: this.http.get<Product>(`/api/products/${productId}`),
      reviews: this.http.get<Review[]>(`/api/products/${productId}/reviews`),
      recommendations: this.http.get<Product[]>(`/api/products/${productId}/recommendations`),
      inventory: this.http.get<Inventory>(`/api/products/${productId}/inventory`)
    }).pipe(
      map(({ product, reviews, recommendations, inventory }) => ({
        product,
        reviews,
        recommendations,
        inventory,
        averageRating: this.calculateAverageRating(reviews),
        isInStock: inventory.quantity > 0
      })),
      catchError(error => {
        console.error('Failed to load product details:', error);
        return throwError(`Product ${productId} could not be loaded`);
      })
    );
  }

  private calculateAverageRating(reviews: Review[]): number {
    if (reviews.length === 0) return 0;
    const sum = reviews.reduce((acc, review) => acc + review.rating, 0);
    return sum / reviews.length;
  }
}
```

**Marble Diagram:**
```
A:        -1-2-3|
B:        --a-b-c|
C:        ---x-y-z|
forkJoin([A, B, C])
Output:   -------[3,c,z]|
```

### 6. race() - First to Emit Wins

The `race()` operator returns an Observable that mirrors the first source Observable to emit a value.

```typescript
import { race, timer, fromEvent } from 'rxjs';
import { map, mapTo } from 'rxjs/operators';

// Basic race
const slow$ = timer(1000).pipe(mapTo('slow'));
const fast$ = timer(500).pipe(mapTo('fast'));

const winner$ = race(slow$, fast$);
winner$.subscribe(value => console.log(value));
// Output: 'fast'

// Race between user action and timeout
@Component({})
export class TimeoutComponent {
  
  showModalWithTimeout(): Observable<ModalResult> {
    const userAction$ = this.modalService.waitForUserAction().pipe(
      map(action => ({ type: 'user-action', action }))
    );
    
    const timeout$ = timer(10000).pipe(
      map(() => ({ type: 'timeout', action: 'auto-close' }))
    );

    return race(userAction$, timeout$).pipe(
      tap(result => {
        if (result.type === 'timeout') {
          this.notificationService.showWarning('Modal auto-closed due to inactivity');
        }
      })
    );
  }
}

// Race between multiple data sources
@Injectable()
export class FallbackDataService {
  
  getDataWithFallback(id: string): Observable<Data> {
    const primarySource$ = this.http.get<Data>(`/api/primary/data/${id}`).pipe(
      delay(100), // Small delay to prefer primary
      map(data => ({ ...data, source: 'primary' }))
    );

    const cacheSource$ = this.cacheService.get<Data>(`data-${id}`).pipe(
      map(data => ({ ...data, source: 'cache' }))
    );

    const backupSource$ = this.http.get<Data>(`/api/backup/data/${id}`).pipe(
      map(data => ({ ...data, source: 'backup' }))
    );

    return race(
      primarySource$,
      cacheSource$,
      backupSource$
    ).pipe(
      catchError(error => {
        console.error('All data sources failed:', error);
        return of({ id, source: 'default', data: null });
      })
    );
  }
}
```

### 7. withLatestFrom() - Include Latest Values

The `withLatestFrom()` operator emits values from the source Observable combined with the latest values from other Observables.

```typescript
import { interval, fromEvent } from 'rxjs';
import { withLatestFrom, map } from 'rxjs/operators';

// Basic usage
const source$ = interval(1000);
const other$ = fromEvent(document, 'click');

const combined$ = source$.pipe(
  withLatestFrom(other$),
  map(([interval, click]) => `Interval: ${interval}, Click: ${click.type}`)
);

// Angular form with calculated fields
@Component({
  template: `
    <form [formGroup]="orderForm">
      <input formControlName="quantity" type="number" placeholder="Quantity">
      <input formControlName="price" type="number" placeholder="Unit Price">
      <input formControlName="discount" type="number" placeholder="Discount %">
      
      <div class="calculation">
        <p>Subtotal: {{ calculation$ | async | async }}?.subtotal | currency }}</p>
        <p>Discount: {{ (calculation$ | async)?.discountAmount | currency }}</p>
        <p>Total: {{ (calculation$ | async)?.total | currency }}</p>
      </div>
    </form>
  `
})
export class CalculatedFormComponent {
  orderForm = new FormGroup({
    quantity: new FormControl(1),
    price: new FormControl(0),
    discount: new FormControl(0)
  });

  calculation$ = this.orderForm.get('quantity')!.valueChanges.pipe(
    startWith(1),
    withLatestFrom(
      this.orderForm.get('price')!.valueChanges.pipe(startWith(0)),
      this.orderForm.get('discount')!.valueChanges.pipe(startWith(0))
    ),
    map(([quantity, price, discount]) => {
      const subtotal = quantity * price;
      const discountAmount = subtotal * (discount / 100);
      const total = subtotal - discountAmount;
      
      return { subtotal, discountAmount, total };
    })
  );
}

// Real-time data with context
@Injectable()
export class ContextualDataService {
  private currentUser$ = new BehaviorSubject<User | null>(null);
  private userPreferences$ = new BehaviorSubject<UserPreferences | null>(null);

  // Combine real-time data with user context
  getPersonalizedData(): Observable<PersonalizedData> {
    return this.realTimeDataStream$.pipe(
      withLatestFrom(
        this.currentUser$,
        this.userPreferences$
      ),
      map(([data, user, preferences]) => ({
        ...data,
        personalizedFor: user?.id,
        filteredByPreferences: this.applyPreferences(data, preferences),
        timestamp: Date.now()
      })),
      filter(data => data.personalizedFor !== null)
    );
  }

  private applyPreferences(data: any, preferences: UserPreferences | null): any {
    if (!preferences) return data;
    
    // Apply user preferences to filter/transform data
    return data.filter((item: any) => 
      preferences.categories.includes(item.category)
    );
  }
}
```

### 8. startWith() - Start with Initial Values

The `startWith()` operator emits specified values before the source Observable starts emitting.

```typescript
import { interval, EMPTY } from 'rxjs';
import { startWith, map } from 'rxjs/operators';

// Basic usage
const numbers$ = interval(1000);
const withInitial$ = numbers$.pipe(
  startWith(-1, -2, -3)
);
// Emits: -1, -2, -3, 0, 1, 2, 3, ...

// Angular loading states
@Component({
  template: `
    <div class="user-list">
      <div *ngIf="users$ | async as users">
        <div *ngFor="let user of users" class="user-card">
          <h3>{{ user.name }}</h3>
          <p>{{ user.email }}</p>
        </div>
      </div>
      
      <div *ngIf="(users$ | async)?.length === 0" class="no-users">
        No users found
      </div>
    </div>
  `
})
export class UserListComponent {
  users$ = this.userService.getUsers().pipe(
    startWith([]), // Start with empty array to show "No users found"
    catchError(error => {
      console.error('Failed to load users:', error);
      return of([]);
    })
  );

  constructor(private userService: UserService) {}
}

// Form with default values
@Component({})
export class DefaultFormComponent {
  form = new FormGroup({
    theme: new FormControl(''),
    language: new FormControl('')
  });

  // Provide default selections
  theme$ = this.form.get('theme')!.valueChanges.pipe(
    startWith('light')
  );

  language$ = this.form.get('language')!.valueChanges.pipe(
    startWith('en')
  );

  settings$ = combineLatest([this.theme$, this.language$]).pipe(
    map(([theme, language]) => ({ theme, language }))
  );
}
```

## Advanced Combination Patterns

### 1. Conditional Combination

```typescript
// Combine based on conditions
@Injectable()
export class ConditionalCombinationService {
  
  getCombinedData(useCache: boolean): Observable<CombinedData> {
    const baseData$ = this.http.get<BaseData>('/api/base');
    
    const additionalData$ = useCache 
      ? this.cacheService.get<AdditionalData>('additional')
      : this.http.get<AdditionalData>('/api/additional');

    return combineLatest([baseData$, additionalData$]).pipe(
      map(([base, additional]) => ({ base, additional }))
    );
  }

  // Dynamic combination based on user role
  getDashboardData(userRole: string): Observable<DashboardData> {
    const commonData$ = this.getCommonData();
    
    if (userRole === 'admin') {
      return combineLatest([
        commonData$,
        this.getAdminData(),
        this.getSystemMetrics()
      ]).pipe(
        map(([common, admin, metrics]) => ({ 
          ...common, 
          admin, 
          metrics 
        }))
      );
    } else {
      return combineLatest([
        commonData$,
        this.getUserSpecificData()
      ]).pipe(
        map(([common, userSpecific]) => ({ 
          ...common, 
          userSpecific 
        }))
      );
    }
  }
}
```

### 2. Error Handling in Combinations

```typescript
// Robust error handling
@Injectable()
export class RobustCombinationService {
  
  getCombinedDataWithFallback(): Observable<CombinedData> {
    return forkJoin({
      primary: this.http.get<PrimaryData>('/api/primary').pipe(
        catchError(error => {
          console.error('Primary data failed:', error);
          return of(this.getDefaultPrimaryData());
        })
      ),
      secondary: this.http.get<SecondaryData>('/api/secondary').pipe(
        catchError(error => {
          console.error('Secondary data failed:', error);
          return of(this.getDefaultSecondaryData());
        })
      ),
      optional: this.http.get<OptionalData>('/api/optional').pipe(
        catchError(error => {
          console.warn('Optional data failed:', error);
          return of(null); // Optional data can be null
        })
      )
    }).pipe(
      map(({ primary, secondary, optional }) => ({
        primary,
        secondary,
        optional,
        hasOptionalData: optional !== null
      }))
    );
  }

  // Partial success handling
  getCriticalData(): Observable<PartialData> {
    return forkJoin({
      critical: this.http.get<CriticalData>('/api/critical'), // Must succeed
      important: this.http.get<ImportantData>('/api/important').pipe(
        catchError(() => of(null)) // Can fail
      ),
      nice: this.http.get<NiceData>('/api/nice').pipe(
        catchError(() => of(null)) // Can fail
      )
    }).pipe(
      map(({ critical, important, nice }) => ({
        critical,
        important,
        nice,
        completeness: this.calculateCompleteness({ critical, important, nice })
      }))
    );
  }

  private calculateCompleteness(data: any): number {
    const total = 3;
    const available = Object.values(data).filter(v => v !== null).length;
    return (available / total) * 100;
  }
}
```

### 3. Performance Optimization

```typescript
// Optimized combinations
@Injectable()
export class OptimizedCombinationService {
  
  // Cache expensive combinations
  private expensiveCombination$ = combineLatest([
    this.expensiveDataSource1$,
    this.expensiveDataSource2$,
    this.expensiveDataSource3$
  ]).pipe(
    map(([source1, source2, source3]) => 
      this.expensiveProcessing(source1, source2, source3)
    ),
    shareReplay(1), // Cache the result
    takeUntil(this.destroy$)
  );

  // Debounced combination for rapid changes
  getDebouncedCombination(): Observable<CombinedResult> {
    return combineLatest([
      this.rapidlyChangingSource1$.pipe(debounceTime(300)),
      this.rapidlyChangingSource2$.pipe(debounceTime(300)),
      this.stableSource$
    ]).pipe(
      map(([source1, source2, source3]) => 
        this.combineData(source1, source2, source3)
      ),
      distinctUntilChanged()
    );
  }

  // Selective combination updates
  getSelectiveCombination(): Observable<SelectiveResult> {
    return combineLatest([
      this.frequentSource$.pipe(distinctUntilChanged()),
      this.infrequentSource$.pipe(startWith(null)),
      this.configSource$.pipe(startWith(this.defaultConfig))
    ]).pipe(
      filter(([frequent, infrequent, config]) => 
        frequent !== null && config !== null
      ),
      map(([frequent, infrequent, config]) => ({
        frequent,
        infrequent,
        config,
        processed: this.processSelective(frequent, infrequent, config)
      }))
    );
  }
}
```

## Real-World Angular Examples

### 1. Advanced Shopping Cart

```typescript
@Component({
  template: `
    <div class="shopping-cart">
      <div class="cart-header">
        <h2>Shopping Cart ({{ itemCount$ | async }} items)</h2>
        <button 
          [disabled]="!(canCheckout$ | async)" 
          (click)="checkout()">
          Checkout - {{ cartTotal$ | async | currency }}
        </button>
      </div>

      <div class="cart-items">
        <div *ngFor="let item of cartItems$ | async" class="cart-item">
          <h4>{{ item.product.name }}</h4>
          <p>{{ item.product.price | currency }} x {{ item.quantity }}</p>
          <p>Subtotal: {{ item.subtotal | currency }}</p>
        </div>
      </div>

      <div class="cart-summary" *ngIf="cartSummary$ | async as summary">
        <p>Subtotal: {{ summary.subtotal | currency }}</p>
        <p>Tax: {{ summary.tax | currency }}</p>
        <p>Shipping: {{ summary.shipping | currency }}</p>
        <p class="total">Total: {{ summary.total | currency }}</p>
      </div>

      <div class="promotions" *ngIf="availablePromotions$ | async as promos">
        <h3>Available Promotions</h3>
        <div *ngFor="let promo of promos" class="promotion">
          <button (click)="applyPromotion(promo.code)">
            {{ promo.description }} - Save {{ promo.discount | currency }}
          </button>
        </div>
      </div>
    </div>
  `
})
export class AdvancedShoppingCartComponent {
  private cartItems$ = this.cartService.getCartItems();
  private userLocation$ = this.locationService.getUserLocation();
  private promotions$ = this.promotionService.getAvailablePromotions();
  private appliedPromotions$ = this.cartService.getAppliedPromotions();

  cartSummary$ = combineLatest([
    this.cartItems$,
    this.userLocation$,
    this.appliedPromotions$
  ]).pipe(
    map(([items, location, promotions]) => {
      const subtotal = items.reduce((sum, item) => sum + item.subtotal, 0);
      const tax = this.calculateTax(subtotal, location);
      const shipping = this.calculateShipping(items, location);
      const promotionDiscount = this.calculatePromotionDiscount(subtotal, promotions);
      const total = subtotal + tax + shipping - promotionDiscount;

      return { subtotal, tax, shipping, promotionDiscount, total };
    })
  );

  cartTotal$ = this.cartSummary$.pipe(
    map(summary => summary.total)
  );

  itemCount$ = this.cartItems$.pipe(
    map(items => items.reduce((count, item) => count + item.quantity, 0))
  );

  canCheckout$ = combineLatest([
    this.cartItems$,
    this.cartTotal$,
    this.authService.isAuthenticated$
  ]).pipe(
    map(([items, total, isAuth]) => 
      items.length > 0 && total > 0 && isAuth
    )
  );

  availablePromotions$ = combineLatest([
    this.promotions$,
    this.cartItems$,
    this.appliedPromotions$
  ]).pipe(
    map(([allPromos, items, applied]) => 
      allPromos.filter(promo => 
        this.isPromotionApplicable(promo, items) && 
        !applied.some(ap => ap.code === promo.code)
      )
    )
  );

  private calculateTax(subtotal: number, location: Location): number {
    return subtotal * (location.taxRate || 0);
  }

  private calculateShipping(items: CartItem[], location: Location): number {
    const weight = items.reduce((w, item) => w + (item.product.weight * item.quantity), 0);
    return this.shippingService.calculateShipping(weight, location);
  }

  private calculatePromotionDiscount(subtotal: number, promotions: AppliedPromotion[]): number {
    return promotions.reduce((discount, promo) => 
      discount + this.promotionService.calculateDiscount(promo, subtotal), 0
    );
  }

  private isPromotionApplicable(promotion: Promotion, items: CartItem[]): boolean {
    return this.promotionService.isApplicable(promotion, items);
  }
}
```

### 2. Real-time Collaborative Editor

```typescript
@Component({
  template: `
    <div class="collaborative-editor">
      <div class="editor-header">
        <div class="collaborators">
          <span *ngFor="let user of activeUsers$ | async" 
                class="collaborator"
                [style.color]="user.color">
            {{ user.name }}
          </span>
        </div>
        <div class="connection-status">
          Status: {{ connectionStatus$ | async }}
        </div>
      </div>

      <textarea 
        #editor
        [(ngModel)]="content"
        (input)="onContentChange($event)"
        class="editor">
      </textarea>

      <div class="editor-footer">
        <span>{{ documentStats$ | async | json }}</span>
        <button 
          [disabled]="!(canSave$ | async)"
          (click)="save()">
          Save {{ hasUnsavedChanges$ | async ? '*' : '' }}
        </button>
      </div>
    </div>
  `
})
export class CollaborativeEditorComponent implements OnInit, OnDestroy {
  content = '';
  private destroy$ = new Subject<void>();
  
  private localChanges$ = new Subject<ContentChange>();
  private remoteChanges$ = this.websocketService.getRemoteChanges();
  private connectionStatus$ = this.websocketService.getConnectionStatus();
  private saveRequests$ = new Subject<void>();

  // Combine local and remote changes
  allChanges$ = merge(
    this.localChanges$.pipe(map(change => ({ ...change, source: 'local' }))),
    this.remoteChanges$.pipe(map(change => ({ ...change, source: 'remote' })))
  );

  // Track active collaborators
  activeUsers$ = combineLatest([
    this.websocketService.getActiveUsers(),
    this.authService.getCurrentUser()
  ]).pipe(
    map(([users, currentUser]) => 
      users.filter(user => user.id !== currentUser.id)
    )
  );

  // Document statistics
  documentStats$ = combineLatest([
    this.localChanges$.pipe(startWith(null)),
    this.remoteChanges$.pipe(startWith(null))
  ]).pipe(
    map(() => ({
      wordCount: this.content.split(/\s+/).filter(w => w.length > 0).length,
      charCount: this.content.length,
      lastModified: new Date()
    }))
  );

  // Save state management
  hasUnsavedChanges$ = this.localChanges$.pipe(
    scan((hasChanges, change) => true, false),
    startWith(false)
  );

  canSave$ = combineLatest([
    this.hasUnsavedChanges$,
    this.connectionStatus$
  ]).pipe(
    map(([hasChanges, status]) => hasChanges && status === 'connected')
  );

  // Auto-save functionality
  autoSave$ = this.localChanges$.pipe(
    debounceTime(2000),
    withLatestFrom(this.canSave$),
    filter(([change, canSave]) => canSave),
    tap(() => this.performSave())
  );

  ngOnInit() {
    // Handle remote changes
    this.remoteChanges$.pipe(
      takeUntil(this.destroy$)
    ).subscribe(change => {
      this.applyRemoteChange(change);
    });

    // Start auto-save
    this.autoSave$.pipe(
      takeUntil(this.destroy$)
    ).subscribe();

    // Handle save requests
    this.saveRequests$.pipe(
      withLatestFrom(this.canSave$),
      filter(([_, canSave]) => canSave),
      exhaustMap(() => this.documentService.save(this.content)),
      takeUntil(this.destroy$)
    ).subscribe(
      result => console.log('Document saved:', result),
      error => console.error('Save failed:', error)
    );
  }

  onContentChange(event: Event) {
    const newContent = (event.target as HTMLTextAreaElement).value;
    const change: ContentChange = {
      content: newContent,
      timestamp: Date.now(),
      userId: this.authService.getCurrentUserId()
    };
    
    this.content = newContent;
    this.localChanges$.next(change);
    this.websocketService.broadcastChange(change);
  }

  save() {
    this.saveRequests$.next();
  }

  private applyRemoteChange(change: ContentChange) {
    // Apply operational transformation logic
    this.content = this.otService.applyChange(this.content, change);
  }

  private performSave() {
    this.saveRequests$.next();
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## Best Practices

### 1. Choose the Right Combination Strategy

```typescript
// ✅ Use combineLatest for: Form validation, UI state management
const formValidation$ = combineLatest([
  emailControl.valueChanges,
  passwordControl.valueChanges
]).pipe(
  map(([email, password]) => ({ email, password }))
);

// ✅ Use forkJoin for: Parallel API calls that must all complete
const pageData$ = forkJoin({
  user: this.userService.getUser(),
  settings: this.settingsService.getSettings(),
  notifications: this.notificationService.getNotifications()
});

// ✅ Use merge for: Event streams, notifications
const allEvents$ = merge(
  clickEvents$,
  keyboardEvents$,
  touchEvents$
);

// ✅ Use concat for: Sequential operations
const sequentialUpdates$ = concat(
  updateProfile$,
  updatePreferences$,
  updateNotifications$
);
```

### 2. Handle Timing Issues

```typescript
// ✅ Use startWith for initial values
const dataWithDefault$ = apiData$.pipe(
  startWith([]), // Provide empty array initially
  catchError(() => of([]))
);

// ✅ Use shareReplay to avoid multiple subscriptions
const expensiveData$ = expensiveOperation$.pipe(
  shareReplay(1)
);

// ✅ Handle different emission rates
const syncedData$ = fastSource$.pipe(
  withLatestFrom(slowSource$),
  map(([fast, slow]) => ({ fast, slow }))
);
```

### 3. Error Handling

```typescript
// ✅ Handle errors in individual streams
const robustCombination$ = combineLatest([
  stream1$.pipe(catchError(err => of(defaultValue1))),
  stream2$.pipe(catchError(err => of(defaultValue2))),
  stream3$.pipe(catchError(err => of(defaultValue3)))
]);

// ✅ Provide fallbacks for failed forkJoin
const dataWithFallback$ = forkJoin({
  primary: primarySource$,
  secondary: secondarySource$.pipe(
    catchError(() => of(null))
  )
}).pipe(
  catchError(error => of({ primary: null, secondary: null }))
);
```

## Common Pitfalls

### 1. Memory Leaks

```typescript
// ❌ Not unsubscribing from combinations
class LeakyComponent {
  ngOnInit() {
    combineLatest([source1$, source2$]).subscribe(); // Never unsubscribed!
  }
}

// ✅ Proper subscription management
class ProperComponent implements OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit() {
    combineLatest([source1$, source2$]).pipe(
      takeUntil(this.destroy$)
    ).subscribe();
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### 2. Performance Issues

```typescript
// ❌ Expensive operations in combination
const inefficient$ = combineLatest([source1$, source2$]).pipe(
  map(([s1, s2]) => expensiveOperation(s1, s2)) // Runs on every emission
);

// ✅ Debounce and optimize
const efficient$ = combineLatest([source1$, source2$]).pipe(
  debounceTime(300),
  map(([s1, s2]) => expensiveOperation(s1, s2)),
  shareReplay(1)
);
```

## Exercises

### Exercise 1: Multi-Source Dashboard
Create a dashboard that combines data from multiple APIs and handles partial failures gracefully.

### Exercise 2: Real-time Form Validation
Build a form that validates fields in real-time by combining multiple validation sources (client-side, server-side, business rules).

### Exercise 3: Collaborative Features
Implement a feature that combines local user actions with real-time updates from other users.

## Summary

Combination operators are essential for working with multiple data streams:

- **combineLatest()**: Combine latest values from all sources
- **zip()**: Pair values by index position
- **merge()**: Merge all emissions from multiple sources
- **concat()**: Sequential concatenation
- **forkJoin()**: Wait for all to complete
- **race()**: First to emit wins
- **withLatestFrom()**: Include latest values when source emits
- **startWith()**: Provide initial values

Choose the right combination strategy based on your timing requirements, error handling needs, and performance considerations.

## Next Steps

In the next lesson, we'll explore **Error Handling Operators**, which provide robust ways to handle and recover from errors in reactive streams.
