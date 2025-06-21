# AsyncPipe Best Practices & Internals

## Overview

The AsyncPipe is one of Angular's most powerful features for reactive programming, automatically subscribing to Observables and handling subscription management. This lesson covers comprehensive patterns for using AsyncPipe effectively, understanding its internals, performance optimization, and advanced techniques for building reactive UIs.

## AsyncPipe Fundamentals

### Basic AsyncPipe Usage

**Simple Observable Display:**
```typescript
@Component({
  selector: 'app-user-profile',
  template: `
    <div class="user-profile">
      <!-- Basic async pipe usage -->
      <h1>{{ (user$ | async)?.name }}</h1>
      <p>{{ (user$ | async)?.email }}</p>
      
      <!-- Multiple subscriptions (inefficient) -->
      <div class="user-details">
        <span>Name: {{ (user$ | async)?.name }}</span>
        <span>Email: {{ (user$ | async)?.email }}</span>
        <span>Role: {{ (user$ | async)?.role }}</span>
      </div>

      <!-- Better approach: single subscription with alias -->
      <div class="user-details-optimized" *ngIf="user$ | async as user">
        <span>Name: {{ user.name }}</span>
        <span>Email: {{ user.email }}</span>
        <span>Role: {{ user.role }}</span>
      </div>
    </div>
  `
})
export class UserProfileComponent {
  user$ = this.userService.getCurrentUser();

  constructor(private userService: UserService) {}
}
```

### AsyncPipe with Complex Data Structures

**Handling Arrays and Nested Objects:**
```typescript
@Component({
  selector: 'app-dashboard',
  template: `
    <div class="dashboard">
      <!-- Loading state -->
      <div *ngIf="loading$ | async" class="loading">
        Loading dashboard data...
      </div>

      <!-- Error state -->
      <div *ngIf="error$ | async as error" class="error">
        Error: {{ error }}
      </div>

      <!-- Success state with data -->
      <div *ngIf="dashboardData$ | async as data" class="dashboard-content">
        <!-- User info -->
        <div class="user-section">
          <h2>Welcome, {{ data.user.name }}!</h2>
          <p>Last login: {{ data.user.lastLogin | date:'medium' }}</p>
        </div>

        <!-- Statistics -->
        <div class="stats-section">
          <div class="stat-card" *ngFor="let stat of data.statistics">
            <h3>{{ stat.label }}</h3>
            <span class="stat-value">{{ stat.value | number }}</span>
          </div>
        </div>

        <!-- Recent activities -->
        <div class="activities-section">
          <h3>Recent Activities</h3>
          <div class="activity-list">
            <div 
              *ngFor="let activity of data.activities; trackBy: trackByActivityId" 
              class="activity-item">
              <span class="activity-type">{{ activity.type }}</span>
              <span class="activity-description">{{ activity.description }}</span>
              <span class="activity-date">{{ activity.timestamp | date:'short' }}</span>
            </div>
          </div>
        </div>

        <!-- Navigation menu -->
        <nav class="dashboard-nav">
          <a 
            *ngFor="let item of data.navigationItems" 
            [routerLink]="item.route"
            class="nav-item">
            {{ item.label }}
          </a>
        </nav>
      </div>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class DashboardComponent implements OnInit {
  dashboardData$: Observable<DashboardData>;
  loading$: Observable<boolean>;
  error$: Observable<string | null>;

  constructor(
    private dashboardService: DashboardService,
    private loadingService: LoadingService
  ) {}

  ngOnInit() {
    this.setupObservables();
  }

  private setupObservables(): void {
    this.loading$ = this.loadingService.getLoading('dashboard');
    
    this.dashboardData$ = this.dashboardService.getDashboardData().pipe(
      tap(() => this.loadingService.setLoading('dashboard', false)),
      shareReplay(1)
    );

    this.error$ = this.dashboardData$.pipe(
      map(() => null),
      catchError(error => {
        this.loadingService.setLoading('dashboard', false);
        return of(error.message || 'Failed to load dashboard data');
      }),
      startWith(null)
    );
  }

  trackByActivityId(index: number, activity: Activity): string {
    return activity.id;
  }
}

interface DashboardData {
  user: {
    name: string;
    lastLogin: Date;
  };
  statistics: Array<{
    label: string;
    value: number;
  }>;
  activities: Activity[];
  navigationItems: Array<{
    label: string;
    route: string;
  }>;
}

