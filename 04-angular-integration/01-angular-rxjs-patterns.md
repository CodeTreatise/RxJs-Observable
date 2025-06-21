# Angular RxJS Patterns

## Overview

Angular and RxJS are deeply integrated, providing powerful reactive programming patterns for building modern web applications. This lesson covers the most common and effective patterns for using RxJS in Angular applications, from basic component interactions to complex data flows.

## Core Angular RxJS Integration Points

### 1. Angular Services with Observables
### 2. Component Communication
### 3. Template Integration with AsyncPipe
### 4. Lifecycle Management
### 5. Event Handling Patterns

## Essential Angular RxJS Patterns

### 1. Service Data Streams

The foundation of reactive Angular applications is services that expose data as Observable streams.

**Basic Service Pattern:**
```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { BehaviorSubject, Observable, throwError } from 'rxjs';
import { catchError, map, shareReplay } from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class UserService {
  private usersSubject = new BehaviorSubject<User[]>([]);
  public users$ = this.usersSubject.asObservable();

  private selectedUserSubject = new BehaviorSubject<User | null>(null);
  public selectedUser$ = this.selectedUserSubject.asObservable();

  constructor(private http: HttpClient) {
    this.loadUsers();
  }

  // Load all users
  loadUsers(): void {
    this.http.get<User[]>('/api/users').pipe(
      catchError(error => {
        console.error('Failed to load users:', error);
        return throwError(() => error);
      })
    ).subscribe(users => {
      this.usersSubject.next(users);
    });
  }

  // Select a user
  selectUser(user: User): void {
    this.selectedUserSubject.next(user);
  }

  // Get user by ID with caching
  getUserById(id: number): Observable<User> {
    return this.http.get<User>(`/api/users/${id}`).pipe(
      shareReplay(1),
      catchError(error => throwError(() => error))
    );
  }

  // Add new user
  addUser(user: Partial<User>): Observable<User> {
    return this.http.post<User>('/api/users', user).pipe(
      map(newUser => {
        const currentUsers = this.usersSubject.value;
        this.usersSubject.next([...currentUsers, newUser]);
        return newUser;
      }),
      catchError(error => throwError(() => error))
    );
  }

  // Update user
  updateUser(id: number, updates: Partial<User>): Observable<User> {
    return this.http.put<User>(`/api/users/${id}`, updates).pipe(
      map(updatedUser => {
        const currentUsers = this.usersSubject.value;
        const index = currentUsers.findIndex(u => u.id === id);
        if (index !== -1) {
          const newUsers = [...currentUsers];
          newUsers[index] = updatedUser;
          this.usersSubject.next(newUsers);
        }
        return updatedUser;
      }),
      catchError(error => throwError(() => error))
    );
  }
}

interface User {
  id: number;
  name: string;
  email: string;
  role: string;
}
```

### 2. Smart Component Pattern

Smart components manage state and delegate presentation to dumb components.

