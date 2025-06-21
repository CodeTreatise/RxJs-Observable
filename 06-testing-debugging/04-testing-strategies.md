# Unit Testing Observable-based Code

## Learning Objectives
- Master unit testing strategies for RxJS observables
- Test Angular services, components, and pipes with observables
- Handle asynchronous testing scenarios effectively
- Mock and stub observable dependencies
- Test error scenarios and edge cases
- Implement comprehensive test coverage strategies

## Prerequisites
- Unit testing fundamentals (Jasmine/Jest)
- Angular testing concepts
- RxJS operators knowledge
- Marble testing basics

---

## Table of Contents
1. [Testing Strategies Overview](#testing-strategies-overview)
2. [Testing Observable Services](#testing-observable-services)
3. [Testing Angular Components](#testing-angular-components)
4. [Testing Reactive Forms](#testing-reactive-forms)
5. [Testing HTTP Interactions](#testing-http-interactions)
6. [Mocking and Stubbing](#mocking-and-stubbing)
7. [Asynchronous Testing](#asynchronous-testing)
8. [Error Testing](#error-testing)
9. [Test Utilities and Helpers](#test-utilities-and-helpers)
10. [Best Practices](#best-practices)

---

## Testing Strategies Overview

### Testing Pyramid for Observables

```typescript
// Unit Tests (70%) - Test individual operators and functions
// Integration Tests (20%) - Test service interactions
// E2E Tests (10%) - Test complete user flows

interface TestStrategy {
  level: 'unit' | 'integration' | 'e2e';
  scope: 'operator' | 'service' | 'component' | 'flow';
  approach: 'synchronous' | 'asynchronous' | 'marble';
}

const testingApproaches: TestStrategy[] = [
  { level: 'unit', scope: 'operator', approach: 'marble' },
  { level: 'unit', scope: 'service', approach: 'asynchronous' },
  { level: 'integration', scope: 'component', approach: 'asynchronous' },
  { level: 'e2e', scope: 'flow', approach: 'asynchronous' }
];
```

### Core Testing Principles

```typescript
// 1. Isolation - Test observables in isolation
// 2. Determinism - Ensure predictable test results
// 3. Coverage - Test all paths and edge cases
// 4. Performance - Fast test execution
// 5. Maintainability - Easy to update and understand

class ObservableTestPrinciples {
  // Principle 1: Isolation
  static isolateObservable<T>(
    observable: Observable<T>,
    dependencies: any[] = []
  ): Observable<T> {
    // Mock all external dependencies
    return observable;
  }
  
  // Principle 2: Determinism
  static makeDeterministic<T>(
    observable: Observable<T>
  ): Observable<T> {
    // Use TestScheduler for time-based operations
    return observable;
  }
  
  // Principle 3: Coverage
  static testAllPaths<T>(
    observable: Observable<T>,
    testCases: Array<{ input: any; expected: T }>
  ): void {
    testCases.forEach(({ input, expected }) => {
      // Test each path
    });
  }
}
```

---

## Testing Observable Services

### Basic Service Testing

```typescript
// Service under test
@Injectable({
  providedIn: 'root'
})
export class UserService {
  constructor(private http: HttpClient) {}

  getUser(id: number): Observable<User> {
    return this.http.get<User>(`/api/users/${id}`).pipe(
      retry(3),
      catchError(() => of(null)),
      filter(user => user !== null),
      shareReplay(1)
    );
  }

  searchUsers(query: string): Observable<User[]> {
    if (!query.trim()) {
      return of([]);
    }
    
    return this.http.get<User[]>(`/api/users/search?q=${query}`).pipe(
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(users => of(users)),
      catchError(() => of([]))
    );
  }

  getUsersWithRefresh(): Observable<User[]> {
    const refresh$ = interval(60000);
    
    return refresh$.pipe(
      startWith(0),
      switchMap(() => this.http.get<User[]>('/api/users')),
      shareReplay(1)
    );
  }
}

// Test suite
describe('UserService', () => {
  let service: UserService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [UserService]
    });

    service = TestBed.inject(UserService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
  });

  describe('getUser', () => {
    it('should return user for valid ID', () => {
      const mockUser: User = { id: 1, name: 'John Doe', email: 'john@example.com' };

      service.getUser(1).subscribe(user => {
        expect(user).toEqual(mockUser);
      });

      const req = httpMock.expectOne('/api/users/1');
      expect(req.request.method).toBe('GET');
      req.flush(mockUser);
    });

    it('should retry on failure and then return null', () => {
      service.getUser(1).subscribe({
        next: user => {
          expect(user).toBeNull();
        },
        error: () => fail('Should not error after retries')
      });

      // Expect 4 requests (initial + 3 retries)
      for (let i = 0; i < 4; i++) {
        const req = httpMock.expectOne('/api/users/1');
        req.error(new ErrorEvent('Network error'));
      }
    });

    it('should filter out null responses', () => {
      let emissionCount = 0;
      
      service.getUser(1).subscribe(() => {
        emissionCount++;
      });

      const req = httpMock.expectOne('/api/users/1');
      req.flush(null);

      expect(emissionCount).toBe(0);
    });

    it('should share replay between multiple subscribers', () => {
      const mockUser: User = { id: 1, name: 'John Doe', email: 'john@example.com' };
      
      // Multiple subscriptions
      service.getUser(1).subscribe();
      service.getUser(1).subscribe();
      service.getUser(1).subscribe();

      // Should only make one HTTP request
      const req = httpMock.expectOne('/api/users/1');
      req.flush(mockUser);
    });
  });

  describe('searchUsers', () => {
    it('should return empty array for empty query', () => {
      service.searchUsers('').subscribe(users => {
        expect(users).toEqual([]);
      });

      service.searchUsers('   ').subscribe(users => {
        expect(users).toEqual([]);
      });

      httpMock.expectNone('/api/users/search');
    });

    it('should search users with debouncing', fakeAsync(() => {
      const mockUsers: User[] = [
        { id: 1, name: 'John Doe', email: 'john@example.com' }
      ];

      let result: User[] | undefined;
      service.searchUsers('john').subscribe(users => {
        result = users;
      });

      // Fast typing simulation
      tick(100);
      service.searchUsers('jo');
      tick(100);
      service.searchUsers('joh');
      tick(100);
      service.searchUsers('john');

      tick(300); // Wait for debounce

      const req = httpMock.expectOne('/api/users/search?q=john');
      req.flush(mockUsers);

      expect(result).toEqual(mockUsers);
    }));

    it('should handle search errors gracefully', () => {
      service.searchUsers('john').subscribe(users => {
        expect(users).toEqual([]);
      });

      const req = httpMock.expectOne('/api/users/search?q=john');
      req.error(new ErrorEvent('Network error'));
    });
  });

  describe('getUsersWithRefresh', () => {
    it('should fetch users immediately and then periodically', fakeAsync(() => {
      const mockUsers: User[] = [
        { id: 1, name: 'John Doe', email: 'john@example.com' }
      ];

      let emissionCount = 0;
      service.getUsersWithRefresh().subscribe(users => {
        expect(users).toEqual(mockUsers);
        emissionCount++;
      });

      // Initial request
      let req = httpMock.expectOne('/api/users');
      req.flush(mockUsers);
      expect(emissionCount).toBe(1);

      // After 60 seconds
      tick(60000);
      req = httpMock.expectOne('/api/users');
      req.flush(mockUsers);
      expect(emissionCount).toBe(2);

      // After another 60 seconds
      tick(60000);
      req = httpMock.expectOne('/api/users');
      req.flush(mockUsers);
      expect(emissionCount).toBe(3);
    }));
  });
});
```

### Advanced Service Testing

```typescript
// Complex service with multiple dependencies
@Injectable({
  providedIn: 'root'
})
export class DataService {
  constructor(
    private http: HttpClient,
    private cache: CacheService,
    private auth: AuthService,
    private logger: LoggerService
  ) {}

  getData(id: string): Observable<Data> {
    return this.auth.getToken().pipe(
      switchMap(token => 
        this.cache.get(id).pipe(
          switchMap(cached => {
            if (cached) {
              this.logger.info('Cache hit', { id });
              return of(cached);
            }
            
            return this.http.get<Data>(`/api/data/${id}`, {
              headers: { Authorization: `Bearer ${token}` }
            }).pipe(
              tap(data => {
                this.cache.set(id, data);
                this.logger.info('Data fetched', { id, data });
              }),
              catchError(error => {
                this.logger.error('Data fetch failed', { id, error });
                return throwError(() => error);
              })
            );
          })
        )
      )
    );
  }

  getDataWithFallback(id: string): Observable<Data | null> {
    return this.getData(id).pipe(
      catchError(() => {
        this.logger.warn('Using fallback data', { id });
        return this.getFallbackData(id);
      })
    );
  }

  private getFallbackData(id: string): Observable<Data | null> {
    return of(null).pipe(delay(100));
  }
}

// Advanced test suite
describe('DataService', () => {
  let service: DataService;
  let httpMock: HttpTestingController;
  let cacheMock: jasmine.SpyObj<CacheService>;
  let authMock: jasmine.SpyObj<AuthService>;
  let loggerMock: jasmine.SpyObj<LoggerService>;

  beforeEach(() => {
    const cacheSpies = jasmine.createSpyObj('CacheService', ['get', 'set']);
    const authSpies = jasmine.createSpyObj('AuthService', ['getToken']);
    const loggerSpies = jasmine.createSpyObj('LoggerService', ['info', 'warn', 'error']);

    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [
        DataService,
        { provide: CacheService, useValue: cacheSpies },
        { provide: AuthService, useValue: authSpies },
        { provide: LoggerService, useValue: loggerSpies }
      ]
    });

    service = TestBed.inject(DataService);
    httpMock = TestBed.inject(HttpTestingController);
    cacheMock = TestBed.inject(CacheService) as jasmine.SpyObj<CacheService>;
    authMock = TestBed.inject(AuthService) as jasmine.SpyObj<AuthService>;
    loggerMock = TestBed.inject(LoggerService) as jasmine.SpyObj<LoggerService>;
  });

  afterEach(() => {
    httpMock.verify();
  });

  describe('getData', () => {
    it('should return cached data when available', fakeAsync(() => {
      const mockData: Data = { id: '1', value: 'test' };
      const mockToken = 'mock-token';

      // Setup mocks
      authMock.getToken.and.returnValue(of(mockToken));
      cacheMock.get.and.returnValue(of(mockData));

      let result: Data | undefined;
      service.getData('1').subscribe(data => {
        result = data;
      });

      tick();

      expect(result).toEqual(mockData);
      expect(cacheMock.get).toHaveBeenCalledWith('1');
      expect(loggerMock.info).toHaveBeenCalledWith('Cache hit', { id: '1' });
      httpMock.expectNone('/api/data/1');
    }));

    it('should fetch from API when not cached', fakeAsync(() => {
      const mockData: Data = { id: '1', value: 'test' };
      const mockToken = 'mock-token';

      // Setup mocks
      authMock.getToken.and.returnValue(of(mockToken));
      cacheMock.get.and.returnValue(of(null));

      let result: Data | undefined;
      service.getData('1').subscribe(data => {
        result = data;
      });

      tick();

      const req = httpMock.expectOne('/api/data/1');
      expect(req.request.headers.get('Authorization')).toBe('Bearer mock-token');
      req.flush(mockData);

      expect(result).toEqual(mockData);
      expect(cacheMock.set).toHaveBeenCalledWith('1', mockData);
      expect(loggerMock.info).toHaveBeenCalledWith('Data fetched', { 
        id: '1', 
        data: mockData 
      });
    }));

    it('should handle authentication errors', fakeAsync(() => {
      authMock.getToken.and.returnValue(throwError(() => new Error('Auth failed')));

      let error: any;
      service.getData('1').subscribe({
        next: () => fail('Should not succeed'),
        error: (err) => error = err
      });

      tick();

      expect(error.message).toBe('Auth failed');
      httpMock.expectNone('/api/data/1');
    }));

    it('should handle API errors and log them', fakeAsync(() => {
      const mockToken = 'mock-token';
      const apiError = new Error('API Error');

      authMock.getToken.and.returnValue(of(mockToken));
      cacheMock.get.and.returnValue(of(null));

      let error: any;
      service.getData('1').subscribe({
        next: () => fail('Should not succeed'),
        error: (err) => error = err
      });

      tick();

      const req = httpMock.expectOne('/api/data/1');
      req.error(new ErrorEvent('Network error', { message: 'API Error' }));

      expect(error).toBeDefined();
      expect(loggerMock.error).toHaveBeenCalledWith('Data fetch failed', {
        id: '1',
        error: jasmine.any(Object)
      });
    }));
  });

  describe('getDataWithFallback', () => {
    it('should return fallback data on error', fakeAsync(() => {
      const mockToken = 'mock-token';

      authMock.getToken.and.returnValue(of(mockToken));
      cacheMock.get.and.returnValue(of(null));

      let result: Data | null | undefined;
      service.getDataWithFallback('1').subscribe(data => {
        result = data;
      });

      tick();

      const req = httpMock.expectOne('/api/data/1');
      req.error(new ErrorEvent('Network error'));

      tick(100); // Wait for fallback delay

      expect(result).toBeNull();
      expect(loggerMock.warn).toHaveBeenCalledWith('Using fallback data', { id: '1' });
    }));
  });
});
```

---

## Testing Angular Components

### Component with Observables

```typescript
// Component under test
@Component({
  selector: 'app-user-list',
  template: `
    <div class="search-container">
      <input 
        [formControl]="searchControl" 
        placeholder="Search users..."
        data-testid="search-input">
    </div>
    
    <div class="user-list" data-testid="user-list">
      <div *ngIf="loading$ | async" data-testid="loading">
        Loading...
      </div>
      
      <div *ngIf="error$ | async as error" 
           class="error" 
           data-testid="error">
        {{ error }}
      </div>
      
      <div *ngFor="let user of users$ | async; trackBy: trackByUserId" 
           class="user-item"
           data-testid="user-item">
        {{ user.name }} ({{ user.email }})
      </div>
      
      <div *ngIf="(users$ | async)?.length === 0 && !(loading$ | async)" 
           data-testid="no-results">
        No users found
      </div>
    </div>
  `
})
export class UserListComponent implements OnInit, OnDestroy {
  searchControl = new FormControl('');
  
  users$: Observable<User[]>;
  loading$: Observable<boolean>;
  error$: Observable<string | null>;
  
  private destroy$ = new Subject<void>();

  constructor(private userService: UserService) {}

  ngOnInit(): void {
    const search$ = this.searchControl.valueChanges.pipe(
      startWith(''),
      debounceTime(300),
      distinctUntilChanged(),
      shareReplay(1)
    );

    const searchResults$ = search$.pipe(
      switchMap(query => 
        query ? this.userService.searchUsers(query) : of([])
      ),
      shareReplay(1)
    );

    this.users$ = searchResults$.pipe(
      map(users => users || []),
      catchError(() => of([]))
    );

    this.loading$ = search$.pipe(
      switchMap(() => 
        concat(
          of(true),
          searchResults$.pipe(take(1), map(() => false))
        )
      ),
      startWith(false)
    );

    this.error$ = searchResults$.pipe(
      map(() => null),
      catchError(error => of(error.message))
    );

    // Auto-cleanup
    merge(this.users$, this.loading$, this.error$)
      .pipe(takeUntil(this.destroy$))
      .subscribe();
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }

  trackByUserId(index: number, user: User): number {
    return user.id;
  }
}

// Component test suite
describe('UserListComponent', () => {
  let component: UserListComponent;
  let fixture: ComponentFixture<UserListComponent>;
  let userService: jasmine.SpyObj<UserService>;

  beforeEach(async () => {
    const userServiceSpy = jasmine.createSpyObj('UserService', ['searchUsers']);

    await TestBed.configureTestingModule({
      declarations: [UserListComponent],
      imports: [ReactiveFormsModule],
      providers: [
        { provide: UserService, useValue: userServiceSpy }
      ]
    }).compileComponents();

    fixture = TestBed.createComponent(UserListComponent);
    component = fixture.componentInstance;
    userService = TestBed.inject(UserService) as jasmine.SpyObj<UserService>;
  });

  describe('initialization', () => {
    it('should create component', () => {
      expect(component).toBeTruthy();
    });

    it('should initialize with empty search', fakeAsync(() => {
      userService.searchUsers.and.returnValue(of([]));
      
      component.ngOnInit();
      tick(300); // debounce time
      
      expect(userService.searchUsers).toHaveBeenCalledWith('');
    }));
  });

  describe('search functionality', () => {
    beforeEach(() => {
      userService.searchUsers.and.returnValue(of([]));
      component.ngOnInit();
      fixture.detectChanges();
    });

    it('should search when typing in search input', fakeAsync(() => {
      const searchInput = fixture.debugElement.query(
        By.css('[data-testid="search-input"]')
      );
      const mockUsers: User[] = [
        { id: 1, name: 'John Doe', email: 'john@example.com' }
      ];

      userService.searchUsers.and.returnValue(of(mockUsers));

      // Simulate typing
      searchInput.nativeElement.value = 'john';
      searchInput.nativeElement.dispatchEvent(new Event('input'));
      
      tick(300); // Wait for debounce
      fixture.detectChanges();

      expect(userService.searchUsers).toHaveBeenCalledWith('john');
      
      const userItems = fixture.debugElement.queryAll(
        By.css('[data-testid="user-item"]')
      );
      expect(userItems.length).toBe(1);
      expect(userItems[0].nativeElement.textContent.trim())
        .toContain('John Doe (john@example.com)');
    }));

    it('should debounce search input', fakeAsync(() => {
      const searchInput = fixture.debugElement.query(
        By.css('[data-testid="search-input"]')
      );

      // Rapid typing
      searchInput.nativeElement.value = 'j';
      searchInput.nativeElement.dispatchEvent(new Event('input'));
      tick(100);

      searchInput.nativeElement.value = 'jo';
      searchInput.nativeElement.dispatchEvent(new Event('input'));
      tick(100);

      searchInput.nativeElement.value = 'joh';
      searchInput.nativeElement.dispatchEvent(new Event('input'));
      tick(100);

      searchInput.nativeElement.value = 'john';
      searchInput.nativeElement.dispatchEvent(new Event('input'));
      tick(300);

      // Should only call service once with final value
      expect(userService.searchUsers).toHaveBeenCalledTimes(2); // Initial + final
      expect(userService.searchUsers).toHaveBeenCalledWith('john');
    }));

    it('should show loading state during search', fakeAsync(() => {
      const searchSubject = new BehaviorSubject<User[]>([]);
      userService.searchUsers.and.returnValue(searchSubject);

      const searchInput = fixture.debugElement.query(
        By.css('[data-testid="search-input"]')
      );

      searchInput.nativeElement.value = 'john';
      searchInput.nativeElement.dispatchEvent(new Event('input'));
      tick(300);
      fixture.detectChanges();

      // Should show loading
      let loadingElement = fixture.debugElement.query(
        By.css('[data-testid="loading"]')
      );
      expect(loadingElement).toBeTruthy();

      // Complete the search
      searchSubject.next([]);
      fixture.detectChanges();

      // Should hide loading
      loadingElement = fixture.debugElement.query(
        By.css('[data-testid="loading"]')
      );
      expect(loadingElement).toBeFalsy();
    }));

    it('should show error state on search failure', fakeAsync(() => {
      userService.searchUsers.and.returnValue(
        throwError(() => new Error('Search failed'))
      );

      const searchInput = fixture.debugElement.query(
        By.css('[data-testid="search-input"]')
      );

      searchInput.nativeElement.value = 'john';
      searchInput.nativeElement.dispatchEvent(new Event('input'));
      tick(300);
      fixture.detectChanges();

      const errorElement = fixture.debugElement.query(
        By.css('[data-testid="error"]')
      );
      expect(errorElement).toBeTruthy();
      expect(errorElement.nativeElement.textContent.trim())
        .toBe('Search failed');
    }));

    it('should show no results message when search returns empty', fakeAsync(() => {
      userService.searchUsers.and.returnValue(of([]));

      const searchInput = fixture.debugElement.query(
        By.css('[data-testid="search-input"]')
      );

      searchInput.nativeElement.value = 'nonexistent';
      searchInput.nativeElement.dispatchEvent(new Event('input'));
      tick(300);
      fixture.detectChanges();

      const noResultsElement = fixture.debugElement.query(
        By.css('[data-testid="no-results"]')
      );
      expect(noResultsElement).toBeTruthy();
      expect(noResultsElement.nativeElement.textContent.trim())
        .toBe('No users found');
    }));
  });

  describe('cleanup', () => {
    it('should unsubscribe on destroy', () => {
      spyOn(component['destroy$'], 'next');
      spyOn(component['destroy$'], 'complete');

      component.ngOnDestroy();

      expect(component['destroy$'].next).toHaveBeenCalled();
      expect(component['destroy$'].complete).toHaveBeenCalled();
    });
  });
});
```

---

## Testing Reactive Forms

### Complex Form Testing

```typescript
// Form component
@Component({
  selector: 'app-user-form',
  template: `
    <form [formGroup]="userForm" (ngSubmit)="onSubmit()">
      <div>
        <label>Name:</label>
        <input formControlName="name" data-testid="name-input">
        <div *ngIf="nameErrors$ | async as errors" class="errors">
          <div *ngFor="let error of errors" data-testid="name-error">
            {{ error }}
          </div>
        </div>
      </div>

      <div>
        <label>Email:</label>
        <input formControlName="email" data-testid="email-input">
        <div *ngIf="emailErrors$ | async as errors" class="errors">
          <div *ngFor="let error of errors" data-testid="email-error">
            {{ error }}
          </div>
        </div>
      </div>

      <div>
        <label>Username:</label>
        <input formControlName="username" data-testid="username-input">
        <div *ngIf="usernameErrors$ | async as errors" class="errors">
          <div *ngFor="let error of errors" data-testid="username-error">
            {{ error }}
          </div>
        </div>
        <div *ngIf="usernameChecking$ | async" data-testid="username-checking">
          Checking availability...
        </div>
      </div>

      <button 
        type="submit" 
        [disabled]="userForm.invalid || (submitting$ | async)"
        data-testid="submit-button">
        <span *ngIf="submitting$ | async">Submitting...</span>
        <span *ngIf="!(submitting$ | async)">Submit</span>
      </button>
    </form>
  `
})
export class UserFormComponent implements OnInit {
  userForm = this.fb.group({
    name: ['', [Validators.required, Validators.minLength(2)]],
    email: ['', [Validators.required, Validators.email]],
    username: ['', [Validators.required], [this.usernameValidator.bind(this)]]
  });

  nameErrors$: Observable<string[]>;
  emailErrors$: Observable<string[]>;
  usernameErrors$: Observable<string[]>;
  usernameChecking$: Observable<boolean>;
  submitting$ = new BehaviorSubject<boolean>(false);

  constructor(
    private fb: FormBuilder,
    private userService: UserService
  ) {}

  ngOnInit(): void {
    this.nameErrors$ = this.getFieldErrors('name');
    this.emailErrors$ = this.getFieldErrors('email');
    this.usernameErrors$ = this.getFieldErrors('username');
    
    this.usernameChecking$ = this.userForm.get('username')!.statusChanges.pipe(
      map(status => status === 'PENDING'),
      startWith(false)
    );
  }

  private getFieldErrors(fieldName: string): Observable<string[]> {
    const control = this.userForm.get(fieldName)!;
    
    return control.statusChanges.pipe(
      startWith(control.status),
      map(() => {
        if (control.errors && (control.dirty || control.touched)) {
          return this.mapErrors(control.errors);
        }
        return [];
      })
    );
  }

  private mapErrors(errors: ValidationErrors): string[] {
    const errorMessages: string[] = [];
    
    if (errors['required']) {
      errorMessages.push('This field is required');
    }
    if (errors['minlength']) {
      errorMessages.push(`Minimum length is ${errors['minlength'].requiredLength}`);
    }
    if (errors['email']) {
      errorMessages.push('Invalid email format');
    }
    if (errors['usernameExists']) {
      errorMessages.push('Username already exists');
    }
    
    return errorMessages;
  }

  private usernameValidator(control: AbstractControl): Observable<ValidationErrors | null> {
    if (!control.value) {
      return of(null);
    }

    return timer(500).pipe(
      switchMap(() => this.userService.checkUsernameAvailability(control.value)),
      map(available => available ? null : { usernameExists: true }),
      catchError(() => of(null))
    );
  }

  onSubmit(): void {
    if (this.userForm.valid) {
      this.submitting$.next(true);
      
      this.userService.createUser(this.userForm.value).subscribe({
        next: () => {
          this.submitting$.next(false);
          // Handle success
        },
        error: () => {
          this.submitting$.next(false);
          // Handle error
        }
      });
    }
  }
}

// Form tests
describe('UserFormComponent', () => {
  let component: UserFormComponent;
  let fixture: ComponentFixture<UserFormComponent>;
  let userService: jasmine.SpyObj<UserService>;

  beforeEach(async () => {
    const userServiceSpy = jasmine.createSpyObj('UserService', [
      'checkUsernameAvailability',
      'createUser'
    ]);

    await TestBed.configureTestingModule({
      declarations: [UserFormComponent],
      imports: [ReactiveFormsModule],
      providers: [
        { provide: UserService, useValue: userServiceSpy }
      ]
    }).compileComponents();

    fixture = TestBed.createComponent(UserFormComponent);
    component = fixture.componentInstance;
    userService = TestBed.inject(UserService) as jasmine.SpyObj<UserService>;
  });

  describe('form validation', () => {
    beforeEach(() => {
      userService.checkUsernameAvailability.and.returnValue(of(true));
      component.ngOnInit();
      fixture.detectChanges();
    });

    it('should show required error for empty name', fakeAsync(() => {
      const nameInput = fixture.debugElement.query(
        By.css('[data-testid="name-input"]')
      );

      // Focus and blur to trigger validation
      nameInput.nativeElement.focus();
      nameInput.nativeElement.blur();
      nameInput.nativeElement.dispatchEvent(new Event('blur'));
      
      tick();
      fixture.detectChanges();

      const nameError = fixture.debugElement.query(
        By.css('[data-testid="name-error"]')
      );
      expect(nameError.nativeElement.textContent.trim())
        .toBe('This field is required');
    }));

    it('should show minimum length error for short name', fakeAsync(() => {
      const nameInput = fixture.debugElement.query(
        By.css('[data-testid="name-input"]')
      );

      nameInput.nativeElement.value = 'a';
      nameInput.nativeElement.dispatchEvent(new Event('input'));
      nameInput.nativeElement.blur();
      
      tick();
      fixture.detectChanges();

      const nameError = fixture.debugElement.query(
        By.css('[data-testid="name-error"]')
      );
      expect(nameError.nativeElement.textContent.trim())
        .toBe('Minimum length is 2');
    }));

    it('should show email format error for invalid email', fakeAsync(() => {
      const emailInput = fixture.debugElement.query(
        By.css('[data-testid="email-input"]')
      );

      emailInput.nativeElement.value = 'invalid-email';
      emailInput.nativeElement.dispatchEvent(new Event('input'));
      emailInput.nativeElement.blur();
      
      tick();
      fixture.detectChanges();

      const emailError = fixture.debugElement.query(
        By.css('[data-testid="email-error"]')
      );
      expect(emailError.nativeElement.textContent.trim())
        .toBe('Invalid email format');
    }));

    it('should check username availability', fakeAsync(() => {
      const usernameInput = fixture.debugElement.query(
        By.css('[data-testid="username-input"]')
      );

      usernameInput.nativeElement.value = 'testuser';
      usernameInput.nativeElement.dispatchEvent(new Event('input'));
      
      tick(500); // Wait for debounce
      
      expect(userService.checkUsernameAvailability)
        .toHaveBeenCalledWith('testuser');
    }));

    it('should show username checking indicator', fakeAsync(() => {
      userService.checkUsernameAvailability.and.returnValue(
        timer(1000).pipe(map(() => true))
      );

      const usernameInput = fixture.debugElement.query(
        By.css('[data-testid="username-input"]')
      );

      usernameInput.nativeElement.value = 'testuser';
      usernameInput.nativeElement.dispatchEvent(new Event('input'));
      
      tick(500);
      fixture.detectChanges();

      const checkingIndicator = fixture.debugElement.query(
        By.css('[data-testid="username-checking"]')
      );
      expect(checkingIndicator).toBeTruthy();
      expect(checkingIndicator.nativeElement.textContent.trim())
        .toBe('Checking availability...');

      tick(1000);
      fixture.detectChanges();

      const checkingIndicatorAfter = fixture.debugElement.query(
        By.css('[data-testid="username-checking"]')
      );
      expect(checkingIndicatorAfter).toBeFalsy();
    }));

    it('should show username exists error', fakeAsync(() => {
      userService.checkUsernameAvailability.and.returnValue(of(false));

      const usernameInput = fixture.debugElement.query(
        By.css('[data-testid="username-input"]')
      );

      usernameInput.nativeElement.value = 'existinguser';
      usernameInput.nativeElement.dispatchEvent(new Event('input'));
      
      tick(500);
      fixture.detectChanges();

      const usernameError = fixture.debugElement.query(
        By.css('[data-testid="username-error"]')
      );
      expect(usernameError.nativeElement.textContent.trim())
        .toBe('Username already exists');
    }));
  });

  describe('form submission', () => {
    beforeEach(() => {
      userService.checkUsernameAvailability.and.returnValue(of(true));
      userService.createUser.and.returnValue(of({ id: 1, name: 'Test User' }));
      component.ngOnInit();
      fixture.detectChanges();
    });

    it('should disable submit button when form is invalid', () => {
      const submitButton = fixture.debugElement.query(
        By.css('[data-testid="submit-button"]')
      );

      expect(submitButton.nativeElement.disabled).toBeTruthy();
    });

    it('should enable submit button when form is valid', fakeAsync(() => {
      // Fill form with valid data
      const nameInput = fixture.debugElement.query(By.css('[data-testid="name-input"]'));
      const emailInput = fixture.debugElement.query(By.css('[data-testid="email-input"]'));
      const usernameInput = fixture.debugElement.query(By.css('[data-testid="username-input"]'));

      nameInput.nativeElement.value = 'John Doe';
      nameInput.nativeElement.dispatchEvent(new Event('input'));

      emailInput.nativeElement.value = 'john@example.com';
      emailInput.nativeElement.dispatchEvent(new Event('input'));

      usernameInput.nativeElement.value = 'johndoe';
      usernameInput.nativeElement.dispatchEvent(new Event('input'));

      tick(500); // Wait for async validation
      fixture.detectChanges();

      const submitButton = fixture.debugElement.query(
        By.css('[data-testid="submit-button"]')
      );
      expect(submitButton.nativeElement.disabled).toBeFalsy();
    }));

    it('should show submitting state during form submission', fakeAsync(() => {
      // Setup valid form
      component.userForm.patchValue({
        name: 'John Doe',
        email: 'john@example.com',
        username: 'johndoe'
      });
      component.userForm.markAllAsTouched();
      
      userService.createUser.and.returnValue(timer(1000).pipe(map(() => ({ id: 1 }))));

      tick(500); // Wait for validation
      fixture.detectChanges();

      const submitButton = fixture.debugElement.query(
        By.css('[data-testid="submit-button"]')
      );
      
      submitButton.nativeElement.click();
      fixture.detectChanges();

      expect(submitButton.nativeElement.textContent.trim())
        .toBe('Submitting...');
      expect(submitButton.nativeElement.disabled).toBeTruthy();

      tick(1000);
      fixture.detectChanges();

      expect(submitButton.nativeElement.textContent.trim())
        .toBe('Submit');
    }));
  });
});
```

---

## Testing HTTP Interactions

### HTTP Service Testing

```typescript
// HTTP service with complex scenarios
@Injectable({
  providedIn: 'root'
})
export class ApiService {
  constructor(private http: HttpClient) {}

  getUserWithRetry(id: number): Observable<User> {
    return this.http.get<User>(`/api/users/${id}`).pipe(
      retry({
        count: 3,
        delay: (error, retryIndex) => timer(retryIndex * 1000)
      }),
      timeout(5000),
      catchError(error => {
        if (error.status === 404) {
          return throwError(() => new Error('User not found'));
        }
        return throwError(() => new Error('Network error'));
      })
    );
  }

  uploadFile(file: File): Observable<UploadProgress> {
    const formData = new FormData();
    formData.append('file', file);

    return this.http.post<any>('/api/upload', formData, {
      reportProgress: true,
      observe: 'events'
    }).pipe(
      map(event => {
        if (event.type === HttpEventType.UploadProgress) {
          const progress = Math.round(100 * event.loaded / event.total!);
          return { type: 'progress', progress };
        } else if (event.type === HttpEventType.Response) {
          return { type: 'complete', result: event.body };
        }
        return { type: 'start' };
      }),
      filter(progress => progress.type !== 'start')
    );
  }

  getDataWithCache(): Observable<Data[]> {
    return this.http.get<Data[]>('/api/data', {
      headers: { 'Cache-Control': 'max-age=300' }
    }).pipe(
      shareReplay({
        bufferSize: 1,
        refCount: false
      })
    );
  }
}

interface UploadProgress {
  type: 'progress' | 'complete';
  progress?: number;
  result?: any;
}

// HTTP tests
describe('ApiService HTTP Interactions', () => {
  let service: ApiService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [ApiService]
    });

    service = TestBed.inject(ApiService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
  });

  describe('getUserWithRetry', () => {
    it('should retry on failure with exponential backoff', fakeAsync(() => {
      let result: User | undefined;
      let error: any;

      service.getUserWithRetry(1).subscribe({
        next: user => result = user,
        error: err => error = err
      });

      // First request fails
      let req = httpMock.expectOne('/api/users/1');
      req.error(new ErrorEvent('Network error'), { status: 500 });

      tick(1000); // First retry delay

      // Second request fails
      req = httpMock.expectOne('/api/users/1');
      req.error(new ErrorEvent('Network error'), { status: 500 });

      tick(2000); // Second retry delay

      // Third request fails
      req = httpMock.expectOne('/api/users/1');
      req.error(new ErrorEvent('Network error'), { status: 500 });

      tick(3000); // Third retry delay

      // Fourth request (final attempt) fails
      req = httpMock.expectOne('/api/users/1');
      req.error(new ErrorEvent('Network error'), { status: 500 });

      expect(error).toBeDefined();
      expect(error.message).toBe('Network error');
    }));

    it('should handle 404 errors specifically', () => {
      let error: any;

      service.getUserWithRetry(1).subscribe({
        error: err => error = err
      });

      const req = httpMock.expectOne('/api/users/1');
      req.error(new ErrorEvent('Not found'), { status: 404 });

      expect(error.message).toBe('User not found');
    });

    it('should timeout after 5 seconds', fakeAsync(() => {
      let error: any;

      service.getUserWithRetry(1).subscribe({
        error: err => error = err
      });

      // Don't respond to the request
      httpMock.expectOne('/api/users/1');
      
      tick(5000);

      expect(error).toBeDefined();
      expect(error.name).toBe('TimeoutError');
    }));
  });

  describe('uploadFile', () => {
    it('should report upload progress', () => {
      const file = new File(['content'], 'test.txt', { type: 'text/plain' });
      const progressEvents: UploadProgress[] = [];

      service.uploadFile(file).subscribe(progress => {
        progressEvents.push(progress);
      });

      const req = httpMock.expectOne('/api/upload');
      expect(req.request.method).toBe('POST');
      expect(req.request.body).toBeInstanceOf(FormData);

      // Simulate progress events
      req.event({
        type: HttpEventType.UploadProgress,
        loaded: 25,
        total: 100
      });

      req.event({
        type: HttpEventType.UploadProgress,
        loaded: 50,
        total: 100
      });

      req.event({
        type: HttpEventType.UploadProgress,
        loaded: 100,
        total: 100
      });

      req.event({
        type: HttpEventType.Response,
        body: { fileId: 'abc123' }
      } as HttpResponse<any>);

      expect(progressEvents).toEqual([
        { type: 'progress', progress: 25 },
        { type: 'progress', progress: 50 },
        { type: 'progress', progress: 100 },
        { type: 'complete', result: { fileId: 'abc123' } }
      ]);
    });
  });

  describe('getDataWithCache', () => {
    it('should set cache headers', () => {
      service.getDataWithCache().subscribe();

      const req = httpMock.expectOne('/api/data');
      expect(req.request.headers.get('Cache-Control')).toBe('max-age=300');
      req.flush([]);
    });

    it('should share replay between subscribers', () => {
      const mockData = [{ id: 1, value: 'test' }];

      // Multiple subscriptions
      service.getDataWithCache().subscribe();
      service.getDataWithCache().subscribe();
      service.getDataWithCache().subscribe();

      // Should only make one request
      const req = httpMock.expectOne('/api/data');
      req.flush(mockData);
    });
  });
});
```

---

## Mocking and Stubbing

### Advanced Mocking Strategies

```typescript
// Mock factory for observables
class ObservableMockFactory {
  static createMockService<T>(methods: string[]): jasmine.SpyObj<T> {
    const spy = jasmine.createSpyObj('MockService', methods);
    
    // Default all methods to return empty observables
    methods.forEach(method => {
      spy[method].and.returnValue(EMPTY);
    });
    
    return spy;
  }
  
  static createControlledObservable<T>(): {
    observable: Observable<T>;
    next: (value: T) => void;
    error: (error: any) => void;
    complete: () => void;
  } {
    const subject = new Subject<T>();
    
    return {
      observable: subject.asObservable(),
      next: (value: T) => subject.next(value),
      error: (error: any) => subject.error(error),
      complete: () => subject.complete()
    };
  }
  
  static createTimedObservable<T>(
    values: T[], 
    interval: number = 1000
  ): Observable<T> {
    return new Observable<T>(subscriber => {
      let index = 0;
      
      const timer = setInterval(() => {
        if (index < values.length) {
          subscriber.next(values[index]);
          index++;
        } else {
          subscriber.complete();
          clearInterval(timer);
        }
      }, interval);
      
      return () => clearInterval(timer);
    });
  }
}

// Complex service mock
describe('Complex Service Mocking', () => {
  let service: DataProcessingService;
  let httpMock: jasmine.SpyObj<HttpClient>;
  let cacheMock: jasmine.SpyObj<CacheService>;
  let loggerMock: jasmine.SpyObj<LoggerService>;

  beforeEach(() => {
    httpMock = ObservableMockFactory.createMockService<HttpClient>([
      'get', 'post', 'put', 'delete'
    ]);
    
    cacheMock = ObservableMockFactory.createMockService<CacheService>([
      'get', 'set', 'delete'
    ]);
    
    loggerMock = jasmine.createSpyObj('LoggerService', [
      'info', 'warn', 'error'
    ]);

    TestBed.configureTestingModule({
      providers: [
        DataProcessingService,
        { provide: HttpClient, useValue: httpMock },
        { provide: CacheService, useValue: cacheMock },
        { provide: LoggerService, useValue: loggerMock }
      ]
    });

    service = TestBed.inject(DataProcessingService);
  });

  it('should process data with controlled timing', fakeAsync(() => {
    const { observable, next, complete } = ObservableMockFactory.createControlledObservable<Data>();
    const mockData = { id: 1, value: 'test' };
    
    httpMock.get.and.returnValue(observable);
    cacheMock.get.and.returnValue(of(null));

    let result: Data | undefined;
    service.processData(1).subscribe(data => {
      result = data;
    });

    // Simulate delayed response
    tick(1000);
    next(mockData);
    complete();
    tick();

    expect(result).toEqual(mockData);
  }));

  it('should handle multiple concurrent requests', fakeAsync(() => {
    const requests = [
      ObservableMockFactory.createControlledObservable<Data>(),
      ObservableMockFactory.createControlledObservable<Data>(),
      ObservableMockFactory.createControlledObservable<Data>()
    ];

    httpMock.get.and.returnValues(...requests.map(r => r.observable));
    cacheMock.get.and.returnValue(of(null));

    const results: Data[] = [];
    
    // Start multiple requests
    service.processData(1).subscribe(data => results.push(data));
    service.processData(2).subscribe(data => results.push(data));
    service.processData(3).subscribe(data => results.push(data));

    // Complete requests in different order
    requests[1].next({ id: 2, value: 'second' });
    requests[1].complete();
    tick();

    requests[0].next({ id: 1, value: 'first' });
    requests[0].complete();
    tick();

    requests[2].next({ id: 3, value: 'third' });
    requests[2].complete();
    tick();

    expect(results).toEqual([
      { id: 2, value: 'second' },
      { id: 1, value: 'first' },
      { id: 3, value: 'third' }
    ]);
  }));
});

// Observable behavior stubs
class ObservableStubs {
  static immediateValue<T>(value: T): Observable<T> {
    return of(value);
  }
  
  static delayedValue<T>(value: T, delay: number = 1000): Observable<T> {
    return of(value).pipe(delay(delay));
  }
  
  static neverCompleting<T>(): Observable<T> {
    return NEVER;
  }
  
  static immediateError(error: any): Observable<never> {
    return throwError(() => error);
  }
  
  static delayedError(error: any, delay: number = 1000): Observable<never> {
    return timer(delay).pipe(
      switchMap(() => throwError(() => error))
    );
  }
  
  static sequence<T>(values: T[], interval: number = 100): Observable<T> {
    return from(values).pipe(
      concatMap((value, index) => 
        of(value).pipe(delay(index * interval))
      )
    );
  }
  
  static intermittentFailure<T>(
    value: T, 
    failureRate: number = 0.5
  ): Observable<T> {
    return defer(() => {
      if (Math.random() < failureRate) {
        return throwError(() => new Error('Intermittent failure'));
      }
      return of(value);
    });
  }
}

// Usage in tests
describe('Observable Stubs Usage', () => {
  let service: ApiService;
  let httpMock: jasmine.SpyObj<HttpClient>;

  beforeEach(() => {
    httpMock = jasmine.createSpyObj('HttpClient', ['get']);
    service = new ApiService(httpMock);
  });

  it('should handle immediate responses', () => {
    const mockData = { id: 1, name: 'Test' };
    httpMock.get.and.returnValue(ObservableStubs.immediateValue(mockData));

    let result: any;
    service.getData(1).subscribe(data => result = data);

    expect(result).toEqual(mockData);
  });

  it('should handle delayed responses', fakeAsync(() => {
    const mockData = { id: 1, name: 'Test' };
    httpMock.get.and.returnValue(ObservableStubs.delayedValue(mockData, 2000));

    let result: any;
    service.getData(1).subscribe(data => result = data);

    expect(result).toBeUndefined();
    
    tick(2000);
    
    expect(result).toEqual(mockData);
  }));

  it('should handle intermittent failures', () => {
    const mockData = { id: 1, name: 'Test' };
    httpMock.get.and.returnValue(
      ObservableStubs.intermittentFailure(mockData, 1.0) // Always fail
    );

    let error: any;
    service.getData(1).subscribe({
      error: err => error = err
    });

    expect(error.message).toBe('Intermittent failure');
  });
});
```

---

## Summary

This lesson covered comprehensive unit testing strategies for RxJS observable-based code:

### Key Testing Concepts:
1. **Testing Strategies Overview** - Testing pyramid and approaches
2. **Observable Services** - Testing HTTP services with retry logic
3. **Angular Components** - Testing reactive components and forms
4. **Reactive Forms** - Testing form validation and async validators
5. **HTTP Interactions** - Testing complex HTTP scenarios
6. **Mocking and Stubbing** - Advanced mocking strategies
7. **Asynchronous Testing** - Handling timing and async operations
8. **Error Testing** - Testing error scenarios and recovery
9. **Test Utilities** - Reusable testing helpers
10. **Best Practices** - Effective testing patterns

### Essential Testing Techniques:
- **Marble testing** for operator behavior
- **fakeAsync/tick** for timing control
- **HttpTestingController** for HTTP interactions
- **Spy objects** for dependency mocking
- **TestScheduler** for complex timing scenarios
- **Observable stubs** for controlled behavior

### Best Practices:
- Test observable behavior, not implementation
- Use appropriate testing tools for each scenario
- Mock external dependencies consistently
- Test error scenarios and edge cases
- Maintain test isolation and determinism
- Create reusable testing utilities
- Focus on observable contracts and behavior

Effective testing of observable-based code ensures reliable, maintainable Angular applications with predictable reactive behavior.

---

## Exercises

1. **Service Testing**: Test a complex service with multiple dependencies and error handling
2. **Component Testing**: Test a reactive component with multiple observable streams
3. **Form Testing**: Test a complex reactive form with async validation
4. **HTTP Testing**: Test HTTP interactions with retries, timeouts, and progress tracking
5. **Mock Factory**: Create a comprehensive mock factory for observable services
6. **Error Testing**: Test various error scenarios and recovery mechanisms
7. **Performance Testing**: Test observable performance and memory usage

## Additional Resources

- [Angular Testing Guide](https://angular.io/guide/testing)
- [RxJS Testing Documentation](https://rxjs.dev/guide/testing)
- [Testing Reactive Forms](https://angular.io/guide/reactive-forms#testing)
- [HTTP Testing in Angular](https://angular.io/guide/http#testing-http-requests)
