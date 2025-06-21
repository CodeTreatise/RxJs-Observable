# Creating Custom RxJS Operators

## Learning Objectives
- Understand operator composition and creation patterns
- Build reusable custom operators
- Master the `pipe()` function and operator chaining
- Create domain-specific operators for Angular applications
- Implement higher-order operators
- Debug and test custom operators

## Introduction to Custom Operators

Custom operators allow you to encapsulate complex reactive logic into reusable, composable units. They promote code reuse, improve readability, and enable domain-specific abstractions.

### Types of Custom Operators
- **Composition Operators**: Combining existing operators
- **Creation Operators**: Building operators from scratch
- **Higher-Order Operators**: Operators that work with observables of observables
- **Lettable Operators**: Modern pipeable operators (post RxJS 5.5)

## Understanding Operator Fundamentals

### Operator Function Signature

```typescript
// Basic operator signature
type OperatorFunction<T, R> = (source: Observable<T>) => Observable<R>;

// Example: Simple operator that doubles values
const double = <T extends number>(): OperatorFunction<T, T> => {
  return (source: Observable<T>) => {
    return new Observable<T>(subscriber => {
      return source.subscribe({
        next: value => subscriber.next((value * 2) as T),
        error: err => subscriber.error(err),
        complete: () => subscriber.complete()
      });
    });
  };
};

// Usage
of(1, 2, 3, 4)
  .pipe(double())
  .subscribe(console.log); // 2, 4, 6, 8
```

### Marble Diagram for Custom Operators

```
Input:   --1--2--3--4--|
double(): --2--4--6--8--|
```

## Building Simple Custom Operators

### Composition-Based Operators

```typescript
// logging.operators.ts
import { Observable, OperatorFunction } from 'rxjs';
import { tap, map, filter, catchError } from 'rxjs/operators';

/**
 * Logs values with optional prefix
 */
export const log = <T>(prefix: string = ''): OperatorFunction<T, T> => {
  return tap(value => console.log(`${prefix}:`, value));
};

/**
 * Logs errors with context
 */
export const logErrors = <T>(context: string = ''): OperatorFunction<T, T> => {
  return catchError(error => {
    console.error(`Error in ${context}:`, error);
    throw error;
  });
};

/**
 * Logs subscription lifecycle
 */
export const logLifecycle = <T>(name: string): OperatorFunction<T, T> => {
  return (source: Observable<T>) => {
    return new Observable<T>(subscriber => {
      console.log(`${name}: Subscribed`);
      
      const subscription = source.subscribe({
        next: value => {
          console.log(`${name}: Next -`, value);
          subscriber.next(value);
        },
        error: error => {
          console.log(`${name}: Error -`, error);
          subscriber.error(error);
        },
        complete: () => {
          console.log(`${name}: Complete`);
          subscriber.complete();
        }
      });

      return () => {
        console.log(`${name}: Unsubscribed`);
        subscription.unsubscribe();
      };
    });
  };
};

// Usage examples
interval(1000).pipe(
  take(3),
  log('Timer'),
  map(x => x * 2),
  log('Doubled'),
  logLifecycle('Interval Stream')
).subscribe();
```

### Conditional and Filtering Operators

