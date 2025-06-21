# RxJS State Management Patterns

## Learning Objectives
- Understand state management principles with RxJS
- Master BehaviorSubject and ReplaySubject for state
- Implement reactive state patterns in Angular
- Build scalable state management solutions
- Handle state synchronization and updates
- Implement state persistence and hydration

## Introduction to RxJS State Management

State management is crucial for maintaining application data consistency and enabling reactive updates across components. RxJS provides powerful patterns for managing state without heavy external libraries.

### Core Principles
- **Single Source of Truth**: Centralized state management
- **Immutability**: State updates through pure functions
- **Reactivity**: Automatic UI updates on state changes
- **Predictability**: Clear state flow and updates

## BehaviorSubject for State Management

BehaviorSubject is ideal for state management as it holds the current value and emits it to new subscribers.

### Basic State Service

```typescript
// user-state.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';
import { map, distinctUntilChanged } from 'rxjs/operators';

export interface User {
  id: string;
  name: string;
  email: string;
  role: string;
}

export interface UserState {
  user: User | null;
  loading: boolean;
  error: string | null;
}

@Injectable({
  providedIn: 'root'
})
export class UserStateService {
  private readonly initialState: UserState = {
    user: null,
    loading: false,
    error: null
  };

  private readonly state$ = new BehaviorSubject<UserState>(this.initialState);

  // Public observable for state
  readonly userState$ = this.state$.asObservable();

  // Derived observables
  readonly user$ = this.userState$.pipe(
    map(state => state.user),
    distinctUntilChanged()
  );

  readonly loading$ = this.userState$.pipe(
    map(state => state.loading),
    distinctUntilChanged()
  );

  readonly error$ = this.userState$.pipe(
    map(state => state.error),
    distinctUntilChanged()
  );

  // State getters
  get currentState(): UserState {
    return this.state$.value;
  }

  get currentUser(): User | null {
    return this.currentState.user;
  }

  // State mutations
  setLoading(loading: boolean): void {
    this.updateState({ loading });
  }

  setUser(user: User): void {
    this.updateState({ user, loading: false, error: null });
  }

  setError(error: string): void {
    this.updateState({ error, loading: false });
  }

  clearUser(): void {
    this.updateState({ user: null, error: null });
  }

  private updateState(partial: Partial<UserState>): void {
    this.state$.next({
      ...this.currentState,
      ...partial
    });
  }
}
```

### Component Integration

```typescript
// user-profile.component.ts
import { Component, OnInit } from '@angular/core';
import { UserStateService } from './user-state.service';
import { UserService } from './user.service';

@Component({
  selector: 'app-user-profile',
  template: `
    <div class="user-profile">
      <div *ngIf="userState.loading$ | async" class="loading">
        Loading user...
      </div>

      <div *ngIf="userState.error$ | async as error" class="error">
        {{ error }}
      </div>

      <div *ngIf="userState.user$ | async as user" class="user-info">
        <h2>{{ user.name }}</h2>
        <p>{{ user.email }}</p>
        <span class="role">{{ user.role }}</span>
      </div>

      <button (click)="loadUser()" 
              [disabled]="userState.loading$ | async">
        Load User
      </button>
    </div>
  `
})
export class UserProfileComponent implements OnInit {
  constructor(
    public userState: UserStateService,
    private userService: UserService
  ) {}

  ngOnInit() {
    this.loadUser();
  }

  loadUser(): void {
    this.userState.setLoading(true);
    
    this.userService.getCurrentUser().subscribe({
      next: user => this.userState.setUser(user),
      error: err => this.userState.setError(err.message)
    });
  }
}
```

## Advanced State Patterns

### Store Pattern with Actions

