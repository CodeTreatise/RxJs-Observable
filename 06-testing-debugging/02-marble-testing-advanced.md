# Advanced Marble Testing Techniques

## Learning Objectives
- Master complex marble testing scenarios
- Test higher-order observables and nested streams
- Handle advanced timing and synchronization testing
- Create reusable testing utilities and patterns
- Debug complex observable chains using marble tests
- Test real-world Angular patterns with marble diagrams

## Prerequisites
- Marble testing fundamentals
- Understanding of higher-order observables
- RxJS operators expertise
- Angular testing knowledge

---

## Table of Contents
1. [Higher-Order Observable Testing](#higher-order-observable-testing)
2. [Complex Timing Scenarios](#complex-timing-scenarios)
3. [Subscription Management Testing](#subscription-management-testing)
4. [Testing State Management](#testing-state-management)
5. [Parameterized Marble Tests](#parameterized-marble-tests)
6. [Custom Marble Matchers](#custom-marble-matchers)
7. [Performance Testing](#performance-testing)
8. [Integration Testing](#integration-testing)
9. [Debugging Techniques](#debugging-techniques)
10. [Advanced Patterns](#advanced-patterns)

---

## Higher-Order Observable Testing

### Testing switchMap with Multiple Inner Streams

```typescript
describe('Higher-Order Observable Testing', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should test switchMap cancellation behavior', () => {
    testScheduler.run(({ hot, cold, expectObservable, expectSubscriptions }) => {
      // Outer observable
      const outer$ = hot('--a---b---c|');
      
      // Inner observables
      const aInner$ = cold('--1--2--3|');
      const bInner$ = cold('    --4--5|');
      const cInner$ = cold('        --6|');
      
      // Expected subscriptions
      const aInnerSub = '  --^---!';      // Cancelled when 'b' arrives
      const bInnerSub = '  ------^---!';  // Cancelled when 'c' arrives
      const cInnerSub = '  ----------^--!';
      
      // Expected output
      const expected = '   ----1-4---6|';
      
      const result$ = outer$.pipe(
        switchMap(value => {
          if (value === 'a') return aInner$;
          if (value === 'b') return bInner$;
          if (value === 'c') return cInner$;
          return EMPTY;
        })
      );
      
      expectObservable(result$).toBe(expected);
      expectSubscriptions(aInner$.subscriptions).toBe(aInnerSub);
      expectSubscriptions(bInner$.subscriptions).toBe(bInnerSub);
      expectSubscriptions(cInner$.subscriptions).toBe(cInnerSub);
    });
  });

  it('should test mergeMap concurrent execution', () => {
    testScheduler.run(({ hot, cold, expectObservable }) => {
      const outer$ = hot('--a--b--c|');
      
      const aInner$ = cold('--1--2|');
      const bInner$ = cold('   --3--4|');
      const cInner$ = cold('      --5|');
      
      const expected = '   ----1-(32)-(45)|';
      
      const result$ = outer$.pipe(
        mergeMap(value => {
          if (value === 'a') return aInner$;
          if (value === 'b') return bInner$;
          if (value === 'c') return cInner$;
          return EMPTY;
        })
      );
      
      expectObservable(result$).toBe(expected);
    });
  });

  it('should test concatMap sequential execution', () => {
    testScheduler.run(({ hot, cold, expectObservable }) => {
      const outer$ = hot('--a--b--c|');
      
      const aInner$ = cold('--1--2|');
      const bInner$ = cold('      --3--4|');
      const cInner$ = cold('            --5|');
      
      const expected = '   ----1--2----3--4----5|';
      
      const result$ = outer$.pipe(
        concatMap(value => {
          if (value === 'a') return aInner$;
          if (value === 'b') return bInner$;
          if (value === 'c') return cInner$;
          return EMPTY;
        })
      );
      
      expectObservable(result$).toBe(expected);
    });
  });

  it('should test exhaustMap overlap handling', () => {
    testScheduler.run(({ hot, cold, expectObservable }) => {
      const outer$ = hot('--a--b--c|');
      
      const aInner$ = cold('--1----2|');
      const bInner$ = cold('   ignored');
      const cInner$ = cold('      --3|');
      
      const expected = '   ----1----2--3|';
      
      const result$ = outer$.pipe(
        exhaustMap(value => {
          if (value === 'a') return aInner$;
          if (value === 'b') return bInner$; // Will be ignored
          if (value === 'c') return cInner$;
          return EMPTY;
        })
      );
      
      expectObservable(result$).toBe(expected);
    });
  });
});
```

### Testing Nested Observable Patterns

```typescript
describe('Nested Observable Patterns', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should test nested switchMap operations', () => {
    testScheduler.run(({ hot, cold, expectObservable }) => {
      const outer$ = hot('--a-----b|');
      const innerOuter$ = cold('--x--y|');
      const xInner$ = cold('    --1|');
      const yInner$ = cold('       --2|');
      
      const expected = '   ------1---2|';
      
      const result$ = outer$.pipe(
        switchMap(() => innerOuter$),
        switchMap(value => {
          if (value === 'x') return xInner$;
          if (value === 'y') return yInner$;
          return EMPTY;
        })
      );
      
      expectObservable(result$).toBe(expected);
    });
  });

  it('should test three-level nesting', () => {
    testScheduler.run(({ hot, cold, expectObservable }) => {
      const level1$ = hot('--a|');
      const level2$ = cold('--b|');
      const level3$ = cold('  --c|');
      
      const expected = '   ------c|';
      
      const result$ = level1$.pipe(
        switchMap(() => level2$),
        switchMap(() => level3$)
      );
      
      expectObservable(result$).toBe(expected);
    });
  });

  it('should test conditional nesting', () => {
    testScheduler.run(({ hot, cold, expectObservable }) => {
      const trigger$ = hot('--a--b--c|', {
        a: { type: 'nested', value: 1 },
        b: { type: 'simple', value: 2 },
        c: { type: 'nested', value: 3 }
      });
      
      const nestedInner$ = cold('--x|');
      const simpleInner$ = cold('-y|');
      
      const expected = '   ----x-y---x|';
      
      const result$ = trigger$.pipe(
        switchMap(item => {
          if (item.type === 'nested') {
            return nestedInner$;
          } else {
            return simpleInner$;
          }
        })
      );
      
      expectObservable(result$).toBe(expected);
    });
  });
});
```

---

## Complex Timing Scenarios

### Multi-Stream Synchronization

```typescript
describe('Complex Timing Scenarios', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should test combineLatest with different emission patterns', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const stream1$ = hot('--a-----c--e|');
      const stream2$ = hot('----b-d---f-|');
      const expected =     '----ab-cd-ef|';
      
      const result$ = combineLatest([stream1$, stream2$]).pipe(
        map(([val1, val2]) => val1 + val2)
      );
      
      expectObservable(result$).toBe(expected, {
        ab: 'ab', cd: 'cd', ef: 'ef'
      });
    });
  });

  it('should test withLatestFrom timing', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const trigger$ = hot('--a-----c--e|');
      const source$ =  hot('----b-d---f-|');
      const expected =     '----a---c--e|';
      
      const result$ = trigger$.pipe(
        withLatestFrom(source$),
        map(([trigger, source]) => trigger)
      );
      
      expectObservable(result$).toBe(expected);
    });
  });

  it('should test complex buffer timing', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('--a-b-c-d-e-f-g|');
      const buffer$ = hot('-------x-----y-|');
      const expected =    '-------u-----v-|';
      
      const result$ = source$.pipe(
        buffer(buffer$)
      );
      
      expectObservable(result$).toBe(expected, {
        u: ['a', 'b', 'c'],
        v: ['d', 'e', 'f']
      });
    });
  });

  it('should test window with complex timing', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('--a-b-c-d-e-f|');
      const window$ = hot('-----x-----y-|');
      const expected =    '-----w-----z-|';
      
      const result$ = source$.pipe(
        window(window$),
        map(window => window.pipe(toArray())),
        switchMap(arrayObs => arrayObs)
      );
      
      expectObservable(result$).toBe(expected, {
        w: ['a', 'b'],
        z: ['c', 'd', 'e']
      });
    });
  });

  it('should test race conditions', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const fast$ =  hot('--a|');
      const slow$ =  hot('----b|');
      const slower$ = hot('------c|');
      const expected =   '--a|';
      
      const result$ = race([fast$, slow$, slower$]);
      
      expectObservable(result$).toBe(expected);
    });
  });
});
```

### Advanced Timing Patterns

```typescript
describe('Advanced Timing Patterns', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should test dynamic debouncing', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-b--c---|', {
        a: { value: 'a', delay: 1 },
        b: { value: 'b', delay: 2 },
        c: { value: 'c', delay: 4 }
      });
      const expected =   '---b-----c|';
      
      const result$ = source$.pipe(
        debounce(item => timer(item.delay)),
        map(item => item.value)
      );
      
      expectObservable(result$).toBe(expected);
    });
  });

  it('should test throttle with dynamic duration', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-b-c-d-e|', {
        a: { value: 'a', throttle: 2 },
        b: { value: 'b', throttle: 1 },
        c: { value: 'c', throttle: 3 },
        d: { value: 'd', throttle: 1 },
        e: { value: 'e', throttle: 2 }
      });
      const expected =   'a---c---e|';
      
      const result$ = source$.pipe(
        throttle(item => timer(item.throttle)),
        map(item => item.value)
      );
      
      expectObservable(result$).toBe(expected);
    });
  });

  it('should test timeout with recovery', () => {
    testScheduler.run(({ hot, cold, expectObservable }) => {
      const source$ = hot('--a-------b|');
      const fallback$ = cold('x|');
      const expected =     '--a---x---b|';
      
      const result$ = source$.pipe(
        timeout({
          each: 4,
          with: () => fallback$
        })
      );
      
      expectObservable(result$).toBe(expected);
    });
  });

  it('should test sampling with multiple sources', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const data$ =   hot('--a-b-c-d-e-f|');
      const sample1$ = hot('-----x----------|');
      const sample2$ = hot('----------y-----|');
      const expected =     '-----c-----f----|';
      
      const result$ = merge(
        data$.pipe(sample(sample1$)),
        data$.pipe(sample(sample2$))
      );
      
      expectObservable(result$).toBe(expected);
    });
  });
});
```

---

## Subscription Management Testing

### Testing Subscription Lifecycle

```typescript
describe('Subscription Management', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should test takeUntil unsubscription', () => {
    testScheduler.run(({ hot, expectObservable, expectSubscriptions }) => {
      const source$ = hot('--a--b--c--d--e|');
      const notifier$ = hot('--------x-------|');
      const sourceSub =    '^-------!';
      const expected =     '--a--b--c';
      
      const result$ = source$.pipe(takeUntil(notifier$));
      
      expectObservable(result$).toBe(expected);
      expectSubscriptions(source$.subscriptions).toBe(sourceSub);
    });
  });

  it('should test manual unsubscription timing', () => {
    testScheduler.run(({ hot, expectObservable, expectSubscriptions }) => {
      const source$ = hot('--a--b--c--d--e|');
      const unsub =       '-------!';
      const expected =    '--a--b--';
      
      expectObservable(source$, unsub).toBe(expected);
      expectSubscriptions(source$.subscriptions).toBe('^------!');
    });
  });

  it('should test shareReplay subscription sharing', () => {
    testScheduler.run(({ hot, expectObservable, expectSubscriptions }) => {
      const source$ = hot('--a--b--c--d|').pipe(shareReplay(1));
      
      const sub1 = '^-----------!';
      const sub2 = '----^-------!';
      const sub3 = '--------^---!';
      
      const expected1 = '--a--b--c--d|';
      const expected2 = '----b--c--d|';
      const expected3 = '--------c--d|';
      
      expectObservable(source$).toBe(expected1);
      expectObservable(source$, '----^-------!').toBe(expected2);
      expectObservable(source$, '--------^---!').toBe(expected3);
      
      // Source should only be subscribed once
      expectSubscriptions(source$.subscriptions).toBe('^-----------!');
    });
  });

  it('should test multiple subscribers with different timing', () => {
    testScheduler.run(({ hot, expectObservable, expectSubscriptions }) => {
      const source$ = hot('--a--b--c--d--e|');
      
      // First subscriber
      const sub1 = '^------!';
      const exp1 = '--a--b--';
      
      // Second subscriber (starts later)
      const sub2 = '----^--------!';
      const exp2 = '----b--c--d--';
      
      expectObservable(source$, sub1).toBe(exp1);
      expectObservable(source$, sub2).toBe(exp2);
      
      expectSubscriptions(source$.subscriptions).toBe([sub1, sub2]);
    });
  });

  it('should test finalize operator timing', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      let finalizeCalled = false;
      const source$ = hot('--a--b--c|');
      const expected =    '--a--b--c|';
      
      const result$ = source$.pipe(
        finalize(() => {
          finalizeCalled = true;
        })
      );
      
      expectObservable(result$).toBe(expected);
      
      // Verify finalize was called after completion
      result$.subscribe().add(() => {
        expect(finalizeCalled).toBe(true);
      });
    });
  });
});
```

---

## Testing State Management

### Redux-style State Testing

```typescript
interface AppState {
  users: User[];
  loading: boolean;
  error: string | null;
}

interface Action {
  type: string;
  payload?: any;
}

describe('State Management Testing', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should test state reducer with actions', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const actions$ = hot('--a--b--c|', {
        a: { type: 'LOAD_USERS_START' },
        b: { type: 'LOAD_USERS_SUCCESS', payload: [{ id: 1, name: 'John' }] },
        c: { type: 'LOAD_USERS_ERROR', payload: 'Network error' }
      });
      
      const initialState: AppState = {
        users: [],
        loading: false,
        error: null
      };
      
      const expected = '--x--y--z|';
      
      const state$ = actions$.pipe(
        scan((state, action) => {
          switch (action.type) {
            case 'LOAD_USERS_START':
              return { ...state, loading: true, error: null };
            case 'LOAD_USERS_SUCCESS':
              return { ...state, loading: false, users: action.payload };
            case 'LOAD_USERS_ERROR':
              return { ...state, loading: false, error: action.payload };
            default:
              return state;
          }
        }, initialState)
      );
      
      expectObservable(state$).toBe(expected, {
        x: { users: [], loading: true, error: null },
        y: { users: [{ id: 1, name: 'John' }], loading: false, error: null },
        z: { users: [{ id: 1, name: 'John' }], loading: false, error: 'Network error' }
      });
    });
  });

  it('should test async actions with effects', () => {
    testScheduler.run(({ hot, cold, expectObservable }) => {
      const actions$ = hot('--a-----b|', {
        a: { type: 'LOAD_USERS' },
        b: { type: 'LOAD_POSTS' }
      });
      
      const httpResponse$ = cold('---x|', {
        x: [{ id: 1, title: 'Post 1' }]
      });
      
      const expected = '-----x-----y|';
      
      const effects$ = actions$.pipe(
        switchMap(action => {
          if (action.type === 'LOAD_USERS') {
            return httpResponse$.pipe(
              map(users => ({ type: 'LOAD_USERS_SUCCESS', payload: users }))
            );
          } else if (action.type === 'LOAD_POSTS') {
            return httpResponse$.pipe(
              map(posts => ({ type: 'LOAD_POSTS_SUCCESS', payload: posts }))
            );
          }
          return EMPTY;
        })
      );
      
      expectObservable(effects$).toBe(expected, {
        x: { type: 'LOAD_USERS_SUCCESS', payload: [{ id: 1, title: 'Post 1' }] },
        y: { type: 'LOAD_POSTS_SUCCESS', payload: [{ id: 1, title: 'Post 1' }] }
      });
    });
  });
});
```

### Component State Testing

```typescript
describe('Component State Testing', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should test form state changes', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const formChanges$ = hot('--a--b--c|', {
        a: { name: 'John', email: '' },
        b: { name: 'John', email: 'john@' },
        c: { name: 'John', email: 'john@example.com' }
      });
      
      const expected = '--x--y--z|';
      
      const formState$ = formChanges$.pipe(
        map(form => ({
          ...form,
          valid: form.name.length > 0 && form.email.includes('@') && form.email.includes('.')
        }))
      );
      
      expectObservable(formState$).toBe(expected, {
        x: { name: 'John', email: '', valid: false },
        y: { name: 'John', email: 'john@', valid: false },
        z: { name: 'John', email: 'john@example.com', valid: true }
      });
    });
  });

  it('should test loading state coordination', () => {
    testScheduler.run(({ hot, cold, expectObservable }) => {
      const userLoad$ = hot('--a-------|');
      const postLoad$ = hot('----b-----|');
      
      const userResponse$ = cold('---x|');
      const postResponse$ = cold('  ---y|');
      
      const expected = '--s--t-u--|';
      
      const loadingState$ = merge(
        userLoad$.pipe(map(() => ({ type: 'user', loading: true }))),
        userLoad$.pipe(switchMap(() => userResponse$), map(() => ({ type: 'user', loading: false }))),
        postLoad$.pipe(map(() => ({ type: 'post', loading: true }))),
        postLoad$.pipe(switchMap(() => postResponse$), map(() => ({ type: 'post', loading: false })))
      ).pipe(
        scan((state, event) => ({
          ...state,
          [event.type]: event.loading
        }), { user: false, post: false })
      );
      
      expectObservable(loadingState$).toBe(expected, {
        s: { user: true, post: false },
        t: { user: true, post: true },
        u: { user: false, post: true }
      });
    });
  });
});
```

---

## Parameterized Marble Tests

### Data-Driven Testing

```typescript
describe('Parameterized Marble Tests', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  const testCases = [
    {
      name: 'should handle single value',
      input: 'a|',
      expected: 'x|',
      values: { a: 1, x: 2 }
    },
    {
      name: 'should handle multiple values',
      input: 'a-b-c|',
      expected: 'x-y-z|',
      values: { a: 1, b: 2, c: 3, x: 2, y: 4, z: 6 }
    },
    {
      name: 'should handle empty stream',
      input: '|',
      expected: '|',
      values: {}
    }
  ];

  testCases.forEach(testCase => {
    it(testCase.name, () => {
      testScheduler.run(({ hot, expectObservable }) => {
        const source$ = hot(testCase.input, testCase.values);
        const result$ = source$.pipe(map(x => x * 2));
        
        expectObservable(result$).toBe(testCase.expected, testCase.values);
      });
    });
  });

  const debounceTestCases = [
    { input: 'a|', time: 1, expected: 'a|' },
    { input: 'a-|', time: 1, expected: '-a|' },
    { input: 'a-b|', time: 2, expected: '--b|' },
    { input: 'a--b|', time: 2, expected: 'a--b|' }
  ];

  debounceTestCases.forEach(({ input, time, expected }) => {
    it(`should debounce "${input}" with time ${time}`, () => {
      testScheduler.run(({ hot, expectObservable }) => {
        const source$ = hot(input);
        const result$ = source$.pipe(debounceTime(time));
        
        expectObservable(result$).toBe(expected);
      });
    });
  });
});
```

### Test Case Generation

```typescript
class MarbleTestGenerator {
  static generateDebounceTests(configs: Array<{
    values: number;
    spacing: number;
    debounceTime: number;
  }>): Array<{
    name: string;
    input: string;
    expected: string;
    debounceTime: number;
  }> {
    return configs.map(config => {
      const input = this.generateInput(config.values, config.spacing);
      const expected = this.calculateDebounceOutput(input, config.debounceTime);
      
      return {
        name: `should debounce ${config.values} values with ${config.spacing}ms spacing and ${config.debounceTime}ms debounce`,
        input,
        expected,
        debounceTime: config.debounceTime
      };
    });
  }

  private static generateInput(values: number, spacing: number): string {
    const chars = 'abcdefghijklmnopqrstuvwxyz';
    let result = '';
    
    for (let i = 0; i < values; i++) {
      if (i > 0) {
        result += '-'.repeat(spacing);
      }
      result += chars[i];
    }
    
    return result + '|';
  }

  private static calculateDebounceOutput(input: string, debounceTime: number): string {
    // Simplified logic for example
    // In real implementation, this would need to properly calculate debounce timing
    const lastValueIndex = input.lastIndexOf(input.match(/[a-z]/)![0]);
    const padding = '-'.repeat(debounceTime);
    
    return input.substring(0, lastValueIndex) + padding + 
           input.charAt(lastValueIndex) + '|';
  }
}

describe('Generated Marble Tests', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  const testConfigs = [
    { values: 2, spacing: 1, debounceTime: 2 },
    { values: 3, spacing: 2, debounceTime: 1 },
    { values: 4, spacing: 1, debounceTime: 3 }
  ];

  const generatedTests = MarbleTestGenerator.generateDebounceTests(testConfigs);

  generatedTests.forEach(test => {
    it(test.name, () => {
      testScheduler.run(({ hot, expectObservable }) => {
        const source$ = hot(test.input);
        const result$ = source$.pipe(debounceTime(test.debounceTime));
        
        expectObservable(result$).toBe(test.expected);
      });
    });
  });
});
```

---

## Custom Marble Matchers

### Custom Assertion Helpers

```typescript
// Custom marble matchers
interface CustomMatchers<R = unknown> {
  toEmitInOrder(expected: string, values?: any): R;
  toCompleteWithin(frames: number): R;
  toEmitErrorOfType(errorType: string): R;
}

declare global {
  namespace jasmine {
    interface Matchers<T> extends CustomMatchers<T> {}
  }
}

const customMatchers: jasmine.CustomMatcherFactories = {
  toEmitInOrder: () => {
    return {
      compare: (actual: Observable<any>, expected: string, values?: any) => {
        let testScheduler: TestScheduler;
        let pass = false;
        let message = '';

        testScheduler = new TestScheduler((actualResult, expectedResult) => {
          pass = JSON.stringify(actualResult) === JSON.stringify(expectedResult);
          if (!pass) {
            message = `Expected observable to emit in order "${expected}" but got different emission pattern`;
          }
        });

        testScheduler.run(({ expectObservable }) => {
          expectObservable(actual).toBe(expected, values);
        });

        return { pass, message };
      }
    };
  },

  toCompleteWithin: () => {
    return {
      compare: (actual: Observable<any>, frames: number) => {
        let testScheduler: TestScheduler;
        let pass = false;
        let message = '';

        testScheduler = new TestScheduler((actualResult, expectedResult) => {
          const completionFrame = actualResult.find((n: any) => n.notification.kind === 'C')?.frame;
          pass = completionFrame !== undefined && completionFrame <= frames;
          message = pass 
            ? `Observable completed within ${frames} frames`
            : `Observable did not complete within ${frames} frames`;
        });

        testScheduler.run(({ expectObservable }) => {
          const pattern = '-'.repeat(frames) + '|';
          expectObservable(actual.pipe(takeUntil(timer(frames + 1)))).toBe(pattern);
        });

        return { pass, message };
      }
    };
  },

  toEmitErrorOfType: () => {
    return {
      compare: (actual: Observable<any>, errorType: string) => {
        let testScheduler: TestScheduler;
        let pass = false;
        let message = '';

        testScheduler = new TestScheduler((actualResult) => {
          const errorNotification = actualResult.find((n: any) => n.notification.kind === 'E');
          if (errorNotification) {
            const error = errorNotification.notification.error;
            pass = error.constructor.name === errorType;
            message = pass
              ? `Observable emitted error of type ${errorType}`
              : `Observable emitted error of type ${error.constructor.name}, expected ${errorType}`;
          } else {
            message = 'Observable did not emit an error';
          }
        });

        testScheduler.run(({ expectObservable }) => {
          expectObservable(actual).toBe('#');
        });

        return { pass, message };
      }
    };
  }
};

describe('Custom Marble Matchers', () => {
  beforeEach(() => {
    jasmine.addMatchers(customMatchers);
  });

  it('should use custom emission order matcher', () => {
    const testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });

    testScheduler.run(({ hot }) => {
      const source$ = hot('a-b-c|');
      expect(source$).toEmitInOrder('a-b-c|');
    });
  });

  it('should use custom completion timing matcher', () => {
    const testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });

    testScheduler.run(({ of }) => {
      const source$ = of(1, 2, 3);
      expect(source$).toCompleteWithin(0); // Synchronous completion
    });
  });

  it('should use custom error type matcher', () => {
    const testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });

    testScheduler.run(() => {
      const source$ = throwError(() => new TypeError('Type error'));
      expect(source$).toEmitErrorOfType('TypeError');
    });
  });
});
```

---

## Performance Testing

### Testing Performance Characteristics

```typescript
describe('Performance Testing with Marbles', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should test operator performance with large datasets', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      // Generate large marble string
      const largeInput = Array(1000).fill(0).map((_, i) => 
        String.fromCharCode(97 + (i % 26))
      ).join('-') + '|';
      
      const source$ = hot(largeInput);
      const result$ = source$.pipe(
        filter(value => value === 'a'),
        take(10)
      );
      
      // Should complete quickly with only 'a' values
      const expected = 'a-------------------------a-------------------------a-------------------------a-------------------------a-------------------------a-------------------------a-------------------------a-------------------------a---------(a|)';
      
      expectObservable(result$).toBe(expected);
    });
  });

  it('should test memory usage patterns', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-b-c-d-e-f-g-h-i-j|');
      
      // Test that scan doesn't leak memory with proper cleanup
      const result$ = source$.pipe(
        scan((acc, curr) => [...acc, curr], [] as string[]),
        takeLast(1)
      );
      
      const expected = '-------------------x|';
      
      expectObservable(result$).toBe(expected, {
        x: ['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j']
      });
    });
  });

  it('should test subscription management performance', () => {
    testScheduler.run(({ hot, expectObservable, expectSubscriptions }) => {
      const source$ = hot('--a--b--c--d--e|');
      
      // Test that switchMap properly unsubscribes
      const result$ = source$.pipe(
        switchMap(value => {
          return hot('---x|').pipe(
            map(x => value + x)
          );
        })
      );
      
      const expected = '-----x-----y-----z|';
      
      expectObservable(result$).toBe(expected, {
        x: 'ax', y: 'bx', z: 'cx'
      });
      
      // Verify proper subscription management
      const subscriptions = source$.subscriptions;
      expectSubscriptions(subscriptions).toBe('^--------------!');
    });
  });

  it('should test backpressure handling', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      // Fast producer
      const fastSource$ = hot('abcdefghij|');
      
      // Slow consumer simulation
      const result$ = fastSource$.pipe(
        concatMap(value => timer(2).pipe(map(() => value)))
      );
      
      // Should handle backpressure by queuing
      const expected = '--a--b--c--d--e--f--g--h--i--(j|)';
      
      expectObservable(result$).toBe(expected);
    });
  });
});
```

---

## Integration Testing

### Testing Angular Services with Marble Tests

```typescript
@Injectable()
export class ApiService {
  constructor(private http: HttpClient) {}

  getUsersWithRetry(): Observable<User[]> {
    return this.http.get<User[]>('/api/users').pipe(
      retry({ count: 3, delay: 1000 }),
      catchError(() => of([]))
    );
  }

  searchUsers(query$: Observable<string>): Observable<User[]> {
    return query$.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(query => 
        query ? this.http.get<User[]>(`/api/users/search?q=${query}`) : of([])
      ),
      catchError(() => of([]))
    );
  }
}

describe('ApiService Integration', () => {
  let service: ApiService;
  let httpMock: HttpTestingController;
  let testScheduler: TestScheduler;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [ApiService]
    });

    service = TestBed.inject(ApiService);
    httpMock = TestBed.inject(HttpTestingController);
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('should test search with debouncing using marbles', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const query$ = hot('a-b--c---|', {
        a: 'jo',
        b: 'joh',
        c: 'john'
      });
      
      const result$ = service.searchUsers(query$);
      const expected =   '------x-y|';
      
      expectObservable(result$).toBe(expected, {
        x: [{ id: 1, name: 'John' }],
        y: [{ id: 1, name: 'John Doe' }]
      });

      // Simulate HTTP responses
      testScheduler.flush();
      
      const requests = httpMock.match('/api/users/search');
      expect(requests.length).toBe(2); // Only 'joh' and 'john' should trigger requests
      
      requests[0].flush([{ id: 1, name: 'John' }]);
      requests[1].flush([{ id: 1, name: 'John Doe' }]);
    });
  });

  it('should test retry behavior with marbles', () => {
    testScheduler.run(({ cold, expectObservable }) => {
      // Mock HTTP to fail 3 times then succeed
      let callCount = 0;
      const mockHttp = {
        get: () => {
          callCount++;
          if (callCount <= 3) {
            return cold('#'); // Error
          } else {
            return cold('(a|)', { a: [{ id: 1, name: 'John' }] });
          }
        }
      };
      
      service = new ApiService(mockHttp as any);
      
      const result$ = service.getUsersWithRetry();
      const expected = '(a|)'; // Should eventually succeed
      
      expectObservable(result$).toBe(expected, {
        a: [{ id: 1, name: 'John' }]
      });
    });
  });
});
```

### Testing Component Interactions

```typescript
@Component({
  selector: 'app-search',
  template: `
    <input [formControl]="searchControl" placeholder="Search users...">
    <div *ngFor="let user of users$ | async">{{ user.name }}</div>
    <div *ngIf="loading$ | async">Loading...</div>
  `
})
export class SearchComponent implements OnInit {
  searchControl = new FormControl('');
  users$: Observable<User[]>;
  loading$: Observable<boolean>;

  constructor(private apiService: ApiService) {}

  ngOnInit(): void {
    const search$ = this.searchControl.valueChanges.pipe(
      startWith(''),
      share()
    );

    this.users$ = this.apiService.searchUsers(search$);
    
    this.loading$ = search$.pipe(
      switchMap(() => 
        concat(
          of(true),
          this.users$.pipe(take(1), map(() => false))
        )
      )
    );
  }
}

describe('SearchComponent Integration', () => {
  let component: SearchComponent;
  let fixture: ComponentFixture<SearchComponent>;
  let mockApiService: jasmine.SpyObj<ApiService>;
  let testScheduler: TestScheduler;

  beforeEach(() => {
    const spy = jasmine.createSpyObj('ApiService', ['searchUsers']);

    TestBed.configureTestingModule({
      declarations: [SearchComponent],
      imports: [ReactiveFormsModule],
      providers: [
        { provide: ApiService, useValue: spy }
      ]
    });

    fixture = TestBed.createComponent(SearchComponent);
    component = fixture.componentInstance;
    mockApiService = TestBed.inject(ApiService) as jasmine.SpyObj<ApiService>;
    
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should coordinate search and loading states', () => {
    testScheduler.run(({ hot, cold, expectObservable }) => {
      const searchResults$ = cold('---a|', {
        a: [{ id: 1, name: 'John' }]
      });
      
      mockApiService.searchUsers.and.returnValue(searchResults$);
      
      component.ngOnInit();
      
      // Simulate form control changes
      const formChanges$ = hot('--x--y|', {
        x: 'jo',
        y: 'john'
      });
      
      formChanges$.subscribe(value => {
        component.searchControl.setValue(value);
      });
      
      // Test loading state coordination
      const expectedLoading = 'a-b--c--d|';
      const expectedUsers =   '-----u-----v|';
      
      expectObservable(component.loading$).toBe(expectedLoading, {
        a: true, b: false, c: true, d: false
      });
      
      expectObservable(component.users$).toBe(expectedUsers, {
        u: [{ id: 1, name: 'John' }],
        v: [{ id: 1, name: 'John' }]
      });
    });
  });
});
```

---

## Debugging Techniques

### Marble-Assisted Debugging

```typescript
describe('Debugging with Marbles', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should debug complex operator chains', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('--a--b--c--d|');
      
      // Add intermediate logging for debugging
      const debug$ = source$.pipe(
        tap(value => console.log('Source:', value)),
        debounceTime(2),
        tap(value => console.log('After debounce:', value)),
        map(value => value.toUpperCase()),
        tap(value => console.log('After map:', value))
      );
      
      const expected = '----b----d|';
      
      expectObservable(debug$).toBe(expected, {
        b: 'B', d: 'D'
      });
    });
  });

  it('should visualize timing issues', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const trigger$ = hot('--a-----b--|');
      const data$ =    hot('----x-y-z--|');
      
      // Problem: withLatestFrom might miss values
      const result$ = trigger$.pipe(
        withLatestFrom(data$),
        map(([trigger, data]) => ({ trigger, data }))
      );
      
      // Visualize what actually happens
      const expected = '----w-----z|';
      
      expectObservable(result$).toBe(expected, {
        w: { trigger: 'a', data: 'x' },
        z: { trigger: 'b', data: 'z' }
      });
    });
  });

  it('should debug subscription timing', () => {
    testScheduler.run(({ hot, cold, expectObservable, expectSubscriptions }) => {
      const outer$ = hot('--a--b--|');
      const inner$ = cold(' --x|');
      
      const result$ = outer$.pipe(
        switchMap(() => inner$)
      );
      
      const expected = '----x--x|';
      const innerSubs = [
        '  --^-!',    // First inner subscription
        '  -----^-!'  // Second inner subscription
      ];
      
      expectObservable(result$).toBe(expected);
      expectSubscriptions(inner$.subscriptions).toBe(innerSubs);
    });
  });

  it('should debug error propagation', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('--a--#--b|');
      
      const result$ = source$.pipe(
        map(value => value.toUpperCase()),
        catchError(error => {
          console.log('Caught error:', error);
          return of('ERROR');
        }),
        tap(value => console.log('Final value:', value))
      );
      
      const expected = '--A--E-----|';
      
      expectObservable(result$).toBe(expected, {
        A: 'A', E: 'ERROR'
      });
    });
  });
});
```

---

## Advanced Patterns

### Testing Custom Operators

```typescript
// Custom operator implementation
export function bufferUntilQuiet<T>(quietTime: number): OperatorFunction<T, T[]> {
  return (source: Observable<T>) => {
    return source.pipe(
      buffer(source.pipe(debounceTime(quietTime))),
      filter(buffer => buffer.length > 0)
    );
  };
}

describe('Custom Operator Testing', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should test bufferUntilQuiet operator', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      const source$ = hot('a-b--c-d----e-f|');
      const expected =   '----x-------y--z|';
      
      const result$ = source$.pipe(bufferUntilQuiet(2));
      
      expectObservable(result$).toBe(expected, {
        x: ['a', 'b'],
        y: ['c', 'd'],
        z: ['e', 'f']
      });
    });
  });

  it('should test edge cases for custom operator', () => {
    testScheduler.run(({ hot, expectObservable }) => {
      // Single value
      const single$ = hot('a---|');
      const expectedSingle = '---x|';
      
      // Empty stream
      const empty$ = hot('|');
      const expectedEmpty = '|';
      
      // Error stream
      const error$ = hot('a-#');
      const expectedError = 'a-#';
      
      expectObservable(single$.pipe(bufferUntilQuiet(2)))
        .toBe(expectedSingle, { x: ['a'] });
      
      expectObservable(empty$.pipe(bufferUntilQuiet(2)))
        .toBe(expectedEmpty);
      
      expectObservable(error$.pipe(bufferUntilQuiet(2)))
        .toBe(expectedError);
    });
  });
});
```

### Testing Reactive Patterns

```typescript
// Event sourcing pattern
interface Event {
  type: string;
  payload: any;
  timestamp: number;
}

class EventStore {
  private events$ = new BehaviorSubject<Event[]>([]);

  append(event: Event): void {
    const currentEvents = this.events$.value;
    this.events$.next([...currentEvents, event]);
  }

  getEvents(): Observable<Event[]> {
    return this.events$.asObservable();
  }

  projectState<T>(initialState: T, reducer: (state: T, event: Event) => T): Observable<T> {
    return this.events$.pipe(
      map(events => events.reduce(reducer, initialState)),
      distinctUntilChanged()
    );
  }
}

describe('Event Sourcing Pattern Testing', () => {
  let testScheduler: TestScheduler;
  let eventStore: EventStore;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
    eventStore = new EventStore();
  });

  it('should test event sourcing projection', () => {
    testScheduler.run(({ expectObservable }) => {
      const initialState = { count: 0 };
      
      const state$ = eventStore.projectState(initialState, (state, event) => {
        switch (event.type) {
          case 'INCREMENT':
            return { count: state.count + 1 };
          case 'DECREMENT':
            return { count: state.count - 1 };
          default:
            return state;
        }
      });

      const expected = 'a-b-c-d|';
      
      // Simulate events being added
      setTimeout(() => {
        eventStore.append({ type: 'INCREMENT', payload: null, timestamp: 1 });
        eventStore.append({ type: 'INCREMENT', payload: null, timestamp: 2 });
        eventStore.append({ type: 'DECREMENT', payload: null, timestamp: 3 });
      });

      expectObservable(state$.pipe(take(4))).toBe(expected, {
        a: { count: 0 },
        b: { count: 1 },
        c: { count: 2 },
        d: { count: 1 }
      });
    });
  });
});
```

---

## Summary

This lesson covered advanced marble testing techniques for complex RxJS scenarios:

### Key Advanced Concepts:
1. **Higher-Order Observables** - Testing nested and flattening operations
2. **Complex Timing** - Multi-stream synchronization and coordination
3. **Subscription Management** - Testing lifecycle and cleanup
4. **State Management** - Testing reactive state patterns
5. **Parameterized Tests** - Data-driven and generated test cases
6. **Custom Matchers** - Extended assertion capabilities
7. **Performance Testing** - Memory and timing characteristics
8. **Integration Testing** - Angular services and components
9. **Debugging Techniques** - Visualizing complex observable chains
10. **Advanced Patterns** - Custom operators and reactive architectures

### Testing Strategies:
- **Comprehensive Coverage** of edge cases and error scenarios
- **Visual Debugging** using marble diagrams for complex flows
- **Performance Verification** for memory and timing constraints
- **Integration Validation** for real-world Angular patterns
- **Custom Utilities** for reusable testing patterns

### Best Practices:
- Use parameterized tests for systematic coverage
- Create custom matchers for domain-specific assertions
- Test subscription management explicitly
- Visualize complex timing relationships
- Generate test cases for comprehensive coverage
- Debug with intermediate marble visualizations

Advanced marble testing provides the tools needed to thoroughly test complex reactive systems, ensuring reliability and performance in production Angular applications.

---

## Exercises

1. **Higher-Order Testing**: Test a complex switchMap scenario with 3 levels of nesting
2. **Custom Operator**: Create and test a custom debouncing operator with dynamic timing
3. **State Management**: Test a complete Redux-style state management system
4. **Performance**: Test memory usage patterns with large datasets
5. **Integration**: Test an Angular component with multiple reactive streams
6. **Debugging**: Debug a complex timing issue using marble visualization
7. **Custom Matcher**: Create a custom matcher for testing observable completion timing

## Additional Resources

- [Advanced RxJS Testing Patterns](https://blog.angular.io/rxjs-advanced-testing-patterns)
- [Testing Higher-Order Observables](https://www.learnrxjs.io/learn-rxjs/operators/testing)
- [Marble Testing Best Practices](https://medium.com/@benlesh/rxjs-testing-with-marble-diagrams)
- [Performance Testing RxJS](https://rxjs.dev/guide/testing/performance)
