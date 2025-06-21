# Marble Testing Fundamentals

## Learning Objectives
- Understand RxJS marble testing concepts and syntax
- Write effective marble tests for observables
- Test synchronous and asynchronous streams
- Debug observable behavior using marble diagrams
- Set up proper testing environments for RxJS

## Prerequisites
- Basic RxJS operators knowledge
- Understanding of marble diagrams
- JavaScript/TypeScript testing fundamentals
- Angular testing concepts

---

## Table of Contents
1. [Introduction to Marble Testing](#introduction-to-marble-testing)
2. [Marble Syntax & Notation](#marble-syntax--notation)
3. [TestScheduler Setup](#testscheduler-setup)
4. [Basic Marble Tests](#basic-marble-tests)
5. [Testing Operators](#testing-operators)
6. [Time-based Testing](#time-based-testing)
7. [Error Testing](#error-testing)
8. [Helper Methods](#helper-methods)
9. [Best Practices](#best-practices)
10. [Common Patterns](#common-patterns)

---

## Introduction to Marble Testing

### What is Marble Testing?

Marble testing is a powerful technique for testing RxJS observables using ASCII-based marble diagrams. It allows you to:

- **Visualize** observable streams as text
- **Test timing** and sequence of emissions
- **Verify transformations** applied by operators
- **Debug complex** observable chains
- **Ensure predictable** behavior

### Why Use Marble Testing?

```typescript
// Traditional testing - harder to understand timing
it('should debounce values', (done) => {
  const source$ = new Subject<string>();
  const result: string[] = [];
  
  source$.pipe(debounceTime(100)).subscribe(value => {
    result.push(value);
    if (result.length === 1) {
      expect(result).toEqual(['c']);
      done();
    }
  });
  
  source$.next('a');
  setTimeout(() => source$.next('b'), 50);
  setTimeout(() => source$.next('c'), 100);
});

// Marble testing - clear and concise
it('should debounce values', () => {
  testScheduler.run(({ hot, expectObservable }) => {
    const source$ = hot('a-b---c---|');
    const expected =   '------c---|';
    
    const result$ = source$.pipe(debounceTime(3));
    expectObservable(result$).toBe(expected);
  });
});
```

---

## Marble Syntax & Notation

### Basic Marble Symbols

| Symbol | Meaning | Example |
|--------|---------|---------|
| `-` | Time frame (1 virtual millisecond) | `'--a--'` |
| `a-z` | Emitted values | `'a-b-c'` |
| `\|` | Completion | `'a-b-\|'` |
| `#` | Error | `'a-b-#'` |
| `^` | Subscription point | `'^--a--'` |
| `!` | Unsubscription point | `'a-b!-'` |
| `()` | Synchronous emissions | `'(ab)-c'` |
| `{}` | Object values | `'{a:1}'` |

### Time Representation

```typescript
// Each character represents 1 frame (virtual millisecond)
'--a--b--c--|'
// ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑
// 0 1 2 3 4 5 6 7 8 9 101112

// Grouping for synchronous emissions
'(abc)---|'  // a, b, c emitted at frame 0
'---(abc)---|'  // a, b, c emitted at frame 3
```

### Value Mapping

```typescript
// Simple values
const marbles = 'a-b-c|';
const values = { a: 1, b: 2, c: 3 };

// Complex objects
const marbles = 'a-b-c|';
const values = {
  a: { id: 1, name: 'John' },
  b: { id: 2, name: 'Jane' },
  c: { id: 3, name: 'Bob' }
};

// Arrays and nested structures
const marbles = 'a-b-c|';
const values = {
  a: [1, 2, 3],
  b: { users: ['John', 'Jane'] },
  c: new Date('2025-01-01')
};
```

---

## TestScheduler Setup

### Basic Setup

```typescript
import { TestScheduler } from 'rxjs/testing';
import { map, debounceTime, throttleTime } from 'rxjs/operators';

describe('RxJS Marble Tests', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should test simple mapping', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-b-c|');
      const expected =   'x-y-z|';
      
      const result$ = source$.pipe(
        map(value => value.toUpperCase())
      );
      
      expectObservable(result$).toBe(expected, {
        x: 'A', y: 'B', z: 'C'
      });
    });
  });
});
```

### Angular TestBed Integration

```typescript
import { TestBed } from '@angular/core/testing';
import { TestScheduler } from 'rxjs/testing';

describe('Service with Observables', () => {
  let service: DataService;
  let testScheduler: TestScheduler;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [DataService]
    });
    
    service = TestBed.inject(DataService);
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should transform data correctly', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const input$ = hot('a-b-c|', {
        a: { value: 1 },
        b: { value: 2 },
        c: { value: 3 }
      });
      
      const result$ = service.processData(input$);
      const expected = 'x-y-z|';
      
      expectObservable(result$).toBe(expected, {
        x: { processed: 1 },
        y: { processed: 2 },
        z: { processed: 3 }
      });
    });
  });
});
```

---

## Basic Marble Tests

### Testing Simple Operators

```typescript
describe('Basic Operators', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should test map operator', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-b-c|');
      const expected =   'x-y-z|';
      
      const result$ = source$.pipe(
        map(value => value + '1')
      );
      
      expectObservable(result$).toBe(expected, {
        x: 'a1', y: 'b1', z: 'c1'
      });
    });
  });

  it('should test filter operator', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-b-c-d|', {
        a: 1, b: 2, c: 3, d: 4
      });
      const expected =   '--b---d|';
      
      const result$ = source$.pipe(
        filter(value => value % 2 === 0)
      );
      
      expectObservable(result$).toBe(expected, {
        b: 2, d: 4
      });
    });
  });

  it('should test take operator', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-b-c-d-e|');
      const expected =   'a-b-c|';
      
      const result$ = source$.pipe(take(3));
      
      expectObservable(result$).toBe(expected);
    });
  });
});
```

### Testing Observable Creation

```typescript
describe('Observable Creation', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should test of operator', () => {
    testScheduler.run(({ expectObservable }) => {
      const source$ = of(1, 2, 3);
      const expected = '(abc|)';
      
      expectObservable(source$).toBe(expected, {
        a: 1, b: 2, c: 3
      });
    });
  });

  it('should test from operator', () => {
    testScheduler.run(({ expectObservable }) => {
      const source$ = from([1, 2, 3]);
      const expected = '(abc|)';
      
      expectObservable(source$).toBe(expected, {
        a: 1, b: 2, c: 3
      });
    });
  });

  it('should test interval operator', () => {
    testScheduler.run(({ expectObservable }) => {
      const source$ = interval(3).pipe(take(4));
      const expected = '---a--b--c--(d|)';
      
      expectObservable(source$).toBe(expected, {
        a: 0, b: 1, c: 2, d: 3
      });
    });
  });
});
```

---

## Testing Operators

### Transformation Operators

```typescript
describe('Transformation Operators', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should test switchMap', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a---b---c|');
      const inner$ =     '--x|';
      const expected =   '---x---x---x|';
      
      const result$ = source$.pipe(
        switchMap(() => hot(inner$))
      );
      
      expectObservable(result$).toBe(expected);
    });
  });

  it('should test mergeMap', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a---b---c|');
      const aInner$ =    '--1-2|';
      const bInner$ =        '--3-4|';
      const cInner$ =            '--5-6|';
      const expected =   '---1-2-3-4-5-6|';
      
      const result$ = source$.pipe(
        mergeMap(value => {
          if (value === 'a') return hot(aInner$);
          if (value === 'b') return hot(bInner$);
          if (value === 'c') return hot(cInner$);
          return EMPTY;
        })
      );
      
      expectObservable(result$).toBe(expected);
    });
  });

  it('should test concatMap', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a---b|');
      const inner$ =     '--x-y|';
      const expected =   '---x-y---x-y|';
      
      const result$ = source$.pipe(
        concatMap(() => hot(inner$))
      );
      
      expectObservable(result$).toBe(expected);
    });
  });

  it('should test scan operator', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-b-c|', { a: 1, b: 2, c: 3 });
      const expected =   'x-y-z|';
      
      const result$ = source$.pipe(
        scan((acc, curr) => acc + curr, 0)
      );
      
      expectObservable(result$).toBe(expected, {
        x: 1, y: 3, z: 6
      });
    });
  });
});
```

### Filtering Operators

```typescript
describe('Filtering Operators', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should test distinctUntilChanged', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-a-b-b-a-c|');
      const expected =   'a---b---a-c|';
      
      const result$ = source$.pipe(distinctUntilChanged());
      
      expectObservable(result$).toBe(expected);
    });
  });

  it('should test skipUntil', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-b-c-d-e|');
      const trigger$ = hot('----x|');
      const expected =   '----d-e|';
      
      const result$ = source$.pipe(skipUntil(trigger$));
      
      expectObservable(result$).toBe(expected);
    });
  });

  it('should test takeWhile', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-b-c-d|', {
        a: 1, b: 2, c: 3, d: 4
      });
      const expected =   'a-b|';
      
      const result$ = source$.pipe(
        takeWhile(value => value < 3)
      );
      
      expectObservable(result$).toBe(expected, {
        a: 1, b: 2
      });
    });
  });
});
```

---

## Time-based Testing

### Testing Timing Operators

```typescript
describe('Time-based Operators', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should test debounceTime', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-b--c---|');
      const expected =   '----b--c-|';
      
      const result$ = source$.pipe(debounceTime(2));
      
      expectObservable(result$).toBe(expected);
    });
  });

  it('should test throttleTime', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-b-c-d-e|');
      const expected =   'a---c---e|';
      
      const result$ = source$.pipe(throttleTime(2));
      
      expectObservable(result$).toBe(expected);
    });
  });

  it('should test delay', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-b-c|');
      const expected =   '---a-b-c|';
      
      const result$ = source$.pipe(delay(3));
      
      expectObservable(result$).toBe(expected);
    });
  });

  it('should test timeout', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-----b|');
      const expected =   'a----#';
      
      const result$ = source$.pipe(timeout(4));
      
      expectObservable(result$).toBe(expected);
    });
  });

  it('should test auditTime', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-b-c-d--e|');
      const expected =   '----c----e|';
      
      const result$ = source$.pipe(auditTime(3));
      
      expectObservable(result$).toBe(expected);
    });
  });
});
```

### Testing Complex Timing

```typescript
describe('Complex Timing Scenarios', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should test debounce with dynamic timing', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-b--c---|', {
        a: { value: 'a', delay: 1 },
        b: { value: 'b', delay: 2 },
        c: { value: 'c', delay: 3 }
      });
      const expected =   '---b----c|';
      
      const result$ = source$.pipe(
        debounce(item => timer(item.delay)),
        map(item => item.value)
      );
      
      expectObservable(result$).toBe(expected, {
        b: 'b', c: 'c'
      });
    });
  });

  it('should test sample with interval', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-b-c-d-e-f|');
      const sample$ = hot('----x----x-|');
      const expected =   '----c----f|';
      
      const result$ = source$.pipe(sample(sample$));
      
      expectObservable(result$).toBe(expected);
    });
  });

  it('should test buffer with time and count', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-b-c-d-e-f|');
      const expected =   '------x----y|';
      
      const result$ = source$.pipe(
        bufferTime(3, 2)
      );
      
      expectObservable(result$).toBe(expected, {
        x: ['a', 'b', 'c'],
        y: ['d', 'e', 'f']
      });
    });
  });
});
```

---

## Error Testing

### Testing Error Scenarios

```typescript
describe('Error Handling', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should test basic error', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-b-#');
      const expected =   'a-b-#';
      
      expectObservable(source$).toBe(expected);
    });
  });

  it('should test catchError', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-b-#');
      const fallback$ = hot('x-y|');
      const expected =   'a-b-x-y|';
      
      const result$ = source$.pipe(
        catchError(() => fallback$)
      );
      
      expectObservable(result$).toBe(expected);
    });
  });

  it('should test retry', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-#');
      const expected =   'a-a-#';
      
      const result$ = source$.pipe(retry(1));
      
      expectObservable(result$).toBe(expected);
    });
  });

  it('should test retryWhen', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-#');
      const retryTrigger$ = hot('--x');
      const expected =   'a---a-#';
      
      const result$ = source$.pipe(
        retryWhen(() => retryTrigger$)
      );
      
      expectObservable(result$).toBe(expected);
    });
  });

  it('should test throwError', () => {
    testScheduler.run(({ expectObservable }) => {
      const error = new Error('Test error');
      const source$ = throwError(() => error);
      const expected = '#';
      
      expectObservable(source$).toBe(expected, {}, error);
    });
  });
});
```

### Custom Error Testing

```typescript
describe('Custom Error Scenarios', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should test map with error', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-b-c|', {
        a: 1, b: 0, c: 3  // Division by zero for 'b'
      });
      const expected =   'x-#';
      
      const result$ = source$.pipe(
        map(value => {
          if (value === 0) {
            throw new Error('Division by zero');
          }
          return 10 / value;
        })
      );
      
      expectObservable(result$).toBe(expected, {
        x: 10
      }, new Error('Division by zero'));
    });
  });

  it('should test switchMap with error recovery', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-b-c|');
      const aInner$ =    'x|';
      const bInner$ =    '#';
      const cInner$ =    'z|';
      const fallback$ =  'y|';
      const expected =   'x-y-z|';
      
      const result$ = source$.pipe(
        switchMap(value => {
          if (value === 'a') return hot(aInner$);
          if (value === 'b') return hot(bInner$).pipe(
            catchError(() => hot(fallback$))
          );
          if (value === 'c') return hot(cInner$);
          return EMPTY;
        })
      );
      
      expectObservable(result$).toBe(expected);
    });
  });
});
```

---

## Helper Methods

### TestScheduler Helper Methods

```typescript
describe('TestScheduler Helpers', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should use hot observables', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      // Hot observable - subscription starts at frame 0
      const source$ = hot('--a-b-c|');
      const expected =   '--a-b-c|';
      
      expectObservable(source$).toBe(expected);
    });
  });

  it('should use cold observables', () => {
    testScheduler.run(({ cold, expectObservable }) => {
      // Cold observable - subscription starts when subscribed
      const source$ = cold('a-b-c|');
      const expected =    'a-b-c|';
      
      expectObservable(source$).toBe(expected);
    });
  });

  it('should test subscription timing', () => {
    testScheduler.run(({ hot, expectObservable, expectSubscriptions }) => {
      const source$ = hot('--a-b-c-d-e|');
      const sub1 =       '^---!';
      const sub2 =       '----^-----!';
      const expected1 =  '--a-b';
      const expected2 =  '----c-d-e';
      
      expectObservable(source$.pipe(take(2))).toBe(expected1);
      expectObservable(source$.pipe(skip(2), take(3))).toBe(expected2);
      expectSubscriptions(source$.subscriptions).toBe([sub1, sub2]);
    });
  });

  it('should flush manually', () => {
    testScheduler.run(({ hot, expectObservable, flush }) => {
      const source$ = hot('a-b-c|');
      const result$ = source$.pipe(
        tap(value => console.log('Value:', value))
      );
      
      expectObservable(result$).toBe('a-b-c|');
      flush(); // Execute the test
    });
  });
});
```

### Custom Test Utilities

```typescript
// Custom test utilities for common patterns
export class RxJSTestUtils {
  static testScheduler: TestScheduler;

  static setup(): void {
    this.testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  }

  static testOperator<T, R>(
    operator: OperatorFunction<T, R>,
    source: string,
    expected: string,
    sourceValues?: any,
    expectedValues?: any
  ): void {
    this.testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot(source, sourceValues);
      const result$ = source$.pipe(operator);
      expectObservable(result$).toBe(expected, expectedValues);
    });
  }

  static testAsyncOperator<T, R>(
    operatorFactory: () => OperatorFunction<T, R>,
    source: string,
    expected: string,
    sourceValues?: any,
    expectedValues?: any
  ): void {
    this.testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot(source, sourceValues);
      const result$ = source$.pipe(operatorFactory());
      expectObservable(result$).toBe(expected, expectedValues);
    });
  }

  static createMockHttp(responses: Record<string, string>): any {
    return {
      get: (url: string) => {
        const response = responses[url];
        if (response) {
          return this.testScheduler.createColdObservable(response);
        }
        return this.testScheduler.createColdObservable('#');
      }
    };
  }
}

// Usage example
describe('Custom Test Utils', () => {
  beforeEach(() => {
    RxJSTestUtils.setup();
  });

  it('should test map operator with utility', () => {
    RxJSTestUtils.testOperator(
      map((x: string) => x.toUpperCase()),
      'a-b-c|',
      'x-y-z|',
      { a: 'a', b: 'b', c: 'c' },
      { x: 'A', y: 'B', z: 'C' }
    );
  });
});
```

---

## Best Practices

### Test Organization

```typescript
describe('RxJS Testing Best Practices', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  describe('Operator: map', () => {
    it('should transform values', () => {
      testScheduler.run(({ hot, expectObservable }) => {
        const source$ = hot('a-b-c|');
        const expected =   'x-y-z|';
        
        const result$ = source$.pipe(
          map(value => value.toUpperCase())
        );
        
        expectObservable(result$).toBe(expected, {
          x: 'A', y: 'B', z: 'C'
        });
      });
    });

    it('should handle empty stream', () => {
      testScheduler.run(({ hot, expectObservable }) => {
        const source$ = hot('|');
        const expected =   '|';
        
        const result$ = source$.pipe(
          map(value => value.toUpperCase())
        );
        
        expectObservable(result$).toBe(expected);
      });
    });

    it('should handle errors', () => {
      testScheduler.run(({ hot, expectObservable }) => {
        const source$ = hot('a-#');
        const expected =   'x-#';
        
        const result$ = source$.pipe(
          map(value => value.toUpperCase())
        );
        
        expectObservable(result$).toBe(expected, {
          x: 'A'
        });
      });
    });
  });
});
```

### Descriptive Test Names

```typescript
describe('Data Processing Service', () => {
  describe('when processing user data', () => {
    it('should transform user objects to display format', () => {
      // Test implementation
    });

    it('should filter out inactive users', () => {
      // Test implementation
    });

    it('should handle empty user list gracefully', () => {
      // Test implementation
    });

    it('should retry failed requests up to 3 times', () => {
      // Test implementation
    });
  });

  describe('when handling errors', () => {
    it('should emit fallback data on network error', () => {
      // Test implementation
    });

    it('should propagate validation errors to UI', () => {
      // Test implementation
    });
  });
});
```

### Shared Test Setup

```typescript
// test-helpers.ts
export const createTestUser = (overrides: Partial<User> = {}): User => ({
  id: 1,
  name: 'John Doe',
  email: 'john@example.com',
  active: true,
  ...overrides
});

export const createMockHttpClient = (testScheduler: TestScheduler) => ({
  get: (url: string) => {
    const responses: Record<string, string> = {
      '/api/users': 'a|',
      '/api/error': '#'
    };
    return testScheduler.createColdObservable(
      responses[url] || '#'
    );
  }
});

// In test files
describe('User Service', () => {
  let testScheduler: TestScheduler;
  let mockHttp: any;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
    mockHttp = createMockHttpClient(testScheduler);
  });

  it('should load users', () => {
    testScheduler.run(({ expectObservable }) => {
      const service = new UserService(mockHttp);
      const result$ = service.getUsers();
      const expected = 'a|';
      
      expectObservable(result$).toBe(expected, {
        a: [createTestUser()]
      });
    });
  });
});
```

---

## Common Patterns

### Testing Angular Services

```typescript
@Injectable()
export class UserService {
  constructor(private http: HttpClient) {}

  getUsers(): Observable<User[]> {
    return this.http.get<User[]>('/api/users').pipe(
      map(users => users.filter(user => user.active)),
      retry(3),
      catchError(() => of([]))
    );
  }

  searchUsers(query: string): Observable<User[]> {
    return this.http.get<User[]>(`/api/users/search?q=${query}`).pipe(
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(users => of(users)),
      catchError(() => of([]))
    );
  }
}

describe('UserService', () => {
  let service: UserService;
  let httpMock: HttpTestingController;
  let testScheduler: TestScheduler;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [UserService]
    });

    service = TestBed.inject(UserService);
    httpMock = TestBed.inject(HttpTestingController);
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('should get active users only', () => {
    const mockUsers = [
      createTestUser({ id: 1, active: true }),
      createTestUser({ id: 2, active: false }),
      createTestUser({ id: 3, active: true })
    ];

    service.getUsers().subscribe(users => {
      expect(users).toEqual([
        mockUsers[0],
        mockUsers[2]
      ]);
    });

    const req = httpMock.expectOne('/api/users');
    req.flush(mockUsers);
  });

  it('should retry failed requests', () => {
    service.getUsers().subscribe();

    // Expect 4 requests (initial + 3 retries)
    for (let i = 0; i < 4; i++) {
      const req = httpMock.expectOne('/api/users');
      req.error(new ErrorEvent('Network error'));
    }
  });
});
```

### Testing Components with Observables

```typescript
@Component({
  selector: 'app-user-list',
  template: `
    <div *ngFor="let user of users$ | async">
      {{ user.name }}
    </div>
    <div *ngIf="loading$ | async">Loading...</div>
    <div *ngIf="error$ | async">Error occurred</div>
  `
})
export class UserListComponent implements OnInit {
  users$: Observable<User[]>;
  loading$: Observable<boolean>;
  error$: Observable<boolean>;

  constructor(private userService: UserService) {}

  ngOnInit(): void {
    const request$ = this.userService.getUsers().pipe(
      share()
    );

    this.users$ = request$.pipe(
      catchError(() => of([]))
    );

    this.loading$ = request$.pipe(
      map(() => false),
      startWith(true),
      catchError(() => of(false))
    );

    this.error$ = request$.pipe(
      map(() => false),
      catchError(() => of(true))
    );
  }
}

describe('UserListComponent', () => {
  let component: UserListComponent;
  let fixture: ComponentFixture<UserListComponent>;
  let userService: jasmine.SpyOf<UserService>;
  let testScheduler: TestScheduler;

  beforeEach(() => {
    const spy = jasmine.createSpyObj('UserService', ['getUsers']);

    TestBed.configureTestingModule({
      declarations: [UserListComponent],
      providers: [
        { provide: UserService, useValue: spy }
      ]
    });

    fixture = TestBed.createComponent(UserListComponent);
    component = fixture.componentInstance;
    userService = TestBed.inject(UserService) as jasmine.SpyOf<UserService>;
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should display users when loaded', fakeAsync(() => {
    const mockUsers = [createTestUser()];
    userService.getUsers.and.returnValue(of(mockUsers));

    component.ngOnInit();
    tick();

    component.users$.subscribe(users => {
      expect(users).toEqual(mockUsers);
    });
  }));

  it('should show loading state', () => {
    testScheduler.run(({ cold, expectObservable }) => {
      const users$ = cold('--a|', { a: [createTestUser()] });
      userService.getUsers.and.returnValue(users$);

      component.ngOnInit();

      const expected = 'a-b|';
      expectObservable(component.loading$).toBe(expected, {
        a: true,
        b: false
      });
    });
  });
});
```

---

## Summary

This lesson covered the fundamentals of marble testing in RxJS:

### Key Concepts Learned:
1. **Marble Syntax** - Understanding the ASCII-based notation
2. **TestScheduler** - Setting up virtual time testing
3. **Basic Testing** - Testing simple operators and observables
4. **Operator Testing** - Testing transformation and filtering operators
5. **Time-based Testing** - Testing timing-dependent operators
6. **Error Testing** - Testing error scenarios and recovery
7. **Helper Methods** - Using TestScheduler helper functions
8. **Best Practices** - Organizing tests and shared utilities

### Benefits of Marble Testing:
- **Visual clarity** of observable behavior
- **Predictable timing** in tests
- **Easy debugging** of complex streams
- **Comprehensive coverage** of edge cases
- **Fast execution** with virtual time

### Testing Patterns:
- **Hot vs Cold** observables in tests
- **Subscription timing** verification
- **Error scenario** testing
- **Angular integration** testing
- **Custom utility** creation

Marble testing provides a powerful and intuitive way to test RxJS observables, making complex asynchronous behavior predictable and verifiable.

---

## Exercises

1. **Basic Operators**: Write marble tests for map, filter, and take
2. **Time Operators**: Test debounceTime, throttleTime, and delay
3. **Error Handling**: Test catchError and retry scenarios
4. **Angular Service**: Test a service with HTTP calls using marble tests
5. **Complex Streams**: Test switchMap with multiple inner observables
6. **Custom Operators**: Create and test a custom operator
7. **Performance**: Compare marble testing vs traditional async testing

## Additional Resources

- [RxJS Marble Testing Documentation](https://rxjs.dev/guide/testing/marble-testing)
- [Angular Testing Guide](https://angular.io/guide/testing)
- [TestScheduler API Reference](https://rxjs.dev/api/testing/TestScheduler)
- [Marble Testing Examples](https://github.com/ReactiveX/rxjs/tree/master/spec)