```typescript
// store.interface.ts
export interface Action {
  type: string;
  payload?: any;
}

export interface Store<T> {
  state$: Observable<T>;
  dispatch(action: Action): void;
  select<K>(selector: (state: T) => K): Observable<K>;
}

// base-store.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';
import { map, distinctUntilChanged } from 'rxjs/operators';

export abstract class BaseStore<T> implements Store<T> {
  protected state$ = new BehaviorSubject<T>(this.initialState);

  abstract get initialState(): T;
  abstract reducer(state: T, action: Action): T;

  get state(): Observable<T> {
    return this.state$.asObservable();
  }

  get currentState(): T {
    return this.state$.value;
  }

  dispatch(action: Action): void {
    const currentState = this.currentState;
    const newState = this.reducer(currentState, action);
    this.state$.next(newState);
  }

  select<K>(selector: (state: T) => K): Observable<K> {
    return this.state$.pipe(
      map(selector),
      distinctUntilChanged()
    );
  }
}
```

### Todo Store Implementation

```typescript
// todo-store.service.ts
export interface Todo {
  id: string;
  text: string;
  completed: boolean;
  createdAt: Date;
}

export interface TodoState {
  todos: Todo[];
  filter: 'all' | 'active' | 'completed';
  loading: boolean;
}

export enum TodoActionTypes {
  ADD_TODO = '[Todo] Add Todo',
  TOGGLE_TODO = '[Todo] Toggle Todo',
  DELETE_TODO = '[Todo] Delete Todo',
  SET_FILTER = '[Todo] Set Filter',
  LOAD_TODOS = '[Todo] Load Todos',
  LOAD_TODOS_SUCCESS = '[Todo] Load Todos Success'
}

@Injectable({
  providedIn: 'root'
})
export class TodoStore extends BaseStore<TodoState> {
  
  get initialState(): TodoState {
    return {
      todos: [],
      filter: 'all',
      loading: false
    };
  }

  // Selectors
  readonly todos$ = this.select(state => state.todos);
  readonly filter$ = this.select(state => state.filter);
  readonly loading$ = this.select(state => state.loading);

  readonly visibleTodos$ = this.select(state => {
    const { todos, filter } = state;
    switch (filter) {
      case 'active':
        return todos.filter(todo => !todo.completed);
      case 'completed':
        return todos.filter(todo => todo.completed);
      default:
        return todos;
    }
  });

  readonly todoCount$ = this.select(state => 
    state.todos.filter(todo => !todo.completed).length
  );

  reducer(state: TodoState, action: Action): TodoState {
    switch (action.type) {
      case TodoActionTypes.ADD_TODO:
        return {
          ...state,
          todos: [...state.todos, action.payload]
        };

      case TodoActionTypes.TOGGLE_TODO:
        return {
          ...state,
          todos: state.todos.map(todo =>
            todo.id === action.payload
              ? { ...todo, completed: !todo.completed }
              : todo
          )
        };

      case TodoActionTypes.DELETE_TODO:
        return {
          ...state,
          todos: state.todos.filter(todo => todo.id !== action.payload)
        };

      case TodoActionTypes.SET_FILTER:
        return {
          ...state,
          filter: action.payload
        };

      case TodoActionTypes.LOAD_TODOS:
        return {
          ...state,
          loading: true
        };

      case TodoActionTypes.LOAD_TODOS_SUCCESS:
        return {
          ...state,
          todos: action.payload,
          loading: false
        };

      default:
        return state;
    }
  }

  // Action creators
  addTodo(text: string): void {
    const todo: Todo = {
      id: Date.now().toString(),
      text,
      completed: false,
      createdAt: new Date()
    };
    this.dispatch({ type: TodoActionTypes.ADD_TODO, payload: todo });
  }

  toggleTodo(id: string): void {
    this.dispatch({ type: TodoActionTypes.TOGGLE_TODO, payload: id });
  }

  deleteTodo(id: string): void {
    this.dispatch({ type: TodoActionTypes.DELETE_TODO, payload: id });
  }

  setFilter(filter: 'all' | 'active' | 'completed'): void {
    this.dispatch({ type: TodoActionTypes.SET_FILTER, payload: filter });
  }
}
```