```typescript
// filtering.operators.ts

/**
 * Filters out null and undefined values with type narrowing
 */
export const filterNullish = <T>(): OperatorFunction<T | null | undefined, T> => {
  return filter((value): value is T => value != null);
};

/**
 * Filters by type with type guards
 */
export const filterByType = <T, U extends T>(
  typeGuard: (value: T) => value is U
): OperatorFunction<T, U> => {
  return filter(typeGuard);
};

/**
 * Takes elements until a condition is met
 */
export const takeUntilCondition = <T>(
  predicate: (value: T) => boolean
): OperatorFunction<T, T> => {
  return (source: Observable<T>) => {
    return new Observable<T>(subscriber => {
      return source.subscribe({
        next: value => {
          if (predicate(value)) {
            subscriber.complete();
          } else {
            subscriber.next(value);
          }
        },
        error: err => subscriber.error(err),
        complete: () => subscriber.complete()
      });
    });
  };
};

/**
 * Skips elements while condition is true
 */
export const skipWhileCondition = <T>(
  predicate: (value: T, index: number) => boolean
): OperatorFunction<T, T> => {
  return (source: Observable<T>) => {
    return new Observable<T>(subscriber => {
      let index = 0;
      let skipping = true;

      return source.subscribe({
        next: value => {
          if (skipping && predicate(value, index)) {
            index++;
            return;
          }
          skipping = false;
          subscriber.next(value);
          index++;
        },
        error: err => subscriber.error(err),
        complete: () => subscriber.complete()
      });
    });
  };
};

// Usage examples
of(null, 1, undefined, 2, null, 3).pipe(
  filterNullish()
).subscribe(console.log); // 1, 2, 3

of(1, 2, 3, 4, 5, 6).pipe(
  takeUntilCondition(x => x > 3)
).subscribe(console.log); // 1, 2, 3
```

## Advanced Custom Operators

### Stateful Operators

```typescript
// stateful.operators.ts

/**
 * Accumulates values in a buffer and emits when buffer reaches size
 */
export const bufferCount = <T>(size: number): OperatorFunction<T, T[]> => {
  return (source: Observable<T>) => {
    return new Observable<T[]>(subscriber => {
      const buffer: T[] = [];

      return source.subscribe({
        next: value => {
          buffer.push(value);
          if (buffer.length === size) {
            subscriber.next([...buffer]);
            buffer.length = 0; // Clear buffer
          }
        },
        error: err => subscriber.error(err),
        complete: () => {
          if (buffer.length > 0) {
            subscriber.next([...buffer]);
          }
          subscriber.complete();
        }
      });
    });
  };
};

/**
 * Emits the previous and current value as a tuple
 */
export const pairwise = <T>(): OperatorFunction<T, [T, T]> => {
  return (source: Observable<T>) => {
    return new Observable<[T, T]>(subscriber => {
      let hasPrevious = false;
      let previous: T;

      return source.subscribe({
        next: value => {
          if (hasPrevious) {
            subscriber.next([previous, value]);
          }
          previous = value;
          hasPrevious = true;
        },
        error: err => subscriber.error(err),
        complete: () => subscriber.complete()
      });
    });
  };
};

/**
 * Tracks changes and emits change objects
 */
export interface Change<T> {
  previous: T | undefined;
  current: T;
  isFirst: boolean;
}

export const trackChanges = <T>(): OperatorFunction<T, Change<T>> => {
  return (source: Observable<T>) => {
    return new Observable<Change<T>>(subscriber => {
      let isFirst = true;
      let previous: T | undefined;

      return source.subscribe({
        next: current => {
          subscriber.next({
            previous,
            current,
            isFirst
          });
          previous = current;
          isFirst = false;
        },
        error: err => subscriber.error(err),
        complete: () => subscriber.complete()
      });
    });
  };
};

// Usage examples
of(1, 2, 3, 4, 5, 6).pipe(
  bufferCount(3)
).subscribe(console.log); // [1,2,3], [4,5,6]

of('a', 'b', 'c', 'd').pipe(
  pairwise()
).subscribe(console.log); // ['a','b'], ['b','c'], ['c','d']
```

### Time-Based Operators