interface Activity {
  id: string;
  type: string;
  description: string;
  timestamp: Date;
}
```

## Advanced AsyncPipe Patterns

### 1. Multiple Observable Composition

**Combining Multiple Streams:**
```typescript
@Component({
  selector: 'app-order-summary',
  template: `
    <div class="order-summary">
      <!-- Single combined subscription -->
      <div *ngIf="orderViewModel$ | async as vm" class="order-content">
        <!-- Loading state -->
        <div *ngIf="vm.loading" class="loading-overlay">
          <div class="spinner"></div>
          <p>{{ vm.loadingMessage }}</p>
        </div>

        <!-- Error state -->
        <div *ngIf="vm.error" class="error-banner">
          {{ vm.error }}
          <button (click)="retry()">Retry</button>
        </div>

        <!-- Success state -->
        <div *ngIf="vm.order && !vm.loading" class="order-details">
          <!-- Order header -->
          <div class="order-header">
            <h2>Order #{{ vm.order.id }}</h2>
            <span class="order-status" [class]="vm.order.status">
              {{ vm.order.status | titlecase }}
            </span>
          </div>

          <!-- Customer info -->
          <div class="customer-section">
            <h3>Customer Information</h3>
            <p>{{ vm.customer.name }}</p>
            <p>{{ vm.customer.email }}</p>
            <p>{{ vm.customer.phone }}</p>
          </div>

          <!-- Order items -->
          <div class="items-section">
            <h3>Order Items</h3>
            <div class="items-list">
              <div *ngFor="let item of vm.order.items" class="order-item">
                <img [src]="item.product.imageUrl" [alt]="item.product.name">
                <div class="item-details">
                  <h4>{{ item.product.name }}</h4>
                  <p>Quantity: {{ item.quantity }}</p>
                  <p>Price: {{ item.price | currency }}</p>
                </div>
              </div>
            </div>
          </div>

          <!-- Pricing breakdown -->
          <div class="pricing-section">
            <div class="price-row">
              <span>Subtotal:</span>
              <span>{{ vm.pricing.subtotal | currency }}</span>
            </div>
            <div class="price-row">
              <span>Tax:</span>
              <span>{{ vm.pricing.tax | currency }}</span>
            </div>
            <div class="price-row">
              <span>Shipping:</span>
              <span>{{ vm.pricing.shipping | currency }}</span>
            </div>
            <div class="price-row total">
              <span>Total:</span>
              <span>{{ vm.pricing.total | currency }}</span>
            </div>
          </div>

          <!-- Actions -->
          <div class="actions-section">
            <button 
              *ngFor="let action of vm.availableActions"
              (click)="executeAction(action)"
              [disabled]="vm.processingAction"
              class="action-btn">
              {{ action.label }}
            </button>
          </div>
        </div>
      </div>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class OrderSummaryComponent implements OnInit, OnDestroy {
  @Input() orderId: string;
  
  private destroy$ = new Subject<void>();
  private retrySubject = new Subject<void>();
  
  orderViewModel$: Observable<OrderViewModel>;

  constructor(
    private orderService: OrderService,
    private customerService: CustomerService,
    private pricingService: PricingService
  ) {}

  ngOnInit() {
    this.setupViewModel();
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }

  private setupViewModel(): void {
    const trigger$ = merge(
      of(null), // Initial trigger
      this.retrySubject.asObservable() // Retry trigger
    );

    this.orderViewModel$ = trigger$.pipe(
      switchMap(() => this.loadOrderData()),
      takeUntil(this.destroy$),
      shareReplay(1)
    );
  }

  private loadOrderData(): Observable<OrderViewModel> {
    const loading: OrderViewModel = {
      loading: true,
      loadingMessage: 'Loading order details...',
      error: null,
      order: null,
      customer: null,
      pricing: null,
      availableActions: [],
      processingAction: false
    };

    return of(loading).pipe(
      switchMap(() => 
        combineLatest([
          this.orderService.getOrder(this.orderId),
          this.customerService.getCustomer(this.orderId),
          this.pricingService.getOrderPricing(this.orderId)
        ]).pipe(
          map(([order, customer, pricing]) => ({
            loading: false,
            loadingMessage: '',
            error: null,
            order,
            customer,
            pricing,
            availableActions: this.getAvailableActions(order),
            processingAction: false
          })),
          catchError(error => of({
            loading: false,
            loadingMessage: '',
            error: 'Failed to load order details. Please try again.',
            order: null,
            customer: null,
            pricing: null,
            availableActions: [],
            processingAction: false
          }))
        )
      ),
      startWith(loading)
    );
  }

  private getAvailableActions(order: Order): OrderAction[] {
    const actions: OrderAction[] = [];

    switch (order.status) {
      case 'pending':
        actions.push(
          { id: 'cancel', label: 'Cancel Order', type: 'danger' },
          { id: 'modify', label: 'Modify Order', type: 'secondary' }
        );
        break;
      case 'confirmed':
        actions.push(
          { id: 'track', label: 'Track Order', type: 'primary' }
        );
        break;
      case 'delivered':
        actions.push(
          { id: 'return', label: 'Return Items', type: 'secondary' },
          { id: 'review', label: 'Write Review', type: 'primary' }
        );
        break;
    }

    return actions;
  }

  retry(): void {
    this.retrySubject.next();
  }

  executeAction(action: OrderAction): void {
    console.log('Executing action:', action);
    // Implementation for order actions
  }
}

interface OrderViewModel {
  loading: boolean;
  loadingMessage: string;
  error: string | null;
  order: Order | null;
  customer: Customer | null;
  pricing: OrderPricing | null;
  availableActions: OrderAction[];
  processingAction: boolean;
}

interface OrderAction {
  id: string;
  label: string;
  type: 'primary' | 'secondary' | 'danger';
}
```

### 2. Conditional Display with AsyncPipe

**Advanced Conditional Rendering:**
```typescript
@Component({
  selector: 'app-content-viewer',
  template: `
    <div class="content-viewer">
      <!-- Method 1: Using multiple async pipes (less efficient) -->
      <div class="approach-1">
        <div *ngIf="(content$ | async)?.type === 'article'" class="article">
          <h1>{{ (content$ | async)?.title }}</h1>
          <div [innerHTML]="(content$ | async)?.body"></div>
        </div>
        
        <div *ngIf="(content$ | async)?.type === 'video'" class="video">
          <h1>{{ (content$ | async)?.title }}</h1>
          <video [src]="(content$ | async)?.videoUrl" controls></video>
        </div>
      </div>

      <!-- Method 2: Single subscription with template alias (efficient) -->
      <div class="approach-2">
        <ng-container *ngIf="content$ | async as content">
          <div [ngSwitch]="content.type">
            <!-- Article content -->
            <div *ngSwitchCase="'article'" class="article">
              <h1>{{ content.title }}</h1>
              <p class="meta">
                By {{ content.author }} on {{ content.publishDate | date }}
              </p>
              <div [innerHTML]="content.body"></div>
              <div class="tags">
                <span *ngFor="let tag of content.tags" class="tag">
                  {{ tag }}
                </span>
              </div>
            </div>

            <!-- Video content -->
            <div *ngSwitchCase="'video'" class="video">
              <h1>{{ content.title }}</h1>
              <video [src]="content.videoUrl" controls [poster]="content.thumbnailUrl">
              </video>
              <p class="description">{{ content.description }}</p>
              <div class="video-stats">
                <span>Duration: {{ content.duration | duration }}</span>
                <span>Views: {{ content.viewCount | number }}</span>
              </div>
            </div>

            <!-- Gallery content -->
            <div *ngSwitchCase="'gallery'" class="gallery">
              <h1>{{ content.title }}</h1>
              <div class="image-grid">
                <img 
                  *ngFor="let image of content.images; trackBy: trackByImageId"
                  [src]="image.url" 
                  [alt]="image.caption"
                  (click)="openLightbox(image)">
              </div>
            </div>

            <!-- Unknown content type -->
            <div *ngSwitchDefault class="unknown">
              <p>Unsupported content type: {{ content.type }}</p>
            </div>
          </div>
        </ng-container>
      </div>

      <!-- Method 3: Multiple observables with combined state -->
      <div class="approach-3">
        <ng-container *ngIf="viewModel$ | async as vm">
          <!-- Loading state -->
          <div *ngIf="vm.loading" class="loading">
            <div class="spinner"></div>
            <p>Loading content...</p>
          </div>

          <!-- Error state -->
          <div *ngIf="vm.error" class="error">
            <h3>Oops! Something went wrong</h3>
            <p>{{ vm.error }}</p>
            <button (click)="retryLoad()">Try Again</button>
          </div>

          <!-- Content state -->
          <div *ngIf="vm.content && !vm.loading" class="content">
            <!-- Content header -->
            <header class="content-header">
              <h1>{{ vm.content.title }}</h1>
              <div class="content-meta">
                <span class="author">{{ vm.content.author }}</span>
                <span class="date">{{ vm.content.publishDate | date:'mediumDate' }}</span>
                <span class="read-time">{{ vm.estimatedReadTime }} min read</span>
              </div>
            </header>

            <!-- Content body -->
            <main class="content-body" [ngSwitch]="vm.content.type">
              <!-- Article rendering -->
              <article *ngSwitchCase="'article'">
                <div [innerHTML]="vm.content.body | sanitizeHtml"></div>
                
                <!-- Related articles -->
                <aside *ngIf="vm.relatedContent?.length" class="related">
                  <h3>Related Articles</h3>
                  <div *ngFor="let related of vm.relatedContent" class="related-item">
                    <a [routerLink]="['/content', related.id]">{{ related.title }}</a>
                  </div>
                </aside>
              </article>

              <!-- Video rendering -->
              <div *ngSwitchCase="'video'" class="video-container">
                <video 
                  [src]="vm.content.videoUrl" 
                  controls 
                  [poster]="vm.content.thumbnailUrl"
                  (loadedmetadata)="onVideoLoaded($event)">
                </video>
                
                <!-- Video controls -->
                <div class="video-controls">
                  <button (click)="togglePlayback()">
                    {{ vm.isPlaying ? 'Pause' : 'Play' }}
                  </button>
                  <button (click)="toggleFullscreen()">Fullscreen</button>
                </div>
              </div>
            </main>

            <!-- Content actions -->
            <footer class="content-actions">
              <button 
                *ngFor="let action of vm.availableActions"
                (click)="executeAction(action)"
                [disabled]="vm.processingAction === action.id"
                class="action-btn">
                <span *ngIf="vm.processingAction === action.id" class="spinner-small"></span>
                {{ action.label }}
              </button>
            </footer>
          </div>
        </ng-container>
      </div>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ContentViewerComponent implements OnInit, OnDestroy {
  @Input() contentId: string;
  
  private destroy$ = new Subject<void>();
  private retrySubject = new Subject<void>();
  
  content$: Observable<Content>;
  viewModel$: Observable<ContentViewModel>;

  constructor(
    private contentService: ContentService,
    private analyticsService: AnalyticsService
  ) {}

  ngOnInit() {
    this.setupObservables();
    this.trackContentView();
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }

  private setupObservables(): void {
    this.content$ = this.contentService.getContent(this.contentId).pipe(
      shareReplay(1)
    );

    const retrigger$ = merge(
      of(null),
      this.retrySubject.asObservable()
    );

    this.viewModel$ = retrigger$.pipe(
      switchMap(() => this.buildViewModel()),
      takeUntil(this.destroy$),
      shareReplay(1)
    );
  }

  private buildViewModel(): Observable<ContentViewModel> {
    const loading: ContentViewModel = {
      loading: true,
      error: null,
      content: null,
      relatedContent: [],
      estimatedReadTime: 0,
      availableActions: [],
      processingAction: null,
      isPlaying: false
    };

    return of(loading).pipe(
      switchMap(() =>
        combineLatest([
          this.content$,
          this.contentService.getRelatedContent(this.contentId),
          this.contentService.getEstimatedReadTime(this.contentId)
        ]).pipe(
          map(([content, related, readTime]) => ({
            loading: false,
            error: null,
            content,
            relatedContent: related,
            estimatedReadTime: readTime,
            availableActions: this.getAvailableActions(content),
            processingAction: null,
            isPlaying: false
          })),
          catchError(error => of({
            loading: false,
            error: 'Failed to load content',
            content: null,
            relatedContent: [],
            estimatedReadTime: 0,
            availableActions: [],
            processingAction: null,
            isPlaying: false
          }))
        )
      ),
      startWith(loading)
    );
  }

  private getAvailableActions(content: Content): ContentAction[] {
    return [
      { id: 'like', label: 'Like', icon: 'heart' },
      { id: 'share', label: 'Share', icon: 'share' },
      { id: 'bookmark', label: 'Bookmark', icon: 'bookmark' },
      { id: 'report', label: 'Report', icon: 'flag' }
    ];
  }

  private trackContentView(): void {
    this.content$.pipe(
      filter(content => !!content),
      take(1)
    ).subscribe(content => {
      this.analyticsService.trackContentView(content.id, content.type);
    });
  }

  trackByImageId(index: number, image: Image): string {
    return image.id;
  }

  retryLoad(): void {
    this.retrySubject.next();
  }

  executeAction(action: ContentAction): void {
    console.log('Executing action:', action);
  }

  onVideoLoaded(event: Event): void {
    console.log('Video loaded:', event);
  }

  togglePlayback(): void {
    // Implementation for video playback toggle
  }

  toggleFullscreen(): void {
    // Implementation for fullscreen toggle
  }

  openLightbox(image: Image): void {
    // Implementation for image lightbox
  }
}

interface ContentViewModel {
  loading: boolean;
  error: string | null;
  content: Content | null;
  relatedContent: Content[];
  estimatedReadTime: number;
  availableActions: ContentAction[];
  processingAction: string | null;
  isPlaying: boolean;
}

interface Content {
  id: string;
  type: 'article' | 'video' | 'gallery';
  title: string;
  author: string;
  publishDate: Date;
  body?: string;
  videoUrl?: string;
  thumbnailUrl?: string;
  images?: Image[];
  tags: string[];
}

interface ContentAction {
  id: string;
  label: string;
  icon: string;
}

interface Image {
  id: string;
  url: string;
  caption: string;
}
```

### 3. Performance Optimization Patterns

**Optimized AsyncPipe Usage:**
```typescript
@Component({
  selector: 'app-product-list',
  template: `
    <div class="product-list">
      <!-- Search and filters -->
      <div class="filters-section">
        <input 
          #searchInput
          type="text" 
          placeholder="Search products..."
          (input)="onSearchChange($event)">
        
        <select #categorySelect (change)="onCategoryChange($event)">
          <option value="">All Categories</option>
          <option *ngFor="let category of categories$ | async" [value]="category.id">
            {{ category.name }}
          </option>
        </select>

        <div class="sort-options">
          <label>Sort by:</label>
          <select (change)="onSortChange($event)">
            <option value="name">Name</option>
            <option value="price-asc">Price (Low to High)</option>
            <option value="price-desc">Price (High to Low)</option>
            <option value="rating">Rating</option>
          </select>
        </div>
      </div>

      <!-- Results section with optimized async pipe usage -->
      <div class="results-section">
        <!-- Using trackBy for performance -->
        <ng-container *ngIf="productViewModel$ | async as vm">
          <!-- Loading skeleton -->
          <div *ngIf="vm.loading" class="loading-skeleton">
            <div *ngFor="let skeleton of skeletonArray" class="product-skeleton">
              <div class="skeleton-image"></div>
              <div class="skeleton-text"></div>
              <div class="skeleton-text short"></div>
            </div>
          </div>

          <!-- Empty state -->
          <div *ngIf="!vm.loading && vm.products.length === 0" class="empty-state">
            <h3>No products found</h3>
            <p>Try adjusting your search criteria</p>
          </div>

          <!-- Products grid -->
          <div *ngIf="!vm.loading && vm.products.length > 0" class="products-grid">
            <div 
              *ngFor="let product of vm.products; trackBy: trackByProductId; index as i"
              class="product-card"
              [class.featured]="product.featured"
              [@slideIn]="i">
              
              <!-- Product image with lazy loading -->
              <div class="product-image">
                <img 
                  [src]="product.imageUrl" 
                  [alt]="product.name"
                  loading="lazy"
                  (error)="onImageError($event)">
                <div *ngIf="product.discount" class="discount-badge">
                  -{{ product.discount }}%
                </div>
              </div>

              <!-- Product info -->
              <div class="product-info">
                <h3 class="product-name">{{ product.name }}</h3>
                <p class="product-description">{{ product.description | slice:0:100 }}...</p>
                
                <div class="product-rating">
                  <div class="stars">
                    <span 
                      *ngFor="let star of getStarsArray(product.rating)"
                      class="star"
                      [class.filled]="star.filled">
                      ★
                    </span>
                  </div>
                  <span class="rating-text">({{ product.reviewCount }})</span>
                </div>

                <div class="product-pricing">
                  <span 
                    *ngIf="product.originalPrice !== product.price" 
                    class="original-price">
                    {{ product.originalPrice | currency }}
                  </span>
                  <span class="current-price">{{ product.price | currency }}</span>
                </div>

                <div class="product-actions">
                  <button 
                    class="add-to-cart-btn"
                    [disabled]="!product.inStock || (vm.addingToCart === product.id)"
                    (click)="addToCart(product)">
                    <span *ngIf="vm.addingToCart === product.id" class="spinner"></span>
                    {{ !product.inStock ? 'Out of Stock' : 'Add to Cart' }}
                  </button>
                  
                  <button 
                    class="wishlist-btn"
                    [class.active]="product.inWishlist"
                    (click)="toggleWishlist(product)">
                    ♡
                  </button>
                </div>
              </div>
            </div>
          </div>

          <!-- Pagination -->
          <div *ngIf="vm.totalPages > 1" class="pagination">
            <button 
              *ngFor="let page of getPagesArray(vm.totalPages)"
              [class.active]="page === vm.currentPage"
              [disabled]="vm.loading"
              (click)="goToPage(page)">
              {{ page }}
            </button>
          </div>

          <!-- Load more button for infinite scroll -->
          <div *ngIf="vm.hasMore && !vm.loading" class="load-more">
            <button 
              (click)="loadMore()"
              [disabled]="vm.loadingMore">
              {{ vm.loadingMore ? 'Loading...' : 'Load More' }}
            </button>
          </div>
        </ng-container>
      </div>
    </div>
  `,
  animations: [
    trigger('slideIn', [
      transition(':enter', [
        style({ opacity: 0, transform: 'translateY(20px)' }),
        animate('300ms ease-in', style({ opacity: 1, transform: 'translateY(0)' }))
      ])
    ])
  ],
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ProductListComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  private searchSubject = new Subject<string>();
  private categorySubject = new Subject<string>();
  private sortSubject = new Subject<string>();
  private pageSubject = new BehaviorSubject<number>(1);

  categories$: Observable<Category[]>;
  productViewModel$: Observable<ProductViewModel>;
  
  skeletonArray = Array(12).fill(0); // For loading skeleton

  constructor(
    private productService: ProductService,
    private categoryService: CategoryService,
    private cartService: CartService,
    private wishlistService: WishlistService
  ) {}

  ngOnInit() {
    this.setupObservables();
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }

  private setupObservables(): void {
    // Categories for filter dropdown
    this.categories$ = this.categoryService.getCategories().pipe(
      shareReplay(1)
    );

    // Combine all filter inputs
    const filters$ = combineLatest([
      this.searchSubject.pipe(
        startWith(''),
        debounceTime(300),
        distinctUntilChanged()
      ),
      this.categorySubject.pipe(startWith('')),
      this.sortSubject.pipe(startWith('name')),
      this.pageSubject.pipe(distinctUntilChanged())
    ]).pipe(
      map(([search, category, sort, page]) => ({
        search: search.trim(),
        category,
        sort,
        page
      }))
    );

    // Product view model with loading states
    this.productViewModel$ = filters$.pipe(
      switchMap(filters => this.loadProducts(filters)),
      shareReplay(1)
    );
  }

  private loadProducts(filters: ProductFilters): Observable<ProductViewModel> {
    const loading: ProductViewModel = {
      loading: true,
      products: [],
      totalPages: 0,
      currentPage: filters.page,
      hasMore: false,
      loadingMore: false,
      addingToCart: null
    };

    return of(loading).pipe(
      switchMap(() =>
        this.productService.getProducts(filters).pipe(
          map(response => ({
            loading: false,
            products: response.products,
            totalPages: response.totalPages,
            currentPage: filters.page,
            hasMore: response.hasMore,
            loadingMore: false,
            addingToCart: null
          })),
          catchError(() => of({
            loading: false,
            products: [],
            totalPages: 0,
            currentPage: 1,
            hasMore: false,
            loadingMore: false,
            addingToCart: null
          }))
        )
      ),
      startWith(loading)
    );
  }

  // Optimized trackBy function
  trackByProductId(index: number, product: Product): string {
    return product.id;
  }

  // Event handlers
  onSearchChange(event: Event): void {
    const target = event.target as HTMLInputElement;
    this.searchSubject.next(target.value);
    this.pageSubject.next(1); // Reset to first page
  }

  onCategoryChange(event: Event): void {
    const target = event.target as HTMLSelectElement;
    this.categorySubject.next(target.value);
    this.pageSubject.next(1);
  }

  onSortChange(event: Event): void {
    const target = event.target as HTMLSelectElement;
    this.sortSubject.next(target.value);
    this.pageSubject.next(1);
  }

  goToPage(page: number): void {
    this.pageSubject.next(page);
  }

  loadMore(): void {
    const currentPage = this.pageSubject.value;
    this.pageSubject.next(currentPage + 1);
  }

  addToCart(product: Product): void {
    this.cartService.addToCart(product).pipe(
      takeUntil(this.destroy$)
    ).subscribe({
      next: () => console.log('Product added to cart'),
      error: (error) => console.error('Failed to add to cart:', error)
    });
  }

  toggleWishlist(product: Product): void {
    const action = product.inWishlist ? 
      this.wishlistService.removeFromWishlist(product.id) :
      this.wishlistService.addToWishlist(product.id);
    
    action.pipe(
      takeUntil(this.destroy$)
    ).subscribe({
      next: () => console.log('Wishlist updated'),
      error: (error) => console.error('Failed to update wishlist:', error)
    });
  }

  getStarsArray(rating: number): Array<{filled: boolean}> {
    return Array(5).fill(0).map((_, index) => ({
      filled: index < Math.floor(rating)
    }));
  }

  getPagesArray(totalPages: number): number[] {
    return Array(totalPages).fill(0).map((_, index) => index + 1);
  }

  onImageError(event: Event): void {
    const img = event.target as HTMLImageElement;
    img.src = '/assets/images/placeholder.jpg';
  }
}

interface ProductViewModel {
  loading: boolean;
  products: Product[];
  totalPages: number;
  currentPage: number;
  hasMore: boolean;
  loadingMore: boolean;
  addingToCart: string | null;
}

interface ProductFilters {
  search: string;
  category: string;
  sort: string;
  page: number;
}

interface Product {
  id: string;
  name: string;
  description: string;
  imageUrl: string;
  price: number;
  originalPrice: number;
  rating: number;
  reviewCount: number;
  inStock: boolean;
  featured: boolean;
  discount: number;
  inWishlist: boolean;
}

interface Category {
  id: string;
  name: string;
}
```

## AsyncPipe Internals and Performance

### Understanding AsyncPipe Implementation

**Custom AsyncPipe Analysis:**
```typescript
// Simplified version of AsyncPipe to understand its internals
@Pipe({
  name: 'customAsync',
  pure: false // Important: AsyncPipe is impure
})
export class CustomAsyncPipe implements OnDestroy, PipeTransform {
  private subscription: Subscription | null = null;
  private obj: Observable<any> | Promise<any> | null = null;
  private latestValue: any = null;
  private markForCheck: () => void;

  constructor(private cdr: ChangeDetectorRef) {
    this.markForCheck = () => this.cdr.markForCheck();
  }

  transform(obj: Observable<any> | Promise<any> | null): any {
    if (!obj) {
      if (this.subscription) {
        this.dispose();
      }
      return null;
    }

    if (obj !== this.obj) {
      this.dispose();
      this.subscribe(obj);
      this.obj = obj;
    }

    return this.latestValue;
  }

  private subscribe(obj: Observable<any> | Promise<any>): void {
    if (obj instanceof Observable) {
      this.subscription = obj.subscribe({
        next: (value) => {
          this.latestValue = value;
          this.markForCheck();
        },
        error: (error) => {
          this.latestValue = null;
          this.markForCheck();
          throw error;
        }
      });
    } else {
      // Handle Promise
      obj.then(
        (value) => {
          this.latestValue = value;
          this.markForCheck();
        },
        (error) => {
          this.latestValue = null;
          this.markForCheck();
          throw error;
        }
      );
    }
  }

  private dispose(): void {
    if (this.subscription) {
      this.subscription.unsubscribe();
      this.subscription = null;
    }
    this.latestValue = null;
  }

  ngOnDestroy(): void {
    this.dispose();
  }
}
```

### Performance Best Practices

**Optimized AsyncPipe Usage:**
```typescript
@Component({
  selector: 'app-performance-demo',
  template: `
    <div class="performance-demo">
      <!-- ❌ Bad: Multiple subscriptions to same observable -->
      <div class="bad-example">
        <h2>{{ (user$ | async)?.name }}</h2>
        <p>Email: {{ (user$ | async)?.email }}</p>
        <p>Role: {{ (user$ | async)?.role }}</p>
        <!-- This creates 3 separate subscriptions! -->
      </div>

      <!-- ✅ Good: Single subscription with alias -->
      <div class="good-example">
        <ng-container *ngIf="user$ | async as user">
          <h2>{{ user.name }}</h2>
          <p>Email: {{ user.email }}</p>
          <p>Role: {{ user.role }}</p>
          <!-- Only one subscription -->
        </ng-container>
      </div>

      <!-- ✅ Better: Combined view model -->
      <div class="better-example">
        <ng-container *ngIf="viewModel$ | async as vm">
          <div *ngIf="vm.loading" class="loading">Loading...</div>
          <div *ngIf="vm.error" class="error">{{ vm.error }}</div>
          <div *ngIf="vm.user" class="user-info">
            <h2>{{ vm.user.name }}</h2>
            <p>Email: {{ vm.user.email }}</p>
            <p>Role: {{ vm.user.role }}</p>
            <p>Last login: {{ vm.user.lastLogin | date }}</p>
          </div>
        </ng-container>
      </div>

      <!-- ✅ Best: Reactive form with OnPush -->
      <div class="reactive-form-example">
        <form [formGroup]="userForm">
          <ng-container *ngIf="formViewModel$ | async as formVm">
            <div class="form-field">
              <label>Name</label>
              <input formControlName="name" [class.error]="formVm.nameInvalid">
              <div *ngIf="formVm.nameError" class="error-message">
                {{ formVm.nameError }}
              </div>
            </div>
            
            <div class="form-field">
              <label>Email</label>
              <input formControlName="email" [class.error]="formVm.emailInvalid">
              <div *ngIf="formVm.emailError" class="error-message">
                {{ formVm.emailError }}
              </div>
            </div>

            <button 
              type="submit" 
              [disabled]="!formVm.isValid || formVm.isSubmitting">
              {{ formVm.isSubmitting ? 'Saving...' : 'Save' }}
            </button>
          </ng-container>
        </form>
      </div>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class PerformanceDemoComponent implements OnInit {
  user$: Observable<User>;
  viewModel$: Observable<UserViewModel>;
  formViewModel$: Observable<FormViewModel>;
  userForm: FormGroup;

  constructor(
    private userService: UserService,
    private fb: FormBuilder
  ) {
    this.buildForm();
  }

  ngOnInit() {
    this.setupObservables();
  }

  private buildForm(): void {
    this.userForm = this.fb.group({
      name: ['', Validators.required],
      email: ['', [Validators.required, Validators.email]]
    });
  }

  private setupObservables(): void {
    this.user$ = this.userService.getCurrentUser().pipe(
      shareReplay(1) // Important: prevent multiple HTTP requests
    );

    // Combined view model approach
    this.viewModel$ = this.user$.pipe(
      map(user => ({ loading: false, error: null, user })),
      startWith({ loading: true, error: null, user: null }),
      catchError(error => of({ loading: false, error: error.message, user: null }))
    );

    // Form view model with validation
    this.formViewModel$ = combineLatest([
      this.userForm.statusChanges.pipe(startWith(this.userForm.status)),
      this.userForm.get('name')!.statusChanges.pipe(startWith(this.userForm.get('name')!.status)),
      this.userForm.get('email')!.statusChanges.pipe(startWith(this.userForm.get('email')!.status))
    ]).pipe(
      map(([formStatus, nameStatus, emailStatus]) => ({
        isValid: formStatus === 'VALID',
        isSubmitting: false,
        nameInvalid: nameStatus === 'INVALID' && this.userForm.get('name')!.touched,
        emailInvalid: emailStatus === 'INVALID' && this.userForm.get('email')!.touched,
        nameError: this.getFieldError('name'),
        emailError: this.getFieldError('email')
      }))
    );
  }

  private getFieldError(fieldName: string): string {
    const control = this.userForm.get(fieldName);
    if (!control || !control.errors || !control.touched) return '';

    if (control.errors['required']) return `${fieldName} is required`;
    if (control.errors['email']) return 'Please enter a valid email';
    return 'Invalid input';
  }
}

interface UserViewModel {
  loading: boolean;
  error: string | null;
  user: User | null;
}

interface FormViewModel {
  isValid: boolean;
  isSubmitting: boolean;
  nameInvalid: boolean;
  emailInvalid: boolean;
  nameError: string;
  emailError: string;
}
```

## Advanced AsyncPipe Techniques

### 1. Error Handling with AsyncPipe

**Comprehensive Error Handling:**
```typescript
@Component({
  selector: 'app-resilient-component',
  template: `
    <div class="resilient-component">
      <!-- Method 1: Error handling in observable stream -->
      <ng-container *ngIf="safeData$ | async as data">
        <div *ngIf="data.success" class="success-content">
          <h2>{{ data.result.title }}</h2>
          <p>{{ data.result.description }}</p>
        </div>
        
        <div *ngIf="!data.success" class="error-content">
          <h3>Something went wrong</h3>
          <p>{{ data.error }}</p>
          <button (click)="retry()">Try Again</button>
        </div>
      </ng-container>

      <!-- Method 2: Separate error observable -->
      <div class="separate-error-handling">
        <div *ngIf="loading$ | async" class="loading">Loading...</div>
        
        <div *ngIf="error$ | async as error" class="error">
          <h3>Error</h3>
          <p>{{ error }}</p>
          <button (click)="retry()">Retry</button>
        </div>
        
        <div *ngIf="data$ | async as data" class="data">
          <h2>{{ data.title }}</h2>
          <p>{{ data.description }}</p>
        </div>
      </div>

      <!-- Method 3: State machine approach -->
      <ng-container *ngIf="state$ | async as state">
        <div [ngSwitch]="state.type">
          <div *ngSwitchCase="'loading'" class="loading">
            <div class="spinner"></div>
            <p>{{ state.message }}</p>
          </div>
          
          <div *ngSwitchCase="'error'" class="error">
            <h3>{{ state.title }}</h3>
            <p>{{ state.message }}</p>
            <button (click)="retry()">{{ state.actionText }}</button>
          </div>
          
          <div *ngSwitchCase="'success'" class="success">
            <h2>{{ state.data.title }}</h2>
            <p>{{ state.data.description }}</p>
            <div class="actions">
              <button *ngFor="let action of state.actions" 
                      (click)="executeAction(action)">
                {{ action.label }}
              </button>
            </div>
          </div>
          
          <div *ngSwitchCase="'empty'" class="empty">
            <h3>{{ state.title }}</h3>
            <p>{{ state.message }}</p>
            <button (click)="refresh()">{{ state.actionText }}</button>
          </div>
        </div>
      </ng-container>
    </div>
  `
})
export class ResilientComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  private retrySubject = new Subject<void>();

  // Method 1: Safe data with error handling
  safeData$: Observable<SafeResult<any>>;
  
  // Method 2: Separate observables
  data$: Observable<any>;
  loading$: Observable<boolean>;
  error$: Observable<string | null>;
  
  // Method 3: State machine
  state$: Observable<AppState>;

  constructor(private dataService: DataService) {}

  ngOnInit() {
    this.setupSafeObservables();
    this.setupSeparateObservables();
    this.setupStateMachine();
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }

  private setupSafeObservables(): void {
    const trigger$ = merge(of(null), this.retrySubject);
    
    this.safeData$ = trigger$.pipe(
      switchMap(() => 
        this.dataService.getData().pipe(
          map(result => ({ success: true, result, error: null })),
          catchError(error => of({ 
            success: false, 
            result: null, 
            error: this.getErrorMessage(error) 
          }))
        )
      ),
      shareReplay(1)
    );
  }

  private setupSeparateObservables(): void {
    const loadingSubject = new BehaviorSubject(false);
    const errorSubject = new BehaviorSubject<string | null>(null);
    
    this.loading$ = loadingSubject.asObservable();
    this.error$ = errorSubject.asObservable();
    
    this.data$ = this.retrySubject.pipe(
      startWith(null),
      tap(() => {
        loadingSubject.next(true);
        errorSubject.next(null);
      }),
      switchMap(() => 
        this.dataService.getData().pipe(
          tap(() => loadingSubject.next(false)),
          catchError(error => {
            loadingSubject.next(false);
            errorSubject.next(this.getErrorMessage(error));
            return EMPTY;
          })
        )
      ),
      shareReplay(1)
    );
  }

  private setupStateMachine(): void {
    const trigger$ = merge(of(null), this.retrySubject);
    
    this.state$ = trigger$.pipe(
      switchMap(() => this.loadDataWithStates()),
      shareReplay(1)
    );
  }

  private loadDataWithStates(): Observable<AppState> {
    const loadingState: AppState = {
      type: 'loading',
      title: 'Loading',
      message: 'Please wait while we load your data...',
      data: null,
      actions: [],
      actionText: ''
    };

    return of(loadingState).pipe(
      switchMap(() =>
        this.dataService.getData().pipe(
          map(data => {
            if (!data || (Array.isArray(data) && data.length === 0)) {
              return {
                type: 'empty' as const,
                title: 'No Data Available',
                message: 'There is no data to display at the moment.',
                data: null,
                actions: [],
                actionText: 'Refresh'
              };
            }

            return {
              type: 'success' as const,
              title: 'Data Loaded Successfully',
              message: '',
              data,
              actions: [
                { id: 'refresh', label: 'Refresh' },
                { id: 'export', label: 'Export' }
              ],
              actionText: ''
            };
          }),
          catchError(error => of({
            type: 'error' as const,
            title: 'Failed to Load Data',
            message: this.getErrorMessage(error),
            data: null,
            actions: [],
            actionText: 'Try Again'
          }))
        )
      ),
      startWith(loadingState)
    );
  }

  private getErrorMessage(error: any): string {
    if (error?.status === 404) return 'Data not found';
    if (error?.status === 403) return 'Access denied';
    if (error?.status >= 500) return 'Server error. Please try again later.';
    return error?.message || 'An unexpected error occurred';
  }

  retry(): void {
    this.retrySubject.next();
  }

  refresh(): void {
    this.retry();
  }

  executeAction(action: any): void {
    console.log('Executing action:', action);
  }
}

interface SafeResult<T> {
  success: boolean;
  result: T | null;
  error: string | null;
}

type AppState = {
  type: 'loading' | 'error' | 'success' | 'empty';
  title: string;
  message: string;
  data: any;
  actions: Array<{ id: string; label: string }>;
  actionText: string;
};
```

### 2. Memory Management and Cleanup

**Preventing Memory Leaks:**
```typescript
@Component({
  selector: 'app-memory-safe',
  template: `
    <div class="memory-safe-component">
      <!-- Using async pipe prevents memory leaks -->
      <div *ngIf="longRunningTask$ | async as result">
        {{ result }}
      </div>
      
      <!-- Alternative: Manual subscription with cleanup -->
      <div>{{ manualResult }}</div>
    </div>
  `
})
export class MemorySafeComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  
  // ✅ Good: AsyncPipe handles cleanup automatically
  longRunningTask$: Observable<string>;
  
  // Manual subscription (requires cleanup)
  manualResult: string = '';

  constructor(private dataService: DataService) {}

  ngOnInit() {
    // AsyncPipe subscription (automatic cleanup)
    this.longRunningTask$ = this.dataService.getLongRunningData().pipe(
      shareReplay(1)
    );

    // Manual subscription (requires manual cleanup)
    this.dataService.getOtherData().pipe(
      takeUntil(this.destroy$) // ✅ Important: prevents memory leaks
    ).subscribe(result => {
      this.manualResult = result;
    });
  }

  ngOnDestroy() {
    // Clean up manual subscriptions
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## Common Pitfalls and Solutions

### 1. Multiple Subscriptions
```typescript
// ❌ Bad: Multiple async pipes create multiple subscriptions
template: `
  <div>{{ (data$ | async)?.name }}</div>
  <div>{{ (data$ | async)?.email }}</div>
  <div>{{ (data$ | async)?.phone }}</div>
`

// ✅ Good: Single subscription with alias
template: `
  <ng-container *ngIf="data$ | async as data">
    <div>{{ data.name }}</div>
    <div>{{ data.email }}</div>
    <div>{{ data.phone }}</div>
  </ng-container>
`
```

### 2. Null Safety
```typescript
// ❌ Risky: No null checking
template: `<div>{{ (user$ | async).name }}</div>`

// ✅ Safe: Proper null checking
template: `<div>{{ (user$ | async)?.name }}</div>`

// ✅ Better: Conditional rendering
template: `
  <div *ngIf="user$ | async as user">
    {{ user.name }}
  </div>
`
```

### 3. Error Handling
```typescript
// ❌ Bad: No error handling
data$ = this.service.getData();

// ✅ Good: Proper error handling
data$ = this.service.getData().pipe(
  catchError(error => {
    console.error('Error loading data:', error);
    return of(null); // Provide fallback
  })
);
```

## Summary

AsyncPipe is a powerful tool for reactive Angular applications:

- **Automatic Subscription Management**: No manual subscribe/unsubscribe needed
- **Performance Optimization**: Use single subscriptions with template aliases
- **Change Detection Integration**: Works seamlessly with OnPush strategy
- **Error Handling**: Implement proper error states and fallbacks
- **Memory Safety**: Prevents memory leaks through automatic cleanup

Key best practices:
- Use template aliases to avoid multiple subscriptions
- Implement proper null safety with optional chaining
- Handle loading, error, and empty states
- Combine with OnPush change detection for optimal performance
- Use shareReplay() for expensive operations
- Implement proper error handling strategies