## Entity State Management

For complex data structures, implement entity-based state management:

```typescript
// entity-store.ts
export interface EntityState<T> {
  ids: string[];
  entities: { [id: string]: T };
  loading: boolean;
  error: string | null;
}

export abstract class EntityStore<T extends { id: string }> extends BaseStore<EntityState<T>> {
  
  get initialState(): EntityState<T> {
    return {
      ids: [],
      entities: {},
      loading: false,
      error: null
    };
  }

  // Selectors
  readonly allEntities$ = this.select(state => 
    state.ids.map(id => state.entities[id])
  );

  readonly entitiesCount$ = this.select(state => state.ids.length);

  selectEntity(id: string): Observable<T | undefined> {
    return this.select(state => state.entities[id]);
  }

  // Actions
  setEntities(entities: T[]): void {
    const normalized = this.normalizeEntities(entities);
    this.dispatch({
      type: 'SET_ENTITIES',
      payload: normalized
    });
  }

  addEntity(entity: T): void {
    this.dispatch({
      type: 'ADD_ENTITY',
      payload: entity
    });
  }

  updateEntity(id: string, changes: Partial<T>): void {
    this.dispatch({
      type: 'UPDATE_ENTITY',
      payload: { id, changes }
    });
  }

  deleteEntity(id: string): void {
    this.dispatch({
      type: 'DELETE_ENTITY',
      payload: id
    });
  }

  private normalizeEntities(entities: T[]): { ids: string[], entities: { [id: string]: T } } {
    return entities.reduce(
      (acc, entity) => ({
        ids: [...acc.ids, entity.id],
        entities: { ...acc.entities, [entity.id]: entity }
      }),
      { ids: [], entities: {} }
    );
  }

  reducer(state: EntityState<T>, action: Action): EntityState<T> {
    switch (action.type) {
      case 'SET_ENTITIES':
        return {
          ...state,
          ...action.payload,
          loading: false,
          error: null
        };

      case 'ADD_ENTITY':
        const entity = action.payload;
        return {
          ...state,
          ids: [...state.ids, entity.id],
          entities: { ...state.entities, [entity.id]: entity }
        };

      case 'UPDATE_ENTITY':
        const { id, changes } = action.payload;
        return {
          ...state,
          entities: {
            ...state.entities,
            [id]: { ...state.entities[id], ...changes }
          }
        };

      case 'DELETE_ENTITY':
        const idToDelete = action.payload;
        const { [idToDelete]: deleted, ...remainingEntities } = state.entities;
        return {
          ...state,
          ids: state.ids.filter(id => id !== idToDelete),
          entities: remainingEntities
        };

      default:
        return state;
    }
  }
}
```

## State Synchronization Patterns

### Cross-Component Communication

```typescript
// notification-state.service.ts
export interface Notification {
  id: string;
  message: string;
  type: 'success' | 'error' | 'warning' | 'info';
  duration?: number;
}

@Injectable({
  providedIn: 'root'
})
export class NotificationStateService {
  private notifications$ = new BehaviorSubject<Notification[]>([]);

  readonly notifications = this.notifications$.asObservable();

  addNotification(notification: Omit<Notification, 'id'>): void {
    const newNotification: Notification = {
      ...notification,
      id: Date.now().toString()
    };

    const current = this.notifications$.value;
    this.notifications$.next([...current, newNotification]);

    // Auto-remove after duration
    if (notification.duration) {
      setTimeout(() => {
        this.removeNotification(newNotification.id);
      }, notification.duration);
    }
  }

  removeNotification(id: string): void {
    const current = this.notifications$.value;
    this.notifications$.next(current.filter(n => n.id !== id));
  }

  clearAll(): void {
    this.notifications$.next([]);
  }
}

// Global error handler
@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  constructor(private notificationState: NotificationStateService) {}

  handleError(error: any): void {
    console.error(error);
    this.notificationState.addNotification({
      message: 'An unexpected error occurred',
      type: 'error',
      duration: 5000
    });
  }
}
```

