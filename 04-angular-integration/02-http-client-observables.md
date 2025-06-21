# HttpClient & Observables

## Overview

Angular's HttpClient is built on RxJS Observables, providing powerful reactive capabilities for HTTP communication. This lesson covers comprehensive patterns for using HttpClient with Observables, including request configuration, response handling, error management, and advanced scenarios like caching, retries, and concurrent requests.

## HttpClient Fundamentals

### Basic HTTP Operations

**Simple GET Request:**
```typescript
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams, HttpHeaders } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class ApiService {
  private readonly baseUrl = 'https://api.example.com';

  constructor(private http: HttpClient) {}

  // Basic GET request
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(`${this.baseUrl}/users`);
  }

  // GET with parameters
  getUserById(id: number): Observable<User> {
    return this.http.get<User>(`${this.baseUrl}/users/${id}`);
  }

  // GET with query parameters
  searchUsers(query: string, page: number = 1): Observable<UserSearchResponse> {
    const params = new HttpParams()
      .set('q', query)
      .set('page', page.toString())
      .set('limit', '10');

    return this.http.get<UserSearchResponse>(`${this.baseUrl}/users/search`, { params });
  }
}

interface User {
  id: number;
  name: string;
  email: string;
  avatar?: string;
}

interface UserSearchResponse {
  users: User[];
  total: number;
  page: number;
  totalPages: number;
}
```

**POST, PUT, DELETE Operations:**
```typescript
@Injectable({
  providedIn: 'root'
})
export class UserService {
  private readonly apiUrl = 'https://api.example.com/users';

  constructor(private http: HttpClient) {}

  // Create user
  createUser(user: Partial<User>): Observable<User> {
    const headers = new HttpHeaders({
      'Content-Type': 'application/json'
    });

    return this.http.post<User>(this.apiUrl, user, { headers });
  }

  // Update user
  updateUser(id: number, updates: Partial<User>): Observable<User> {
    return this.http.put<User>(`${this.apiUrl}/${id}`, updates);
  }

  // Partial update
  patchUser(id: number, updates: Partial<User>): Observable<User> {
    return this.http.patch<User>(`${this.apiUrl}/${id}`, updates);
  }

  // Delete user
  deleteUser(id: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`);
  }

  // Upload file
  uploadAvatar(userId: number, file: File): Observable<{ avatarUrl: string }> {
    const formData = new FormData();
    formData.append('avatar', file);

    return this.http.post<{ avatarUrl: string }>(
      `${this.apiUrl}/${userId}/avatar`, 
      formData
    );
  }
}
```

## Advanced HttpClient Patterns

### 1. Request and Response Transformation

```typescript
import { map, switchMap, catchError } from 'rxjs/operators';
import { throwError, of } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class TransformationService {
  constructor(private http: HttpClient) {}

  // Transform request data
  createUserWithValidation(userData: UserFormData): Observable<User> {
    // Transform form data to API format
    const apiUser = this.transformToApiFormat(userData);
    
    return this.http.post<ApiUserResponse>('/api/users', apiUser).pipe(
      map(response => this.transformFromApiFormat(response)),
      catchError(this.handleValidationErrors)
    );
  }

  // Combine multiple requests
  getUserWithPosts(userId: number): Observable<UserWithPosts> {
    return this.http.get<User>(`/api/users/${userId}`).pipe(
      switchMap(user => 
        this.http.get<Post[]>(`/api/users/${userId}/posts`).pipe(
          map(posts => ({ ...user, posts }))
        )
      )
    );
  }

  // Transform response data
  getFormattedUsers(): Observable<FormattedUser[]> {
    return this.http.get<User[]>('/api/users').pipe(
      map(users => users.map(user => ({
        ...user,
        displayName: `${user.firstName} ${user.lastName}`,
        initials: this.getInitials(user.firstName, user.lastName),
        isActive: user.lastLogin > Date.now() - 30 * 24 * 60 * 60 * 1000
      })))
    );
  }

  private transformToApiFormat(formData: UserFormData): ApiUser {
    return {
      first_name: formData.firstName,
      last_name: formData.lastName,
      email_address: formData.email,
      birth_date: formData.birthDate?.toISOString()
    };
  }

  private transformFromApiFormat(apiResponse: ApiUserResponse): User {
    return {
      id: apiResponse.id,
      firstName: apiResponse.first_name,
      lastName: apiResponse.last_name,
      email: apiResponse.email_address,
      birthDate: apiResponse.birth_date ? new Date(apiResponse.birth_date) : null
    };
  }

  private handleValidationErrors(error: any): Observable<never> {
    if (error.status === 422 && error.error.validation_errors) {
      const formattedErrors = this.formatValidationErrors(error.error.validation_errors);
      return throwError(() => ({ type: 'validation', errors: formattedErrors }));
    }
    return throwError(() => error);
  }

  private getInitials(firstName: string, lastName: string): string {
    return `${firstName.charAt(0)}${lastName.charAt(0)}`.toUpperCase();
  }
}
```

### 2. HTTP Interceptors with Observables

**Authentication Interceptor:**
```typescript
import { Injectable } from '@angular/core';
import { HttpInterceptor, HttpRequest, HttpHandler, HttpEvent } from '@angular/common/http';
import { Observable, throwError, BehaviorSubject } from 'rxjs';
import { catchError, switchMap, filter, take } from 'rxjs/operators';

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  private isRefreshing = false;
  private refreshTokenSubject = new BehaviorSubject<any>(null);

  constructor(private authService: AuthService) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    // Add auth token to request
    const authToken = this.authService.getToken();
    const authReq = authToken ? this.addToken(req, authToken) : req;

    return next.handle(authReq).pipe(
      catchError(error => {
        if (error.status === 401 && authToken) {
          return this.handle401Error(authReq, next);
        }
        return throwError(() => error);
      })
    );
  }

  private addToken(request: HttpRequest<any>, token: string): HttpRequest<any> {
    return request.clone({
      setHeaders: {
        Authorization: `Bearer ${token}`
      }
    });
  }

  private handle401Error(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    if (!this.isRefreshing) {
      this.isRefreshing = true;
      this.refreshTokenSubject.next(null);

      return this.authService.refreshToken().pipe(
        switchMap((token: any) => {
          this.isRefreshing = false;
          this.refreshTokenSubject.next(token.accessToken);
          return next.handle(this.addToken(request, token.accessToken));
        }),
        catchError(error => {
          this.isRefreshing = false;
          this.authService.logout();
          return throwError(() => error);
        })
      );
    } else {
      return this.refreshTokenSubject.pipe(
        filter(token => token != null),
        take(1),
        switchMap(jwt => next.handle(this.addToken(request, jwt)))
      );
    }
  }
}
```

**Loading Interceptor:**
```typescript
@Injectable()
export class LoadingInterceptor implements HttpInterceptor {
  constructor(private loadingService: LoadingService) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    // Skip loading for certain requests
    if (req.headers.has('X-Skip-Loading')) {
      return next.handle(req);
    }