```typescript
// time.operators.ts

/**
 * Throttles emissions with custom timing logic
 */
export const throttleWithCondition = <T>(
  durationSelector: (value: T) => Observable<any>
): OperatorFunction<T, T> => {
  return (source: Observable<T>) => {
    return new Observable<T>(subscriber => {
      let isThrottled = false;
      let throttleSubscription: Subscription;

      return source.subscribe({
        next: value => {
          if (!isThrottled) {
            subscriber.next(value);
            isThrottled = true;

            throttleSubscription = durationSelector(value).subscribe({
              next: () => {
                isThrottled = false;
                throttleSubscription?.unsubscribe();
              },
              complete: () => {
                isThrottled = false;
              }
            });
          }
        },
        error: err => {
          throttleSubscription?.unsubscribe();
          subscriber.error(err);
        },
        complete: () => {
          throttleSubscription?.unsubscribe();
          subscriber.complete();
        }
      });
    });
  };
};

/**
 * Delays emission based on dynamic duration
 */
export const delayBy = <T>(
  delayDurationSelector: (value: T, index: number) => number
): OperatorFunction<T, T> => {
  return (source: Observable<T>) => {
    return new Observable<T>(subscriber => {
      let index = 0;

      return source.subscribe({
        next: value => {
          const delayMs = delayDurationSelector(value, index++);
          
          setTimeout(() => {
            subscriber.next(value);
          }, delayMs);
        },
        error: err => subscriber.error(err),
        complete: () => {
          // Note: This simplified version doesn't handle completion timing perfectly
          subscriber.complete();
        }
      });
    });
  };
};

/**
 * Rate limits emissions to a maximum frequency
 */
export const rateLimit = <T>(
  maxEmissions: number,
  timeWindow: number
): OperatorFunction<T, T> => {
  return (source: Observable<T>) => {
    return new Observable<T>(subscriber => {
      const emissions: number[] = [];

      return source.subscribe({
        next: value => {
          const now = Date.now();
          
          // Remove old emissions outside time window
          while (emissions.length > 0 && now - emissions[0] > timeWindow) {
            emissions.shift();
          }

          if (emissions.length < maxEmissions) {
            emissions.push(now);
            subscriber.next(value);
          }
          // Drop emission if rate limit exceeded
        },
        error: err => subscriber.error(err),
        complete: () => subscriber.complete()
      });
    });
  };
};

// Usage
interval(100).pipe(
  rateLimit(5, 1000), // Max 5 emissions per second
  take(20)
).subscribe(console.log);
```

## Domain-Specific Angular Operators

### HTTP and Loading State Operators

```typescript
// http.operators.ts
import { HttpErrorResponse } from '@angular/common/http';

export interface LoadingState<T> {
  loading: boolean;
  data: T | null;
  error: string | null;
}

/**
 * Wraps HTTP requests with loading state
 */
export const withLoadingState = <T>(): OperatorFunction<T, LoadingState<T>> => {
  return (source: Observable<T>) => {
    return source.pipe(
      map(data => ({
        loading: false,
        data,
        error: null
      })),
      startWith({
        loading: true,
        data: null,
        error: null
      }),
      catchError(error => {
        const errorMessage = error instanceof HttpErrorResponse 
          ? error.message 
          : 'An error occurred';
        
        return of({
          loading: false,
          data: null,
          error: errorMessage
        });
      })
    );
  };
};

/**
 * Retries HTTP requests with exponential backoff
 */
export const retryWithBackoff = (
  maxRetries: number = 3,
  baseDelay: number = 1000,
  maxDelay: number = 30000
): OperatorFunction<any, any> => {
  return (source: Observable<any>) => {
    return source.pipe(
      retryWhen(errors => 
        errors.pipe(
          scan((retryCount, error) => {
            if (retryCount >= maxRetries) {
              throw error;
            }
            return retryCount + 1;
          }, 0),
          delayWhen(retryCount => {
            const delay = Math.min(baseDelay * Math.pow(2, retryCount), maxDelay);
            console.log(`Retrying in ${delay}ms (attempt ${retryCount + 1})`);
            return timer(delay);
          })
        )
      )
    );
  };
};

/**
 * Caches successful HTTP responses
 */
export const cacheResponse = <T>(
  cacheTime: number = 5 * 60 * 1000 // 5 minutes
): OperatorFunction<T, T> => {
  return (source: Observable<T>) => {
    return source.pipe(
      shareReplay({
        bufferSize: 1,
        windowTime: cacheTime,
        refCount: true
      })
    );
  };
};

// Usage
this.http.get<User[]>('/api/users').pipe(
  retryWithBackoff(3, 1000),
  cacheResponse(300000), // Cache for 5 minutes
  withLoadingState()
).subscribe(state => {
  this.loadingState = state;
});
```

