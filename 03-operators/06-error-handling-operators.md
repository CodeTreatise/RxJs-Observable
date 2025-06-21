# Error Handling Operators

## Overview

Error handling is crucial in reactive programming, especially in real-world applications where network requests, user input, and external services can fail. RxJS provides powerful error handling operators that allow you to gracefully handle errors, implement retry strategies, and provide fallback mechanisms to ensure your application remains stable and user-friendly.

## Learning Objectives

After completing this lesson, you will be able to:
- Handle errors gracefully in Observable streams
- Implement retry strategies for failed operations
- Provide fallback values and recovery mechanisms
- Build resilient applications with proper error boundaries
- Apply error handling patterns in Angular applications

## Core Error Handling Operators

### 1. catchError() - Handle and Recover from Errors

The `catchError()` operator catches errors in the Observable stream and allows you to return a new Observable or throw a different error.

```typescript
import { of, throwError } from 'rxjs';
import { catchError, map } from 'rxjs/operators';

// Basic error handling
const source$ = throwError('Something went wrong!');
const handled$ = source$.pipe(
  catchError(error => {
    console.error('Caught error:', error);
    return of('Default value');
  })
);

handled$.subscribe(
  value => console.log('Value:', value),
  error => console.log('Error:', error) // Won't be called
);
// Output: "Value: Default value"

// Angular HTTP error handling
@Injectable()
export class ApiService {
  constructor(private http: HttpClient) {}

  getUser(id: string): Observable<User> {
    return this.http.get<User>(`/api/users/${id}`).pipe(
      catchError(error => {
        console.error('Failed to fetch user:', error);
        
        if (error.status === 404) {
          return of(this.createGuestUser());
        } else if (error.status === 500) {
          this.notificationService.showError('Server error occurred');
          return throwError('Server temporarily unavailable');
        } else {
          return throwError('Failed to load user data');
        }
      })
    );
  }

  private createGuestUser(): User {
    return {
      id: 'guest',
      name: 'Guest User',
      email: 'guest@example.com',
      role: 'guest'
    };
  }
}

// Form validation with error recovery
@Component({})
export class FormComponent {
  userForm = new FormGroup({
    email: new FormControl('', [Validators.required, Validators.email])
  });

  emailValidation$ = this.userForm.get('email')!.valueChanges.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap(email => 
      this.validationService.validateEmail(email).pipe(
        map(result => ({ isValid: true, message: 'Email is available' })),
        catchError(error => {
          if (error.status === 409) {
            return of({ isValid: false, message: 'Email already exists' });
          } else {
            return of({ isValid: false, message: 'Validation temporarily unavailable' });
          }
        })
      )
    )
  );
}
```

**Marble Diagram:**
```
Input:  -1-2-X
catchError(err => of('fallback'))
Output: -1-2-fallback|
```

### 2. retry() - Retry Failed Operations

The `retry()` operator resubscribes to the source Observable when an error occurs, attempting the operation again.

```typescript
import { interval, throwError } from 'rxjs';
import { mergeMap, retry } from 'rxjs/operators';

// Basic retry
const unreliableSource$ = interval(1000).pipe(
  mergeMap(value => {
    if (Math.random() < 0.7) {
      return throwError('Random failure');
    }
    return of(value);
  }),
  retry(3) // Retry up to 3 times
);

// Angular HTTP with retry
@Injectable()
export class DataService {
  constructor(private http: HttpClient) {}

  fetchCriticalData(): Observable<CriticalData> {
    return this.http.get<CriticalData>('/api/critical-data').pipe(
      retry(3), // Retry 3 times before giving up
      catchError(error => {
        console.error('All retry attempts failed:', error);
        return throwError('Critical data unavailable after multiple attempts');
      })
    );
  }

  // Conditional retry based on error type
  fetchDataWithConditionalRetry(): Observable<Data> {
    return this.http.get<Data>('/api/data').pipe(
      retryWhen(errors => 
        errors.pipe(
          mergeMap((error, index) => {
            // Only retry on network errors, not client errors
            if (error.status >= 500 && index < 2) {
              console.log(`Retrying... Attempt ${index + 1}`);
              return of(error).pipe(delay(1000 * (index + 1))); // Exponential backoff
            }
            return throwError(error);
          })
        )
      )
    );
  }
}

// File upload with retry
@Component({})
export class FileUploadComponent {
  uploadFile(file: File): Observable<UploadResult> {
    return this.http.post<UploadResult>('/api/upload', this.createFormData(file)).pipe(
      retry({
        count: 3,
        delay: (error, retryCount) => {
          console.log(`Upload failed, retrying... Attempt ${retryCount}`);
          return timer(Math.pow(2, retryCount) * 1000); // Exponential backoff
        }
      }),
      catchError(error => {
        this.notificationService.showError(`Failed to upload ${file.name}`);
        return of({ success: false, fileName: file.name, error: error.message });
      })
    );
  }

  private createFormData(file: File): FormData {
    const formData = new FormData();
    formData.append('file', file);
    return formData;
  }
}
```

