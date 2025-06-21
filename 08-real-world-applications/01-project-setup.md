# Setting up Real-World RxJS Projects

## Table of Contents
- [Introduction](#introduction)
- [Project Architecture Setup](#project-architecture-setup)
- [Development Environment Configuration](#development-environment-configuration)
- [RxJS Integration Patterns](#rxjs-integration-patterns)
- [State Management Setup](#state-management-setup)
- [Testing Infrastructure](#testing-infrastructure)
- [Performance Monitoring](#performance-monitoring)
- [Error Handling Strategy](#error-handling-strategy)
- [Code Organization](#code-organization)
- [Build and Deployment](#build-and-deployment)
- [Team Collaboration](#team-collaboration)
- [Best Practices](#best-practices)

## Introduction

Setting up a real-world RxJS project requires careful planning and architecture decisions. This lesson covers the essential setup patterns, configurations, and best practices for building scalable, maintainable applications with RxJS and Angular.

### Project Types and Considerations

```typescript
// Different project types require different RxJS setups
export enum ProjectType {
  ENTERPRISE_APP = 'enterprise-app',        // Complex business logic
  E_COMMERCE = 'e-commerce',                // High-performance, user-focused
  DASHBOARD = 'dashboard',                  // Real-time data visualization
  MOBILE_PWA = 'mobile-pwa',               // Offline-first, performance critical
  MICROSERVICES = 'microservices',          // Distributed architecture
  REAL_TIME_APP = 'real-time-app'          // WebSocket-heavy applications
}

// Setup configuration per project type
export const ProjectSetupConfig = {
  [ProjectType.ENTERPRISE_APP]: {
    rxjsFeatures: ['operators', 'subjects', 'schedulers', 'testing'],
    stateManagement: 'NgRx',
    errorHandling: 'centralized',
    performance: 'balanced',
    testing: 'comprehensive'
  },
  
  [ProjectType.E_COMMERCE]: {
    rxjsFeatures: ['operators', 'ajax', 'performance-optimized'],
    stateManagement: 'Akita',
    errorHandling: 'user-friendly',
    performance: 'high-priority',
    testing: 'critical-paths'
  },
  
  [ProjectType.DASHBOARD]: {
    rxjsFeatures: ['operators', 'websocket', 'schedulers', 'subjects'],
    stateManagement: 'custom-reactive',
    errorHandling: 'resilient',
    performance: 'real-time',
    testing: 'stream-focused'
  }
};
```

## Project Architecture Setup

### Folder Structure

```typescript
// Recommended project structure for RxJS applications
export const RecommendedStructure = {
  src: {
    app: {
      core: {
        services: ['api.service.ts', 'error-handler.service.ts', 'state.service.ts'],
        interceptors: ['auth.interceptor.ts', 'error.interceptor.ts'],
        guards: ['auth.guard.ts', 'permission.guard.ts'],
        models: ['user.model.ts', 'api-response.model.ts']
      },
      
      shared: {
        components: ['loading.component.ts', 'error-display.component.ts'],
        directives: ['auto-unsubscribe.directive.ts'],
        pipes: ['async-error.pipe.ts'],
        operators: ['custom-operators/'],
        utils: ['rxjs-utils.ts', 'stream-helpers.ts']
      },
      
      features: {
        'feature-name': {
          components: [],
          services: ['feature.service.ts', 'feature-state.service.ts'],
          models: ['feature.model.ts'],
          guards: ['feature.guard.ts'],
          'feature.module.ts': null
        }
      },
      
      'rxjs-setup': {
        'operators.ts': 'Custom operator exports',
        'subjects.ts': 'Shared subjects and streams',
        'schedulers.ts': 'Scheduler configurations',
        'error-handling.ts': 'Error handling utilities'
      }
    }
  }
};
```

### Core Architecture Components

```typescript
// Core RxJS architecture setup
@Injectable({
  providedIn: 'root'
})
export class RxJSCoreService {
  // Global error subject for application-wide error handling
  private readonly globalError$ = new Subject<Error>();
  
  // Global loading state
  private readonly loadingState$ = new BehaviorSubject<boolean>(false);
  
  // Application state streams
  private readonly appState$ = new BehaviorSubject<AppState>(initialAppState);
  
  constructor() {
    this.setupErrorHandling();
    this.setupGlobalStreams();
  }
  
  // Public getters for global streams
  get errors$(): Observable<Error> {
    return this.globalError$.asObservable();
  }
  
  get loading$(): Observable<boolean> {
    return this.loadingState$.asObservable();
  }
  
  get state$(): Observable<AppState> {
    return this.appState$.asObservable();
  }
  
  private setupErrorHandling(): void {
    // Handle all unhandled errors
    this.globalError$.pipe(
      filter(error => error != null),
      tap(error => console.error('Global error:', error)),
      // Add error reporting service here
      debounceTime(100), // Prevent spam
      distinctUntilChanged()
    ).subscribe();
  }
  
  private setupGlobalStreams(): void {
    // Set up any global reactive streams
    this.appState$.pipe(
      distinctUntilChanged(),
      tap(state => this.persistState(state))
    ).subscribe();
  }
  
  // Helper methods
  emitError(error: Error): void {
    this.globalError$.next(error);
  }
  
  setLoading(loading: boolean): void {
    this.loadingState$.next(loading);
  }
  
  updateState(partialState: Partial<AppState>): void {
    const currentState = this.appState$.value;
    this.appState$.next({ ...currentState, ...partialState });
  }
  
  private persistState(state: AppState): void {
    // Persist critical state to localStorage
    try {
      localStorage.setItem('app-state', JSON.stringify(state));
    } catch (error) {
      console.warn('Failed to persist state:', error);
    }
  }
}

interface AppState {
  user: User | null;
  theme: 'light' | 'dark';
  language: string;
  preferences: UserPreferences;
}
```

## Development Environment Configuration

### TypeScript Configuration

```json
// tsconfig.json optimized for RxJS development
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ES2020",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "strict": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noImplicitOverride": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    
    // Path mapping for clean imports
    "baseUrl": "./src",
    "paths": {
      "@core/*": ["app/core/*"],
      "@shared/*": ["app/shared/*"],
      "@features/*": ["app/features/*"],
      "@rxjs-setup/*": ["app/rxjs-setup/*"],
      "@environments/*": ["environments/*"]
    },
    
    // RxJS-specific settings
    "moduleResolution": "node",
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  },
  
  "angularCompilerOptions": {
    "enableI18nLegacyMessageIdFormat": false,
    "strictInjectionParameters": true,
    "strictInputAccessModifiers": true,
    "strictTemplates": true
  }
}
```

### ESLint Configuration

```json
// .eslintrc.json for RxJS best practices
{
  "extends": [
    "@angular-eslint/recommended",
    "@angular-eslint/template/process-inline-templates",
    "plugin:rxjs/recommended"
  ],
  "rules": {
    // RxJS-specific rules
    "rxjs/no-async-subscribe": "error",
    "rxjs/no-create": "error",
    "rxjs/no-ignored-observable": "error",
    "rxjs/no-ignored-subscription": "error",
    "rxjs/no-nested-subscribe": "error",
    "rxjs/no-unbound-methods": "error",
    "rxjs/prefer-observer": "error",
    "rxjs/no-implicit-any-catch": "error",
    "rxjs/no-index": "error",
    "rxjs/no-internal": "error",
    "rxjs/no-subject-unsubscribe": "error",
    "rxjs/no-unsafe-takeuntil": "error",
    
    // Memory leak prevention
    "rxjs/no-ignored-notifier": "error",
    "rxjs/no-sharereplay": "warn",
    
    // Performance rules
    "rxjs/prefer-pipeable-operators": "error",
    "rxjs/no-redundant-notify": "error"
  }
}
```

### Development Tools Setup

```typescript
// Development utilities for RxJS debugging
export class RxJSDevTools {
  private static isDevelopment = !environment.production;
  
  // Enhanced tap operator for debugging
  static debug<T>(label: string) {
    return (source: Observable<T>) => {
      if (!this.isDevelopment) {
        return source;
      }
      
      return source.pipe(
        tap({
          next: (value) => console.log(`[${label}] Next:`, value),
          error: (error) => console.error(`[${label}] Error:`, error),
          complete: () => console.log(`[${label}] Complete`)
        })
      );
    };
  }
  
  // Stream performance monitoring
  static monitor<T>(streamName: string) {
    return (source: Observable<T>) => {
      if (!this.isDevelopment) {
        return source;
      }
      
      const startTime = performance.now();
      let emissionCount = 0;
      
      return source.pipe(
        tap(() => {
          emissionCount++;
          const currentTime = performance.now();
          const duration = currentTime - startTime;
          
          console.log(`[${streamName}] Emission #${emissionCount} at ${duration.toFixed(2)}ms`);
        }),
        finalize(() => {
          const endTime = performance.now();
          const totalDuration = endTime - startTime;
          console.log(`[${streamName}] Stream completed. Total: ${totalDuration.toFixed(2)}ms, Emissions: ${emissionCount}`);
        })
      );
    };
  }
  
  // Memory leak detection
  static trackSubscriptions() {
    if (!this.isDevelopment) {
      return;
    }
    
    const subscriptions = new Set<Subscription>();
    const originalSubscribe = Observable.prototype.subscribe;
    
    Observable.prototype.subscribe = function(this: Observable<any>, ...args: any[]) {
      const subscription = originalSubscribe.apply(this, args);
      subscriptions.add(subscription);
      
      const originalUnsubscribe = subscription.unsubscribe;
      subscription.unsubscribe = function() {
        subscriptions.delete(subscription);
        originalUnsubscribe.call(this);
      };
      
      return subscription;
    };
    
    // Log active subscriptions periodically
    interval(10000).subscribe(() => {
      console.log(`Active subscriptions: ${subscriptions.size}`);
    });
  }
}
```

## RxJS Integration Patterns

### Service Layer Pattern

```typescript
// Base service with common RxJS patterns
export abstract class BaseApiService {
  protected readonly http = inject(HttpClient);
  protected readonly errorHandler = inject(ErrorHandlerService);
  protected readonly loadingService = inject(LoadingService);
  
  // Generic HTTP method with error handling and loading states
  protected request<T>(
    request: Observable<T>,
    options: {
      showLoading?: boolean;
      errorMessage?: string;
      retryCount?: number;
    } = {}
  ): Observable<T> {
    const { showLoading = true, errorMessage = 'Request failed', retryCount = 3 } = options;
    
    return request.pipe(
      // Show loading indicator
      tap(() => showLoading && this.loadingService.setLoading(true)),
      
      // Retry logic
      retry({
        count: retryCount,
        delay: (error, retryIndex) => timer(Math.pow(2, retryIndex) * 1000)
      }),
      
      // Error handling
      catchError(error => {
        console.error(errorMessage, error);
        this.errorHandler.handleError(error, errorMessage);
        return throwError(() => error);
      }),
      
      // Hide loading indicator
      finalize(() => showLoading && this.loadingService.setLoading(false))
    );
  }
  
  // Cached HTTP requests
  protected cachedRequest<T>(
    key: string,
    request: Observable<T>,
    cacheTime: number = 300000 // 5 minutes
  ): Observable<T> {
    return this.cacheService.get(key).pipe(
      switchMap(cached => {
        if (cached && Date.now() - cached.timestamp < cacheTime) {
          return of(cached.data);
        }
        
        return request.pipe(
          tap(data => this.cacheService.set(key, { data, timestamp: Date.now() }))
        );
      })
    );
  }
}

// Example implementation
@Injectable({
  providedIn: 'root'
})
export class UserService extends BaseApiService {
  private readonly usersSubject = new BehaviorSubject<User[]>([]);
  
  users$ = this.usersSubject.asObservable();
  
  getUsers(): Observable<User[]> {
    return this.cachedRequest(
      'users',
      this.request(
        this.http.get<User[]>('/api/users'),
        { errorMessage: 'Failed to load users' }
      )
    ).pipe(
      tap(users => this.usersSubject.next(users))
    );
  }
  
  createUser(user: CreateUserRequest): Observable<User> {
    return this.request(
      this.http.post<User>('/api/users', user),
      { errorMessage: 'Failed to create user' }
    ).pipe(
      tap(newUser => {
        const currentUsers = this.usersSubject.value;
        this.usersSubject.next([...currentUsers, newUser]);
      })
    );
  }
}
```

### Component Integration Pattern

```typescript
// Base component with RxJS lifecycle management
export abstract class BaseComponent implements OnInit, OnDestroy {
  protected readonly destroy$ = new Subject<void>();
  protected readonly loadingService = inject(LoadingService);
  protected readonly errorService = inject(ErrorService);
  
  // Convenience method for auto-unsubscribing
  protected takeUntilDestroy<T>() {
    return takeUntil(this.destroy$);
  }
  
  ngOnInit(): void {
    this.setupSubscriptions();
    this.onInit();
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
  
  // Override in child components
  protected abstract onInit(): void;
  
  private setupSubscriptions(): void {
    // Global loading state
    this.loadingService.loading$.pipe(
      this.takeUntilDestroy()
    ).subscribe(loading => {
      // Handle loading state in UI
    });
    
    // Global error handling
    this.errorService.errors$.pipe(
      this.takeUntilDestroy()
    ).subscribe(error => {
      // Handle errors in UI
    });
  }
}

// Example feature component
@Component({
  selector: 'app-user-list',
  template: `
    <div class="user-list">
      <div *ngIf="loading$ | async" class="loading">Loading...</div>
      <div *ngIf="error$ | async as error" class="error">{{ error }}</div>
      
      <div class="users">
        <app-user-card 
          *ngFor="let user of users$ | async; trackBy: trackByUserId" 
          [user]="user"
          (edit)="editUser($event)"
          (delete)="deleteUser($event)">
        </app-user-card>
      </div>
      
      <button (click)="loadUsers()" [disabled]="loading$ | async">
        Refresh Users
      </button>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserListComponent extends BaseComponent {
  private readonly userService = inject(UserService);
  
  // Component streams
  users$ = this.userService.users$;
  loading$ = this.loadingService.loading$;
  error$ = this.errorService.errors$;
  
  // User actions
  private readonly userActions$ = new Subject<UserAction>();
  
  protected onInit(): void {
    // Load initial data
    this.loadUsers();
    
    // Handle user actions
    this.userActions$.pipe(
      switchMap(action => this.handleUserAction(action)),
      this.takeUntilDestroy()
    ).subscribe();
  }
  
  loadUsers(): void {
    this.userService.getUsers().pipe(
      this.takeUntilDestroy()
    ).subscribe();
  }
  
  editUser(user: User): void {
    this.userActions$.next({ type: 'EDIT', user });
  }
  
  deleteUser(user: User): void {
    this.userActions$.next({ type: 'DELETE', user });
  }
  
  trackByUserId(index: number, user: User): string {
    return user.id;
  }
  
  private handleUserAction(action: UserAction): Observable<any> {
    switch (action.type) {
      case 'EDIT':
        return this.openEditDialog(action.user);
      case 'DELETE':
        return this.confirmDelete(action.user);
      default:
        return EMPTY;
    }
  }
  
  private openEditDialog(user: User): Observable<any> {
    // Implementation for edit dialog
    return EMPTY;
  }
  
  private confirmDelete(user: User): Observable<any> {
    // Implementation for delete confirmation
    return EMPTY;
  }
}

interface UserAction {
  type: 'EDIT' | 'DELETE';
  user: User;
}
```

## State Management Setup

### Custom Reactive State Management

```typescript
// Generic reactive state management
export abstract class ReactiveState<T> {
  private readonly state$ = new BehaviorSubject<T>(this.getInitialState());
  private readonly loading$ = new BehaviorSubject<boolean>(false);
  private readonly error$ = new Subject<Error>();
  
  // Public getters
  get currentState(): T {
    return this.state$.value;
  }
  
  get state(): Observable<T> {
    return this.state$.asObservable().pipe(distinctUntilChanged());
  }
  
  get loading(): Observable<boolean> {
    return this.loading$.asObservable();
  }
  
  get error(): Observable<Error> {
    return this.error$.asObservable();
  }
  
  // State management methods
  protected setState(newState: T): void {
    this.state$.next(newState);
  }
  
  protected patchState(partialState: Partial<T>): void {
    this.setState({ ...this.currentState, ...partialState });
  }
  
  protected setLoading(loading: boolean): void {
    this.loading$.next(loading);
  }
  
  protected setError(error: Error): void {
    this.error$.next(error);
  }
  
  // Async operation wrapper
  protected executeAsync<R>(
    operation: Observable<R>,
    options: {
      updateState?: (result: R, currentState: T) => T;
      errorHandler?: (error: Error) => T;
    } = {}
  ): Observable<R> {
    this.setLoading(true);
    
    return operation.pipe(
      tap(result => {
        if (options.updateState) {
          this.setState(options.updateState(result, this.currentState));
        }
      }),
      catchError(error => {
        this.setError(error);
        if (options.errorHandler) {
          this.setState(options.errorHandler(error));
        }
        return throwError(() => error);
      }),
      finalize(() => this.setLoading(false))
    );
  }
  
  protected abstract getInitialState(): T;
}

// Example state implementation
interface TodoState {
  todos: Todo[];
  filter: 'all' | 'active' | 'completed';
  selectedTodo: Todo | null;
}

@Injectable({
  providedIn: 'root'
})
export class TodoStateService extends ReactiveState<TodoState> {
  private readonly http = inject(HttpClient);
  
  // Computed selectors
  filteredTodos$ = this.state.pipe(
    map(state => {
      switch (state.filter) {
        case 'active':
          return state.todos.filter(todo => !todo.completed);
        case 'completed':
          return state.todos.filter(todo => todo.completed);
        default:
          return state.todos;
      }
    })
  );
  
  activeTodoCount$ = this.state.pipe(
    map(state => state.todos.filter(todo => !todo.completed).length)
  );
  
  protected getInitialState(): TodoState {
    return {
      todos: [],
      filter: 'all',
      selectedTodo: null
    };
  }
  
  // Actions
  loadTodos(): Observable<Todo[]> {
    return this.executeAsync(
      this.http.get<Todo[]>('/api/todos'),
      {
        updateState: (todos, state) => ({ ...state, todos })
      }
    );
  }
  
  addTodo(title: string): Observable<Todo> {
    return this.executeAsync(
      this.http.post<Todo>('/api/todos', { title, completed: false }),
      {
        updateState: (newTodo, state) => ({
          ...state,
          todos: [...state.todos, newTodo]
        })
      }
    );
  }
  
  toggleTodo(id: string): Observable<Todo> {
    const todo = this.currentState.todos.find(t => t.id === id);
    if (!todo) return throwError(() => new Error('Todo not found'));
    
    return this.executeAsync(
      this.http.patch<Todo>(`/api/todos/${id}`, { completed: !todo.completed }),
      {
        updateState: (updatedTodo, state) => ({
          ...state,
          todos: state.todos.map(t => t.id === id ? updatedTodo : t)
        })
      }
    );
  }
  
  setFilter(filter: TodoState['filter']): void {
    this.patchState({ filter });
  }
  
  selectTodo(todo: Todo | null): void {
    this.patchState({ selectedTodo: todo });
  }
}

interface Todo {
  id: string;
  title: string;
  completed: boolean;
  createdAt: Date;
}
```

## Testing Infrastructure

### RxJS Testing Setup

```typescript
// Testing utilities for RxJS streams
export class RxJSTestUtils {
  static createMockObservable<T>(values: T[], error?: Error): Observable<T> {
    const scheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
    
    return scheduler.createColdObservable(
      error 
        ? `--a--b--#` 
        : `--a--b--c--|`,
      error 
        ? { a: values[0], b: values[1] }
        : { a: values[0], b: values[1], c: values[2] }
    ).pipe(
      map((value, index) => values[index])
    );
  }
  
  static expectStream<T>(
    source: Observable<T>,
    expectedMarble: string,
    expectedValues?: any,
    expectedError?: any
  ): void {
    const scheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
    
    scheduler.run(({ expectObservable }) => {
      expectObservable(source).toBe(expectedMarble, expectedValues, expectedError);
    });
  }
  
  static advanceTime(ms: number): void {
    jasmine.clock().tick(ms);
  }
}

// Example service test
describe('UserService', () => {
  let service: UserService;
  let httpMock: jasmine.SpyObj<HttpClient>;
  let scheduler: TestScheduler;
  
  beforeEach(() => {
    const httpSpy = jasmine.createSpyObj('HttpClient', ['get', 'post', 'put', 'delete']);
    
    TestBed.configureTestingModule({
      providers: [
        UserService,
        { provide: HttpClient, useValue: httpSpy }
      ]
    });
    
    service = TestBed.inject(UserService);
    httpMock = TestBed.inject(HttpClient) as jasmine.SpyObj<HttpClient>;
    
    scheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });
  
  it('should load users and update state', () => {
    const mockUsers: User[] = [
      { id: '1', name: 'John', email: 'john@example.com' },
      { id: '2', name: 'Jane', email: 'jane@example.com' }
    ];
    
    httpMock.get.and.returnValue(of(mockUsers));
    
    scheduler.run(({ expectObservable }) => {
      const result$ = service.getUsers();
      
      expectObservable(result$).toBe('(a|)', { a: mockUsers });
      expectObservable(service.users$).toBe('a', { a: mockUsers });
    });
  });
  
  it('should handle errors properly', () => {
    const error = new Error('Network error');
    httpMock.get.and.returnValue(throwError(() => error));
    
    scheduler.run(({ expectObservable }) => {
      const result$ = service.getUsers();
      
      expectObservable(result$).toBe('#', null, error);
    });
  });
  
  it('should cache requests', fakeAsync(() => {
    const mockUsers: User[] = [{ id: '1', name: 'John', email: 'john@example.com' }];
    httpMock.get.and.returnValue(of(mockUsers));
    
    // First request
    service.getUsers().subscribe();
    expect(httpMock.get).toHaveBeenCalledTimes(1);
    
    // Second request (should use cache)
    service.getUsers().subscribe();
    expect(httpMock.get).toHaveBeenCalledTimes(1);
    
    // Advance time beyond cache expiry
    tick(300001);
    
    // Third request (should make new HTTP call)
    service.getUsers().subscribe();
    expect(httpMock.get).toHaveBeenCalledTimes(2);
  }));
});
```

## Performance Monitoring

### Stream Performance Monitoring

```typescript
// Performance monitoring for RxJS streams
@Injectable({
  providedIn: 'root'
})
export class StreamPerformanceMonitorService {
  private readonly metrics = new Map<string, StreamMetrics>();
  
  // Monitor stream performance
  monitor<T>(streamName: string) {
    return (source: Observable<T>) => {
      if (!environment.production) {
        return this.monitorInDevelopment(streamName, source);
      }
      
      return this.monitorInProduction(streamName, source);
    };
  }
  
  private monitorInDevelopment<T>(streamName: string, source: Observable<T>): Observable<T> {
    const startTime = performance.now();
    let emissionCount = 0;
    let lastEmissionTime = startTime;
    
    return source.pipe(
      tap(value => {
        emissionCount++;
        const currentTime = performance.now();
        const timeSinceStart = currentTime - startTime;
        const timeSinceLastEmission = currentTime - lastEmissionTime;
        
        console.group(`🔍 Stream: ${streamName}`);
        console.log(`📊 Emission #${emissionCount}`);
        console.log(`⏱️ Time since start: ${timeSinceStart.toFixed(2)}ms`);
        console.log(`⏱️ Time since last: ${timeSinceLastEmission.toFixed(2)}ms`);
        console.log(`📦 Value:`, value);
        console.groupEnd();
        
        lastEmissionTime = currentTime;
      }),
      finalize(() => {
        const endTime = performance.now();
        const totalDuration = endTime - startTime;
        
        console.log(`✅ Stream ${streamName} completed:`);
        console.log(`📊 Total duration: ${totalDuration.toFixed(2)}ms`);
        console.log(`📊 Total emissions: ${emissionCount}`);
        console.log(`📊 Average time per emission: ${(totalDuration / emissionCount).toFixed(2)}ms`);
      })
    );
  }
  
  private monitorInProduction<T>(streamName: string, source: Observable<T>): Observable<T> {
    const startTime = performance.now();
    let emissionCount = 0;
    
    return source.pipe(
      tap(() => emissionCount++),
      finalize(() => {
        const endTime = performance.now();
        const duration = endTime - startTime;
        
        this.recordMetrics(streamName, {
          duration,
          emissionCount,
          averageEmissionTime: duration / emissionCount
        });
      })
    );
  }
  
  private recordMetrics(streamName: string, metrics: StreamMetrics): void {
    const existing = this.metrics.get(streamName);
    
    if (existing) {
      // Update rolling averages
      existing.totalDuration += metrics.duration;
      existing.totalEmissions += metrics.emissionCount;
      existing.executionCount++;
      existing.averageDuration = existing.totalDuration / existing.executionCount;
    } else {
      this.metrics.set(streamName, {
        ...metrics,
        totalDuration: metrics.duration,
        totalEmissions: metrics.emissionCount,
        executionCount: 1,
        averageDuration: metrics.duration
      });
    }
  }
  
  getMetrics(): Map<string, StreamMetrics> {
    return new Map(this.metrics);
  }
  
  getSlowStreams(threshold: number = 1000): string[] {
    return Array.from(this.metrics.entries())
      .filter(([_, metrics]) => metrics.averageDuration > threshold)
      .map(([name]) => name);
  }
}

interface StreamMetrics {
  duration: number;
  emissionCount: number;
  averageEmissionTime: number;
  totalDuration?: number;
  totalEmissions?: number;
  executionCount?: number;
  averageDuration?: number;
}
```

## Error Handling Strategy

### Centralized Error Handling

```typescript
// Centralized error handling service
@Injectable({
  providedIn: 'root'
})
export class ErrorHandlerService {
  private readonly errorSubject = new Subject<AppError>();
  private readonly notificationService = inject(NotificationService);
  
  errors$ = this.errorSubject.asObservable();
  
  constructor() {
    this.setupGlobalErrorHandling();
  }
  
  handleError(error: any, context?: string): void {
    const appError = this.transformError(error, context);
    
    // Log error
    console.error('Application Error:', appError);
    
    // Emit to subscribers
    this.errorSubject.next(appError);
    
    // Show user notification if needed
    if (appError.showToUser) {
      this.notificationService.showError(appError.userMessage);
    }
    
    // Report to external service in production
    if (environment.production) {
      this.reportError(appError);
    }
  }
  
  private transformError(error: any, context?: string): AppError {
    let appError: AppError;
    
    if (error instanceof HttpErrorResponse) {
      appError = this.handleHttpError(error, context);
    } else if (error instanceof TypeError) {
      appError = this.handleTypeError(error, context);
    } else if (error instanceof Error) {
      appError = this.handleGenericError(error, context);
    } else {
      appError = {
        id: this.generateErrorId(),
        type: ErrorType.UNKNOWN,
        message: 'An unknown error occurred',
        userMessage: 'Something went wrong. Please try again.',
        context,
        timestamp: new Date(),
        showToUser: true,
        originalError: error
      };
    }
    
    return appError;
  }
  
  private handleHttpError(error: HttpErrorResponse, context?: string): AppError {
    let userMessage: string;
    let type: ErrorType;
    
    switch (error.status) {
      case 400:
        type = ErrorType.VALIDATION;
        userMessage = 'Invalid request. Please check your input.';
        break;
      case 401:
        type = ErrorType.AUTHENTICATION;
        userMessage = 'Please log in to continue.';
        break;
      case 403:
        type = ErrorType.AUTHORIZATION;
        userMessage = 'You don\'t have permission to perform this action.';
        break;
      case 404:
        type = ErrorType.NOT_FOUND;
        userMessage = 'The requested resource was not found.';
        break;
      case 500:
        type = ErrorType.SERVER;
        userMessage = 'Server error. Please try again later.';
        break;
      default:
        type = ErrorType.NETWORK;
        userMessage = 'Network error. Please check your connection.';
    }
    
    return {
      id: this.generateErrorId(),
      type,
      message: error.message,
      userMessage,
      context,
      timestamp: new Date(),
      showToUser: true,
      originalError: error,
      statusCode: error.status
    };
  }
  
  private setupGlobalErrorHandling(): void {
    // Handle RxJS errors globally
    window.addEventListener('unhandledrejection', (event) => {
      this.handleError(event.reason, 'Unhandled Promise Rejection');
    });
    
    // Handle Angular errors
    window.addEventListener('error', (event) => {
      this.handleError(event.error, 'Global Error Handler');
    });
  }
  
  private generateErrorId(): string {
    return `error_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
  
  private reportError(error: AppError): void {
    // Implement error reporting to external service
    // Example: Sentry, LogRocket, etc.
  }
}

enum ErrorType {
  NETWORK = 'network',
  VALIDATION = 'validation',
  AUTHENTICATION = 'authentication',
  AUTHORIZATION = 'authorization',
  NOT_FOUND = 'not_found',
  SERVER = 'server',
  UNKNOWN = 'unknown'
}

interface AppError {
  id: string;
  type: ErrorType;
  message: string;
  userMessage: string;
  context?: string;
  timestamp: Date;
  showToUser: boolean;
  originalError: any;
  statusCode?: number;
}
```

## Code Organization

### Custom Operators Organization

```typescript
// Custom operators organized by category
export * from './transformation/index';
export * from './filtering/index';
export * from './utility/index';
export * from './error-handling/index';

// Example custom operators
// transformation/index.ts
export { mapToProperty } from './map-to-property';
export { transformData } from './transform-data';

// utility/index.ts
export { debug } from './debug';
export { loading } from './loading';
export { cache } from './cache';

// Example: mapToProperty operator
export function mapToProperty<T, K extends keyof T>(property: K) {
  return (source: Observable<T>): Observable<T[K]> => {
    return source.pipe(
      map(obj => obj[property])
    );
  };
}

// Example: debug operator
export function debug<T>(label: string, options: DebugOptions = {}) {
  return (source: Observable<T>): Observable<T> => {
    if (!environment.production || options.forceLog) {
      return source.pipe(
        tap({
          next: value => console.log(`[${label}] Next:`, value),
          error: error => console.error(`[${label}] Error:`, error),
          complete: () => console.log(`[${label}] Complete`)
        })
      );
    }
    return source;
  };
}

interface DebugOptions {
  forceLog?: boolean;
}
```

## Build and Deployment

### Production Build Configuration

```json
// angular.json production configuration
{
  "configurations": {
    "production": {
      "optimization": true,
      "outputHashing": "all",
      "sourceMap": false,
      "namedChunks": false,
      "extractLicenses": true,
      "vendorChunk": false,
      "buildOptimizer": true,
      "budgets": [
        {
          "type": "initial",
          "maximumWarning": "2mb",
          "maximumError": "5mb"
        },
        {
          "type": "anyComponentStyle",
          "maximumWarning": "6kb",
          "maximumError": "10kb"
        }
      ],
      "fileReplacements": [
        {
          "replace": "src/environments/environment.ts",
          "with": "src/environments/environment.prod.ts"
        }
      ]
    }
  }
}
```

### Environment Configuration

```typescript
// environments/environment.ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:3000/api',
  enableRxJSDebugging: true,
  enablePerformanceMonitoring: true,
  logLevel: 'debug'
};

// environments/environment.prod.ts
export const environment = {
  production: true,
  apiUrl: 'https://api.myapp.com',
  enableRxJSDebugging: false,
  enablePerformanceMonitoring: false,
  logLevel: 'error'
};
```

## Team Collaboration

### Code Standards and Guidelines

```typescript
// Team coding standards for RxJS
export const RxJSCodingStandards = {
  // Naming conventions
  naming: {
    observables: 'Use $ suffix for observable variables',
    subjects: 'Use descriptive names with Subject suffix',
    operators: 'Use camelCase for custom operators'
  },
  
  // Structure guidelines
  structure: {
    imports: 'Group RxJS imports at the top',
    operators: 'Chain operators vertically for readability',
    subscriptions: 'Always handle unsubscription'
  },
  
  // Performance guidelines
  performance: {
    shareReplay: 'Use shareReplay with bufferSize=1 for caching',
    distinctUntilChanged: 'Use to prevent unnecessary emissions',
    takeUntil: 'Use for component cleanup'
  }
};

// Code review checklist
export const CodeReviewChecklist = [
  '✅ All observables are properly unsubscribed',
  '✅ Error handling is implemented',
  '✅ Loading states are managed',
  '✅ No nested subscriptions',
  '✅ Operators are used efficiently',
  '✅ Custom operators are well-documented',
  '✅ Tests cover stream behavior',
  '✅ Performance impact is considered'
];
```

## Best Practices

### Summary of Best Practices

```typescript
// Comprehensive best practices for RxJS projects
export const RxJSBestPractices = {
  architecture: [
    'Use reactive state management patterns',
    'Implement centralized error handling',
    'Create reusable base classes and services',
    'Organize code by feature and responsibility'
  ],
  
  performance: [
    'Monitor stream performance in development',
    'Use lazy loading for non-critical features',
    'Implement proper caching strategies',
    'Optimize bundle size with tree shaking'
  ],
  
  maintenance: [
    'Write comprehensive tests for streams',
    'Document complex operator chains',
    'Use TypeScript for type safety',
    'Implement proper error boundaries'
  ],
  
  team: [
    'Establish coding standards and guidelines',
    'Use code review checklists',
    'Provide RxJS training for team members',
    'Share reusable operators and patterns'
  ]
};
```

## Summary

Setting up a real-world RxJS project requires careful planning across multiple dimensions:

1. **Architecture**: Establish clear patterns for services, components, and state management
2. **Development Environment**: Configure TypeScript, ESLint, and development tools for RxJS
3. **Testing**: Set up comprehensive testing infrastructure with marble testing
4. **Performance**: Implement monitoring and optimization strategies
5. **Error Handling**: Create centralized, user-friendly error management
6. **Team Collaboration**: Establish standards, guidelines, and review processes

By following these patterns and practices, teams can build scalable, maintainable RxJS applications that perform well and provide excellent developer experience.

---

**Next:** [Complex Data Flow Patterns](./02-data-flow-patterns.md)
**Previous:** [Browser Compatibility & Polyfills](../07-performance-optimization/05-browser-compatibility.md)
