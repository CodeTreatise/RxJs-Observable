# Router & Observables

## Learning Objectives
- Master Angular Router's Observable APIs
- Implement navigation patterns with RxJS
- Handle route parameters and query strings reactively
- Build dynamic routing solutions
- Implement guards with Observable patterns
- Manage route-based data loading

## Introduction to Router Observables

Angular Router provides extensive Observable APIs for reactive navigation and route state management. Understanding these patterns is crucial for building dynamic, responsive applications.

### Core Router Observables
- **Route Parameters**: `params$`, `paramMap$`
- **Query Parameters**: `queryParams$`, `queryParamMap$`
- **Route Data**: `data$`
- **Navigation Events**: Router event streams
- **URL Changes**: Location service observables

## Route Parameters & Navigation

### Basic Parameter Handling

```typescript
// product-detail.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute, Router } from '@angular/router';
import { Observable, combineLatest } from 'rxjs';
import { map, switchMap, filter } from 'rxjs/operators';

@Component({
  selector: 'app-product-detail',
  template: `
    <div class="product-detail">
      <div *ngIf="product$ | async as product">
        <h1>{{ product.name }}</h1>
        <p>{{ product.description }}</p>
        <span class="price">{{ product.price | currency }}</span>
        
        <div class="variants" *ngIf="selectedVariant$ | async as variant">
          <h3>Selected Variant: {{ variant.name }}</h3>
        </div>
      </div>

      <div class="navigation">
        <button (click)="goToPrevious()" 
                [disabled]="!hasPrevious$ | async">
          Previous
        </button>
        <button (click)="goToNext()" 
                [disabled]="!hasNext$ | async">
          Next
        </button>
      </div>
    </div>
  `
})
export class ProductDetailComponent implements OnInit {
  
  // Primary route parameter
  readonly productId$ = this.route.paramMap.pipe(
    map(params => params.get('id')),
    filter(id => id !== null)
  );

  // Query parameters
  readonly variantId$ = this.route.queryParamMap.pipe(
    map(params => params.get('variant'))
  );

  // Combined parameter handling
  readonly routeParams$ = combineLatest([
    this.productId$,
    this.variantId$
  ]).pipe(
    map(([productId, variantId]) => ({ productId, variantId }))
  );

  // Data loading based on parameters
  readonly product$ = this.productId$.pipe(
    switchMap(id => this.productService.getProduct(id))
  );

  readonly selectedVariant$ = combineLatest([
    this.product$,
    this.variantId$
  ]).pipe(
    map(([product, variantId]) => 
      variantId ? product.variants.find(v => v.id === variantId) : null
    )
  );

  // Navigation state
  readonly hasPrevious$ = this.productId$.pipe(
    switchMap(id => this.productService.hasPrevious(id))
  );

  readonly hasNext$ = this.productId$.pipe(
    switchMap(id => this.productService.hasNext(id))
  );

  constructor(
    private route: ActivatedRoute,
    private router: Router,
    private productService: ProductService
  ) {}

  ngOnInit() {
    // React to route changes
    this.routeParams$.subscribe(({ productId, variantId }) => {
      console.log('Route changed:', { productId, variantId });
      // Additional side effects
    });
  }

  goToPrevious(): void {
    this.productId$.pipe(
      switchMap(id => this.productService.getPreviousId(id)),
      take(1)
    ).subscribe(prevId => {
      if (prevId) {
        this.router.navigate(['/products', prevId]);
      }
    });
  }

  goToNext(): void {
    this.productId$.pipe(
      switchMap(id => this.productService.getNextId(id)),
      take(1)
    ).subscribe(nextId => {
      if (nextId) {
        this.router.navigate(['/products', nextId]);
      }
    });
  }

  selectVariant(variantId: string): void {
    this.router.navigate([], {
      relativeTo: this.route,
      queryParams: { variant: variantId },
      queryParamsHandling: 'merge'
    });
  }
}
```

### Advanced Parameter Patterns