**Marble Diagram:**
```
Input:  -1-2-X
retry(2)
Retry 1: -1-2-X
Retry 2: -1-2-X
Output:  -1-2-X (error after all retries)
```

### 3. retryWhen() - Custom Retry Logic

The `retryWhen()` operator provides more control over retry behavior, allowing custom logic for when and how to retry.

```typescript
import { timer, throwError } from 'rxjs';
import { retryWhen, mergeMap, take } from 'rxjs/operators';

// Exponential backoff retry
const exponentialBackoffRetry = (maxRetries: number = 3) => {
  return retryWhen(errors => 
    errors.pipe(
      mergeMap((error, index) => {
        const retryAttempt = index + 1;
        
        if (retryAttempt > maxRetries) {
          return throwError(error);
        }
        
        const delay = Math.pow(2, retryAttempt) * 1000;
        console.log(`Retry attempt ${retryAttempt} after ${delay}ms`);
        
        return timer(delay);
      })
    )
  );
};

// Angular service with sophisticated retry logic
@Injectable()
export class RobustApiService {
  constructor(private http: HttpClient) {}

  fetchDataWithSmartRetry<T>(url: string): Observable<T> {
    return this.http.get<T>(url).pipe(
      retryWhen(errors => 
        errors.pipe(
          mergeMap((error, retryCount) => {
            // Don't retry client errors (4xx)
            if (error.status >= 400 && error.status < 500) {
              return throwError(error);
            }
            
            // Don't retry after 3 attempts
            if (retryCount >= 3) {
              return throwError(error);
            }
            
            // Calculate delay with jitter to prevent thundering herd
            const baseDelay = Math.pow(2, retryCount) * 1000;
            const jitter = Math.random() * 1000;
            const delay = baseDelay + jitter;
            
            console.log(`Retrying request to ${url} in ${delay}ms`);
            return timer(delay);
          })
        )
      ),
      catchError(error => {
        this.errorService.logError(error, url);
        return throwError(`Failed to fetch data from ${url}`);
      })
    );
  }

  // Retry with circuit breaker pattern
  private circuitBreakerState = new Map<string, { failures: number; lastFailure: number }>();

  fetchWithCircuitBreaker<T>(url: string): Observable<T> {
    const state = this.circuitBreakerState.get(url) || { failures: 0, lastFailure: 0 };
    
    // Circuit is open if too many recent failures
    if (state.failures >= 3 && Date.now() - state.lastFailure < 60000) {
      return throwError('Circuit breaker is open');
    }
    
    return this.http.get<T>(url).pipe(
      tap(() => {
        // Reset on success
        this.circuitBreakerState.set(url, { failures: 0, lastFailure: 0 });
      }),
      retryWhen(errors => 
        errors.pipe(
          mergeMap((error, retryCount) => {
            if (retryCount >= 2) {
              // Update circuit breaker state
              state.failures++;
              state.lastFailure = Date.now();
              this.circuitBreakerState.set(url, state);
              return throwError(error);
            }
            
            return timer(1000 * (retryCount + 1));
          })
        )
      )
    );
  }
}
```

### 4. onErrorResumeNext() - Continue with Alternative Observable

The `onErrorResumeNext()` operator continues with alternative Observables when the source Observable encounters an error.