### State Persistence

```typescript
// persistent-store.ts
export abstract class PersistentStore<T> extends BaseStore<T> {
  constructor(private storageKey: string) {
    super();
    this.loadFromStorage();
    this.setupPersistence();
  }

  private loadFromStorage(): void {
    try {
      const stored = localStorage.getItem(this.storageKey);
      if (stored) {
        const state = JSON.parse(stored);
        this.state$.next({ ...this.initialState, ...state });
      }
    } catch (error) {
      console.warn('Failed to load state from storage:', error);
    }
  }

  private setupPersistence(): void {
    this.state$.pipe(
      debounceTime(1000), // Avoid too frequent saves
      distinctUntilChanged()
    ).subscribe(state => {
      try {
        localStorage.setItem(this.storageKey, JSON.stringify(state));
      } catch (error) {
        console.warn('Failed to persist state:', error);
      }
    });
  }

  clearStorage(): void {
    localStorage.removeItem(this.storageKey);
  }
}

// Usage
@Injectable({
  providedIn: 'root'
})
export class UserPreferencesStore extends PersistentStore<UserPreferences> {
  constructor() {
    super('user-preferences');
  }

  get initialState(): UserPreferences {
    return {
      theme: 'light',
      language: 'en',
      notifications: true
    };
  }

  // ... reducer and actions
}
```

## State Testing Strategies

### Testing State Services

```typescript
// user-state.service.spec.ts
describe('UserStateService', () => {
  let service: UserStateService;

  beforeEach(() => {
    TestBed.configureTestingModule({});
    service = TestBed.inject(UserStateService);
  });

  it('should initialize with empty state', () => {
    expect(service.currentUser).toBeNull();
    expect(service.currentState.loading).toBeFalse();
    expect(service.currentState.error).toBeNull();
  });

  it('should set loading state', (done) => {
    service.setLoading(true);
    
    service.loading$.subscribe(loading => {
      expect(loading).toBeTrue();
      done();
    });
  });

  it('should set user and clear loading/error', (done) => {
    const user: User = {
      id: '1',
      name: 'John Doe',
      email: 'john@example.com',
      role: 'user'
    };

    service.setUser(user);

    service.userState$.subscribe(state => {
      expect(state.user).toEqual(user);
      expect(state.loading).toBeFalse();
      expect(state.error).toBeNull();
      done();
    });
  });

  it('should handle errors correctly', (done) => {
    const errorMessage = 'Something went wrong';
    service.setError(errorMessage);

    service.error$.subscribe(error => {
      expect(error).toBe(errorMessage);
      done();
    });
  });
});
```

### Testing with Marble Diagrams

```typescript
// todo-store.service.spec.ts
import { TestScheduler } from 'rxjs/testing';

describe('TodoStore', () => {
  let testScheduler: TestScheduler;
  let store: TodoStore;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
    
    TestBed.configureTestingModule({});
    store = TestBed.inject(TodoStore);
  });

  it('should add todos over time', () => {
    testScheduler.run(({ cold, expectObservable }) => {
      // Actions timeline: add todo at frame 10 and 20
      cold('----------a---------b').subscribe(frame => {
        if (frame === 'a') {
          store.addTodo('First todo');
        } else if (frame === 'b') {
          store.addTodo('Second todo');
        }
      });

      // Expected state changes
      const expected = 'a---------b---------c';
      const values = {
        a: [], // Initial empty state
        b: [jasmine.objectContaining({ text: 'First todo' })],
        c: [
          jasmine.objectContaining({ text: 'First todo' }),
          jasmine.objectContaining({ text: 'Second todo' })
        ]
      };

      expectObservable(store.todos$).toBe(expected, values);
    });
  });
});
```

## Performance Optimization

### Memoized Selectors