### Form Validation Operators

```typescript
// validation.operators.ts

export interface ValidationResult {
  valid: boolean;
  errors: { [key: string]: any } | null;
}

/**
 * Debounces form validation
 */
export const debounceValidation = (
  debounceTime: number = 300
): OperatorFunction<any, any> => {
  return (source: Observable<any>) => {
    return source.pipe(
      debounceTime(debounceTime),
      distinctUntilChanged()
    );
  };
};

/**
 * Validates form values asynchronously
 */
export const validateAsync = <T>(
  validator: (value: T) => Observable<ValidationResult>
): OperatorFunction<T, ValidationResult> => {
  return (source: Observable<T>) => {
    return source.pipe(
      switchMap(value => 
        validator(value).pipe(
          catchError(error => {
            return of({
              valid: false,
              errors: { validationError: error.message }
            });
          })
        )
      )
    );
  };
};

/**
 * Combines multiple validation results
 */
export const combineValidations = (): OperatorFunction<ValidationResult[], ValidationResult> => {
  return map(validations => {
    const valid = validations.every(v => v.valid);
    const errors = validations.reduce((acc, v) => ({
      ...acc,
      ...(v.errors || {})
    }), {});

    return {
      valid,
      errors: valid ? null : errors
    };
  });
};

// Usage in reactive forms
this.emailControl.valueChanges.pipe(
  debounceValidation(500),
  validateAsync(email => this.validateEmailUnique(email)),
  takeUntil(this.destroy$)
).subscribe(result => {
  this.emailValidation = result;
});
```

### State Management Operators

```typescript
// state.operators.ts

/**
 * Updates state immutably
 */
export const updateState = <T>(
  updater: (state: T) => Partial<T>
): OperatorFunction<T, T> => {
  return map(state => ({
    ...state,
    ...updater(state)
  }));
};

/**
 * Filters state changes by specific properties
 */
export const whenPropertyChanges = <T, K extends keyof T>(
  property: K
): OperatorFunction<T, T[K]> => {
  return (source: Observable<T>) => {
    return source.pipe(
      map(state => state[property]),
      distinctUntilChanged()
    );
  };
};

/**
 * Selects nested properties safely
 */
export const selectProperty = <T, K extends keyof T>(
  property: K
): OperatorFunction<T, T[K] | undefined> => {
  return map(state => state && state[property]);
};

/**
 * Filters out loading states
 */
export const whenNotLoading = <T extends { loading: boolean }>(): OperatorFunction<T, T> => {
  return filter(state => !state.loading);
};

/**
 * Selects successful data from loading states
 */
export const selectData = <T, D>(): OperatorFunction<LoadingState<D>, D> => {
  return (source: Observable<LoadingState<D>>) => {
    return source.pipe(
      filter(state => !state.loading && state.data !== null),
      map(state => state.data!),
      distinctUntilChanged()
    );
  };
};

// Usage
this.store.state$.pipe(
  whenPropertyChanges('user'),
  filterNullish(),
  selectProperty('preferences')
).subscribe(preferences => {
  this.userPreferences = preferences;
});
```

## Higher-Order Custom Operators

### Advanced Flattening Operators