```typescript
import { onErrorResumeNext, of, throwError } from 'rxjs';

// Continue with alternatives
const primary$ = throwError('Primary failed');
const secondary$ = throwError('Secondary failed');
const fallback$ = of('Fallback value');

const resilient$ = onErrorResumeNext(primary$, secondary$, fallback$);
resilient$.subscribe(value => console.log(value));
// Output: "Fallback value"

// Data source fallback chain
@Injectable()
export class FallbackDataService {
  constructor(
    private http: HttpClient,
    private cacheService: CacheService,
    private localStorageService: LocalStorageService
  ) {}

  getUserData(userId: string): Observable<UserData> {
    const primarySource$ = this.http.get<UserData>(`/api/users/${userId}`);
    const cacheSource$ = this.cacheService.get<UserData>(`user-${userId}`);
    const localSource$ = this.localStorageService.get<UserData>(`user-${userId}`);
    const defaultSource$ = of(this.createDefaultUserData(userId));

    return onErrorResumeNext(
      primarySource$,
      cacheSource$,
      localSource$,
      defaultSource$
    ).pipe(
      take(1), // Take the first successful result
      tap(data => console.log('Data source used:', this.getDataSource(data)))
    );
  }

  private createDefaultUserData(userId: string): UserData {
    return {
      id: userId,
      name: 'Unknown User',
      email: '',
      isDefault: true
    };
  }

  private getDataSource(data: UserData): string {
    if (data.isDefault) return 'default';
    // Additional logic to determine source
    return 'unknown';
  }
}
```

### 5. throwError() - Create Error Observable

The `throwError()` operator creates an Observable that immediately emits an error.

```typescript
import { throwError, of } from 'rxjs';
import { mergeMap } from 'rxjs/operators';

// Conditional error throwing
const conditionalError$ = of(1, 2, 3, 4, 5).pipe(
  mergeMap(value => {
    if (value === 4) {
      return throwError(`Value ${value} is not allowed`);
    }
    return of(value);
  }),
  catchError(error => {
    console.error('Caught:', error);
    return of('Error handled');
  })
);

// Angular validation service
@Injectable()
export class ValidationService {
  validateUserInput(input: UserInput): Observable<ValidationResult> {
    // Client-side validation
    if (!input.email || !this.isValidEmail(input.email)) {
      return throwError('Invalid email format');
    }
    
    if (!input.password || input.password.length < 8) {
      return throwError('Password must be at least 8 characters');
    }
    
    // Server-side validation
    return this.http.post<ValidationResult>('/api/validate', input).pipe(
      mergeMap(result => {
        if (!result.isValid) {
          return throwError(result.errorMessage);
        }
        return of(result);
      })
    );
  }

  private isValidEmail(email: string): boolean {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(email);
  }
}

// Error boundary for components
@Injectable()
export class ErrorBoundaryService {
  handleComponentError(error: any, component: string): Observable<never> {
    this.logError(error, component);
    this.notificationService.showError(`An error occurred in ${component}`);
    
    // Navigate to error page for critical errors
    if (this.isCriticalError(error)) {
      this.router.navigate(['/error']);
    }
    
    return throwError(`Component error in ${component}: ${error.message}`);
  }

  private logError(error: any, context: string): void {
    console.error(`[${context}] Error:`, error);
    // Send to error tracking service
    this.errorTrackingService.captureError(error, { context });
  }

  private isCriticalError(error: any): boolean {
    return error.name === 'ChunkLoadError' || 
           error.message?.includes('Loading chunk') ||
           error.status >= 500;
  }
}
```

## Advanced Error Handling Patterns

### 1. Global Error Handler