```typescript
// search.component.ts
@Component({
  selector: 'app-search',
  template: `
    <div class="search-page">
      <form [formGroup]="searchForm" class="search-form">
        <input formControlName="query" placeholder="Search...">
        <select formControlName="category">
          <option value="">All Categories</option>
          <option *ngFor="let cat of categories" [value]="cat.id">
            {{ cat.name }}
          </option>
        </select>
        <input type="number" formControlName="minPrice" placeholder="Min Price">
        <input type="number" formControlName="maxPrice" placeholder="Max Price">
      </form>

      <div class="results">
        <div *ngIf="loading$ | async" class="loading">Searching...</div>
        <div *ngFor="let item of results$ | async" class="result-item">
          {{ item.name }}
        </div>
      </div>

      <div class="pagination">
        <button (click)="previousPage()" 
                [disabled]="(currentPage$ | async) <= 1">
          Previous
        </button>
        <span>Page {{ currentPage$ | async }} of {{ totalPages$ | async }}</span>
        <button (click)="nextPage()" 
                [disabled]="(currentPage$ | async) >= (totalPages$ | async)">
          Next
        </button>
      </div>
    </div>
  `
})
export class SearchComponent implements OnInit {
  searchForm = this.fb.group({
    query: [''],
    category: [''],
    minPrice: [null],
    maxPrice: [null]
  });

  // Extract all query parameters
  readonly queryParams$ = this.route.queryParamMap.pipe(
    map(params => ({
      query: params.get('q') || '',
      category: params.get('category') || '',
      minPrice: params.get('minPrice') ? +params.get('minPrice') : null,
      maxPrice: params.get('maxPrice') ? +params.get('maxPrice') : null,
      page: params.get('page') ? +params.get('page') : 1,
      sortBy: params.get('sort') || 'relevance'
    }))
  );

  readonly currentPage$ = this.queryParams$.pipe(
    map(params => params.page)
  );

  readonly searchCriteria$ = this.queryParams$.pipe(
    map(params => ({
      query: params.query,
      category: params.category,
      minPrice: params.minPrice,
      maxPrice: params.maxPrice,
      sortBy: params.sortBy
    })),
    distinctUntilChanged((a, b) => JSON.stringify(a) === JSON.stringify(b))
  );

  readonly searchResults$ = combineLatest([
    this.searchCriteria$,
    this.currentPage$
  ]).pipe(
    debounceTime(300), // Avoid excessive API calls
    switchMap(([criteria, page]) => 
      this.searchService.search(criteria, page)
    ),
    shareReplay(1)
  );

  readonly results$ = this.searchResults$.pipe(
    map(response => response.items)
  );

  readonly totalPages$ = this.searchResults$.pipe(
    map(response => response.totalPages)
  );

  readonly loading$ = this.searchResults$.pipe(
    map(() => false),
    startWith(true),
    distinctUntilChanged()
  );

  constructor(
    private route: ActivatedRoute,
    private router: Router,
    private fb: FormBuilder,
    private searchService: SearchService
  ) {}

  ngOnInit() {
    // Sync form with URL parameters
    this.queryParams$.subscribe(params => {
      this.searchForm.patchValue({
        query: params.query,
        category: params.category,
        minPrice: params.minPrice,
        maxPrice: params.maxPrice
      }, { emitEvent: false });
    });

    // Update URL when form changes
    this.searchForm.valueChanges.pipe(
      debounceTime(500),
      distinctUntilChanged()
    ).subscribe(formValue => {
      this.updateUrl(formValue);
    });
  }

  private updateUrl(formValue: any): void {
    const queryParams: any = {};
    
    if (formValue.query) queryParams.q = formValue.query;
    if (formValue.category) queryParams.category = formValue.category;
    if (formValue.minPrice) queryParams.minPrice = formValue.minPrice;
    if (formValue.maxPrice) queryParams.maxPrice = formValue.maxPrice;

    this.router.navigate([], {
      relativeTo: this.route,
      queryParams,
      queryParamsHandling: 'merge'
    });
  }

  nextPage(): void {
    this.currentPage$.pipe(take(1)).subscribe(currentPage => {
      this.router.navigate([], {
        relativeTo: this.route,
        queryParams: { page: currentPage + 1 },
        queryParamsHandling: 'merge'
      });
    });
  }

  previousPage(): void {
    this.currentPage$.pipe(take(1)).subscribe(currentPage => {
      if (currentPage > 1) {
        this.router.navigate([], {
          relativeTo: this.route,
          queryParams: { page: currentPage - 1 },
          queryParamsHandling: 'merge'
        });
      }
    });
  }
}
```

## Router Events & Navigation Monitoring

### Navigation Event Streams

