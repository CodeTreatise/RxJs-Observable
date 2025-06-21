# Pagination & Infinite Scroll Patterns

## Learning Objectives
- Implement efficient pagination with RxJS Observables
- Build infinite scroll mechanisms using reactive patterns
- Handle state management for paginated data
- Optimize performance with virtual scrolling
- Manage loading states and error handling
- Implement search and filtering with pagination

## Prerequisites
- Understanding of RxJS operators
- Angular HTTP client knowledge
- Component state management
- Change detection concepts

---

## Table of Contents
1. [Pagination Fundamentals](#pagination-fundamentals)
2. [Basic Pagination Implementation](#basic-pagination-implementation)
3. [Infinite Scroll Patterns](#infinite-scroll-patterns)
4. [Virtual Scrolling with CDK](#virtual-scrolling-with-cdk)
5. [State Management](#state-management)
6. [Search & Filtering](#search--filtering)
7. [Performance Optimization](#performance-optimization)
8. [Error Handling](#error-handling)
9. [Testing Strategies](#testing-strategies)
10. [Real-World Examples](#real-world-examples)

---

## Pagination Fundamentals

### Core Concepts

Pagination involves breaking large datasets into manageable chunks:

```typescript
interface PaginationState {
  page: number;
  pageSize: number;
  total: number;
  hasNext: boolean;
  hasPrevious: boolean;
}

interface PaginatedResponse<T> {
  items: T[];
  pagination: PaginationState;
}

interface PaginationRequest {
  page: number;
  pageSize: number;
  sortBy?: string;
  sortOrder?: 'asc' | 'desc';
  filters?: Record<string, any>;
}
```

### Marble Diagram - Pagination Flow

```
Page requests:     1----2----3----4--->
                   |    |    |    |
API responses:     A----B----C----D--->
                   |    |    |    |
Combined state:    S1---S2---S3---S4-->

Where:
- Numbers: Page requests
- Letters: API responses with data
- S: Combined pagination state
```

---

## Basic Pagination Implementation

### Service Layer

```typescript
@Injectable({
  providedIn: 'root'
})
export class PaginationService<T> {
  private readonly baseUrl = 'api';
  
  constructor(private http: HttpClient) {}

  /**
   * Get paginated data
   */
  getPaginatedData(
    endpoint: string,
    request: PaginationRequest
  ): Observable<PaginatedResponse<T>> {
    const params = new HttpParams()
      .set('page', request.page.toString())
      .set('pageSize', request.pageSize.toString());

    return this.http.get<PaginatedResponse<T>>(
      `${this.baseUrl}/${endpoint}`,
      { params }
    ).pipe(
      catchError(this.handleError)
    );
  }

  /**
   * Create pagination state observable
   */
  createPaginationState(
    endpoint: string,
    initialRequest: PaginationRequest
  ): Observable<PaginatedResponse<T>> {
    const pageRequest$ = new BehaviorSubject(initialRequest);
    
    return pageRequest$.pipe(
      debounceTime(300),
      distinctUntilChanged((a, b) => 
        JSON.stringify(a) === JSON.stringify(b)
      ),
      switchMap(request => 
        this.getPaginatedData(endpoint, request)
      ),
      shareReplay(1)
    );
  }

  private handleError(error: HttpErrorResponse): Observable<never> {
    console.error('Pagination error:', error);
    return throwError(() => error);
  }
}
```

### Component Implementation

```typescript
@Component({
  selector: 'app-paginated-list',
  template: `
    <div class="pagination-container">
      <!-- Loading State -->
      <div *ngIf="loading$ | async" class="loading">
        Loading...
      </div>

      <!-- Error State -->
      <div *ngIf="error$ | async as error" class="error">
        Error: {{ error.message }}
        <button (click)="retry()">Retry</button>
      </div>

      <!-- Data List -->
      <div *ngIf="data$ | async as response" class="data-container">
        <div class="items">
          <div *ngFor="let item of response.items" class="item">
            {{ item.name }}
          </div>
        </div>

        <!-- Pagination Controls -->
        <div class="pagination-controls">
          <button 
            [disabled]="!response.pagination.hasPrevious"
            (click)="previousPage()">
            Previous
          </button>
          
          <span class="page-info">
            Page {{ response.pagination.page }} of 
            {{ getTotalPages(response.pagination) }}
          </span>
          
          <button 
            [disabled]="!response.pagination.hasNext"
            (click)="nextPage()">
            Next
          </button>
        </div>
      </div>
    </div>
  `
})
export class PaginatedListComponent implements OnInit, OnDestroy {
  private readonly pageRequest$ = new BehaviorSubject<PaginationRequest>({
    page: 1,
    pageSize: 10
  });

  private readonly destroy$ = new Subject<void>();

  // Public observables
  data$ = this.pageRequest$.pipe(
    switchMap(request => 
      this.paginationService.getPaginatedData('users', request)
    ),
    shareReplay(1)
  );

  loading$ = this.pageRequest$.pipe(
    switchMap(() => 
      concat(
        of(true),
        this.data$.pipe(map(() => false))
      )
    )
  );

  error$ = this.data$.pipe(
    map(() => null),
    catchError(error => of(error))
  );

  constructor(
    private paginationService: PaginationService<User>
  ) {}

  ngOnInit(): void {
    // Auto-retry on error
    this.error$.pipe(
      filter(error => !!error),
      delay(5000),
      takeUntil(this.destroy$)
    ).subscribe(() => this.retry());
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }

  nextPage(): void {
    const current = this.pageRequest$.value;
    this.pageRequest$.next({
      ...current,
      page: current.page + 1
    });
  }

  previousPage(): void {
    const current = this.pageRequest$.value;
    this.pageRequest$.next({
      ...current,
      page: Math.max(1, current.page - 1)
    });
  }

  goToPage(page: number): void {
    const current = this.pageRequest$.value;
    this.pageRequest$.next({
      ...current,
      page
    });
  }

  changePageSize(pageSize: number): void {
    this.pageRequest$.next({
      ...this.pageRequest$.value,
      page: 1,
      pageSize
    });
  }

  retry(): void {
    const current = this.pageRequest$.value;
    this.pageRequest$.next({ ...current });
  }

  getTotalPages(pagination: PaginationState): number {
    return Math.ceil(pagination.total / pagination.pageSize);
  }
}
```

---

## Infinite Scroll Patterns

### Infinite Scroll Service

```typescript
@Injectable({
  providedIn: 'root'
})
export class InfiniteScrollService<T> {
  private readonly scrollThreshold = 200; // pixels from bottom

  constructor(private http: HttpClient) {}

  /**
   * Create infinite scroll observable
   */
  createInfiniteScroll(
    endpoint: string,
    pageSize: number = 20
  ): Observable<T[]> {
    let currentPage = 0;
    let hasMore = true;
    const allItems: T[] = [];

    const loadMore$ = new Subject<void>();
    
    return loadMore$.pipe(
      startWith(null), // Trigger initial load
      scan((acc, _) => ({ page: ++currentPage, items: acc.items }), 
           { page: 0, items: [] as T[] }),
      filter(() => hasMore),
      switchMap(({ page }) => 
        this.loadPage(endpoint, page, pageSize)
      ),
      tap(response => {
        hasMore = response.items.length === pageSize;
        allItems.push(...response.items);
      }),
      map(() => [...allItems]),
      shareReplay(1)
    );
  }

  /**
   * Create scroll trigger observable
   */
  createScrollTrigger(element: ElementRef): Observable<void> {
    return fromEvent(element.nativeElement, 'scroll').pipe(
      map(() => element.nativeElement),
      filter(el => {
        const threshold = this.scrollThreshold;
        return el.scrollTop + el.clientHeight >= 
               el.scrollHeight - threshold;
      }),
      debounceTime(100),
      map(() => void 0)
    );
  }

  private loadPage(
    endpoint: string, 
    page: number, 
    pageSize: number
  ): Observable<PaginatedResponse<T>> {
    const params = new HttpParams()
      .set('page', page.toString())
      .set('pageSize', pageSize.toString());

    return this.http.get<PaginatedResponse<T>>(endpoint, { params });
  }
}
```

### Infinite Scroll Component

```typescript
@Component({
  selector: 'app-infinite-scroll',
  template: `
    <div class="infinite-scroll-container" 
         #scrollContainer
         (scroll)="onScroll()">
      
      <div *ngFor="let item of items$ | async; trackBy: trackByFn" 
           class="item">
        {{ item.name }}
      </div>

      <div *ngIf="loading$ | async" class="loading">
        Loading more...
      </div>

      <div *ngIf="error$ | async as error" class="error">
        Error loading more items
        <button (click)="retryLoad()">Retry</button>
      </div>
    </div>
  `,
  styles: [`
    .infinite-scroll-container {
      height: 400px;
      overflow-y: auto;
    }
    .item {
      padding: 10px;
      border-bottom: 1px solid #eee;
    }
    .loading, .error {
      text-align: center;
      padding: 20px;
    }
  `]
})
export class InfiniteScrollComponent implements OnInit, OnDestroy {
  @ViewChild('scrollContainer') scrollContainer!: ElementRef;

  private readonly loadMore$ = new Subject<void>();
  private readonly destroy$ = new Subject<void>();

  // State observables
  items$: Observable<Item[]>;
  loading$: Observable<boolean>;
  error$: Observable<any>;

  constructor(
    private infiniteScrollService: InfiniteScrollService<Item>
  ) {
    this.setupStreams();
  }

  ngOnInit(): void {
    // Trigger initial load
    this.loadMore$.next();
  }

  ngAfterViewInit(): void {
    // Setup scroll trigger
    this.infiniteScrollService
      .createScrollTrigger(this.scrollContainer)
      .pipe(takeUntil(this.destroy$))
      .subscribe(() => this.loadMore$.next());
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }

  onScroll(): void {
    // Additional scroll handling if needed
  }

  retryLoad(): void {
    this.loadMore$.next();
  }

  trackByFn(index: number, item: Item): any {
    return item.id;
  }

  private setupStreams(): void {
    // Create data stream
    this.items$ = this.loadMore$.pipe(
      scan((acc, _) => ({ ...acc, page: acc.page + 1 }), 
           { page: 0, items: [] as Item[] }),
      switchMap(({ page }) => 
        this.loadPage(page).pipe(
          map(response => response.items),
          scan((allItems, newItems) => [...allItems, ...newItems], [] as Item[])
        )
      ),
      shareReplay(1)
    );

    // Loading state
    this.loading$ = this.loadMore$.pipe(
      switchMap(() => 
        concat(
          of(true),
          this.items$.pipe(
            take(1),
            map(() => false)
          )
        )
      ),
      startWith(false)
    );

    // Error state
    this.error$ = this.items$.pipe(
      map(() => null),
      catchError(error => of(error))
    );
  }

  private loadPage(page: number): Observable<PaginatedResponse<Item>> {
    return this.infiniteScrollService.loadPage('items', page, 20);
  }
}
```

---

## Virtual Scrolling with CDK

### Virtual Scroll Implementation

```typescript
@Component({
  selector: 'app-virtual-scroll',
  template: `
    <div class="virtual-scroll-container">
      <cdk-virtual-scroll-viewport 
        itemSize="50" 
        class="viewport"
        (scrolledIndexChange)="onScrolledIndexChange($event)">
        
        <div *cdkVirtualFor="let item of items$ | async; 
                             trackBy: trackByFn; 
                             templateCacheSize: 0" 
             class="item">
          <app-item-component [item]="item"></app-item-component>
        </div>

        <!-- Loading indicator at the bottom -->
        <div *ngIf="loading$ | async" class="loading-indicator">
          Loading more items...
        </div>
      </cdk-virtual-scroll-viewport>
    </div>
  `,
  styles: [`
    .virtual-scroll-container {
      height: 500px;
    }
    .viewport {
      height: 100%;
    }
    .item {
      height: 50px;
      display: flex;
      align-items: center;
      padding: 0 16px;
      border-bottom: 1px solid #e0e0e0;
    }
    .loading-indicator {
      padding: 16px;
      text-align: center;
    }
  `]
})
export class VirtualScrollComponent implements OnInit {
  private readonly pageSize = 50;
  private readonly loadThreshold = 10; // Load when 10 items from end
  
  private readonly loadMore$ = new BehaviorSubject<number>(1);
  
  items$: Observable<Item[]>;
  loading$: Observable<boolean>;

  constructor(
    private dataService: DataService
  ) {
    this.setupVirtualScrollStream();
  }

  ngOnInit(): void {
    // Initial load
    this.loadMore$.next(1);
  }

  onScrolledIndexChange(index: number): void {
    // Trigger load more when near the end
    const currentItems = this.items$.pipe(take(1)).subscribe(items => {
      if (index >= items.length - this.loadThreshold) {
        const nextPage = Math.floor(items.length / this.pageSize) + 1;
        this.loadMore$.next(nextPage);
      }
    });
  }

  trackByFn(index: number, item: Item): any {
    return item.id;
  }

  private setupVirtualScrollStream(): void {
    this.items$ = this.loadMore$.pipe(
      distinctUntilChanged(),
      switchMap(page => 
        this.dataService.getPage(page, this.pageSize)
      ),
      scan((allItems, newItems) => {
        // Avoid duplicates
        const existingIds = new Set(allItems.map(item => item.id));
        const uniqueNewItems = newItems.filter(item => 
          !existingIds.has(item.id)
        );
        return [...allItems, ...uniqueNewItems];
      }, [] as Item[]),
      shareReplay(1)
    );

    this.loading$ = this.loadMore$.pipe(
      switchMap(() => 
        concat(
          of(true),
          this.items$.pipe(
            take(1),
            map(() => false)
          )
        )
      )
    );
  }
}
```

---

## State Management

### Pagination State Manager

```typescript
interface PaginationState<T> {
  items: T[];
  currentPage: number;
  pageSize: number;
  totalItems: number;
  totalPages: number;
  loading: boolean;
  error: string | null;
  filters: Record<string, any>;
  sortBy: string | null;
  sortOrder: 'asc' | 'desc';
}

@Injectable()
export class PaginationStateService<T> {
  private readonly state$ = new BehaviorSubject<PaginationState<T>>(
    this.getInitialState()
  );

  // Selectors
  readonly items$ = this.state$.pipe(
    map(state => state.items),
    distinctUntilChanged()
  );

  readonly pagination$ = this.state$.pipe(
    map(state => ({
      currentPage: state.currentPage,
      pageSize: state.pageSize,
      totalItems: state.totalItems,
      totalPages: state.totalPages
    })),
    distinctUntilChanged((a, b) => JSON.stringify(a) === JSON.stringify(b))
  );

  readonly loading$ = this.state$.pipe(
    map(state => state.loading),
    distinctUntilChanged()
  );

  readonly error$ = this.state$.pipe(
    map(state => state.error),
    distinctUntilChanged()
  );

  // Actions
  setLoading(loading: boolean): void {
    this.updateState({ loading });
  }

  setError(error: string | null): void {
    this.updateState({ error, loading: false });
  }

  setItems(items: T[], totalItems: number): void {
    const totalPages = Math.ceil(totalItems / this.state$.value.pageSize);
    this.updateState({ 
      items, 
      totalItems, 
      totalPages, 
      loading: false, 
      error: null 
    });
  }

  appendItems(newItems: T[]): void {
    const currentState = this.state$.value;
    this.updateState({
      items: [...currentState.items, ...newItems],
      loading: false
    });
  }

  setPage(page: number): void {
    this.updateState({ currentPage: page });
  }

  setPageSize(pageSize: number): void {
    this.updateState({ 
      pageSize, 
      currentPage: 1,
      items: [] // Reset items when page size changes
    });
  }

  setFilters(filters: Record<string, any>): void {
    this.updateState({ 
      filters, 
      currentPage: 1,
      items: [] // Reset items when filters change
    });
  }

  setSort(sortBy: string, sortOrder: 'asc' | 'desc'): void {
    this.updateState({ 
      sortBy, 
      sortOrder,
      currentPage: 1,
      items: [] // Reset items when sort changes
    });
  }

  reset(): void {
    this.state$.next(this.getInitialState());
  }

  private updateState(partial: Partial<PaginationState<T>>): void {
    this.state$.next({
      ...this.state$.value,
      ...partial
    });
  }

  private getInitialState(): PaginationState<T> {
    return {
      items: [],
      currentPage: 1,
      pageSize: 10,
      totalItems: 0,
      totalPages: 0,
      loading: false,
      error: null,
      filters: {},
      sortBy: null,
      sortOrder: 'asc'
    };
  }
}
```

---

## Search & Filtering

### Search-Enabled Pagination

```typescript
@Component({
  selector: 'app-searchable-pagination',
  template: `
    <div class="search-pagination-container">
      <!-- Search Controls -->
      <div class="search-controls">
        <input 
          type="text" 
          placeholder="Search..."
          [formControl]="searchControl"
          class="search-input">
        
        <select [formControl]="categoryControl" class="filter-select">
          <option value="">All Categories</option>
          <option *ngFor="let category of categories" [value]="category">
            {{ category }}
          </option>
        </select>

        <select [formControl]="sortControl" class="sort-select">
          <option value="name_asc">Name (A-Z)</option>
          <option value="name_desc">Name (Z-A)</option>
          <option value="date_asc">Date (Oldest)</option>
          <option value="date_desc">Date (Newest)</option>
        </select>
      </div>

      <!-- Results -->
      <div class="results-container">
        <div *ngIf="loading$ | async" class="loading">Searching...</div>
        
        <div *ngIf="items$ | async as items" class="items">
          <div *ngFor="let item of items" class="item">
            {{ item.name }} - {{ item.category }}
          </div>
        </div>

        <!-- Pagination -->
        <div class="pagination" *ngIf="pagination$ | async as pagination">
          <button 
            [disabled]="pagination.currentPage === 1"
            (click)="previousPage()">
            Previous
          </button>
          
          <span>
            Page {{ pagination.currentPage }} of {{ pagination.totalPages }}
            ({{ pagination.totalItems }} total items)
          </span>
          
          <button 
            [disabled]="pagination.currentPage === pagination.totalPages"
            (click)="nextPage()">
            Next
          </button>
        </div>
      </div>
    </div>
  `
})
export class SearchablePaginationComponent implements OnInit, OnDestroy {
  searchControl = new FormControl('');
  categoryControl = new FormControl('');
  sortControl = new FormControl('name_asc');

  categories = ['Electronics', 'Books', 'Clothing', 'Home'];

  private readonly destroy$ = new Subject<void>();
  private readonly paginationState = new PaginationStateService<Item>();

  // Combined search parameters
  private readonly searchParams$ = combineLatest([
    this.searchControl.valueChanges.pipe(startWith('')),
    this.categoryControl.valueChanges.pipe(startWith('')),
    this.sortControl.valueChanges.pipe(startWith('name_asc')),
    this.paginationState.pagination$
  ]).pipe(
    debounceTime(300),
    distinctUntilChanged((a, b) => JSON.stringify(a) === JSON.stringify(b))
  );

  // Public observables
  items$ = this.paginationState.items$;
  loading$ = this.paginationState.loading$;
  pagination$ = this.paginationState.pagination$;
  error$ = this.paginationState.error$;

  constructor(
    private searchService: SearchService
  ) {}

  ngOnInit(): void {
    this.setupSearchSubscription();
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }

  nextPage(): void {
    const currentPage = this.paginationState.state$.value.currentPage;
    this.paginationState.setPage(currentPage + 1);
  }

  previousPage(): void {
    const currentPage = this.paginationState.state$.value.currentPage;
    this.paginationState.setPage(Math.max(1, currentPage - 1));
  }

  private setupSearchSubscription(): void {
    this.searchParams$.pipe(
      tap(() => this.paginationState.setLoading(true)),
      switchMap(([search, category, sort, pagination]) => {
        const [sortBy, sortOrder] = sort.split('_') as [string, 'asc' | 'desc'];
        
        const searchRequest = {
          query: search || '',
          category: category || '',
          sortBy,
          sortOrder,
          page: pagination.currentPage,
          pageSize: pagination.pageSize
        };

        return this.searchService.search(searchRequest).pipe(
          catchError(error => {
            this.paginationState.setError(error.message);
            return EMPTY;
          })
        );
      }),
      takeUntil(this.destroy$)
    ).subscribe(response => {
      this.paginationState.setItems(response.items, response.totalCount);
    });

    // Reset to first page when search parameters change
    merge(
      this.searchControl.valueChanges,
      this.categoryControl.valueChanges,
      this.sortControl.valueChanges
    ).pipe(
      debounceTime(100),
      takeUntil(this.destroy$)
    ).subscribe(() => {
      this.paginationState.setPage(1);
    });
  }
}
```

---

## Performance Optimization

### Optimized Pagination Strategy

```typescript
@Injectable()
export class OptimizedPaginationService<T> {
  private readonly cache = new Map<string, Observable<PaginatedResponse<T>>>();
  private readonly cacheTimeout = 5 * 60 * 1000; // 5 minutes

  constructor(private http: HttpClient) {}

  /**
   * Get paginated data with caching and optimization
   */
  getOptimizedPaginatedData(
    endpoint: string,
    request: PaginationRequest
  ): Observable<PaginatedResponse<T>> {
    const cacheKey = this.getCacheKey(endpoint, request);
    
    if (this.cache.has(cacheKey)) {
      return this.cache.get(cacheKey)!;
    }

    const request$ = this.http.get<PaginatedResponse<T>>(
      endpoint,
      { params: this.buildParams(request) }
    ).pipe(
      retry({
        count: 3,
        delay: (error, retryIndex) => timer(retryIndex * 1000)
      }),
      shareReplay({
        bufferSize: 1,
        refCount: true
      }),
      finalize(() => {
        // Remove from cache after timeout
        timer(this.cacheTimeout).subscribe(() => {
          this.cache.delete(cacheKey);
        });
      })
    );

    this.cache.set(cacheKey, request$);
    return request$;
  }

  /**
   * Preload adjacent pages
   */
  preloadAdjacentPages(
    endpoint: string,
    currentRequest: PaginationRequest
  ): void {
    const preloadRequests = [
      { ...currentRequest, page: currentRequest.page + 1 },
      { ...currentRequest, page: Math.max(1, currentRequest.page - 1) }
    ];

    preloadRequests.forEach(request => {
      this.getOptimizedPaginatedData(endpoint, request)
        .pipe(take(1))
        .subscribe(); // Just trigger the request
    });
  }

  /**
   * Batch multiple page requests
   */
  getBatchedPages(
    endpoint: string,
    requests: PaginationRequest[]
  ): Observable<PaginatedResponse<T>[]> {
    const requests$ = requests.map(request =>
      this.getOptimizedPaginatedData(endpoint, request)
    );

    return forkJoin(requests$);
  }

  /**
   * Clear cache
   */
  clearCache(): void {
    this.cache.clear();
  }

  private getCacheKey(endpoint: string, request: PaginationRequest): string {
    return `${endpoint}_${JSON.stringify(request)}`;
  }

  private buildParams(request: PaginationRequest): HttpParams {
    let params = new HttpParams()
      .set('page', request.page.toString())
      .set('pageSize', request.pageSize.toString());

    if (request.sortBy) {
      params = params.set('sortBy', request.sortBy);
    }
    
    if (request.sortOrder) {
      params = params.set('sortOrder', request.sortOrder);
    }

    if (request.filters) {
      Object.entries(request.filters).forEach(([key, value]) => {
        if (value !== null && value !== undefined && value !== '') {
          params = params.set(key, value.toString());
        }
      });
    }

    return params;
  }
}
```

### Performance Monitoring

```typescript
@Injectable()
export class PaginationPerformanceMonitor {
  private readonly performanceMetrics$ = new Subject<PerformanceMetric>();

  constructor() {}

  /**
   * Monitor pagination performance
   */
  monitorPagination<T>(
    source$: Observable<PaginatedResponse<T>>,
    operation: string
  ): Observable<PaginatedResponse<T>> {
    const startTime = performance.now();
    
    return source$.pipe(
      tap(response => {
        const endTime = performance.now();
        const metric: PerformanceMetric = {
          operation,
          duration: endTime - startTime,
          itemCount: response.items.length,
          timestamp: new Date()
        };
        
        this.performanceMetrics$.next(metric);
      }),
      catchError(error => {
        const endTime = performance.now();
        const metric: PerformanceMetric = {
          operation,
          duration: endTime - startTime,
          itemCount: 0,
          error: error.message,
          timestamp: new Date()
        };
        
        this.performanceMetrics$.next(metric);
        return throwError(() => error);
      })
    );
  }

  /**
   * Get performance metrics stream
   */
  getMetrics(): Observable<PerformanceMetric> {
    return this.performanceMetrics$.asObservable();
  }

  /**
   * Get performance summary
   */
  getPerformanceSummary(): Observable<PerformanceSummary> {
    return this.performanceMetrics$.pipe(
      bufferTime(10000), // Buffer for 10 seconds
      filter(metrics => metrics.length > 0),
      map(metrics => {
        const successfulMetrics = metrics.filter(m => !m.error);
        const failedMetrics = metrics.filter(m => m.error);
        
        return {
          totalRequests: metrics.length,
          successfulRequests: successfulMetrics.length,
          failedRequests: failedMetrics.length,
          averageDuration: successfulMetrics.length > 0 
            ? successfulMetrics.reduce((sum, m) => sum + m.duration, 0) / successfulMetrics.length
            : 0,
          totalItemsLoaded: successfulMetrics.reduce((sum, m) => sum + m.itemCount, 0)
        };
      })
    );
  }
}

interface PerformanceMetric {
  operation: string;
  duration: number;
  itemCount: number;
  error?: string;
  timestamp: Date;
}

interface PerformanceSummary {
  totalRequests: number;
  successfulRequests: number;
  failedRequests: number;
  averageDuration: number;
  totalItemsLoaded: number;
}
```

---

## Error Handling

### Robust Error Handling

```typescript
@Injectable()
export class PaginationErrorHandler {
  /**
   * Handle pagination errors with retry strategies
   */
  handlePaginationError<T>(
    source$: Observable<PaginatedResponse<T>>,
    config: ErrorHandlingConfig = {}
  ): Observable<PaginatedResponse<T>> {
    const {
      maxRetries = 3,
      retryDelay = 1000,
      fallbackData = null,
      showUserMessage = true
    } = config;

    return source$.pipe(
      retryWhen(errors =>
        errors.pipe(
          scan((retryCount, error) => {
            if (retryCount >= maxRetries) {
              throw error;
            }
            
            console.warn(`Pagination attempt ${retryCount + 1} failed:`, error);
            return retryCount + 1;
          }, 0),
          delayWhen(retryCount => timer(retryDelay * retryCount))
        )
      ),
      catchError(error => {
        if (showUserMessage) {
          this.showErrorMessage(error);
        }
        
        if (fallbackData) {
          return of(fallbackData);
        }
        
        return throwError(() => this.enhanceError(error));
      })
    );
  }

  /**
   * Handle network-specific errors
   */
  handleNetworkErrors<T>(
    source$: Observable<T>
  ): Observable<T> {
    return source$.pipe(
      catchError(error => {
        if (error.status === 0) {
          // Network error
          return throwError(() => new Error('Network connection lost. Please check your internet connection.'));
        } else if (error.status >= 500) {
          // Server error
          return throwError(() => new Error('Server error. Please try again later.'));
        } else if (error.status === 429) {
          // Rate limiting
          return throwError(() => new Error('Too many requests. Please wait a moment and try again.'));
        }
        
        return throwError(() => error);
      })
    );
  }

  private showErrorMessage(error: any): void {
    // Show user-friendly error message
    console.error('Pagination error:', error);
    // You could integrate with a notification service here
  }

  private enhanceError(error: any): Error {
    const enhancedError = new Error(
      `Pagination failed: ${error.message || 'Unknown error'}`
    );
    (enhancedError as any).originalError = error;
    return enhancedError;
  }
}

interface ErrorHandlingConfig {
  maxRetries?: number;
  retryDelay?: number;
  fallbackData?: any;
  showUserMessage?: boolean;
}
```

---

## Testing Strategies

### Unit Tests

```typescript
describe('PaginationService', () => {
  let service: PaginationService<TestItem>;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [PaginationService]
    });
    
    service = TestBed.inject(PaginationService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
  });

  describe('getPaginatedData', () => {
    it('should fetch paginated data successfully', () => {
      const mockResponse: PaginatedResponse<TestItem> = {
        items: [
          { id: 1, name: 'Item 1' },
          { id: 2, name: 'Item 2' }
        ],
        pagination: {
          page: 1,
          pageSize: 10,
          total: 2,
          hasNext: false,
          hasPrevious: false
        }
      };

      const request: PaginationRequest = {
        page: 1,
        pageSize: 10
      };

      service.getPaginatedData('test-endpoint', request).subscribe(response => {
        expect(response).toEqual(mockResponse);
      });

      const req = httpMock.expectOne(req => 
        req.url.includes('test-endpoint') && 
        req.params.get('page') === '1' &&
        req.params.get('pageSize') === '10'
      );
      
      expect(req.request.method).toBe('GET');
      req.flush(mockResponse);
    });

    it('should handle HTTP errors', () => {
      const request: PaginationRequest = {
        page: 1,
        pageSize: 10
      };

      service.getPaginatedData('test-endpoint', request).subscribe({
        next: () => fail('Should have failed'),
        error: (error) => {
          expect(error.status).toBe(500);
        }
      });

      const req = httpMock.expectOne(req => req.url.includes('test-endpoint'));
      req.flush('Server Error', { status: 500, statusText: 'Internal Server Error' });
    });
  });

  describe('createPaginationState', () => {
    it('should create paginated state observable', fakeAsync(() => {
      const mockResponse: PaginatedResponse<TestItem> = {
        items: [{ id: 1, name: 'Item 1' }],
        pagination: {
          page: 1,
          pageSize: 10,
          total: 1,
          hasNext: false,
          hasPrevious: false
        }
      };

      const request: PaginationRequest = {
        page: 1,
        pageSize: 10
      };

      let result: PaginatedResponse<TestItem> | null = null;
      
      service.createPaginationState('test-endpoint', request)
        .subscribe(response => result = response);

      tick(300); // Wait for debounce

      const req = httpMock.expectOne(req => req.url.includes('test-endpoint'));
      req.flush(mockResponse);

      expect(result).toEqual(mockResponse);
    }));
  });
});
```

### Integration Tests

```typescript
describe('InfiniteScrollComponent', () => {
  let component: InfiniteScrollComponent;
  let fixture: ComponentFixture<InfiniteScrollComponent>;
  let mockInfiniteScrollService: jasmine.SpyObj<InfiniteScrollService<TestItem>>;

  beforeEach(async () => {
    const spy = jasmine.createSpyObj('InfiniteScrollService', 
      ['createInfiniteScroll', 'createScrollTrigger']);

    await TestBed.configureTestingModule({
      declarations: [InfiniteScrollComponent],
      providers: [
        { provide: InfiniteScrollService, useValue: spy }
      ]
    }).compileComponents();

    fixture = TestBed.createComponent(InfiniteScrollComponent);
    component = fixture.componentInstance;
    mockInfiniteScrollService = TestBed.inject(InfiniteScrollService) as jasmine.SpyObj<InfiniteScrollService<TestItem>>;
  });

  it('should load initial items on init', () => {
    const mockItems = [
      { id: 1, name: 'Item 1' },
      { id: 2, name: 'Item 2' }
    ];

    mockInfiniteScrollService.createInfiniteScroll.and.returnValue(of(mockItems));
    
    component.ngOnInit();
    
    expect(mockInfiniteScrollService.createInfiniteScroll).toHaveBeenCalled();
  });

  it('should trigger load more on scroll', fakeAsync(() => {
    const scrollContainer = fixture.debugElement.query(By.css('.infinite-scroll-container'));
    const mockScrollTrigger = new Subject<void>();
    
    mockInfiniteScrollService.createScrollTrigger.and.returnValue(mockScrollTrigger);
    
    component.ngAfterViewInit();
    
    // Simulate scroll trigger
    mockScrollTrigger.next();
    tick();
    
    expect(component.loadMore$.observers.length).toBeGreaterThan(0);
  }));
});
```

---

## Real-World Examples

### E-commerce Product Listing

```typescript
@Component({
  selector: 'app-product-listing',
  template: `
    <div class="product-listing">
      <!-- Filters -->
      <div class="filters">
        <input 
          type="text" 
          placeholder="Search products..."
          [formControl]="searchControl">
        
        <select [formControl]="categoryControl">
          <option value="">All Categories</option>
          <option *ngFor="let category of categories" [value]="category.id">
            {{ category.name }}
          </option>
        </select>

        <div class="price-range">
          <input 
            type="range" 
            [formControl]="minPriceControl"
            min="0" 
            max="1000">
          <input 
            type="range" 
            [formControl]="maxPriceControl"
            min="0" 
            max="1000">
        </div>
      </div>

      <!-- Product Grid -->
      <div class="product-grid">
        <div *ngFor="let product of products$ | async; trackBy: trackByProduct" 
             class="product-card">
          <img [src]="product.imageUrl" [alt]="product.name">
          <h3>{{ product.name }}</h3>
          <p class="price">{{ product.price | currency }}</p>
          <button (click)="addToCart(product)">Add to Cart</button>
        </div>
      </div>

      <!-- Load More Button -->
      <div class="load-more" *ngIf="hasMore$ | async">
        <button 
          (click)="loadMore()"
          [disabled]="loading$ | async">
          <span *ngIf="loading$ | async">Loading...</span>
          <span *ngIf="!(loading$ | async)">Load More Products</span>
        </button>
      </div>
    </div>
  `
})
export class ProductListingComponent implements OnInit, OnDestroy {
  searchControl = new FormControl('');
  categoryControl = new FormControl('');
  minPriceControl = new FormControl(0);
  maxPriceControl = new FormControl(1000);

  categories$ = this.productService.getCategories();
  
  private readonly destroy$ = new Subject<void>();
  private readonly loadMore$ = new Subject<void>();
  
  // State
  products$: Observable<Product[]>;
  loading$: Observable<boolean>;
  hasMore$: Observable<boolean>;

  constructor(
    private productService: ProductService
  ) {
    this.setupProductStream();
  }

  ngOnInit(): void {
    this.loadMore$.next(); // Initial load
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }

  loadMore(): void {
    this.loadMore$.next();
  }

  addToCart(product: Product): void {
    this.cartService.addItem(product);
  }

  trackByProduct(index: number, product: Product): any {
    return product.id;
  }

  private setupProductStream(): void {
    const filters$ = combineLatest([
      this.searchControl.valueChanges.pipe(startWith('')),
      this.categoryControl.valueChanges.pipe(startWith('')),
      this.minPriceControl.valueChanges.pipe(startWith(0)),
      this.maxPriceControl.valueChanges.pipe(startWith(1000))
    ]).pipe(
      debounceTime(300),
      distinctUntilChanged((a, b) => JSON.stringify(a) === JSON.stringify(b))
    );

    // Reset products when filters change
    filters$.pipe(
      skip(1), // Skip initial emission
      takeUntil(this.destroy$)
    ).subscribe(() => {
      this.resetProducts();
    });

    // Product loading stream
    this.products$ = this.loadMore$.pipe(
      withLatestFrom(filters$),
      scan((acc, [_, [search, category, minPrice, maxPrice]]) => {
        const page = acc.products.length === 0 ? 1 : acc.page + 1;
        const filters = { search, category, minPrice, maxPrice };
        
        return { ...acc, page, filters };
      }, { products: [] as Product[], page: 0, filters: {} }),
      switchMap(({ page, filters }) =>
        this.productService.getProducts(page, 20, filters).pipe(
          map(response => ({
            products: page === 1 ? response.items : 
                     [...acc.products, ...response.items],
            hasMore: response.items.length === 20
          }))
        )
      ),
      map(({ products }) => products),
      shareReplay(1)
    );

    this.loading$ = this.loadMore$.pipe(
      switchMap(() => 
        concat(
          of(true),
          this.products$.pipe(take(1), map(() => false))
        )
      )
    );

    this.hasMore$ = this.products$.pipe(
      switchMap(() => 
        this.productService.hasMoreProducts()
      )
    );
  }

  private resetProducts(): void {
    // Trigger reset by emitting initial load
    this.loadMore$.next();
  }
}
```

### Social Media Feed

```typescript
@Component({
  selector: 'app-social-feed',
  template: `
    <div class="social-feed">
      <!-- Feed Content -->
      <cdk-virtual-scroll-viewport 
        itemSize="200" 
        class="feed-viewport"
        (scrolledIndexChange)="onScrollIndexChange($event)">
        
        <div *cdkVirtualFor="let post of posts$ | async; 
                             trackBy: trackByPost" 
             class="post">
          <div class="post-header">
            <img [src]="post.author.avatar" class="avatar">
            <div class="author-info">
              <h4>{{ post.author.name }}</h4>
              <span class="timestamp">{{ post.createdAt | date }}</span>
            </div>
          </div>
          
          <div class="post-content">
            <p>{{ post.content }}</p>
            <img *ngIf="post.imageUrl" [src]="post.imageUrl" class="post-image">
          </div>
          
          <div class="post-actions">
            <button (click)="likePost(post)" 
                    [class.liked]="post.isLiked">
              ❤️ {{ post.likeCount }}
            </button>
            <button (click)="sharePost(post)">
              🔄 Share
            </button>
            <button (click)="commentOnPost(post)">
              💬 {{ post.commentCount }}
            </button>
          </div>
        </div>

        <!-- Loading indicator -->
        <div *ngIf="loading$ | async" class="loading">
          Loading more posts...
        </div>
      </cdk-virtual-scroll-viewport>

      <!-- Pull to refresh -->
      <div class="pull-to-refresh" 
           (swipedown)="refreshFeed()"
           *ngIf="refreshing$ | async">
        Refreshing feed...
      </div>
    </div>
  `
})
export class SocialFeedComponent implements OnInit, OnDestroy {
  private readonly refreshTrigger$ = new Subject<void>();
  private readonly loadMoreTrigger$ = new Subject<void>();
  private readonly destroy$ = new Subject<void>();

  posts$: Observable<Post[]>;
  loading$: Observable<boolean>;
  refreshing$: Observable<boolean>;

  constructor(
    private socialService: SocialService,
    private cdr: ChangeDetectorRef
  ) {
    this.setupFeedStream();
  }

  ngOnInit(): void {
    // Initial load
    this.loadMoreTrigger$.next();
    
    // Auto-refresh every 5 minutes
    timer(0, 5 * 60 * 1000).pipe(
      takeUntil(this.destroy$)
    ).subscribe(() => this.refreshTrigger$.next());
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }

  onScrollIndexChange(index: number): void {
    // Load more when near the end
    const threshold = 5;
    this.posts$.pipe(take(1)).subscribe(posts => {
      if (index >= posts.length - threshold) {
        this.loadMoreTrigger$.next();
      }
    });
  }

  refreshFeed(): void {
    this.refreshTrigger$.next();
  }

  likePost(post: Post): void {
    this.socialService.toggleLike(post.id).subscribe();
  }

  sharePost(post: Post): void {
    this.socialService.sharePost(post.id).subscribe();
  }

  commentOnPost(post: Post): void {
    // Navigate to post detail or open comment modal
  }

  trackByPost(index: number, post: Post): any {
    return post.id;
  }

  private setupFeedStream(): void {
    // Refresh stream (replaces all posts)
    const refreshedPosts$ = this.refreshTrigger$.pipe(
      switchMap(() => 
        this.socialService.getFeedPosts(1, 20)
      ),
      map(response => response.items)
    );

    // Load more stream (appends posts)
    const loadMorePosts$ = this.loadMoreTrigger$.pipe(
      scan(page => page + 1, 0),
      switchMap(page => 
        this.socialService.getFeedPosts(page, 20)
      ),
      scan((allPosts, response) => {
        // Avoid duplicates
        const existingIds = new Set(allPosts.map(p => p.id));
        const newPosts = response.items.filter(p => !existingIds.has(p.id));
        return [...allPosts, ...newPosts];
      }, [] as Post[])
    );

    // Combine streams
    this.posts$ = merge(
      refreshedPosts$,
      loadMorePosts$
    ).pipe(
      shareReplay(1)
    );

    this.loading$ = this.loadMoreTrigger$.pipe(
      switchMap(() => 
        concat(
          of(true),
          this.posts$.pipe(take(1), map(() => false))
        )
      )
    );

    this.refreshing$ = this.refreshTrigger$.pipe(
      switchMap(() => 
        concat(
          of(true),
          refreshedPosts$.pipe(take(1), map(() => false))
        )
      )
    );
  }
}
```

---

## Summary

This lesson covers comprehensive pagination and infinite scroll patterns using RxJS Observables in Angular:

### Key Concepts Learned:
1. **Basic Pagination** - Traditional page-based data loading
2. **Infinite Scroll** - Continuous data loading on scroll
3. **Virtual Scrolling** - Performance optimization for large lists
4. **State Management** - Managing pagination state with Observables
5. **Search & Filtering** - Combining search with pagination
6. **Performance Optimization** - Caching, preloading, and monitoring
7. **Error Handling** - Robust error recovery strategies
8. **Testing** - Unit and integration testing approaches

### Best Practices:
- Use `shareReplay()` for caching pagination results
- Implement debouncing for search inputs
- Handle loading and error states gracefully
- Optimize with virtual scrolling for large datasets
- Preload adjacent pages for better UX
- Monitor performance metrics
- Test pagination scenarios thoroughly

### Common Patterns:
- **Page-based pagination** for traditional interfaces
- **Infinite scroll** for social media feeds
- **Virtual scrolling** for performance-critical lists
- **Search-enabled pagination** for data exploration
- **Cached pagination** for improved performance

These patterns provide the foundation for building scalable, performant data presentation in Angular applications with excellent user experience.

---

## Exercises

1. **Basic Pagination**: Implement a paginated table with sorting
2. **Infinite Scroll**: Build a photo gallery with infinite scroll
3. **Virtual Scrolling**: Create a contact list with 10,000+ entries
4. **Search Integration**: Add real-time search to pagination
5. **Performance Optimization**: Implement caching and preloading
6. **Error Recovery**: Add robust error handling with retry logic
7. **Testing**: Write comprehensive tests for pagination scenarios

## Additional Resources

- [Angular CDK Virtual Scrolling](https://material.angular.io/cdk/scrolling/overview)
- [RxJS Operators for Pagination](https://rxjs.dev/guide/operators)
- [Performance Best Practices](https://angular.io/guide/performance-checklist)
- [Testing Angular Observables](https://angular.io/guide/testing-services)