**Smart Component:**
```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Observable, Subject, combineLatest } from 'rxjs';
import { map, takeUntil, startWith } from 'rxjs/operators';

@Component({
  selector: 'app-user-dashboard',
  template: `
    <div class="user-dashboard">
      <!-- Search and filters -->
      <app-user-filters
        [filters]="filters$ | async"
        (filtersChange)="onFiltersChange($event)">
      </app-user-filters>

      <!-- User list -->
      <app-user-list
        [users]="filteredUsers$ | async"
        [loading]="loading$ | async"
        [selectedUser]="selectedUser$ | async"
        (userSelect)="onUserSelect($event)"
        (userDelete)="onUserDelete($event)">
      </app-user-list>

      <!-- User details -->
      <app-user-details
        [user]="selectedUser$ | async"
        (userUpdate)="onUserUpdate($event)">
      </app-user-details>
    </div>
  `,
  styleUrls: ['./user-dashboard.component.scss']
})
export class UserDashboardComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  private filtersSubject = new Subject<UserFilters>();

  // Observable streams
  users$ = this.userService.users$;
  selectedUser$ = this.userService.selectedUser$;
  loading$ = new BehaviorSubject(false);
  filters$ = this.filtersSubject.pipe(
    startWith({ search: '', role: 'all' })
  );

  // Derived streams
  filteredUsers$ = combineLatest([this.users$, this.filters$]).pipe(
    map(([users, filters]) => this.filterUsers(users, filters))
  );

  constructor(private userService: UserService) {}

  ngOnInit() {
    // Component initialization logic
    this.loadInitialData();
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }

  private loadInitialData(): void {
    this.loading$.next(true);
    this.userService.loadUsers();
    
    // Simulate loading completion
    this.users$.pipe(
      takeUntil(this.destroy$)
    ).subscribe(() => {
      this.loading$.next(false);
    });
  }

  onFiltersChange(filters: UserFilters): void {
    this.filtersSubject.next(filters);
  }

  onUserSelect(user: User): void {
    this.userService.selectUser(user);
  }

  onUserDelete(user: User): void {
    this.userService.deleteUser(user.id).pipe(
      takeUntil(this.destroy$)
    ).subscribe({
      next: () => console.log('User deleted successfully'),
      error: (error) => console.error('Failed to delete user:', error)
    });
  }

  onUserUpdate(user: User): void {
    this.userService.updateUser(user.id, user).pipe(
      takeUntil(this.destroy$)
    ).subscribe({
      next: () => console.log('User updated successfully'),
      error: (error) => console.error('Failed to update user:', error)
    });
  }

  private filterUsers(users: User[], filters: UserFilters): User[] {
    return users.filter(user => {
      const matchesSearch = !filters.search || 
        user.name.toLowerCase().includes(filters.search.toLowerCase()) ||
        user.email.toLowerCase().includes(filters.search.toLowerCase());
      
      const matchesRole = filters.role === 'all' || user.role === filters.role;
      
      return matchesSearch && matchesRole;
    });
  }
}

interface UserFilters {
  search: string;
  role: string;
}
```

### 3. Component Communication Patterns

**Parent-Child Communication:**
```typescript
// Parent Component
@Component({
  selector: 'app-parent',
  template: `
    <app-child
      [data]="parentData$ | async"
      (dataChange)="onChildDataChange($event)">
    </app-child>
  `
})
export class ParentComponent {
  parentData$ = new BehaviorSubject<string>('initial data');

  onChildDataChange(newData: string): void {
    this.parentData$.next(newData);
  }
}

// Child Component
@Component({
  selector: 'app-child',
  template: `
    <div>
      <p>Data: {{ data }}</p>
      <button (click)="updateData()">Update Data</button>
    </div>
  `
})
export class ChildComponent {
  @Input() data: string | null = null;
  @Output() dataChange = new EventEmitter<string>();

  updateData(): void {
    this.dataChange.emit(`Updated at ${new Date().toISOString()}`);
  }
}
```

**Sibling Communication via Service:**
```typescript
@Injectable({
  providedIn: 'root'
})
export class CommunicationService {
  private messageSubject = new Subject<string>();
  public message$ = this.messageSubject.asObservable();

  sendMessage(message: string): void {
    this.messageSubject.next(message);
  }
}

// Sender Component
@Component({
  selector: 'app-sender',
  template: `
    <button (click)="sendMessage()">Send Message</button>
  `
})
export class SenderComponent {
  constructor(private communicationService: CommunicationService) {}

  sendMessage(): void {
    this.communicationService.sendMessage('Hello from sender!');
  }
}

// Receiver Component
@Component({
  selector: 'app-receiver',
  template: `
    <p>Received: {{ message$ | async }}</p>
  `
})
export class ReceiverComponent implements OnDestroy {
  private destroy$ = new Subject<void>();
  message$ = this.communicationService.message$;

  constructor(private communicationService: CommunicationService) {}

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### 4. State Management Pattern

**Simple State Management with BehaviorSubject:**
```typescript
@Injectable({
  providedIn: 'root'
})
export class AppStateService {
  // Private state
  private stateSubject = new BehaviorSubject<AppState>(initialState);
  
  // Public state observable
  public state$ = this.stateSubject.asObservable();