```typescript
// Global error interceptor
@Injectable()
export class ErrorInterceptor implements HttpInterceptor {
  constructor(
    private notificationService: NotificationService,
    private authService: AuthService,
    private router: Router
  ) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    return next.handle(req).pipe(
      catchError((error: HttpErrorResponse) => {
        let errorMessage = 'An error occurred';
        
        if (error.error instanceof ErrorEvent) {
          // Client-side error
          errorMessage = `Client Error: ${error.error.message}`;
        } else {
          // Server-side error
          switch (error.status) {
            case 401:
              this.authService.logout();
              this.router.navigate(['/login']);
              errorMessage = 'Please log in again';
              break;
            case 403:
              errorMessage = 'You do not have permission to perform this action';
              break;
            case 404:
              errorMessage = 'The requested resource was not found';
              break;
            case 500:
              errorMessage = 'Server error occurred. Please try again later';
              break;
            default:
              errorMessage = `Server Error: ${error.status} - ${error.message}`;
          }
        }
        
        this.notificationService.showError(errorMessage);
        return throwError(errorMessage);
      })
    );
  }
}

// Global error handler
@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  constructor(
    private errorService: ErrorService,
    private notificationService: NotificationService
  ) {}

  handleError(error: any): void {
    console.error('Global error:', error);
    
    // Log error to external service
    this.errorService.logError(error);
    
    // Show user-friendly message
    if (this.isUserFacingError(error)) {
      this.notificationService.showError(this.getUserFriendlyMessage(error));
    }
  }

  private isUserFacingError(error: any): boolean {
    return !error.message?.includes('ExpressionChangedAfterItHasBeenCheckedError');
  }

  private getUserFriendlyMessage(error: any): string {
    if (error.name === 'ChunkLoadError') {
      return 'Please refresh the page to load the latest version';
    }
    return 'An unexpected error occurred. Please try again';
  }
}
```

### 2. Resilient Data Loading

```typescript
@Component({})
export class ResilientDataComponent implements OnInit {
  private destroy$ = new Subject<void>();
  private refreshTrigger$ = new Subject<void>();

  // Multi-layered error handling for data loading
  data$ = this.refreshTrigger$.pipe(
    startWith(null), // Initial load
    switchMap(() => this.loadDataWithResilience()),
    shareReplay(1),
    takeUntil(this.destroy$)
  );

  isLoading$ = new BehaviorSubject<boolean>(false);
  error$ = new BehaviorSubject<string | null>(null);

  ngOnInit() {
    this.refresh();
  }

  refresh() {
    this.refreshTrigger$.next();
  }

  private loadDataWithResilience(): Observable<Data[]> {
    this.isLoading$.next(true);
    this.error$.next(null);

    return this.dataService.getData().pipe(
      // Primary strategy: retry with exponential backoff
      retryWhen(errors => 
        errors.pipe(
          mergeMap((error, retryCount) => {
            if (retryCount >= 3) {
              console.log('Switching to fallback data source');
              return this.loadFallbackData();
            }
            
            const delay = Math.pow(2, retryCount) * 1000;
            console.log(`Retrying in ${delay}ms...`);
            return timer(delay);
          }),
          take(4) // 3 retries + 1 fallback attempt
        )
      ),
      // Final fallback
      catchError(error => {
        console.error('All data loading strategies failed:', error);
        this.error$.next('Unable to load data. Please check your connection.');
        return of([]); // Return empty array as final fallback
      }),
      finalize(() => this.isLoading$.next(false))
    );
  }

  private loadFallbackData(): Observable<Data[]> {
    return forkJoin({
      cached: this.cacheService.getCachedData().pipe(
        catchError(() => of([]))
      ),
      local: this.localStorageService.getLocalData().pipe(
        catchError(() => of([]))
      )
    }).pipe(
      map(({ cached, local }) => {
        if (cached.length > 0) return cached;
        if (local.length > 0) return local;
        return this.getStaticFallbackData();
      })
    );
  }

  private getStaticFallbackData(): Data[] {
    return [
      { id: 'fallback', name: 'Offline Data', isOffline: true }
    ];
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### 3. Form Error Handling

```typescript
@Component({
  template: `
    <form [formGroup]="userForm" (ngSubmit)="onSubmit()">
      <div class="form-group">
        <input formControlName="email" placeholder="Email">
        <div class="error" *ngIf="emailError$ | async as error">
          {{ error }}
        </div>
      </div>

      <div class="form-group">
        <input formControlName="password" type="password" placeholder="Password">
        <div class="error" *ngIf="passwordError$ | async as error">
          {{ error }}
        </div>
      </div>

      <button 
        type="submit" 
        [disabled]="isSubmitting$ | async"
        [class.loading]="isSubmitting$ | async">
        {{ (isSubmitting$ | async) ? 'Submitting...' : 'Submit' }}
      </button>

      <div class="form-error" *ngIf="formError$ | async as error">
        {{ error }}
      </div>
    </form>
  `
})
export class RobustFormComponent {
  userForm = new FormGroup({
    email: new FormControl('', [Validators.required, Validators.email]),
    password: new FormControl('', [Validators.required, Validators.minLength(8)])
  });