    this.loadingService.setLoading(true);

    return next.handle(req).pipe(
      finalize(() => this.loadingService.setLoading(false))
    );
  }
}
```

### 3. Caching Strategies

**HTTP Cache Service:**
```typescript
@Injectable({
  providedIn: 'root'
})
export class HttpCacheService {
  private cache = new Map<string, { data: any; timestamp: number; ttl: number }>();

  constructor(private http: HttpClient) {}

  // Get with caching
  get<T>(url: string, ttlMs: number = 300000): Observable<T> { // 5 minutes default TTL
    const cacheKey = this.getCacheKey(url);
    const cached = this.cache.get(cacheKey);

    if (cached && Date.now() - cached.timestamp < cached.ttl) {
      return of(cached.data);
    }

    return this.http.get<T>(url).pipe(
      tap(data => {
        this.cache.set(cacheKey, {
          data,
          timestamp: Date.now(),
          ttl: ttlMs
        });
      })
    );
  }

  // Invalidate cache for specific URL
  invalidate(url: string): void {
    const cacheKey = this.getCacheKey(url);
    this.cache.delete(cacheKey);
  }

  // Clear all cache
  clearCache(): void {
    this.cache.clear();
  }

  // Cache with shareReplay
  getCachedObservable<T>(url: string): Observable<T> {
    return this.http.get<T>(url).pipe(
      shareReplay(1)
    );
  }

  private getCacheKey(url: string): string {
    return btoa(url); // Simple base64 encoding
  }
}
```

**Smart Caching Service:**
```typescript
@Injectable({
  providedIn: 'root'
})
export class SmartCacheService {
  private observableCache = new Map<string, Observable<any>>();
  private dataCache = new Map<string, { data: any; timestamp: number }>();