  // Derived state selectors
  public user$ = this.state$.pipe(
    map(state => state.user),
    distinctUntilChanged()
  );

  public isLoading$ = this.state$.pipe(
    map(state => state.loading),
    distinctUntilChanged()
  );

  public notifications$ = this.state$.pipe(
    map(state => state.notifications),
    distinctUntilChanged()
  );

  // State update methods
  updateUser(user: User): void {
    this.updateState({ user });
  }

  setLoading(loading: boolean): void {
    this.updateState({ loading });
  }

  addNotification(notification: Notification): void {
    const currentState = this.stateSubject.value;
    const notifications = [...currentState.notifications, notification];
    this.updateState({ notifications });
  }

  removeNotification(id: string): void {
    const currentState = this.stateSubject.value;
    const notifications = currentState.notifications.filter(n => n.id !== id);
    this.updateState({ notifications });
  }

  private updateState(partialState: Partial<AppState>): void {
    const currentState = this.stateSubject.value;
    const newState = { ...currentState, ...partialState };
    this.stateSubject.next(newState);
  }

  // Get current state snapshot
  getCurrentState(): AppState {
    return this.stateSubject.value;
  }
}

interface AppState {
  user: User | null;
  loading: boolean;
  notifications: Notification[];
}

const initialState: AppState = {
  user: null,
  loading: false,
  notifications: []
};
```

### 5. Error Handling Pattern

**Global Error Handling:**
```typescript
@Injectable({
  providedIn: 'root'
})
export class ErrorHandlingService {
  private errorSubject = new Subject<AppError>();
  public errors$ = this.errorSubject.asObservable();

  handleError(error: any, context?: string): Observable<never> {
    const appError: AppError = {
      id: this.generateId(),
      message: this.getErrorMessage(error),
      context,
      timestamp: new Date(),
      severity: this.getErrorSeverity(error)
    };

    this.errorSubject.next(appError);
    return throwError(() => appError);
  }

  private getErrorMessage(error: any): string {
    if (error?.error?.message) return error.error.message;
    if (error?.message) return error.message;
    return 'An unexpected error occurred';
  }

  private getErrorSeverity(error: any): 'low' | 'medium' | 'high' {
    if (error?.status >= 500) return 'high';
    if (error?.status >= 400) return 'medium';
    return 'low';
  }

  private generateId(): string {
    return Math.random().toString(36).substr(2, 9);
  }
}

// Usage in service
@Injectable({
  providedIn: 'root'
})
export class DataService {
  constructor(
    private http: HttpClient,
    private errorHandler: ErrorHandlingService
  ) {}

  getData(): Observable<any[]> {
    return this.http.get<any[]>('/api/data').pipe(
      catchError(error => this.errorHandler.handleError(error, 'DataService.getData'))
    );
  }
}
```

### 6. Loading States Pattern

**Loading State Management:**
```typescript
@Injectable({
  providedIn: 'root'
})
export class LoadingService {
  private loadingSubjects = new Map<string, BehaviorSubject<boolean>>();

  // Set loading state for a specific operation
  setLoading(key: string, loading: boolean): void {
    if (!this.loadingSubjects.has(key)) {
      this.loadingSubjects.set(key, new BehaviorSubject<boolean>(false));
    }
    this.loadingSubjects.get(key)!.next(loading);
  }

  // Get loading state for a specific operation
  getLoading(key: string): Observable<boolean> {
    if (!this.loadingSubjects.has(key)) {
      this.loadingSubjects.set(key, new BehaviorSubject<boolean>(false));
    }
    return this.loadingSubjects.get(key)!.asObservable();
  }

  // Check if any operation is loading
  get isAnyLoading$(): Observable<boolean> {
    return combineLatest(
      Array.from(this.loadingSubjects.values())
    ).pipe(
      map(loadingStates => loadingStates.some(loading => loading)),
      startWith(false)
    );
  }