  private submitAttempts$ = new Subject<FormValue>();
  private formError$ = new BehaviorSubject<string | null>(null);
  
  isSubmitting$ = new BehaviorSubject<boolean>(false);

  // Field-specific error handling
  emailError$ = combineLatest([
    this.userForm.get('email')!.valueChanges.pipe(startWith('')),
    this.userForm.get('email')!.statusChanges.pipe(startWith('INVALID'))
  ]).pipe(
    map(([value, status]) => {
      if (status === 'VALID' || !value) return null;
      
      const control = this.userForm.get('email')!;
      if (control.hasError('required')) return 'Email is required';
      if (control.hasError('email')) return 'Please enter a valid email';
      return null;
    })
  );

  passwordError$ = combineLatest([
    this.userForm.get('password')!.valueChanges.pipe(startWith('')),
    this.userForm.get('password')!.statusChanges.pipe(startWith('INVALID'))
  ]).pipe(
    map(([value, status]) => {
      if (status === 'VALID' || !value) return null;
      
      const control = this.userForm.get('password')!;
      if (control.hasError('required')) return 'Password is required';
      if (control.hasError('minlength')) return 'Password must be at least 8 characters';
      return null;
    })
  );

  // Form submission with error handling
  submitResult$ = this.submitAttempts$.pipe(
    tap(() => {
      this.isSubmitting$.next(true);
      this.formError$.next(null);
    }),
    switchMap(formValue => 
      this.userService.createUser(formValue).pipe(
        map(result => ({ success: true, result })),
        retry(2), // Retry failed submissions
        catchError(error => {
          let errorMessage = 'Submission failed. Please try again.';
          
          if (error.status === 409) {
            errorMessage = 'Email already exists. Please use a different email.';
          } else if (error.status === 422) {
            errorMessage = 'Please check your input and try again.';
          } else if (error.status >= 500) {
            errorMessage = 'Server error. Please try again later.';
          }
          
          return of({ success: false, error: errorMessage });
        })
      )
    ),
    tap(result => {
      this.isSubmitting$.next(false);
      if (!result.success) {
        this.formError$.next(result.error);
      }
    }),
    share()
  );

  constructor(private userService: UserService) {
    // Subscribe to submission results
    this.submitResult$.subscribe(result => {
      if (result.success) {
        this.onSubmissionSuccess(result.result);
      }
    });
  }

  onSubmit() {
    if (this.userForm.valid) {
      this.submitAttempts$.next(this.userForm.value);
    } else {
      this.markFormGroupTouched();
    }
  }

  private markFormGroupTouched() {
    Object.keys(this.userForm.controls).forEach(key => {
      this.userForm.get(key)?.markAsTouched();
    });
  }

  private onSubmissionSuccess(result: any) {
    this.userForm.reset();
    // Handle success (e.g., navigate, show success message)
  }
}
```

## Best Practices

### 1. Error Handling Strategy

```typescript
// ✅ Handle errors at appropriate levels
class DataService {
  // Service level: Log and transform errors
  getData(): Observable<Data> {
    return this.http.get<Data>('/api/data').pipe(
      catchError(error => {
        this.logger.error('Data fetch failed:', error);
        return throwError('Data unavailable');
      })
    );
  }
}

class Component {
  // Component level: Handle UI concerns
  data$ = this.dataService.getData().pipe(
    catchError(error => {
      this.showUserFriendlyError(error);
      return of([]); // Provide fallback for UI
    })
  );
}

// ✅ Provide meaningful error messages
const userFriendlyError$ = apiCall$.pipe(
  catchError(error => {
    let message = 'Something went wrong';
    
    if (error.status === 0) {
      message = 'Please check your internet connection';
    } else if (error.status === 404) {
      message = 'The requested data was not found';
    }
    
    return throwError(message);
  })
);
```

### 2. Performance Considerations

```typescript
// ✅ Use shareReplay for expensive error-prone operations
const expensiveData$ = this.expensiveOperation().pipe(
  retry(3),
  catchError(error => of(fallbackData)),
  shareReplay(1) // Cache result to avoid repeated attempts
);