```typescript
// navigation.service.ts
import { Injectable } from '@angular/core';
import { Router, NavigationStart, NavigationEnd, NavigationError, NavigationCancel } from '@angular/router';
import { Observable, BehaviorSubject } from 'rxjs';
import { filter, map, distinctUntilChanged } from 'rxjs/operators';

export interface NavigationState {
  loading: boolean;
  url: string | null;
  error: string | null;
}

@Injectable({
  providedIn: 'root'
})
export class NavigationService {
  private navigationState$ = new BehaviorSubject<NavigationState>({
    loading: false,
    url: null,
    error: null
  });

  readonly navigationState = this.navigationState$.asObservable();

  readonly loading$ = this.navigationState.pipe(
    map(state => state.loading),
    distinctUntilChanged()
  );

  readonly currentUrl$ = this.navigationState.pipe(
    map(state => state.url),
    distinctUntilChanged()
  );

  readonly navigationError$ = this.navigationState.pipe(
    map(state => state.error),
    filter(error => error !== null)
  );

  // Specific event streams
  readonly navigationStart$ = this.router.events.pipe(
    filter(event => event instanceof NavigationStart)
  ) as Observable<NavigationStart>;

  readonly navigationEnd$ = this.router.events.pipe(
    filter(event => event instanceof NavigationEnd)
  ) as Observable<NavigationEnd>;

  readonly navigationError$ = this.router.events.pipe(
    filter(event => event instanceof NavigationError)
  ) as Observable<NavigationError>;

  readonly navigationCancel$ = this.router.events.pipe(
    filter(event => event instanceof NavigationCancel)
  ) as Observable<NavigationCancel>;

  constructor(private router: Router) {
    this.setupNavigationTracking();
  }

  private setupNavigationTracking(): void {
    // Track navigation start
    this.navigationStart$.subscribe(event => {
      this.updateState({
        loading: true,
        url: event.url,
        error: null
      });
    });

    // Track navigation end
    this.navigationEnd$.subscribe(event => {
      this.updateState({
        loading: false,
        url: event.url,
        error: null
      });
    });

    // Track navigation errors
    this.navigationError$.subscribe(event => {
      this.updateState({
        loading: false,
        url: event.url,
        error: event.error?.message || 'Navigation failed'
      });
    });

    // Track navigation cancellations
    this.navigationCancel$.subscribe(event => {
      this.updateState({
        loading: false,
        url: event.url,
        error: 'Navigation cancelled'
      });
    });
  }

  private updateState(partial: Partial<NavigationState>): void {
    const currentState = this.navigationState$.value;
    this.navigationState$.next({ ...currentState, ...partial });
  }
}

// Usage in app component
@Component({
  selector: 'app-root',
  template: `
    <div class="app">
      <div *ngIf="navigationService.loading$ | async" class="loading-bar">
        <div class="progress"></div>
      </div>

      <div *ngIf="navigationService.navigationError$ | async as error" 
           class="error-banner">
        {{ error }}
      </div>

      <router-outlet></router-outlet>
    </div>
  `
})
export class AppComponent {
  constructor(public navigationService: NavigationService) {}
}
```

### Route Data & Resolvers

```typescript
// data-resolver.service.ts
@Injectable()
export class UserDataResolver implements Resolve<User> {
  constructor(
    private userService: UserService,
    private router: Router
  ) {}

  resolve(route: ActivatedRouteSnapshot): Observable<User> {
    const userId = route.paramMap.get('id');
    
    if (!userId) {
      this.router.navigate(['/users']);
      return EMPTY;
    }

    return this.userService.getUser(userId).pipe(
      catchError(error => {
        console.error('Failed to load user:', error);
        this.router.navigate(['/users']);
        return EMPTY;
      })
    );
  }
}

// Route configuration
const routes: Routes = [
  {
    path: 'user/:id',
    component: UserDetailComponent,
    resolve: {
      user: UserDataResolver
    },
    data: {
      title: 'User Details',
      breadcrumb: 'User'
    }
  }
];

// Component using resolved data
@Component({
  selector: 'app-user-detail',
  template: `
    <div class="user-detail">
      <div *ngIf="user$ | async as user">
        <h1>{{ user.name }}</h1>
        <p>{{ user.email }}</p>
      </div>

      <nav class="breadcrumb">
        <span *ngFor="let crumb of breadcrumbs$ | async">
          {{ crumb }}
        </span>
      </nav>
    </div>
  `
})
export class UserDetailComponent implements OnInit {
  readonly user$ = this.route.data.pipe(
    map(data => data.user as User)
  );

  readonly routeData$ = this.route.data.pipe(
    distinctUntilChanged()
  );

  readonly breadcrumbs$ = this.routeData$.pipe(
    map(data => [
      'Home',
      'Users',
      data.breadcrumb || 'Details'
    ])
  );

  readonly pageTitle$ = this.routeData$.pipe(
    map(data => data.title as string)
  );

  constructor(private route: ActivatedRoute) {}

  ngOnInit() {
    // Set page title dynamically
    this.pageTitle$.subscribe(title => {
      document.title = `MyApp - ${title}`;
    });
  }
}
```