  constructor(private http: HttpClient) {}

  // Get with observable caching (prevents duplicate requests)
  get<T>(url: string, forceRefresh: boolean = false): Observable<T> {
    const cacheKey = this.getCacheKey(url);

    if (!forceRefresh && this.observableCache.has(cacheKey)) {
      return this.observableCache.get(cacheKey)!;
    }

    const request$ = this.http.get<T>(url).pipe(
      tap(data => {
        this.dataCache.set(cacheKey, {
          data,
          timestamp: Date.now()
        });
      }),
      share(),
      finalize(() => {
        // Remove from observable cache when complete
        this.observableCache.delete(cacheKey);
      })
    );

    this.observableCache.set(cacheKey, request$);
    return request$;
  }

  // Get cached data or fetch if not available
  getWithFallback<T>(url: string, maxAge: number = 300000): Observable<T> {
    const cacheKey = this.getCacheKey(url);
    const cached = this.dataCache.get(cacheKey);

    if (cached && Date.now() - cached.timestamp < maxAge) {
      return of(cached.data);
    }

    return this.get<T>(url);
  }

  private getCacheKey(url: string): string {
    return url;
  }
}
```

### 4. Error Handling Strategies

**Comprehensive Error Handler:**
```typescript
@Injectable({
  providedIn: 'root'
})
export class HttpErrorHandler {
  constructor(private notificationService: NotificationService) {}

  handleError<T>(operation = 'operation', result?: T) {
    return (error: any): Observable<T> => {
      console.error(`${operation} failed:`, error);

      switch (error.status) {
        case 400:
          this.handleBadRequest(error);
          break;
        case 401:
          this.handleUnauthorized(error);
          break;
        case 403:
          this.handleForbidden(error);
          break;
        case 404:
          this.handleNotFound(error);
          break;
        case 422:
          this.handleValidationError(error);
          break;
        case 500:
          this.handleServerError(error);
          break;
        default:
          this.handleGenericError(error);
      }

      // Return a safe fallback value
      return of(result as T);
    };
  }

  private handleBadRequest(error: any): void {
    this.notificationService.showError('Invalid request. Please check your input.');
  }

  private handleUnauthorized(error: any): void {
    this.notificationService.showError('You are not authorized. Please log in.');
    // Redirect to login
  }

  private handleForbidden(error: any): void {
    this.notificationService.showError('You do not have permission to perform this action.');
  }

  private handleNotFound(error: any): void {
    this.notificationService.showError('The requested resource was not found.');
  }

  private handleValidationError(error: any): void {
    const messages = this.extractValidationMessages(error);
    messages.forEach(message => this.notificationService.showError(message));
  }

  private handleServerError(error: any): void {
    this.notificationService.showError('A server error occurred. Please try again later.');
  }

  private handleGenericError(error: any): void {
    this.notificationService.showError('An unexpected error occurred.');
  }

  private extractValidationMessages(error: any): string[] {
    if (error.error && error.error.errors) {
      return Object.values(error.error.errors).flat() as string[];
    }
    return ['Validation failed'];
  }
}

// Usage in service
@Injectable({
  providedIn: 'root'
})
export class UserService {
  constructor(
    private http: HttpClient,
    private errorHandler: HttpErrorHandler
  ) {}

  getUsers(): Observable<User[]> {
    return this.http.get<User[]>('/api/users').pipe(
      catchError(this.errorHandler.handleError<User[]>('getUsers', []))
    );
  }
}
```

### 5. Retry Strategies

**Advanced Retry Logic:**
```typescript
@Injectable({
  providedIn: 'root'
})
export class RetryService {
  constructor(private http: HttpClient) {}

  // Exponential backoff retry
  getWithExponentialBackoff<T>(url: string): Observable<T> {
    return this.http.get<T>(url).pipe(
      retryWhen(errors =>
        errors.pipe(
          scan((retryCount, error) => {
            if (retryCount >= 3 || error.status < 500) {
              throw error;
            }
            return retryCount + 1;
          }, 0),
          delay(1000), // Wait 1 second before retry
          tap(retryCount => console.log(`Retry attempt ${retryCount}`))
        )
      )
    );
  }