```typescript
// memoized-selectors.ts
import { memoize } from 'lodash-es';

export class MemoizedSelectors {
  // Expensive computed property
  static readonly expensiveSelector = memoize(
    (state: AppState) => {
      // Complex calculation
      return state.items
        .filter(item => item.active)
        .map(item => ({
          ...item,
          computed: this.expensiveComputation(item)
        }))
        .sort((a, b) => a.priority - b.priority);
    },
    // Custom hash function for cache key
    (state: AppState) => `${state.items.length}-${state.lastModified}`
  );

  private static expensiveComputation(item: any): any {
    // Simulate expensive operation
    return item;
  }
}

// Usage in store
export class AppStore extends BaseStore<AppState> {
  readonly expensiveData$ = this.select(
    state => MemoizedSelectors.expensiveSelector(state)
  );
}
```

### OnPush Change Detection

```typescript
// optimized.component.ts
@Component({
  selector: 'app-optimized',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="todos">
      <div *ngFor="let todo of todos$ | async; trackBy: trackByTodoId"
           class="todo-item">
        {{ todo.text }}
      </div>
    </div>
  `
})
export class OptimizedComponent {
  readonly todos$ = this.todoStore.visibleTodos$;

  constructor(private todoStore: TodoStore) {}

  trackByTodoId(index: number, todo: Todo): string {
    return todo.id;
  }
}
```

## Best Practices

### 1. State Shape Design
- Keep state flat and normalized
- Separate loading states from data
- Use consistent naming conventions
- Implement proper TypeScript types

### 2. Action Patterns
- Use descriptive action types
- Include necessary payload data
- Implement action creators
- Consider action effects for side effects

### 3. Selector Optimization
- Use memoized selectors for expensive computations
- Create specific selectors for components
- Combine selectors using RxJS operators
- Avoid deep object selections

### 4. Error Handling
- Implement global error states
- Provide fallback values
- Use proper error boundaries
- Log errors appropriately

### 5. Testing Strategy
- Test state transitions
- Mock external dependencies
- Use marble testing for complex flows
- Test component integration

## Common Pitfalls

### 1. Memory Leaks
```typescript
// ❌ Bad: Not unsubscribing
export class BadComponent {
  ngOnInit() {
    this.store.state$.subscribe(state => {
      // Handle state
    }); // Memory leak!
  }
}

// ✅ Good: Proper cleanup
export class GoodComponent implements OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit() {
    this.store.state$
      .pipe(takeUntil(this.destroy$))
      .subscribe(state => {
        // Handle state
      });
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### 2. State Mutations
```typescript
// ❌ Bad: Mutating state directly
updateUser(changes: Partial<User>) {
  this.currentState.user = { ...this.currentState.user, ...changes }; // Mutation!
}

// ✅ Good: Immutable updates
updateUser(changes: Partial<User>) {
  this.updateState({
    user: { ...this.currentState.user, ...changes }
  });
}
```

## Exercises

### Exercise 1: Shopping Cart State
Create a shopping cart state service with the following features:
- Add/remove items
- Update quantities
- Calculate totals
- Apply discounts
- Persist to localStorage

### Exercise 2: Multi-Step Form State
Build a multi-step form state manager:
- Track current step
- Validate step data
- Store form progress
- Handle navigation
- Implement auto-save

### Exercise 3: Real-time Chat State
Implement a chat application state:
- Manage chat rooms
- Handle messages
- Track user presence
- Implement message search
- Handle offline/online states

## Summary

RxJS provides powerful patterns for state management in Angular applications. Key takeaways:

- Use BehaviorSubject for holding current state
- Implement proper state shapes with TypeScript
- Create derived observables for computed properties
- Use proper error handling and loading states
- Implement testing strategies with marble diagrams
- Optimize performance with memoization and OnPush
- Follow immutability principles for predictable state updates
- Clean up subscriptions to prevent memory leaks

These patterns provide a solid foundation for managing application state without heavy external dependencies, while maintaining the reactive benefits of RxJS.