## Guards with Observables

### Authentication Guard

```typescript
// auth.guard.ts
@Injectable()
export class AuthGuard implements CanActivate, CanActivateChild {
  constructor(
    private authService: AuthService,
    private router: Router
  ) {}

  canActivate(route: ActivatedRouteSnapshot): Observable<boolean> {
    return this.checkAuth(route);
  }

  canActivateChild(route: ActivatedRouteSnapshot): Observable<boolean> {
    return this.checkAuth(route);
  }

  private checkAuth(route: ActivatedRouteSnapshot): Observable<boolean> {
    const requiredRoles = route.data.roles as string[] || [];
    
    return this.authService.currentUser$.pipe(
      map(user => {
        if (!user) {
          this.router.navigate(['/login'], {
            queryParams: { returnUrl: route.url.join('/') }
          });
          return false;
        }

        if (requiredRoles.length > 0 && !this.hasRequiredRole(user, requiredRoles)) {
          this.router.navigate(['/unauthorized']);
          return false;
        }

        return true;
      }),
      catchError(error => {
        console.error('Auth check failed:', error);
        this.router.navigate(['/login']);
        return of(false);
      })
    );
  }

  private hasRequiredRole(user: User, requiredRoles: string[]): boolean {
    return requiredRoles.some(role => user.roles.includes(role));
  }
}
```

### Confirmation Guard

```typescript
// unsaved-changes.guard.ts
export interface HasUnsavedChanges {
  hasUnsavedChanges(): Observable<boolean> | Promise<boolean> | boolean;
}

@Injectable()
export class UnsavedChangesGuard implements CanDeactivate<HasUnsavedChanges> {
  constructor(private dialog: MatDialog) {}

  canDeactivate(component: HasUnsavedChanges): Observable<boolean> | boolean {
    // Check if component has unsaved changes
    const hasChanges = component.hasUnsavedChanges();
    
    if (hasChanges instanceof Observable) {
      return hasChanges.pipe(
        switchMap(hasUnsaved => 
          hasUnsaved ? this.confirmLeave() : of(true)
        )
      );
    }
    
    if (hasChanges instanceof Promise) {
      return from(hasChanges).pipe(
        switchMap(hasUnsaved => 
          hasUnsaved ? this.confirmLeave() : of(true)
        )
      );
    }

    return hasChanges ? this.confirmLeave() : true;
  }

  private confirmLeave(): Observable<boolean> {
    const dialogRef = this.dialog.open(ConfirmLeaveDialogComponent, {
      data: {
        title: 'Unsaved Changes',
        message: 'You have unsaved changes. Are you sure you want to leave?'
      }
    });

    return dialogRef.afterClosed().pipe(
      map(result => result === true)
    );
  }
}

// Component implementing the interface
@Component({
  selector: 'app-form-editor',
  template: `...`
})
export class FormEditorComponent implements HasUnsavedChanges {
  @ViewChild('form') form: NgForm;

  hasUnsavedChanges(): Observable<boolean> {
    return this.form.statusChanges.pipe(
      map(() => this.form.dirty),
      startWith(false)
    );
  }
}
```

## Dynamic Routing & Route Building

### Dynamic Route Configuration

```typescript
// dynamic-routing.service.ts
@Injectable({
  providedIn: 'root'
})
export class DynamicRoutingService {
  constructor(
    private router: Router,
    private configService: ConfigService
  ) {}

  loadDynamicRoutes(): Observable<void> {
    return this.configService.getRouteConfig().pipe(
      map(routeConfig => {
        const dynamicRoutes = this.buildRoutesFromConfig(routeConfig);
        this.router.config.push(...dynamicRoutes);
        return;
      })
    );
  }

  private buildRoutesFromConfig(config: RouteConfig[]): Route[] {
    return config.map(routeConf => ({
      path: routeConf.path,
      loadChildren: () => import(routeConf.module).then(m => m[routeConf.moduleName]),
      data: routeConf.data,
      canActivate: this.resolveGuards(routeConf.guards)
    }));
  }

  private resolveGuards(guardNames: string[]): any[] {
    // Resolve guard classes from names
    return guardNames.map(name => this.getGuardByName(name));
  }

  navigateWithParams(route: string, params: any): Observable<boolean> {
    return from(this.router.navigate([route], { queryParams: params }));
  }

  buildUrl(segments: string[], queryParams?: any): string {
    const tree = this.router.createUrlTree(segments, { queryParams });
    return this.router.serializeUrl(tree);
  }
}
```