```typescript
// flattening.operators.ts

/**
 * Switches to inner observable with cancellation support
 */
export const switchMapWithCancel = <T, R>(
  project: (value: T) => Observable<R>,
  onCancel?: (cancelledValue: T) => void
): OperatorFunction<T, R> => {
  return (source: Observable<T>) => {
    return new Observable<R>(subscriber => {
      let innerSubscription: Subscription | null = null;
      let currentValue: T;

      return source.subscribe({
        next: value => {
          // Cancel previous inner subscription
          if (innerSubscription) {
            innerSubscription.unsubscribe();
            if (onCancel) {
              onCancel(currentValue);
            }
          }

          currentValue = value;
          innerSubscription = project(value).subscribe({
            next: innerValue => subscriber.next(innerValue),
            error: err => subscriber.error(err),
            complete: () => {
              innerSubscription = null;
            }
          });
        },
        error: err => {
          innerSubscription?.unsubscribe();
          subscriber.error(err);
        },
        complete: () => {
          innerSubscription?.unsubscribe();
          subscriber.complete();
        }
      });
    });
  };
};

/**
 * Merges with concurrency limit and queue management
 */
export const mergeMapWithQueue = <T, R>(
  project: (value: T) => Observable<R>,
  concurrent: number = 1,
  queueSize: number = Infinity
): OperatorFunction<T, R> => {
  return (source: Observable<T>) => {
    return new Observable<R>(subscriber => {
      const activeStreams = new Set<Subscription>();
      const queue: T[] = [];

      const processNext = () => {
        if (activeStreams.size < concurrent && queue.length > 0) {
          const value = queue.shift()!;
          const innerSubscription = project(value).subscribe({
            next: innerValue => subscriber.next(innerValue),
            error: err => subscriber.error(err),
            complete: () => {
              activeStreams.delete(innerSubscription);
              processNext();
            }
          });
          activeStreams.add(innerSubscription);
        }
      };

      return source.subscribe({
        next: value => {
          if (queue.length < queueSize) {
            queue.push(value);
            processNext();
          }
          // Drop value if queue is full
        },
        error: err => {
          activeStreams.forEach(sub => sub.unsubscribe());
          subscriber.error(err);
        },
        complete: () => {
          if (activeStreams.size === 0 && queue.length === 0) {
            subscriber.complete();
          }
        }
      });
    });
  };
};

// Usage
searchTerms$.pipe(
  switchMapWithCancel(
    term => this.searchService.search(term),
    cancelledTerm => console.log('Cancelled search for:', cancelledTerm)
  )
).subscribe(results => {
  this.searchResults = results;
});
```

## Testing Custom Operators

### Unit Testing Operators

```typescript
// custom-operators.spec.ts
import { TestScheduler } from 'rxjs/testing';
import { cold, hot } from 'jasmine-marbles';

describe('Custom Operators', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  describe('bufferCount', () => {
    it('should buffer values and emit when count is reached', () => {
      testScheduler.run(({ cold, expectObservable }) => {
        const source = cold('a-b-c-d-e-f|');
        const expected =     '-----x-----y|';
        const values = {
          x: ['a', 'b', 'c'],
          y: ['d', 'e', 'f']
        };

        const result = source.pipe(bufferCount(3));
        expectObservable(result).toBe(expected, values);
      });
    });

    it('should emit remaining buffer on completion', () => {
      testScheduler.run(({ cold, expectObservable }) => {
        const source = cold('a-b-c-d|');
        const expected =     '-----x-y|';
        const values = {
          x: ['a', 'b', 'c'],
          y: ['d']
        };

        const result = source.pipe(bufferCount(3));
        expectObservable(result).toBe(expected, values);
      });
    });
  });

  describe('rateLimit', () => {
    it('should limit emissions to max count per time window', () => {
      testScheduler.run(({ cold, expectObservable }) => {
        const source = cold('abcdefghij|');
        const expected =     'ab--------c|'; // Only first 2, then 1 after window
        
        // Mock time window of 100ms, max 2 emissions
        const result = source.pipe(rateLimit(2, 100));
        expectObservable(result).toBe(expected);
      });
    });
  });

  describe('withLoadingState', () => {
    it('should start with loading state', () => {
      testScheduler.run(({ cold, expectObservable }) => {
        const source = cold('--a|');
        const expected =     'a-b|';
        const values = {
          a: { loading: true, data: null, error: null },
          b: { loading: false, data: 'a', error: null }
        };

        const result = source.pipe(withLoadingState());
        expectObservable(result).toBe(expected, values);
      });
    });

    it('should handle errors correctly', () => {
      testScheduler.run(({ cold, expectObservable }) => {
        const source = cold('--#');
        const expected =     'a-b';
        const values = {
          a: { loading: true, data: null, error: null },
          b: { loading: false, data: null, error: 'An error occurred' }
        };

        const result = source.pipe(withLoadingState());
        expectObservable(result).toBe(expected, values);
      });
    });
  });
});
```

