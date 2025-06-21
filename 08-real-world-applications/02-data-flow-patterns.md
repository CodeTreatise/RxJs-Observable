# Complex Data Flow Patterns

## Table of Contents
- [Introduction](#introduction)
- [Multi-Source Data Coordination](#multi-source-data-coordination)
- [Master-Detail Patterns](#master-detail-patterns)
- [Search and Filtering Patterns](#search-and-filtering-patterns)
- [Real-Time Data Synchronization](#real-time-data-synchronization)
- [Nested Resource Loading](#nested-resource-loading)
- [Progressive Data Loading](#progressive-data-loading)
- [Cross-Component Communication](#cross-component-communication)
- [State Synchronization Patterns](#state-synchronization-patterns)
- [Data Transformation Pipelines](#data-transformation-pipelines)
- [Error Recovery Patterns](#error-recovery-patterns)
- [Performance Optimization Patterns](#performance-optimization-patterns)
- [Testing Complex Flows](#testing-complex-flows)

## Introduction

Complex data flow patterns are essential for building sophisticated Angular applications that handle multiple data sources, real-time updates, and intricate user interactions. This lesson covers advanced RxJS patterns that solve real-world data coordination challenges.

### Pattern Categories

```typescript
// Classification of complex data flow patterns
export enum DataFlowPatternType {
  COORDINATION = 'coordination',         // Multiple sources working together
  TRANSFORMATION = 'transformation',     // Data processing pipelines
  SYNCHRONIZATION = 'synchronization',   // Real-time data sync
  COMPOSITION = 'composition',           // Combining different data types
  ORCHESTRATION = 'orchestration',       // Managing complex workflows
  CACHING = 'caching',                   // Smart caching strategies
  RECOVERY = 'recovery'                  // Error handling and recovery
}

// Pattern complexity levels
export interface DataFlowPattern {
  type: DataFlowPatternType;
  complexity: 'simple' | 'moderate' | 'complex' | 'expert';
  useCase: string;
  performance: 'low' | 'medium' | 'high';
  maintainability: 'easy' | 'moderate' | 'challenging';
}

export const PatternCatalog: DataFlowPattern[] = [
  {
    type: DataFlowPatternType.COORDINATION,
    complexity: 'complex',
    useCase: 'Dashboard with multiple real-time widgets',
    performance: 'high',
    maintainability: 'moderate'
  },
  {
    type: DataFlowPatternType.TRANSFORMATION,
    complexity: 'expert',
    useCase: 'Data analytics with complex calculations',
    performance: 'medium',
    maintainability: 'challenging'
  }
];
```

## Multi-Source Data Coordination

### Coordinated Data Loading

```typescript
// Service for coordinating multiple data sources
@Injectable({
  providedIn: 'root'
})
export class DataCoordinationService {
  private readonly http = inject(HttpClient);
  private readonly cacheService = inject(CacheService);
  
  // Coordinate loading of related data with dependencies
  loadUserDashboard(userId: string): Observable<UserDashboard> {
    return forkJoin({
      // Primary data (required)
      user: this.loadUser(userId),
      profile: this.loadUserProfile(userId),
      
      // Secondary data (can fail gracefully)
      preferences: this.loadUserPreferences(userId).pipe(
        catchError(error => {
          console.warn('Failed to load preferences:', error);
          return of(null);
        })
      ),
      
      // Tertiary data (optional)
      notifications: this.loadNotifications(userId).pipe(
        catchError(() => of([]))
      )
    }).pipe(
      // Switch to load dependent data
      switchMap(baseData => {
        return this.loadDependentData(baseData).pipe(
          map(dependentData => ({
            ...baseData,
            ...dependentData
          }))
        );
      }),
      
      // Transform to final dashboard structure
      map(data => this.transformToDashboard(data)),
      
      // Cache the result
      tap(dashboard => this.cacheService.set(`dashboard_${userId}`, dashboard)),
      
      // Handle errors gracefully
      catchError(error => this.handleDashboardError(error, userId))
    );
  }
  
  private loadDependentData(baseData: any): Observable<any> {
    const dependentRequests: { [key: string]: Observable<any> } = {};
    
    // Load team data if user is a manager
    if (baseData.user.role === 'manager') {
      dependentRequests.team = this.loadTeamMembers(baseData.user.teamId);
    }
    
    // Load projects based on user permissions
    if (baseData.profile.permissions.includes('view_projects')) {
      dependentRequests.projects = this.loadUserProjects(baseData.user.id);
    }
    
    // Load analytics if user has access
    if (baseData.profile.permissions.includes('view_analytics')) {
      dependentRequests.analytics = this.loadUserAnalytics(baseData.user.id);
    }
    
    return Object.keys(dependentRequests).length > 0
      ? forkJoin(dependentRequests)
      : of({});
  }
  
  // Progressive data loading with fallbacks
  loadDataWithFallbacks<T>(
    primarySource: Observable<T>,
    fallbackSources: Observable<T>[],
    timeout: number = 5000
  ): Observable<T> {
    return primarySource.pipe(
      timeout(timeout),
      catchError(primaryError => {
        console.warn('Primary source failed, trying fallbacks:', primaryError);
        
        return this.tryFallbacks(fallbackSources, 0).pipe(
          catchError(fallbackError => {
            console.error('All sources failed:', { primaryError, fallbackError });
            return throwError(() => new Error('All data sources failed'));
          })
        );
      })
    );
  }
  
  private tryFallbacks<T>(sources: Observable<T>[], index: number): Observable<T> {
    if (index >= sources.length) {
      return throwError(() => new Error('No more fallback sources'));
    }
    
    return sources[index].pipe(
      catchError(error => {
        console.warn(`Fallback ${index} failed:`, error);
        return this.tryFallbacks(sources, index + 1);
      })
    );
  }
}

// Example usage in component
@Component({
  selector: 'app-dashboard',
  template: `
    <div class="dashboard" *ngIf="dashboard$ | async as dashboard">
      <!-- User info section -->
      <app-user-info [user]="dashboard.user" [profile]="dashboard.profile"></app-user-info>
      
      <!-- Conditional sections based on loaded data -->
      <app-team-section *ngIf="dashboard.team" [team]="dashboard.team"></app-team-section>
      <app-projects-section *ngIf="dashboard.projects" [projects]="dashboard.projects"></app-projects-section>
      <app-analytics-section *ngIf="dashboard.analytics" [analytics]="dashboard.analytics"></app-analytics-section>
      
      <!-- Loading states for optional data -->
      <div *ngIf="loadingStates$ | async as loading">
        <app-loading *ngIf="loading.team">Loading team data...</app-loading>
        <app-loading *ngIf="loading.projects">Loading projects...</app-loading>
      </div>
    </div>
  `
})
export class DashboardComponent implements OnInit {
  private readonly coordinationService = inject(DataCoordinationService);
  private readonly route = inject(ActivatedRoute);
  
  dashboard$!: Observable<UserDashboard>;
  loadingStates$!: Observable<LoadingStates>;
  
  ngOnInit(): void {
    const userId$ = this.route.params.pipe(
      map(params => params['userId']),
      distinctUntilChanged()
    );
    
    this.dashboard$ = userId$.pipe(
      switchMap(userId => this.coordinationService.loadUserDashboard(userId)),
      shareReplay(1)
    );
    
    this.loadingStates$ = this.createLoadingStates();
  }
  
  private createLoadingStates(): Observable<LoadingStates> {
    return combineLatest([
      this.dashboard$.pipe(startWith(null)),
      // Add other loading observables
    ]).pipe(
      map(([dashboard]) => ({
        team: !dashboard?.team,
        projects: !dashboard?.projects,
        analytics: !dashboard?.analytics
      }))
    );
  }
}

interface UserDashboard {
  user: User;
  profile: UserProfile;
  preferences?: UserPreferences;
  notifications: Notification[];
  team?: TeamMember[];
  projects?: Project[];
  analytics?: AnalyticsData;
}

interface LoadingStates {
  team: boolean;
  projects: boolean;
  analytics: boolean;
}
```

### Parallel Data Processing

```typescript
// Advanced parallel data processing patterns
@Injectable({
  providedIn: 'root'
})
export class ParallelDataProcessor {
  
  // Process multiple data sets in parallel with different transformations
  processDataSets<T, R>(
    dataSets: Observable<T>[],
    processors: ((data: T) => Observable<R>)[],
    options: {
      failFast?: boolean;
      maxConcurrency?: number;
      timeout?: number;
    } = {}
  ): Observable<R[]> {
    const { failFast = false, maxConcurrency = 5, timeout = 10000 } = options;
    
    // Create processing observables
    const processedSets = dataSets.map((dataSet, index) => {
      const processor = processors[index] || processors[0]; // Fallback to first processor
      
      return dataSet.pipe(
        switchMap(data => processor(data)),
        timeout(timeout),
        catchError(error => {
          if (failFast) {
            return throwError(() => error);
          }
          console.warn(`Dataset ${index} processing failed:`, error);
          return of(null as R);
        })
      );
    });
    
    // Control concurrency
    return from(processedSets).pipe(
      mergeMap(processedSet => processedSet, maxConcurrency),
      toArray(),
      map(results => results.filter(result => result !== null))
    );
  }
  
  // Batch processing with progress tracking
  processBatchWithProgress<T, R>(
    items: T[],
    processor: (item: T) => Observable<R>,
    batchSize: number = 10
  ): Observable<BatchProgress<R>> {
    const totalItems = items.length;
    const batches = this.createBatches(items, batchSize);
    
    return from(batches).pipe(
      concatMap((batch, batchIndex) => {
        return forkJoin(
          batch.map(item => processor(item).pipe(
            catchError(error => {
              console.warn('Item processing failed:', error);
              return of(null as R);
            })
          ))
        ).pipe(
          map(results => ({
            batchIndex,
            results: results.filter(r => r !== null),
            processed: (batchIndex + 1) * batchSize,
            total: totalItems,
            progress: Math.min(((batchIndex + 1) * batchSize) / totalItems * 100, 100)
          }))
        );
      }),
      scan((acc, curr) => ({
        ...curr,
        allResults: [...acc.allResults, ...curr.results]
      }), { allResults: [] } as BatchProgress<R> & { allResults: R[] })
    );
  }
  
  private createBatches<T>(items: T[], batchSize: number): T[][] {
    const batches: T[][] = [];
    for (let i = 0; i < items.length; i += batchSize) {
      batches.push(items.slice(i, i + batchSize));
    }
    return batches;
  }
}

interface BatchProgress<T> {
  batchIndex: number;
  results: T[];
  processed: number;
  total: number;
  progress: number;
}
```

## Master-Detail Patterns

### Dynamic Master-Detail Flow

```typescript
// Advanced master-detail pattern with caching and optimization
@Injectable({
  providedIn: 'root'
})
export class MasterDetailService<TMaster, TDetail> {
  private readonly masterCache = new Map<string, TMaster>();
  private readonly detailCache = new Map<string, TDetail>();
  private readonly masterSubject = new BehaviorSubject<TMaster[]>([]);
  private readonly selectedMasterSubject = new BehaviorSubject<TMaster | null>(null);
  
  // Master list stream
  masters$ = this.masterSubject.asObservable();
  
  // Selected master stream
  selectedMaster$ = this.selectedMasterSubject.asObservable();
  
  // Details for selected master
  selectedDetails$ = this.selectedMaster$.pipe(
    switchMap(master => {
      if (!master) return of(null);
      return this.loadDetails(this.getMasterId(master));
    }),
    shareReplay(1)
  );
  
  // Combined master-detail stream
  masterWithDetails$ = combineLatest([
    this.selectedMaster$,
    this.selectedDetails$
  ]).pipe(
    map(([master, details]) => master ? { master, details } : null),
    shareReplay(1)
  );
  
  constructor(
    private masterLoader: (filters?: any) => Observable<TMaster[]>,
    private detailLoader: (masterId: string) => Observable<TDetail>,
    private getMasterId: (master: TMaster) => string
  ) {}
  
  // Load masters with optional filtering
  loadMasters(filters?: any): Observable<TMaster[]> {
    return this.masterLoader(filters).pipe(
      tap(masters => {
        // Update cache
        masters.forEach(master => {
          this.masterCache.set(this.getMasterId(master), master);
        });
        
        // Update stream
        this.masterSubject.next(masters);
      }),
      catchError(error => {
        console.error('Failed to load masters:', error);
        // Return cached data if available
        const cachedMasters = Array.from(this.masterCache.values());
        if (cachedMasters.length > 0) {
          return of(cachedMasters);
        }
        return throwError(() => error);
      })
    );
  }
  
  // Select a master and preload its details
  selectMaster(master: TMaster | null): void {
    this.selectedMasterSubject.next(master);
    
    // Preload details
    if (master) {
      this.preloadDetails(this.getMasterId(master));
    }
  }
  
  // Load details with caching
  private loadDetails(masterId: string): Observable<TDetail> {
    // Check cache first
    if (this.detailCache.has(masterId)) {
      return of(this.detailCache.get(masterId)!);
    }
    
    return this.detailLoader(masterId).pipe(
      tap(details => this.detailCache.set(masterId, details)),
      shareReplay(1)
    );
  }
  
  // Preload details for performance
  private preloadDetails(masterId: string): void {
    if (!this.detailCache.has(masterId)) {
      this.loadDetails(masterId).subscribe();
    }
  }
  
  // Smart prefetching based on user behavior
  enableSmartPrefetching(): void {
    this.selectedMaster$.pipe(
      debounceTime(500), // Wait for user to settle on a selection
      switchMap(master => {
        if (!master) return of([]);
        
        // Get adjacent masters for prefetching
        const masters = this.masterSubject.value;
        const currentIndex = masters.findIndex(m => 
          this.getMasterId(m) === this.getMasterId(master)
        );
        
        const toPreload = [
          masters[currentIndex - 1],
          masters[currentIndex + 1]
        ].filter(Boolean);
        
        return from(toPreload);
      })
    ).subscribe(master => {
      this.preloadDetails(this.getMasterId(master));
    });
  }
  
  // Update master and invalidate related caches
  updateMaster(master: TMaster): Observable<TMaster> {
    const masterId = this.getMasterId(master);
    
    return this.updateMasterOnServer(master).pipe(
      tap(updatedMaster => {
        // Update caches
        this.masterCache.set(masterId, updatedMaster);
        this.detailCache.delete(masterId); // Invalidate details cache
        
        // Update streams
        const masters = this.masterSubject.value.map(m => 
          this.getMasterId(m) === masterId ? updatedMaster : m
        );
        this.masterSubject.next(masters);
        
        if (this.selectedMasterSubject.value && 
            this.getMasterId(this.selectedMasterSubject.value) === masterId) {
          this.selectedMasterSubject.next(updatedMaster);
        }
      })
    );
  }
  
  private updateMasterOnServer(master: TMaster): Observable<TMaster> {
    // Implementation depends on your API
    return of(master); // Placeholder
  }
}

// Example: Product-Orders master-detail
interface Product {
  id: string;
  name: string;
  category: string;
  price: number;
}

interface OrderDetails {
  orders: Order[];
  totalRevenue: number;
  averageOrderValue: number;
  topCustomers: Customer[];
}

@Component({
  selector: 'app-product-orders',
  template: `
    <div class="master-detail-container">
      <!-- Master list -->
      <div class="master-panel">
        <h3>Products</h3>
        <app-search-box (search)="onSearch($event)"></app-search-box>
        
        <div class="product-list">
          <div 
            *ngFor="let product of masters$ | async; trackBy: trackByProductId"
            class="product-item"
            [class.selected]="isSelected(product)"
            (click)="selectProduct(product)">
            
            <h4>{{ product.name }}</h4>
            <p>{{ product.category }} - {{ product.price | currency }}</p>
          </div>
        </div>
      </div>
      
      <!-- Detail panel -->
      <div class="detail-panel">
        <div *ngIf="masterWithDetails$ | async as data">
          <h3>Orders for {{ data.master.name }}</h3>
          
          <div *ngIf="data.details" class="order-details">
            <div class="summary">
              <p>Total Revenue: {{ data.details.totalRevenue | currency }}</p>
              <p>Average Order: {{ data.details.averageOrderValue | currency }}</p>
            </div>
            
            <app-orders-list [orders]="data.details.orders"></app-orders-list>
            <app-customers-list [customers]="data.details.topCustomers"></app-customers-list>
          </div>
          
          <div *ngIf="!data.details" class="loading">
            Loading order details...
          </div>
        </div>
        
        <div *ngIf="!(selectedMaster$ | async)" class="no-selection">
          Select a product to view order details
        </div>
      </div>
    </div>
  `
})
export class ProductOrdersComponent implements OnInit {
  private readonly productService = inject(ProductService);
  private readonly orderService = inject(OrderService);
  
  private masterDetailService!: MasterDetailService<Product, OrderDetails>;
  
  masters$!: Observable<Product[]>;
  selectedMaster$!: Observable<Product | null>;
  masterWithDetails$!: Observable<{ master: Product; details: OrderDetails | null } | null>;
  
  ngOnInit(): void {
    // Initialize master-detail service
    this.masterDetailService = new MasterDetailService(
      (filters) => this.productService.getProducts(filters),
      (productId) => this.orderService.getOrderDetails(productId),
      (product) => product.id
    );
    
    // Set up streams
    this.masters$ = this.masterDetailService.masters$;
    this.selectedMaster$ = this.masterDetailService.selectedMaster$;
    this.masterWithDetails$ = this.masterDetailService.masterWithDetails$;
    
    // Enable smart prefetching
    this.masterDetailService.enableSmartPrefetching();
    
    // Load initial data
    this.loadProducts();
  }
  
  loadProducts(filters?: any): void {
    this.masterDetailService.loadMasters(filters).subscribe();
  }
  
  selectProduct(product: Product): void {
    this.masterDetailService.selectMaster(product);
  }
  
  onSearch(searchTerm: string): void {
    this.loadProducts({ search: searchTerm });
  }
  
  isSelected(product: Product): boolean {
    const selected = this.masterDetailService.selectedMaster$.value;
    return selected ? selected.id === product.id : false;
  }
  
  trackByProductId(index: number, product: Product): string {
    return product.id;
  }
}
```

## Search and Filtering Patterns

### Advanced Search with Debouncing and Caching

```typescript
// Comprehensive search service with advanced features
@Injectable({
  providedIn: 'root'
})
export class AdvancedSearchService<T> {
  private readonly searchCache = new Map<string, SearchResult<T>>();
  private readonly searchSubject = new Subject<SearchQuery>();
  private readonly resultsSubject = new BehaviorSubject<SearchResult<T> | null>(null);
  private readonly loadingSubject = new BehaviorSubject<boolean>(false);
  
  // Public streams
  results$ = this.resultsSubject.asObservable();
  loading$ = this.loadingSubject.asObservable();
  
  constructor(
    private searchFn: (query: SearchQuery) => Observable<T[]>,
    private options: SearchOptions = {}
  ) {
    this.setupSearchStream();
  }
  
  // Main search method
  search(query: SearchQuery): void {
    this.searchSubject.next(query);
  }
  
  // Clear search results
  clear(): void {
    this.resultsSubject.next(null);
  }
  
  private setupSearchStream(): void {
    this.searchSubject.pipe(
      // Debounce user input
      debounceTime(this.options.debounceTime || 300),
      
      // Avoid duplicate searches
      distinctUntilChanged((prev, curr) => this.compareQueries(prev, curr)),
      
      // Don't search for empty or too short queries
      filter(query => this.isValidQuery(query)),
      
      // Handle loading state
      tap(() => this.loadingSubject.next(true)),
      
      // Switch to new search (cancel previous)
      switchMap(query => this.performSearch(query)),
      
      // Handle errors
      catchError(error => {
        console.error('Search error:', error);
        this.loadingSubject.next(false);
        return of(null);
      })
    ).subscribe(result => {
      this.loadingSubject.next(false);
      if (result) {
        this.resultsSubject.next(result);
      }
    });
  }
  
  private performSearch(query: SearchQuery): Observable<SearchResult<T>> {
    const cacheKey = this.getCacheKey(query);
    
    // Check cache first
    if (this.searchCache.has(cacheKey)) {
      const cached = this.searchCache.get(cacheKey)!;
      const cacheAge = Date.now() - cached.timestamp;
      
      if (cacheAge < (this.options.cacheTimeout || 300000)) { // 5 minutes default
        return of(cached);
      }
    }
    
    // Perform actual search
    return this.searchFn(query).pipe(
      map(items => ({
        query,
        items,
        totalCount: items.length,
        timestamp: Date.now(),
        facets: this.extractFacets(items),
        suggestions: this.generateSuggestions(query, items)
      })),
      tap(result => {
        // Cache the result
        this.searchCache.set(cacheKey, result);
        
        // Cleanup old cache entries
        this.cleanupCache();
      })
    );
  }
  
  // Advanced filtering with facets
  searchWithFacets(
    baseQuery: string,
    facets: SearchFacet[],
    pagination: PaginationOptions = {}
  ): Observable<FacetedSearchResult<T>> {
    const query: SearchQuery = {
      text: baseQuery,
      facets,
      ...pagination
    };
    
    return this.performSearch(query).pipe(
      map(result => ({
        ...result,
        facets: this.calculateFacetCounts(result.items, facets),
        pagination: this.calculatePagination(result.items, pagination)
      }))
    );
  }
  
  // Real-time search suggestions
  getSuggestions(partialQuery: string): Observable<string[]> {
    return of(partialQuery).pipe(
      debounceTime(150),
      filter(query => query.length >= 2),
      switchMap(query => this.generateSuggestionsFromCache(query)),
      distinctUntilChanged()
    );
  }
  
  private compareQueries(prev: SearchQuery, curr: SearchQuery): boolean {
    return JSON.stringify(prev) === JSON.stringify(curr);
  }
  
  private isValidQuery(query: SearchQuery): boolean {
    const minLength = this.options.minQueryLength || 2;
    return query.text ? query.text.length >= minLength : true;
  }
  
  private getCacheKey(query: SearchQuery): string {
    return JSON.stringify(query);
  }
  
  private extractFacets(items: T[]): SearchFacet[] {
    // Implementation depends on your data structure
    return [];
  }
  
  private generateSuggestions(query: SearchQuery, items: T[]): string[] {
    // Generate search suggestions based on results
    return [];
  }
  
  private generateSuggestionsFromCache(partialQuery: string): Observable<string[]> {
    const suggestions: string[] = [];
    
    this.searchCache.forEach(result => {
      if (result.query.text?.toLowerCase().includes(partialQuery.toLowerCase())) {
        suggestions.push(result.query.text);
      }
    });
    
    return of(suggestions.slice(0, 10)); // Limit suggestions
  }
  
  private calculateFacetCounts(items: T[], facets: SearchFacet[]): SearchFacet[] {
    // Calculate facet counts based on current results
    return facets;
  }
  
  private calculatePagination(items: T[], options: PaginationOptions): PaginationResult {
    const page = options.page || 1;
    const pageSize = options.pageSize || 20;
    const totalItems = items.length;
    const totalPages = Math.ceil(totalItems / pageSize);
    
    return {
      page,
      pageSize,
      totalItems,
      totalPages,
      hasNext: page < totalPages,
      hasPrevious: page > 1
    };
  }
  
  private cleanupCache(): void {
    const maxCacheSize = this.options.maxCacheSize || 100;
    
    if (this.searchCache.size > maxCacheSize) {
      // Remove oldest entries
      const entries = Array.from(this.searchCache.entries())
        .sort(([, a], [, b]) => a.timestamp - b.timestamp);
      
      const toRemove = entries.slice(0, entries.length - maxCacheSize);
      toRemove.forEach(([key]) => this.searchCache.delete(key));
    }
  }
}

// Usage example: Product search component
@Component({
  selector: 'app-product-search',
  template: `
    <div class="search-container">
      <!-- Search input -->
      <div class="search-input">
        <input 
          type="text" 
          [formControl]="searchControl"
          placeholder="Search products..."
          (keyup.enter)="performSearch()">
        
        <button (click)="performSearch()" [disabled]="loading$ | async">
          Search
        </button>
      </div>
      
      <!-- Search suggestions -->
      <div class="suggestions" *ngIf="suggestions$ | async as suggestions">
        <div 
          *ngFor="let suggestion of suggestions"
          class="suggestion"
          (click)="applySuggestion(suggestion)">
          {{ suggestion }}
        </div>
      </div>
      
      <!-- Facets -->
      <div class="facets">
        <h4>Filter by:</h4>
        <app-facet-filter 
          *ngFor="let facet of availableFacets"
          [facet]="facet"
          (change)="onFacetChange($event)">
        </app-facet-filter>
      </div>
      
      <!-- Results -->
      <div class="results">
        <div *ngIf="loading$ | async" class="loading">Searching...</div>
        
        <div *ngIf="results$ | async as results" class="search-results">
          <div class="results-summary">
            Found {{ results.totalCount }} products
          </div>
          
          <app-product-grid [products]="results.items"></app-product-grid>
          
          <app-pagination 
            [pagination]="results.pagination"
            (pageChange)="onPageChange($event)">
          </app-pagination>
        </div>
      </div>
    </div>
  `
})
export class ProductSearchComponent implements OnInit {
  private readonly productService = inject(ProductService);
  
  searchControl = new FormControl('');
  private searchService!: AdvancedSearchService<Product>;
  
  // Component streams
  results$!: Observable<FacetedSearchResult<Product> | null>;
  loading$!: Observable<boolean>;
  suggestions$!: Observable<string[]>;
  
  // Search state
  availableFacets: SearchFacet[] = [
    { name: 'category', label: 'Category', values: [] },
    { name: 'brand', label: 'Brand', values: [] },
    { name: 'price', label: 'Price Range', values: [] }
  ];
  
  private currentFacets: SearchFacet[] = [];
  private currentPagination: PaginationOptions = { page: 1, pageSize: 20 };
  
  ngOnInit(): void {
    this.setupSearchService();
    this.setupStreams();
    this.setupSearchControl();
  }
  
  private setupSearchService(): void {
    this.searchService = new AdvancedSearchService(
      (query) => this.productService.searchProducts(query),
      {
        debounceTime: 300,
        minQueryLength: 2,
        cacheTimeout: 300000,
        maxCacheSize: 50
      }
    );
  }
  
  private setupStreams(): void {
    this.loading$ = this.searchService.loading$;
    
    this.results$ = this.searchService.results$ as Observable<FacetedSearchResult<Product> | null>;
    
    this.suggestions$ = this.searchControl.valueChanges.pipe(
      debounceTime(200),
      switchMap(value => 
        value ? this.searchService.getSuggestions(value) : of([])
      )
    );
  }
  
  private setupSearchControl(): void {
    this.searchControl.valueChanges.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      filter(value => !value || value.length >= 2)
    ).subscribe(value => {
      if (value) {
        this.performSearch();
      } else {
        this.searchService.clear();
      }
    });
  }
  
  performSearch(): void {
    const searchText = this.searchControl.value || '';
    
    this.searchService.searchWithFacets(
      searchText,
      this.currentFacets,
      this.currentPagination
    ).subscribe();
  }
  
  onFacetChange(facet: SearchFacet): void {
    const index = this.currentFacets.findIndex(f => f.name === facet.name);
    
    if (index >= 0) {
      this.currentFacets[index] = facet;
    } else {
      this.currentFacets.push(facet);
    }
    
    // Reset pagination when filters change
    this.currentPagination.page = 1;
    this.performSearch();
  }
  
  onPageChange(page: number): void {
    this.currentPagination.page = page;
    this.performSearch();
  }
  
  applySuggestion(suggestion: string): void {
    this.searchControl.setValue(suggestion);
    this.performSearch();
  }
}

// Supporting interfaces
interface SearchQuery {
  text?: string;
  facets?: SearchFacet[];
  page?: number;
  pageSize?: number;
  sortBy?: string;
  sortOrder?: 'asc' | 'desc';
}

interface SearchResult<T> {
  query: SearchQuery;
  items: T[];
  totalCount: number;
  timestamp: number;
  facets?: SearchFacet[];
  suggestions?: string[];
}

interface FacetedSearchResult<T> extends SearchResult<T> {
  facets: SearchFacet[];
  pagination: PaginationResult;
}

interface SearchFacet {
  name: string;
  label: string;
  values: FacetValue[];
}

interface FacetValue {
  value: string;
  label: string;
  count?: number;
  selected?: boolean;
}

interface SearchOptions {
  debounceTime?: number;
  minQueryLength?: number;
  cacheTimeout?: number;
  maxCacheSize?: number;
}

interface PaginationOptions {
  page?: number;
  pageSize?: number;
}

interface PaginationResult {
  page: number;
  pageSize: number;
  totalItems: number;
  totalPages: number;
  hasNext: boolean;
  hasPrevious: boolean;
}
```

## Real-Time Data Synchronization

### WebSocket Integration with Conflict Resolution

```typescript
// Real-time data synchronization service
@Injectable({
  providedIn: 'root'
})
export class RealTimeSyncService<T> {
  private readonly webSocketService = inject(WebSocketService);
  private readonly conflictResolver = inject(ConflictResolverService);
  
  private readonly localUpdates$ = new Subject<DataUpdate<T>>();
  private readonly remoteUpdates$ = new Subject<DataUpdate<T>>();
  private readonly conflictedUpdates$ = new Subject<DataConflict<T>>();
  
  // Master data stream that combines local and remote changes
  data$: Observable<T[]>;
  
  // Conflict notifications
  conflicts$ = this.conflictedUpdates$.asObservable();
  
  constructor(private entityType: string) {
    this.data$ = this.createSynchronizedDataStream();
    this.setupWebSocketConnection();
  }
  
  private createSynchronizedDataStream(): Observable<T[]> {
    // Merge local and remote updates
    const allUpdates$ = merge(
      this.localUpdates$,
      this.remoteUpdates$
    );
    
    return allUpdates$.pipe(
      // Scan to accumulate state
      scan((currentData, update) => this.applyUpdate(currentData, update), [] as T[]),
      
      // Share to avoid multiple subscriptions
      shareReplay(1),
      
      // Handle errors
      catchError(error => {
        console.error('Data synchronization error:', error);
        return of([] as T[]);
      })
    );
  }
  
  private setupWebSocketConnection(): void {
    // Subscribe to entity-specific updates
    this.webSocketService.on(`${this.entityType}:updated`).pipe(
      map(data => ({
        type: 'update' as const,
        entity: data.entity,
        timestamp: data.timestamp,
        source: 'remote' as const,
        version: data.version
      }))
    ).subscribe(update => this.handleRemoteUpdate(update));
    
    this.webSocketService.on(`${this.entityType}:deleted`).pipe(
      map(data => ({
        type: 'delete' as const,
        entityId: data.entityId,
        timestamp: data.timestamp,
        source: 'remote' as const,
        version: data.version
      }))
    ).subscribe(update => this.handleRemoteUpdate(update));
  }
  
  // Public methods for local updates
  updateEntity(entity: T): Observable<T> {
    const update: DataUpdate<T> = {
      type: 'update',
      entity,
      timestamp: Date.now(),
      source: 'local',
      version: this.generateVersion()
    };
    
    // Optimistically apply local update
    this.localUpdates$.next(update);
    
    // Send to server and handle response
    return this.sendUpdateToServer(update).pipe(
      tap(serverResponse => {
        // Server confirmed - update with server version
        if (serverResponse.version !== update.version) {
          this.remoteUpdates$.next({
            ...update,
            entity: serverResponse.entity,
            version: serverResponse.version,
            source: 'remote'
          });
        }
      }),
      catchError(error => {
        console.error('Failed to sync update:', error);
        // Revert optimistic update
        this.revertUpdate(update);
        return throwError(() => error);
      })
    );
  }
  
  deleteEntity(entityId: string): Observable<void> {
    const update: DataUpdate<T> = {
      type: 'delete',
      entityId,
      timestamp: Date.now(),
      source: 'local',
      version: this.generateVersion()
    };
    
    this.localUpdates$.next(update);
    
    return this.sendDeleteToServer(update).pipe(
      catchError(error => {
        this.revertUpdate(update);
        return throwError(() => error);
      })
    );
  }
  
  private handleRemoteUpdate(update: DataUpdate<T>): void {
    // Check for conflicts with local updates
    const conflict = this.detectConflict(update);
    
    if (conflict) {
      this.handleConflict(conflict);
    } else {
      this.remoteUpdates$.next(update);
    }
  }
  
  private detectConflict(remoteUpdate: DataUpdate<T>): DataConflict<T> | null {
    // Get current local data
    const currentData$ = this.data$.pipe(take(1));
    
    // This is simplified - in reality you'd check timestamps, versions, etc.
    // For now, assume conflict if we have a local update for the same entity
    return null; // Placeholder
  }
  
  private handleConflict(conflict: DataConflict<T>): void {
    // Emit conflict for UI to handle
    this.conflictedUpdates$.next(conflict);
    
    // Auto-resolve using strategy
    this.conflictResolver.resolve(conflict).subscribe(resolution => {
      switch (resolution.strategy) {
        case 'use-local':
          // Keep local version
          break;
        case 'use-remote':
          this.remoteUpdates$.next(conflict.remoteUpdate);
          break;
        case 'merge':
          this.remoteUpdates$.next({
            ...conflict.remoteUpdate,
            entity: resolution.mergedEntity!
          });
          break;
      }
    });
  }
  
  private applyUpdate(currentData: T[], update: DataUpdate<T>): T[] {
    switch (update.type) {
      case 'update':
        const entityId = this.getEntityId(update.entity!);
        const index = currentData.findIndex(item => 
          this.getEntityId(item) === entityId
        );
        
        if (index >= 0) {
          return [
            ...currentData.slice(0, index),
            update.entity!,
            ...currentData.slice(index + 1)
          ];
        } else {
          return [...currentData, update.entity!];
        }
        
      case 'delete':
        return currentData.filter(item => 
          this.getEntityId(item) !== update.entityId
        );
        
      default:
        return currentData;
    }
  }
  
  private revertUpdate(update: DataUpdate<T>): void {
    // Implement revert logic
    // This would emit a compensating update
  }
  
  private generateVersion(): string {
    return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
  
  private getEntityId(entity: T): string {
    // Implement based on your entity structure
    return (entity as any).id;
  }
  
  private sendUpdateToServer(update: DataUpdate<T>): Observable<any> {
    // Implement server communication
    return of({ entity: update.entity, version: update.version });
  }
  
  private sendDeleteToServer(update: DataUpdate<T>): Observable<void> {
    // Implement server communication
    return of(undefined);
  }
}

// Usage in component
@Component({
  selector: 'app-collaborative-editor',
  template: `
    <div class="collaborative-editor">
      <!-- Real-time data display -->
      <div class="data-list">
        <div 
          *ngFor="let item of data$ | async; trackBy: trackById"
          class="data-item"
          [class.updating]="isUpdating(item)">
          
          <app-data-editor 
            [item]="item"
            (update)="updateItem($event)"
            (delete)="deleteItem(item.id)">
          </app-data-editor>
        </div>
      </div>
      
      <!-- Conflict resolution UI -->
      <app-conflict-resolver 
        *ngIf="conflicts$ | async as conflicts"
        [conflicts]="conflicts"
        (resolve)="resolveConflict($event)">
      </app-conflict-resolver>
      
      <!-- Connection status -->
      <div class="status-bar">
        <span [class]="connectionStatus$ | async">
          {{ (connectionStatus$ | async) === 'connected' ? 'Online' : 'Offline' }}
        </span>
        
        <span *ngIf="pendingUpdates$ | async as pending">
          {{ pending }} pending changes
        </span>
      </div>
    </div>
  `
})
export class CollaborativeEditorComponent implements OnInit {
  private syncService!: RealTimeSyncService<DocumentItem>;
  
  data$!: Observable<DocumentItem[]>;
  conflicts$!: Observable<DataConflict<DocumentItem>[]>;
  connectionStatus$!: Observable<'connected' | 'disconnected'>;
  pendingUpdates$!: Observable<number>;
  
  ngOnInit(): void {
    this.syncService = new RealTimeSyncService<DocumentItem>('document-item');
    
    this.data$ = this.syncService.data$;
    this.conflicts$ = this.syncService.conflicts$.pipe(
      scan((acc, conflict) => [...acc, conflict], [] as DataConflict<DocumentItem>[])
    );
    
    // Set up other streams...
  }
  
  updateItem(item: DocumentItem): void {
    this.syncService.updateEntity(item).subscribe({
      next: () => console.log('Item updated successfully'),
      error: (error) => console.error('Failed to update item:', error)
    });
  }
  
  deleteItem(itemId: string): void {
    this.syncService.deleteEntity(itemId).subscribe();
  }
  
  trackById(index: number, item: DocumentItem): string {
    return item.id;
  }
  
  isUpdating(item: DocumentItem): boolean {
    // Check if item is currently being updated
    return false; // Placeholder
  }
  
  resolveConflict(resolution: ConflictResolution<DocumentItem>): void {
    // Handle conflict resolution
  }
}

// Supporting interfaces
interface DataUpdate<T> {
  type: 'update' | 'delete';
  entity?: T;
  entityId?: string;
  timestamp: number;
  source: 'local' | 'remote';
  version: string;
}

interface DataConflict<T> {
  localUpdate: DataUpdate<T>;
  remoteUpdate: DataUpdate<T>;
  conflictType: 'concurrent-update' | 'update-delete' | 'version-mismatch';
}

interface ConflictResolution<T> {
  strategy: 'use-local' | 'use-remote' | 'merge';
  mergedEntity?: T;
}

interface DocumentItem {
  id: string;
  title: string;
  content: string;
  lastModified: Date;
  version: string;
}
```

## Summary

Complex data flow patterns are essential for building sophisticated Angular applications. Key patterns covered include:

1. **Multi-Source Coordination**: Managing dependencies between multiple data sources with graceful fallbacks
2. **Master-Detail**: Optimized patterns with caching, prefetching, and smart loading
3. **Advanced Search**: Debounced search with caching, faceting, and suggestions
4. **Real-Time Sync**: WebSocket integration with conflict resolution and optimistic updates
5. **Progressive Loading**: Batch processing with progress tracking and concurrency control

These patterns provide the foundation for handling complex data scenarios while maintaining performance and user experience.

---

**Next:** [API Integration & HTTP Patterns](./03-api-integration.md)
**Previous:** [Project Setup & Architecture](./01-project-setup.md)