  // Operator to automatically manage loading state
  withLoading<T>(key: string) {
    return (source: Observable<T>) => {
      return new Observable<T>(subscriber => {
        this.setLoading(key, true);
        
        const subscription = source.subscribe({
          next: value => subscriber.next(value),
          error: error => {
            this.setLoading(key, false);
            subscriber.error(error);
          },
          complete: () => {
            this.setLoading(key, false);
            subscriber.complete();
          }
        });

        return () => {
          this.setLoading(key, false);
          subscription.unsubscribe();
        };
      });
    };
  }
}

// Usage in component
@Component({
  selector: 'app-data-component',
  template: `
    <div>
      <div *ngIf="isLoading$ | async" class="loading-spinner">
        Loading...
      </div>
      <div *ngFor="let item of data$ | async">
        {{ item.name }}
      </div>
    </div>
  `
})
export class DataComponent implements OnInit {
  data$: Observable<any[]>;
  isLoading$ = this.loadingService.getLoading('data-fetch');

  constructor(
    private dataService: DataService,
    private loadingService: LoadingService
  ) {}

  ngOnInit() {
    this.data$ = this.dataService.getData().pipe(
      this.loadingService.withLoading('data-fetch')
    );
  }
}
```

## Advanced Patterns

### 1. Reactive Forms Integration

```typescript
@Component({
  selector: 'app-reactive-form',
  template: `
    <form [formGroup]="userForm">
      <input formControlName="email" placeholder="Email">
      <input formControlName="name" placeholder="Name">
      
      <div *ngIf="emailValidationMessage$ | async as message" class="error">
        {{ message }}
      </div>
      
      <button 
        [disabled]="!userForm.valid || (isSubmitting$ | async)"
        (click)="onSubmit()">
        {{ (isSubmitting$ | async) ? 'Submitting...' : 'Submit' }}
      </button>
    </form>
  `
})
export class ReactiveFormComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  private submitSubject = new Subject<void>();
  
  userForm = this.fb.group({
    email: ['', [Validators.required, Validators.email]],
    name: ['', [Validators.required, Validators.minLength(2)]]
  });

  // Form state observables
  emailValidationMessage$ = this.userForm.get('email')!.statusChanges.pipe(
    startWith(this.userForm.get('email')!.status),
    map(() => this.getEmailValidationMessage()),
    distinctUntilChanged()
  );

  isSubmitting$ = this.submitSubject.pipe(
    switchMap(() => this.userService.createUser(this.userForm.value).pipe(
      map(() => false),
      startWith(true),
      catchError(error => {
        console.error('Submission failed:', error);
        return of(false);
      })
    )),
    startWith(false),
    shareReplay(1)
  );

  constructor(
    private fb: FormBuilder,
    private userService: UserService
  ) {}

  ngOnInit() {
    // Auto-save draft
    this.userForm.valueChanges.pipe(
      debounceTime(1000),
      distinctUntilChanged(),
      takeUntil(this.destroy$)
    ).subscribe(formValue => {
      localStorage.setItem('userFormDraft', JSON.stringify(formValue));
    });

    // Load draft
    const draft = localStorage.getItem('userFormDraft');
    if (draft) {
      this.userForm.patchValue(JSON.parse(draft));
    }
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }

  onSubmit() {
    if (this.userForm.valid) {
      this.submitSubject.next();
    }
  }

  private getEmailValidationMessage(): string {
    const emailControl = this.userForm.get('email')!;
    if (emailControl.valid || emailControl.untouched) return '';
    
    if (emailControl.errors?.['required']) return 'Email is required';
    if (emailControl.errors?.['email']) return 'Please enter a valid email';
    return '';
  }
}
```

### 2. Debounced Search Pattern

```typescript
@Component({
  selector: 'app-search',
  template: `
    <div class="search-container">
      <input 
        #searchInput
        type="text" 
        placeholder="Search users..."
        (input)="onSearchInput($event)">
      
      <div *ngIf="isSearching$ | async" class="searching">
        Searching...
      </div>
      
      <div class="results">
        <div *ngFor="let user of searchResults$ | async" class="result-item">
          {{ user.name }} - {{ user.email }}
        </div>
      </div>
      
      <div *ngIf="searchError$ | async as error" class="error">
        {{ error }}
      </div>
    </div>
  `
})
export class SearchComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  private searchSubject = new Subject<string>();

  searchResults$ = this.searchSubject.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    filter(term => term.length > 2 || term.length === 0),
    switchMap(term => 
      term.length === 0 
        ? of([])
        : this.userService.searchUsers(term).pipe(
            catchError(error => {
              console.error('Search failed:', error);
              return of([]);
            })
          )
    ),
    shareReplay(1)
  );

  isSearching$ = this.searchSubject.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    filter(term => term.length > 2),
    switchMap(term => 
      this.userService.searchUsers(term).pipe(
        map(() => false),
        startWith(true),
        catchError(() => of(false))
      )
    ),
    startWith(false)
  );

  searchError$ = this.searchSubject.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    filter(term => term.length > 2),
    switchMap(term =>
      this.userService.searchUsers(term).pipe(
        map(() => null),
        catchError(error => of('Search failed. Please try again.'))
      )
    ),
    startWith(null)
  );

  ngOnInit() {
    // Initial empty search
    this.searchSubject.next('');
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }

  onSearchInput(event: Event) {
    const target = event.target as HTMLInputElement;
    this.searchSubject.next(target.value);
  }
}
```

## Best Practices

### 1. Subscription Management

```typescript
// ✅ Good: Use takeUntil pattern
@Component({})
export class GoodComponent implements OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit() {
    this.dataService.getData().pipe(
      takeUntil(this.destroy$)
    ).subscribe(data => {
      // Handle data
    });
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// ✅ Good: Use async pipe when possible
@Component({
  template: `<div>{{ data$ | async | json }}</div>`
})
export class AsyncPipeComponent {
  data$ = this.dataService.getData();
  