### Route-Based Data Loading

```typescript
// route-data.service.ts
@Injectable({
  providedIn: 'root'
})
export class RouteDataService {
  constructor(
    private router: Router,
    private dataService: DataService
  ) {}

  // Load data based on current route
  getCurrentRouteData(): Observable<any> {
    return this.router.events.pipe(
      filter(event => event instanceof NavigationEnd),
      map(() => this.router.routerState.root),
      map(route => {
        while (route.firstChild) {
          route = route.firstChild;
        }
        return route;
      }),
      switchMap(route => 
        combineLatest([
          route.params,
          route.queryParams,
          route.data
        ])
      ),
      switchMap(([params, queryParams, data]) => 
        this.dataService.loadData({ params, queryParams, data })
      )
    );
  }

  // Preload data for specific route
  preloadRouteData(route: string, params: any): Observable<any> {
    return this.dataService.loadData({ route, params });
  }

  // Cache route data
  private routeDataCache = new Map<string, Observable<any>>();

  getCachedRouteData(routeKey: string): Observable<any> {
    if (!this.routeDataCache.has(routeKey)) {
      const data$ = this.dataService.loadData({ route: routeKey }).pipe(
        shareReplay(1)
      );
      this.routeDataCache.set(routeKey, data$);
    }
    return this.routeDataCache.get(routeKey)!;
  }
}
```

## Route State Management

### Router State Service

```typescript
// router-state.service.ts
export interface RouterState {
  url: string;
  params: { [key: string]: any };
  queryParams: { [key: string]: any };
  data: { [key: string]: any };
  navigationId: number;
}

@Injectable({
  providedIn: 'root'
})
export class RouterStateService {
  private routerState$ = new BehaviorSubject<RouterState>({
    url: '',
    params: {},
    queryParams: {},
    data: {},
    navigationId: 0
  });

  readonly state$ = this.routerState$.asObservable();

  readonly url$ = this.state$.pipe(
    map(state => state.url),
    distinctUntilChanged()
  );

  readonly params$ = this.state$.pipe(
    map(state => state.params),
    distinctUntilChanged((a, b) => JSON.stringify(a) === JSON.stringify(b))
  );

  readonly queryParams$ = this.state$.pipe(
    map(state => state.queryParams),
    distinctUntilChanged((a, b) => JSON.stringify(a) === JSON.stringify(b))
  );

  constructor(private router: Router) {
    this.trackRouterState();
  }

  private trackRouterState(): void {
    this.router.events.pipe(
      filter(event => event instanceof NavigationEnd)
    ).subscribe(() => {
      const route = this.getLeafRoute(this.router.routerState.root);
      
      const state: RouterState = {
        url: this.router.url,
        params: route.snapshot.params,
        queryParams: route.snapshot.queryParams,
        data: route.snapshot.data,
        navigationId: (this.router as any).navigationId
      };

      this.routerState$.next(state);
    });
  }

  private getLeafRoute(route: ActivatedRoute): ActivatedRoute {
    while (route.firstChild) {
      route = route.firstChild;
    }
    return route;
  }

  // Helper methods
  getParam(key: string): Observable<string | null> {
    return this.params$.pipe(
      map(params => params[key] || null)
    );
  }

  getQueryParam(key: string): Observable<string | null> {
    return this.queryParams$.pipe(
      map(params => params[key] || null)
    );
  }

  hasParam(key: string): Observable<boolean> {
    return this.params$.pipe(
      map(params => key in params)
    );
  }
}
```

## Testing Router Observables

### Testing Route Parameters