// ✅ Implement circuit breaker for failing services
class CircuitBreakerService {
  private failures = 0;
  private lastFailureTime = 0;
  private readonly maxFailures = 5;
  private readonly resetTimeout = 60000;

  makeRequest<T>(operation: () => Observable<T>): Observable<T> {
    if (this.isCircuitOpen()) {
      return throwError('Circuit breaker is open');
    }

    return operation().pipe(
      tap(() => this.onSuccess()),
      catchError(error => {
        this.onFailure();
        return throwError(error);
      })
    );
  }

  private isCircuitOpen(): boolean {
    return this.failures >= this.maxFailures && 
           Date.now() - this.lastFailureTime < this.resetTimeout;
  }

  private onSuccess(): void {
    this.failures = 0;
  }

  private onFailure(): void {
    this.failures++;
    this.lastFailureTime = Date.now();
  }
}
```

### 3. Testing Error Scenarios

```typescript
// ✅ Test error handling
describe('DataService', () => {
  it('should handle network errors gracefully', () => {
    const errorResponse = new HttpErrorResponse({
      error: 'Network error',
      status: 0
    });

    spyOn(httpClient, 'get').and.returnValue(throwError(errorResponse));

    service.getData().subscribe(
      data => expect(data).toEqual([]), // Fallback data
      error => fail('Should have handled error gracefully')
    );
  });

  it('should retry failed requests', () => {
    const httpSpy = spyOn(httpClient, 'get')
      .and.returnValues(
        throwError('First failure'),
        throwError('Second failure'),
        of(mockData)
      );

    service.getDataWithRetry().subscribe(
      data => expect(data).toEqual(mockData)
    );

    expect(httpSpy).toHaveBeenCalledTimes(3);
  });
});
```

## Common Pitfalls

### 1. Swallowing Errors

```typescript
// ❌ Silently ignoring errors
const bad$ = source$.pipe(
  catchError(() => of([])) // Error is lost
);

// ✅ Log and handle appropriately
const good$ = source$.pipe(
  catchError(error => {
    console.error('Operation failed:', error);
    this.notificationService.showError('Unable to load data');
    return of([]); // Provide fallback
  })
);
```

### 2. Infinite Retry Loops

```typescript
// ❌ Unlimited retries
const dangerous$ = source$.pipe(
  retry() // Will retry forever
);

// ✅ Limited retries with backoff
const safe$ = source$.pipe(
  retryWhen(errors =>
    errors.pipe(
      mergeMap((error, index) => {
        if (index >= 3) return throwError(error);
        return timer(1000 * Math.pow(2, index));
      })
    )
  )
);
```

### 3. Not Handling Different Error Types

```typescript
// ❌ Generic error handling
const generic$ = source$.pipe(
  catchError(error => of('Error occurred'))
);

// ✅ Specific error handling
const specific$ = source$.pipe(
  catchError(error => {
    if (error.status === 401) {
      this.authService.logout();
      return throwError('Please log in again');
    } else if (error.status === 403) {
      return throwError('Access denied');
    } else {
      return of(defaultValue);
    }
  })
);
```

## Exercises

### Exercise 1: Robust API Service
Create an API service that implements exponential backoff retry, circuit breaker pattern, and comprehensive error handling for different HTTP status codes.

### Exercise 2: Form with Advanced Error Handling
Build a form component that handles validation errors, submission failures, and network issues with appropriate user feedback.

### Exercise 3: Resilient Data Pipeline
Implement a data processing pipeline that can handle partial failures, provide fallback data sources, and recover gracefully from various error conditions.

## Summary

Error handling operators are essential for building resilient reactive applications:

- **catchError()**: Handle and recover from errors
- **retry()**: Retry failed operations with limits
- **retryWhen()**: Custom retry logic with backoff strategies
- **onErrorResumeNext()**: Continue with alternative streams
- **throwError()**: Create error observables for testing and validation

Key principles:
- Handle errors at appropriate levels
- Provide meaningful error messages
- Implement retry strategies with limits
- Use fallback mechanisms
- Test error scenarios thoroughly

## Next Steps

In the next lesson, we'll explore **Utility Operators**, which provide essential utilities for debugging, side effects, and stream manipulation.