  // Conditional retry
  getWithConditionalRetry<T>(url: string): Observable<T> {
    return this.http.get<T>(url).pipe(
      retryWhen(errors =>
        errors.pipe(
          tap(error => console.log('Error occurred:', error)),
          delayWhen((error, index) => {
            // Only retry for server errors (5xx) and network errors
            if (error.status >= 500 || error.status === 0) {
              const delayTime = Math.pow(2, index) * 1000; // Exponential backoff
              console.log(`Retrying in ${delayTime}ms...`);
              return timer(delayTime);
            }
            // Don't retry for client errors (4xx)
            throw error;
          }),
          take(3) // Maximum 3 retries
        )
      )
    );
  }

  // Custom retry with circuit breaker pattern
  getWithCircuitBreaker<T>(url: string): Observable<T> {
    return this.http.get<T>(url).pipe(
      retryWhen(errors => {
        let consecutiveFailures = 0;
        
        return errors.pipe(
          scan((acc, error) => {
            consecutiveFailures++;
            
            if (consecutiveFailures >= 5) {
              throw new Error('Circuit breaker opened - too many failures');
            }
            
            return acc + 1;
          }, 0),
          delayWhen(() => timer(2000))
        );
      })
    );
  }
}
```

### 6. Concurrent Request Management

**Managing Multiple HTTP Requests:**
```typescript
@Injectable({
  providedIn: 'root'
})
export class ConcurrentRequestService {
  constructor(private http: HttpClient) {}

  // Parallel requests with forkJoin
  getUserDashboardData(userId: number): Observable<UserDashboard> {
    return forkJoin({
      user: this.http.get<User>(`/api/users/${userId}`),
      posts: this.http.get<Post[]>(`/api/users/${userId}/posts`),
      followers: this.http.get<User[]>(`/api/users/${userId}/followers`),
      stats: this.http.get<UserStats>(`/api/users/${userId}/stats`)
    }).pipe(
      map(({ user, posts, followers, stats }) => ({
        user,
        posts,
        followers,
        stats,
        lastUpdated: new Date()
      }))
    );
  }

  // Sequential requests with dependency
  createUserWithProfile(userData: CreateUserData): Observable<UserProfile> {
    return this.http.post<User>('/api/users', userData).pipe(
      switchMap(user =>
        this.http.post<Profile>(`/api/users/${user.id}/profile`, userData.profile).pipe(
          map(profile => ({ user, profile }))
        )
      )
    );
  }

  // Batch operations
  updateMultipleUsers(updates: UserUpdate[]): Observable<User[]> {
    const updateRequests = updates.map(update =>
      this.http.put<User>(`/api/users/${update.id}`, update.data)
    );

    return forkJoin(updateRequests);
  }

  // Progressive loading
  loadUserDataProgressively(userId: number): Observable<Partial<UserDashboard>> {
    const user$ = this.http.get<User>(`/api/users/${userId}`);
    const posts$ = this.http.get<Post[]>(`/api/users/${userId}/posts`);
    const stats$ = this.http.get<UserStats>(`/api/users/${userId}/stats`);

    return merge(
      user$.pipe(map(user => ({ user }))),
      posts$.pipe(map(posts => ({ posts }))),
      stats$.pipe(map(stats => ({ stats })))
    ).pipe(
      scan((acc, data) => ({ ...acc, ...data }), {} as Partial<UserDashboard>)
    );
  }

  // Request cancellation
  searchWithCancellation(query: string): Observable<SearchResult[]> {
    return new Observable<SearchResult[]>(subscriber => {
      const request$ = this.http.get<SearchResult[]>(`/api/search?q=${query}`);
      const subscription = request$.subscribe(subscriber);

      // Return cleanup function
      return () => {
        subscription.unsubscribe();
        console.log('Search request cancelled');
      };
    });
  }
}
```

### 7. File Upload and Download

**File Upload with Progress:**
```typescript
@Injectable({
  providedIn: 'root'
})
export class FileService {
  constructor(private http: HttpClient) {}

  // Upload file with progress tracking
  uploadFile(file: File, url: string): Observable<UploadProgress> {
    const formData = new FormData();
    formData.append('file', file);

    const req = new HttpRequest('POST', url, formData, {
      reportProgress: true
    });

    return this.http.request(req).pipe(
      map(event => {
        switch (event.type) {
          case HttpEventType.UploadProgress:
            const progress = Math.round(100 * event.loaded / (event.total || 1));
            return { status: 'progress', progress };
          
          case HttpEventType.Response:
            return { status: 'complete', result: event.body };
          
          default:
            return { status: 'uploading', progress: 0 };
        }
      })
    );
  }