### Integration Testing

```typescript
// operator-integration.spec.ts
describe('Custom Operator Integration', () => {
  let component: TestComponent;
  let fixture: ComponentFixture<TestComponent>;

  beforeEach(() => {
    TestBed.configureTestingModule({
      declarations: [TestComponent]
    });
    fixture = TestBed.createComponent(TestComponent);
    component = fixture.componentInstance;
  });

  it('should handle form validation with custom operators', fakeAsync(() => {
    const emailInput = 'test@example.com';
    
    component.emailControl.setValue(emailInput);
    
    // Wait for debounce
    tick(500);
    
    expect(component.emailValidation).toEqual({
      valid: true,
      errors: null
    });
  }));

  it('should manage loading state correctly', fakeAsync(() => {
    component.loadData();
    
    expect(component.loadingState.loading).toBe(true);
    
    tick(1000); // Simulate async operation
    
    expect(component.loadingState.loading).toBe(false);
    expect(component.loadingState.data).toBeTruthy();
  }));
});
```

## Best Practices for Custom Operators

### 1. Operator Design Principles
- **Single Responsibility**: Each operator should have one clear purpose
- **Composability**: Operators should work well together
- **Immutability**: Don't mutate input values
- **Error Handling**: Properly propagate and handle errors
- **Resource Cleanup**: Ensure proper subscription management

### 2. Performance Considerations
- **Memory Management**: Avoid memory leaks in stateful operators
- **Subscription Cleanup**: Always unsubscribe from inner subscriptions
- **Efficient Implementation**: Use existing operators when possible
- **Avoid Blocking**: Don't perform synchronous heavy operations

### 3. TypeScript Best Practices
- **Strong Typing**: Provide accurate type definitions
- **Generic Constraints**: Use type constraints when appropriate
- **Type Guards**: Implement proper type narrowing
- **Documentation**: Document operator behavior and usage

### 4. Testing Strategy
- **Marble Testing**: Use TestScheduler for timing-dependent tests
- **Unit Tests**: Test operators in isolation
- **Integration Tests**: Test operators within components
- **Edge Cases**: Test error conditions and boundary cases

## Common Pitfalls

### 1. Memory Leaks
```typescript
// ❌ Bad: Not cleaning up inner subscriptions
const badOperator = <T>() => (source: Observable<T>) => {
  return new Observable<T>(subscriber => {
    source.subscribe(subscriber); // No cleanup!
  });
};

// ✅ Good: Proper cleanup
const goodOperator = <T>() => (source: Observable<T>) => {
  return new Observable<T>(subscriber => {
    const subscription = source.subscribe(subscriber);
    return () => subscription.unsubscribe();
  });
};
```

### 2. Error Propagation
```typescript
// ❌ Bad: Swallowing errors
const badOperator = <T>() => (source: Observable<T>) => {
  return source.pipe(
    catchError(() => EMPTY) // Silently swallows all errors
  );
};

// ✅ Good: Proper error handling
const goodOperator = <T>() => (source: Observable<T>) => {
  return source.pipe(
    catchError(error => {
      console.error('Operator error:', error);
      throw error; // Re-throw or handle appropriately
    })
  );
};
```

## Summary

Custom RxJS operators enable powerful abstractions and code reuse. Key takeaways:

- Build operators using composition or from scratch
- Follow single responsibility and composability principles
- Implement proper error handling and resource cleanup
- Use TypeScript for strong typing and better developer experience
- Test operators thoroughly with marble diagrams and unit tests
- Consider performance implications and memory management
- Create domain-specific operators for Angular applications
- Leverage higher-order operators for complex scenarios

Custom operators are essential for building maintainable, reusable reactive code that expresses domain logic clearly and concisely.