  constructor(private dataService: DataService) {}
}
```

### 2. Error Handling

```typescript
// ✅ Good: Comprehensive error handling
@Component({})
export class ErrorHandlingComponent {
  data$ = this.dataService.getData().pipe(
    retry(3),
    catchError(error => {
      this.errorService.handleError(error);
      return of([]); // Provide fallback
    }),
    shareReplay(1)
  );
}
```

### 3. Performance Optimization

```typescript
// ✅ Good: Use OnPush change detection with observables
@Component({
  selector: 'app-optimized',
  template: `
    <div>{{ data$ | async | json }}</div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class OptimizedComponent {
  data$ = this.dataService.getData().pipe(
    shareReplay(1)
  );
}
```

## Common Pitfalls

### 1. Memory Leaks

```typescript
// ❌ Bad: Subscription without cleanup
@Component({})
export class BadComponent implements OnInit {
  ngOnInit() {
    this.dataService.getData().subscribe(data => {
      // This subscription will never be cleaned up!
    });
  }
}

// ✅ Good: Proper cleanup
@Component({})
export class GoodComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit() {
    this.dataService.getData().pipe(
      takeUntil(this.destroy$)
    ).subscribe(data => {
      // This subscription will be cleaned up
    });
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### 2. Inappropriate Operators

```typescript
// ❌ Bad: Using mergeMap for HTTP requests
searchUsers(term: string) {
  return this.searchSubject.pipe(
    mergeMap(term => this.http.get(`/api/search?q=${term}`))
    // This can cause race conditions!
  );
}

// ✅ Good: Using switchMap for HTTP requests
searchUsers(term: string) {
  return this.searchSubject.pipe(
    switchMap(term => this.http.get(`/api/search?q=${term}`))
    // This cancels previous requests
  );
}
```

## Summary

Angular RxJS patterns provide powerful tools for building reactive applications:

- **Service Patterns**: Use BehaviorSubject for state management
- **Component Communication**: Leverage observables for parent-child and sibling communication
- **State Management**: Implement reactive state with proper selectors
- **Error Handling**: Create centralized error handling strategies
- **Loading States**: Manage loading states reactively
- **Form Integration**: Combine reactive forms with observables
- **Search Patterns**: Implement debounced search with proper error handling

Key principles:
- Always clean up subscriptions
- Use async pipe when possible
- Handle errors gracefully
- Implement loading states
- Use appropriate operators for different scenarios
- Follow reactive programming principles