  // Multiple file upload
  uploadMultipleFiles(files: File[]): Observable<MultipleUploadProgress> {
    const uploads = files.map((file, index) =>
      this.uploadFile(file, '/api/upload').pipe(
        map(progress => ({ index, file: file.name, ...progress }))
      )
    );

    return merge(...uploads).pipe(
      scan((acc, upload) => {
        acc.files[upload.index] = upload;
        acc.overall = this.calculateOverallProgress(acc.files);
        return acc;
      }, { files: [], overall: 0 } as MultipleUploadProgress)
    );
  }

  // Download file
  downloadFile(url: string, filename: string): Observable<DownloadProgress> {
    return this.http.get(url, {
      responseType: 'blob',
      reportProgress: true,
      observe: 'events'
    }).pipe(
      map(event => {
        switch (event.type) {
          case HttpEventType.DownloadProgress:
            const progress = Math.round(100 * event.loaded / (event.total || 1));
            return { status: 'progress', progress };
          
          case HttpEventType.Response:
            this.saveFile(event.body as Blob, filename);
            return { status: 'complete', progress: 100 };
          
          default:
            return { status: 'downloading', progress: 0 };
        }
      })
    );
  }

  private saveFile(blob: Blob, filename: string): void {
    const url = window.URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.download = filename;
    link.click();
    window.URL.revokeObjectURL(url);
  }

  private calculateOverallProgress(files: any[]): number {
    const total = files.reduce((sum, file) => sum + (file.progress || 0), 0);
    return Math.round(total / files.length);
  }
}

interface UploadProgress {
  status: 'uploading' | 'progress' | 'complete';
  progress: number;
  result?: any;
}

interface MultipleUploadProgress {
  files: any[];
  overall: number;
}

interface DownloadProgress {
  status: 'downloading' | 'progress' | 'complete';
  progress: number;
}
```

## Real-World Integration Patterns

### 1. Data Service with CRUD Operations

```typescript
@Injectable({
  providedIn: 'root'
})
export class DataService<T> {
  constructor(
    private http: HttpClient,
    private errorHandler: HttpErrorHandler,
    private cacheService: SmartCacheService
  ) {}

  // Generic CRUD operations
  getAll(endpoint: string, useCache: boolean = true): Observable<T[]> {
    const request$ = this.http.get<T[]>(endpoint);
    
    return useCache 
      ? this.cacheService.getWithFallback(endpoint)
      : request$.pipe(
          catchError(this.errorHandler.handleError<T[]>('getAll', []))
        );
  }

  getById(endpoint: string, id: number | string): Observable<T> {
    return this.http.get<T>(`${endpoint}/${id}`).pipe(
      catchError(this.errorHandler.handleError<T>('getById'))
    );
  }

  create(endpoint: string, item: Partial<T>): Observable<T> {
    return this.http.post<T>(endpoint, item).pipe(
      tap(() => this.cacheService.invalidate(endpoint)),
      catchError(this.errorHandler.handleError<T>('create'))
    );
  }

  update(endpoint: string, id: number | string, item: Partial<T>): Observable<T> {
    return this.http.put<T>(`${endpoint}/${id}`, item).pipe(
      tap(() => {
        this.cacheService.invalidate(endpoint);
        this.cacheService.invalidate(`${endpoint}/${id}`);
      }),
      catchError(this.errorHandler.handleError<T>('update'))
    );
  }