```typescript
// product-detail.component.spec.ts
describe('ProductDetailComponent', () => {
  let component: ProductDetailComponent;
  let fixture: ComponentFixture<ProductDetailComponent>;
  let mockActivatedRoute: jasmine.SpyObj<ActivatedRoute>;

  beforeEach(() => {
    const routeSpy = jasmine.createSpyObj('ActivatedRoute', [], {
      paramMap: new BehaviorSubject(convertToParamMap({ id: '123' })),
      queryParamMap: new BehaviorSubject(convertToParamMap({}))
    });

    TestBed.configureTestingModule({
      declarations: [ProductDetailComponent],
      providers: [
        { provide: ActivatedRoute, useValue: routeSpy }
      ]
    });

    fixture = TestBed.createComponent(ProductDetailComponent);
    component = fixture.componentInstance;
    mockActivatedRoute = TestBed.inject(ActivatedRoute) as jasmine.SpyObj<ActivatedRoute>;
  });

  it('should load product when route parameter changes', fakeAsync(() => {
    const productService = TestBed.inject(ProductService);
    spyOn(productService, 'getProduct').and.returnValue(of(mockProduct));

    // Simulate route parameter change
    (mockActivatedRoute.paramMap as BehaviorSubject<ParamMap>)
      .next(convertToParamMap({ id: '456' }));

    tick();

    expect(productService.getProduct).toHaveBeenCalledWith('456');
  }));

  it('should handle query parameter changes', fakeAsync(() => {
    // Simulate query parameter change
    (mockActivatedRoute.queryParamMap as BehaviorSubject<ParamMap>)
      .next(convertToParamMap({ variant: 'red' }));

    tick();

    component.selectedVariant$.subscribe(variant => {
      expect(variant?.id).toBe('red');
    });
  }));
});
```

### Testing Navigation

```typescript
// navigation.service.spec.ts
describe('NavigationService', () => {
  let service: NavigationService;
  let router: Router;
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });

    TestBed.configureTestingModule({
      imports: [RouterTestingModule],
      providers: [NavigationService]
    });

    service = TestBed.inject(NavigationService);
    router = TestBed.inject(Router);
  });

  it('should track navigation loading state', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const navigationEvents = hot('a-b-c', {
        a: new NavigationStart(1, '/test'),
        b: new NavigationEnd(1, '/test', '/test'),
        c: new NavigationStart(2, '/other')
      });

      // Mock router events
      spyOnProperty(router, 'events').and.returnValue(navigationEvents);

      const expected = 'a-b-c';
      const values = {
        a: true,  // Loading starts
        b: false, // Loading ends
        c: true   // Loading starts again
      };

      expectObservable(service.loading$).toBe(expected, values);
    });
  });
});
```

## Best Practices

### 1. Parameter Handling
- Use `paramMap` and `queryParamMap` for reactive parameters
- Combine multiple parameters with `combineLatest`
- Filter out null/undefined parameters
- Use `distinctUntilChanged` to avoid unnecessary updates

### 2. Data Loading
- Use `switchMap` for parameter-based data loading
- Implement proper error handling
- Consider caching with `shareReplay`
- Preload data when possible

### 3. Navigation Patterns
- Use relative navigation when appropriate
- Merge query parameters carefully
- Implement proper loading states
- Handle navigation errors gracefully

### 4. Guard Implementation
- Return Observables for async checks
- Provide user feedback for failed guards
- Handle errors in guard logic
- Use proper TypeScript interfaces

### 5. Testing Strategy
- Mock ActivatedRoute properly
- Test parameter changes over time
- Use marble testing for complex flows
- Test navigation side effects

## Common Pitfalls

### 1. Memory Leaks
```typescript
// ❌ Bad: Not unsubscribing
ngOnInit() {
  this.route.params.subscribe(params => {
    // Handle params
  }); // Memory leak!
}

// ✅ Good: Proper cleanup
ngOnInit() {
  this.route.params
    .pipe(takeUntil(this.destroy$))
    .subscribe(params => {
      // Handle params
    });
}
```

### 2. Excessive API Calls
```typescript
// ❌ Bad: No debouncing
this.route.queryParams.subscribe(params => {
  this.searchService.search(params.query); // Called on every keystroke
});

// ✅ Good: Proper debouncing
this.route.queryParams.pipe(
  debounceTime(300),
  distinctUntilChanged()
).subscribe(params => {
  this.searchService.search(params.query);
});
```

## Summary

Angular Router's Observable APIs provide powerful patterns for reactive navigation and route state management. Key takeaways:

- Use paramMap and queryParamMap for reactive parameter handling
- Combine parameters with combineLatest for complex scenarios
- Implement proper data loading patterns with switchMap
- Use router events for navigation state tracking
- Create reusable guards with Observable patterns
- Implement proper error handling and loading states
- Test router interactions thoroughly
- Follow cleanup patterns to prevent memory leaks

These patterns enable building sophisticated, reactive routing solutions that respond dynamically to navigation changes and provide excellent user experiences.
