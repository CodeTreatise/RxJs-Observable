# Core Theory & Reactive Programming Concepts 🟢

## 🎯 Learning Objectives
By the end of this lesson, you will understand:
- The fundamental principles of reactive programming
- Core reactive programming concepts and terminology
- How reactive programming differs from imperative programming
- The mathematical foundations behind RxJS
- Key reactive patterns and their applications

## 🌊 What is Reactive Programming?

**Reactive Programming** is a programming paradigm oriented around **data flows** and the **propagation of change**. It's about building systems that **react** to changes automatically.

### Core Principle
> "In reactive programming, everything is a stream"

### The Reactive Manifesto
Modern reactive systems should be:
- **Responsive** - React quickly to users
- **Resilient** - Stay responsive in the face of failure  
- **Elastic** - React to changes in input rate
- **Message Driven** - Rely on asynchronous message-passing

## 🔄 Imperative vs Reactive Programming

### Imperative Approach (Traditional)
```javascript
// Imperative: We tell HOW to do something step by step
let total = 0;
let numbers = [1, 2, 3, 4, 5];

for (let i = 0; i < numbers.length; i++) {
  if (numbers[i] % 2 === 0) {
    total += numbers[i] * 2;
  }
}
console.log(total); // 12
```

### Reactive Approach
```javascript
// Reactive: We declare WHAT we want to happen
const numbers$ = from([1, 2, 3, 4, 5]);

numbers$.pipe(
  filter(n => n % 2 === 0),  // Keep even numbers
  map(n => n * 2),           // Double them
  reduce((acc, n) => acc + n, 0) // Sum them up
).subscribe(total => console.log(total)); // 12
```

### Key Differences

| Imperative | Reactive |
|------------|----------|
| **How** to do it | **What** should happen |
| Manual state management | Automatic propagation |
| Pull-based | Push-based |
| Synchronous by default | Asynchronous by design |
| Hard to compose | Easy to compose |

## 📊 Core Reactive Concepts

### 1. **Streams (Observables)**
A stream is a sequence of ongoing events ordered in time.

```
Time ──────────────────────────────────▶
      │    │    │    │    │    │
      ▼    ▼    ▼    ▼    ▼    ▼
     [a]  [b]  [c]  [d]  [e]  [|]
```

```javascript
// Examples of streams in real applications
const mouseClicks$ = fromEvent(document, 'click');
const httpRequests$ = this.http.get('/api/data');
const userInput$ = fromEvent(searchInput, 'keyup');
const timer$ = interval(1000);
```

### 2. **Observers (Subscribers)**
Observers consume the values emitted by streams.

```javascript
const observer = {
  next: value => console.log('Next value:', value),
  error: err => console.error('Error occurred:', err),
  complete: () => console.log('Stream completed')
};

observable$.subscribe(observer);
```

### 3. **Operators (Stream Transformers)**
Functions that take an Observable and return a new Observable.

```javascript
// Operators transform streams
const transformedStream$ = originalStream$.pipe(
  map(x => x * 2),        // Transform each value
  filter(x => x > 10),    // Keep only certain values
  debounceTime(300),      // Control timing
  distinctUntilChanged()  // Remove duplicates
);
```

## 🧮 Mathematical Foundations

RxJS is based on **mathematical principles** from functional programming:

### 1. **Functors**
Objects that can be mapped over (have a `map` function).

```javascript
// Arrays are functors
[1, 2, 3].map(x => x * 2); // [2, 4, 6]

// Observables are functors
of(1, 2, 3).pipe(
  map(x => x * 2)
); // Stream: 2, 4, 6
```

### 2. **Monads**
Functors with additional `flatMap` (chain) operations.

```javascript
// Array flatMap
[1, 2, 3].flatMap(x => [x, x * 2]); // [1, 2, 2, 4, 3, 6]

// Observable flatMap (switchMap, mergeMap, etc.)
of(1, 2, 3).pipe(
  switchMap(x => of(x, x * 2))
); // Stream: 1, 2, 2, 4, 3, 6
```

### 3. **Category Theory**
Provides the mathematical framework for composition.

```javascript
// Composition laws
const f = x => x * 2;
const g = x => x + 1;

// Function composition
const composed = x => g(f(x));

// Observable composition
source$.pipe(
  map(f),
  map(g)
); // Equivalent to map(x => g(f(x)))
```

## 🔄 Key Reactive Patterns

### 1. **Data Flow Pattern**
```
Input Stream → Transformation → Output Stream
```

```javascript
// Search with reactive data flow
const searchResults$ = searchInput$.pipe(
  debounceTime(300),          // Wait for user to stop typing
  distinctUntilChanged(),     // Skip duplicate queries
  switchMap(query =>          // Switch to new search
    this.api.search(query)
  ),
  catchError(err =>          // Handle errors gracefully
    of([])
  )
);
```