  delete(endpoint: string, id: number | string): Observable<void> {
    return this.http.delete<void>(`${endpoint}/${id}`).pipe(
      tap(() => {
        this.cacheService.invalidate(endpoint);
        this.cacheService.invalidate(`${endpoint}/${id}`);
      }),
      catchError(this.errorHandler.handleError<void>('delete'))
    );
  }
}
```

### 2. Component Integration Example

```typescript
@Component({
  selector: 'app-user-management',
  template: `
    <div class="user-management">
      <!-- Loading indicator -->
      <div *ngIf="loading$ | async" class="loading">
        Loading users...
      </div>

      <!-- Error display -->
      <div *ngIf="error$ | async as error" class="error">
        {{ error }}
      </div>

      <!-- User list -->
      <div class="user-list">
        <div *ngFor="let user of users$ | async" class="user-item">
          <span>{{ user.name }}</span>
          <button (click)="editUser(user)">Edit</button>
          <button (click)="deleteUser(user.id)">Delete</button>
        </div>
      </div>

      <!-- Add user form -->
      <form [formGroup]="userForm" (ngSubmit)="addUser()">
        <input formControlName="name" placeholder="Name">
        <input formControlName="email" placeholder="Email">
        <button type="submit" [disabled]="userForm.invalid || (submitting$ | async)">
          {{ (submitting$ | async) ? 'Adding...' : 'Add User' }}
        </button>
      </form>
    </div>
  `
})
export class UserManagementComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  private refreshSubject = new Subject<void>();

  userForm = this.fb.group({
    name: ['', Validators.required],
    email: ['', [Validators.required, Validators.email]]
  });

  // Observable streams
  users$ = this.refreshSubject.pipe(
    startWith(null),
    switchMap(() => this.userService.getUsers()),
    shareReplay(1)
  );

  loading$ = new BehaviorSubject(false);
  submitting$ = new BehaviorSubject(false);
  error$ = new Subject<string>();

  constructor(
    private userService: UserService,
    private fb: FormBuilder
  ) {}

  ngOnInit() {
    this.loadUsers();
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }

  loadUsers() {
    this.loading$.next(true);
    this.userService.getUsers().pipe(
      takeUntil(this.destroy$),
      finalize(() => this.loading$.next(false))
    ).subscribe({
      next: () => this.refreshSubject.next(),
      error: (error) => this.error$.next('Failed to load users')
    });
  }

  addUser() {
    if (this.userForm.valid) {
      this.submitting$.next(true);
      
      this.userService.createUser(this.userForm.value).pipe(
        takeUntil(this.destroy$),
        finalize(() => this.submitting$.next(false))
      ).subscribe({
        next: () => {
          this.userForm.reset();
          this.refreshSubject.next();
        },
        error: (error) => this.error$.next('Failed to add user')
      });
    }
  }

  editUser(user: User) {
    // Implementation for editing
  }

  deleteUser(id: number) {
    this.userService.deleteUser(id).pipe(
      takeUntil(this.destroy$)
    ).subscribe({
      next: () => this.refreshSubject.next(),
      error: (error) => this.error$.next('Failed to delete user')
    });
  }
}
```

## Best Practices

### 1. Type Safety
```typescript
// ✅ Always use TypeScript interfaces
interface ApiResponse<T> {
  data: T;
  message: string;
  status: number;
}

// ✅ Type HTTP responses
getUsers(): Observable<ApiResponse<User[]>> {
  return this.http.get<ApiResponse<User[]>>('/api/users');
}
```

### 2. Error Handling
```typescript
// ✅ Comprehensive error handling
return this.http.get<User[]>('/api/users').pipe(
  retry(3),
  catchError(error => {
    this.logError(error);
    this.showUserFriendlyMessage(error);
    return of([]); // Provide fallback
  })
);
```

### 3. Resource Management
```typescript
// ✅ Always unsubscribe
ngOnDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

## Common Pitfalls

### 1. Memory Leaks
```typescript
// ❌ Bad: No unsubscription
ngOnInit() {
  this.http.get('/api/data').subscribe(data => {
    // This subscription never gets cleaned up!
  });
}

// ✅ Good: Proper cleanup
ngOnInit() {
  this.http.get('/api/data').pipe(
    takeUntil(this.destroy$)
  ).subscribe(data => {
    // Subscription is properly managed
  });
}
```

### 2. Nested Subscriptions
```typescript
// ❌ Bad: Nested subscriptions
this.http.get<User>('/api/user').subscribe(user => {
  this.http.get<Post[]>(`/api/posts/${user.id}`).subscribe(posts => {
    // Nested subscription hell!
  });
});

// ✅ Good: Use operators
this.http.get<User>('/api/user').pipe(
  switchMap(user => this.http.get<Post[]>(`/api/posts/${user.id}`))
).subscribe(posts => {
  // Clean and readable
});
```

## Summary

HttpClient with Observables provides powerful patterns for HTTP communication:

- **Basic Operations**: GET, POST, PUT, DELETE with proper typing
- **Advanced Patterns**: Caching, retries, error handling, interceptors
- **Concurrent Requests**: Parallel and sequential request management
- **File Operations**: Upload/download with progress tracking
- **Real-world Integration**: Complete CRUD services with Angular components

Key principles:
- Always handle errors gracefully
- Use proper TypeScript typing
- Implement caching strategies
- Manage subscriptions properly
- Use appropriate operators for different scenarios
- Provide user feedback for loading states