### 2. **Event Sourcing Pattern**
```javascript
// All state changes as events
const userActions$ = merge(
  loginClicks$.pipe(map(() => ({ type: 'LOGIN' }))),
  logoutClicks$.pipe(map(() => ({ type: 'LOGOUT' }))),
  profileUpdates$.pipe(map(data => ({ type: 'UPDATE_PROFILE', data })))
);

const appState$ = userActions$.pipe(
  scan((state, action) => reducer(state, action), initialState)
);
```

### 3. **CQRS (Command Query Responsibility Segregation)**
```javascript
// Commands (write operations)
const commands$ = merge(
  createUser$,
  updateUser$,
  deleteUser$
);

// Queries (read operations)  
const queries$ = merge(
  getUserList$,
  getUserDetails$,
  searchUsers$
);

// Separate read and write models
commands$.subscribe(command => writeModel.handle(command));
queries$.subscribe(query => readModel.handle(query));
```

## 🎭 Observable Contract

Every Observable must follow this contract:

```
OnNext* (OnError | OnCompleted)?
```

- **OnNext**: Can emit zero or more values
- **OnError**: Can emit at most one error (terminal)
- **OnCompleted**: Can emit at most one completion signal (terminal)

```javascript
// Valid Observable sequences
// 1. Empty
// ──|

// 2. Single value
// ──a──|

// 3. Multiple values
// ──a──b──c──|

// 4. Error
// ──a──b──X

// 5. Infinite
// ──a──b──c──d──e──...
```

## 🔧 Reactive State Management

### Traditional State Management Problems
```javascript
// Imperative state management
class UserService {
  private users = [];
  private loading = false;
  private error = null;

  async loadUsers() {
    this.loading = true;
    try {
      this.users = await this.api.getUsers();
      this.error = null;
    } catch (err) {
      this.error = err;
    } finally {
      this.loading = false;
    }
    this.notifyComponents(); // Manual notification
  }
}
```

### Reactive State Management
```javascript
// Reactive state management
class UserService {
  private loadUsers$ = new Subject<void>();
  
  users$ = this.loadUsers$.pipe(
    switchMap(() => this.api.getUsers()),
    shareReplay(1)
  );
  
  loading$ = merge(
    this.loadUsers$.pipe(map(() => true)),
    this.users$.pipe(map(() => false))
  );
  
  error$ = this.users$.pipe(
    map(() => null),
    catchError(err => of(err))
  );
  
  // Automatic reactive updates
  state$ = combineLatest([
    this.users$,
    this.loading$,
    this.error$
  ]).pipe(
    map(([users, loading, error]) => ({ users, loading, error }))
  );
}
```

## 🌊 Stream Composition Patterns

### 1. **Sequential Composition**
```javascript
// One operation after another
const sequential$ = firstOperation$.pipe(
  switchMap(result1 => secondOperation$(result1)),
  switchMap(result2 => thirdOperation$(result2))
);
```

### 2. **Parallel Composition**
```javascript
// Multiple operations at once
const parallel$ = combineLatest([
  operation1$,
  operation2$,
  operation3$
]).pipe(
  map(([result1, result2, result3]) => ({
    result1, result2, result3
  }))
);
```

### 3. **Conditional Composition**
```javascript
// Based on conditions
const conditional$ = source$.pipe(
  switchMap(value => 
    value > 10 
      ? expensiveOperation$(value)
      : of(value)
  )
);
```

## 🎯 Reactive Principles in Practice

### 1. **Single Responsibility**
Each Observable should have one clear purpose:

```javascript
// Good: Single responsibility
const userClicks$ = fromEvent(button, 'click');
const validatedClicks$ = userClicks$.pipe(
  filter(() => form.valid)
);
const apiCalls$ = validatedClicks$.pipe(
  switchMap(() => this.api.submitForm())
);

// Avoid: Multiple responsibilities in one stream
const messyStream$ = fromEvent(button, 'click').pipe(
  filter(() => form.valid),
  switchMap(() => this.api.submitForm()),
  tap(result => this.showNotification(result)),
  map(result => result.data),
  retry(3)
);
```

### 2. **Immutability**
Don't modify existing data, create new data:

```javascript
// Good: Immutable transformations
const updatedUsers$ = users$.pipe(
  map(users => users.map(user => ({
    ...user,
    lastSeen: new Date()
  })))
);

// Avoid: Mutating existing data
const badUsers$ = users$.pipe(
  tap(users => {
    users.forEach(user => user.lastSeen = new Date()); // Mutation!
  })
);
```

### 3. **Declarative Style**
Describe what you want, not how to get it:

```javascript
// Good: Declarative
const activeUsers$ = users$.pipe(
  map(users => users.filter(user => user.active)),
  map(activeUsers => activeUsers.length)
);

// Avoid: Imperative style in reactive code
const badActiveCount$ = users$.pipe(
  tap(users => {
    let count = 0;
    for (let i = 0; i < users.length; i++) {
      if (users[i].active) count++;
    }
    return count;
  })
);
```

## 🎨 Advanced Reactive Concepts

### 1. **Higher-Order Observables**
Observables that emit other Observables:

```javascript
// Observable of Observables
const higherOrder$ = interval(1000).pipe(
  map(i => timer(i * 100).pipe(map(() => `Inner ${i}`)))
);

// Flatten with different strategies
const flattened$ = higherOrder$.pipe(
  switchMap(inner$ => inner$) // or mergeMap, concatMap, exhaustMap
);
```

### 2. **Reactive Extensions**
Custom operators for domain-specific logic:

```javascript
// Custom operator
function retryWithBackoff(maxRetries: number, delayMs: number) {
  return <T>(source: Observable<T>) => 
    source.pipe(
      retryWhen(errors => 
        errors.pipe(
          scan((retryCount, err) => {
            if (retryCount >= maxRetries) throw err;
            return retryCount + 1;
          }, 0),
          delay(delayMs)
        )
      )
    );
}

// Usage
api.getData().pipe(
  retryWithBackoff(3, 1000)
).subscribe();
```

### 3. **Reactive Pipelines**
Complex data processing chains:

```javascript
const dataPipeline$ = rawData$.pipe(
  // Data validation
  filter(data => data.isValid),
  
  // Data transformation
  map(data => ({
    ...data,
    processedAt: new Date(),
    normalized: normalizeData(data)
  })),
  
  // Error handling
  catchError(err => {
    console.error('Pipeline error:', err);
    return of(null);
  }),
  
  // Remove null results
  filter(data => data !== null),
  
  // Batch processing
  bufferTime(1000),
  filter(batch => batch.length > 0),
  
  // Final processing
  switchMap(batch => processBatch(batch))
);
```

## 🧪 Testing Reactive Code

### Marble Testing Concepts
```javascript
// Testing with marble diagrams
it('should transform values correctly', () => {
  const source = cold('--a--b--c--|');
  const expected =    '--x--y--z--|';
  
  const result = source.pipe(
    map(value => value.toUpperCase())
  );
  
  expectObservable(result).toBe(expected, {
    x: 'A', y: 'B', z: 'C'
  });
});
```

## 💡 Best Practices

### 1. **Naming Conventions**
```javascript
// Use $ suffix for Observables
const users$ = this.userService.getUsers();
const clicks$ = fromEvent(button, 'click');

// Use descriptive names
const validatedFormSubmissions$ = formSubmissions$.pipe(
  filter(form => form.valid)
);
```

### 2. **Error Handling Strategy**
```javascript
const robustOperation$ = riskyOperation$.pipe(
  retry(3),                    // Retry failed operations
  catchError(err => {
    this.logger.error(err);    // Log errors
    return of(defaultValue);   // Provide fallback
  })
);
```

### 3. **Memory Management**
```javascript
class Component implements OnDestroy {
  private destroy$ = new Subject<void>();
  
  ngOnInit() {
    this.dataService.data$.pipe(
      takeUntil(this.destroy$)   // Automatic cleanup
    ).subscribe(data => {
      // Handle data
    });
  }
  
  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## 🎯 Quick Assessment

Test your understanding:

1. What are the four principles of the Reactive Manifesto?
2. How does reactive programming differ from imperative programming?
3. What is the Observable contract?
4. Name three mathematical concepts that RxJS is based on.

### Answers:
1. **Responsive**, **Resilient**, **Elastic**, **Message Driven**
2. **Reactive**: Declarative, what to do, push-based, automatic propagation
   **Imperative**: Step-by-step, how to do, pull-based, manual state management
3. **OnNext* (OnError | OnCompleted)?** - Zero or more values, at most one error or completion
4. **Functors** (map), **Monads** (flatMap), **Category Theory** (composition)

## 🌟 Key Takeaways

- **Reactive programming** is about automatic propagation of change
- **Streams** are the fundamental building blocks
- **Declarative style** makes code more readable and maintainable
- **Composition** allows building complex operations from simple parts
- **Mathematical foundations** ensure consistent and predictable behavior
- **Immutability** and **single responsibility** are core principles

## 🚀 Next Steps

You now understand the theoretical foundations of reactive programming. Next, we'll learn how to visualize reactive streams using **Marble Diagrams** - a powerful tool for understanding Observable behavior.

**Next Lesson**: [Understanding Marble Diagrams](./03-marble-diagrams-intro.md) 🟢

---

🎉 **Excellent work!** You've mastered the core concepts of reactive programming. These principles will guide you throughout your RxJS journey and help you write more elegant, maintainable reactive code.
