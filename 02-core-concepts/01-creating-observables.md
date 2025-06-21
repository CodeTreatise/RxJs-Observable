# Creating Observables with RxJS 🟢

## 🎯 Learning Objectives
By the end of this lesson, you will understand:
- Different ways to create Observables in RxJS
- When to use each creation method
- How to create custom Observables from scratch
- Best practices for Observable creation
- Performance considerations for different creation patterns

## 🏗️ Observable Creation Methods Overview

RxJS provides multiple ways to create Observables, each suited for different scenarios:

### Creation Method Categories

```
┌─────────────────────────────────────────────────────────┐
│                Observable Creation                      │
├─────────────────────────────────────────────────────────┤
│ 1. From Values                                          │
│    ├── of()           - Static values                   │
│    ├── from()         - Arrays, Promises, Iterables    │
│    └── range()        - Number sequences               │
├─────────────────────────────────────────────────────────┤
│ 2. From Events                                          │
│    ├── fromEvent()    - DOM events                     │
│    ├── fromEventPattern() - Custom event APIs          │
│    └── interval()     - Timer-based events             │
├─────────────────────────────────────────────────────────┤
│ 3. From Async Sources                                   │
│    ├── fromPromise()  - Promise conversion             │
│    ├── ajax()         - HTTP requests                  │
│    └── webSocket()    - WebSocket connections          │
├─────────────────────────────────────────────────────────┤
│ 4. Custom Creation                                      │
│    ├── new Observable() - Manual construction          │
│    ├── defer()        - Lazy creation                  │
│    └── generate()     - Stateful generation            │
└─────────────────────────────────────────────────────────┘
```

## 📊 1. Creating from Values

### of() - Static Values

Creates an Observable that emits provided values synchronously and then completes.

```typescript
import { of } from 'rxjs';

// Single value
const single$ = of(42);
// Marble: 42|

// Multiple values
const multiple$ = of(1, 2, 3, 'hello', true);
// Marble: 1-2-3-hello-true|

// Objects and arrays
const complex$ = of(
  { id: 1, name: 'John' },
  [1, 2, 3],
  new Date()
);

// Usage
single$.subscribe(value => console.log('Single:', value));
// Output: Single: 42

multiple$.subscribe(value => console.log('Multiple:', value));
// Output: Multiple: 1, Multiple: 2, Multiple: 3, Multiple: hello, Multiple: true
```

### from() - Converting Collections and Promises

Converts various data sources into Observables.

```typescript
import { from } from 'rxjs';

// From array
const fromArray$ = from([1, 2, 3, 4, 5]);
// Marble: 1-2-3-4-5|

// From Promise
const promise = Promise.resolve('Hello Promise');
const fromPromise$ = from(promise);
// Marble: ----Hello Promise|

// From string (iterable)
const fromString$ = from('Hello');
// Marble: H-e-l-l-o|

// From Set
const set = new Set([1, 2, 3, 2, 1]); // Set removes duplicates
const fromSet$ = from(set);
// Marble: 1-2-3|

// From Map
const map = new Map([['a', 1], ['b', 2], ['c', 3]]);
const fromMap$ = from(map);
// Marble: ['a',1]-['b',2]-['c',3]|

// From Generator function
function* numberGenerator() {
  yield 1;
  yield 2;
  yield 3;
}
const fromGenerator$ = from(numberGenerator());
// Marble: 1-2-3|

// Usage examples
fromArray$.subscribe(value => console.log('Array:', value));
fromPromise$.subscribe(value => console.log('Promise:', value));
```

### range() - Number Sequences

Creates an Observable that emits a sequence of numbers.

```typescript
import { range } from 'rxjs';

// Range from 1 to 5
const range1to5$ = range(1, 5);
// Marble: 1-2-3-4-5|

// Range from 0 to 3
const range0to3$ = range(0, 4);
// Marble: 0-1-2-3|

// Empty range
const emptyRange$ = range(1, 0);
// Marble: |

// Usage
range1to5$.subscribe(value => console.log('Range:', value));
// Output: Range: 1, Range: 2, Range: 3, Range: 4, Range: 5
```

## ⏰ 2. Creating from Time-Based Events

### interval() - Regular Intervals

Creates an Observable that emits sequential numbers at specified intervals.

```typescript
import { interval } from 'rxjs';
import { take } from 'rxjs/operators';

// Emit every 1000ms (1 second)
const every1s$ = interval(1000);
// Marble: ----0----1----2----3----4... (infinite)

// Limited to first 5 values
const limited$ = interval(1000).pipe(take(5));
// Marble: ----0----1----2----3----4|

// Usage
limited$.subscribe({
  next: value => console.log('Interval:', value),
  complete: () => console.log('Interval complete')
});
```

### timer() - Delayed and Interval Emissions

Creates an Observable that starts emitting after a delay, optionally at intervals.

```typescript
import { timer } from 'rxjs';

// Single emission after 2 seconds
const delayed$ = timer(2000);
// Marble: --------0|

// First emission after 1s, then every 500ms
const delayedInterval$ = timer(1000, 500);
// Marble: ----0--1--2--3--4... (infinite)

// With take to limit emissions
const limitedTimer$ = timer(1000, 500).pipe(take(4));
// Marble: ----0--1--2--3|

// Usage
delayed$.subscribe(value => console.log('Timer delayed:', value));
delayedInterval$.pipe(take(3)).subscribe(value => console.log('Timer interval:', value));
```

## 🎯 3. Creating from Events

### fromEvent() - DOM and Node Events

Converts event targets into Observables.

```typescript
import { fromEvent } from 'rxjs';
import { map, debounceTime } from 'rxjs/operators';

// Mouse clicks
const clicks$ = fromEvent(document, 'click');
// Marble: ----c----c--c--------c... (user clicks)

// Button clicks with target element
const button = document.getElementById('myButton');
const buttonClicks$ = fromEvent(button, 'click');

// Keyboard events
const keyPresses$ = fromEvent(document, 'keydown');

// Window resize events
const resize$ = fromEvent(window, 'resize');

// Usage examples
clicks$.subscribe(event => {
  console.log('Click at:', event.clientX, event.clientY);
});

// Advanced: Search input with debouncing
const searchInput = document.getElementById('search') as HTMLInputElement;
const searchValue$ = fromEvent(searchInput, 'input').pipe(
  map((event: Event) => (event.target as HTMLInputElement).value),
  debounceTime(300) // Wait 300ms after user stops typing
);

searchValue$.subscribe(value => {
  console.log('Search for:', value);
});
```

### fromEventPattern() - Custom Event APIs

For event APIs that don't follow standard addEventListener pattern.

```typescript
import { fromEventPattern } from 'rxjs';

// Custom event emitter
class CustomEmitter {
  private listeners: Function[] = [];

  addListener(handler: Function): void {
    this.listeners.push(handler);
  }

  removeListener(handler: Function): void {
    const index = this.listeners.indexOf(handler);
    if (index > -1) {
      this.listeners.splice(index, 1);
    }
  }

  emit(data: any): void {
    this.listeners.forEach(handler => handler(data));
  }
}

const emitter = new CustomEmitter();

// Convert to Observable
const customEvents$ = fromEventPattern(
  handler => emitter.addListener(handler),    // Add handler
  handler => emitter.removeListener(handler)  // Remove handler
);

// Usage
customEvents$.subscribe(data => console.log('Custom event:', data));

// Emit some events
emitter.emit('Hello');
emitter.emit('World');
```

## 🌐 4. Creating from Async Sources

### AJAX Requests

```typescript
import { ajax } from 'rxjs/ajax';
import { map, catchError } from 'rxjs/operators';
import { of } from 'rxjs';

// Simple GET request
const users$ = ajax.getJSON('https://jsonplaceholder.typicode.com/users');

// POST request
const postData$ = ajax({
  url: 'https://jsonplaceholder.typicode.com/posts',
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: {
    title: 'My Post',
    body: 'Post content',
    userId: 1
  }
});

// With error handling
const safeRequest$ = ajax.getJSON('https://api.example.com/data').pipe(
  catchError(error => {
    console.error('Request failed:', error);
    return of([]); // Return empty array on error
  })
);

// Usage
users$.subscribe(users => console.log('Users:', users));
```

### WebSocket Connections

```typescript
import { webSocket } from 'rxjs/webSocket';

// WebSocket Observable
const socket$ = webSocket('ws://localhost:8080');

// Send and receive messages
socket$.subscribe({
  next: message => console.log('Received:', message),
  error: err => console.error('WebSocket error:', err),
  complete: () => console.log('Connection closed')
});

// Send message
socket$.next({ type: 'ping', data: 'Hello Server' });

// Close connection
socket$.complete();
```

## 🛠️ 5. Custom Observable Creation

### new Observable() - Manual Construction

For complete control over Observable behavior.

```typescript
import { Observable } from 'rxjs';

// Basic custom Observable
const custom$ = new Observable(observer => {
  console.log('Observable started');
  
  // Emit values
  observer.next('First');
  observer.next('Second');
  
  // Async emission
  setTimeout(() => {
    observer.next('Delayed');
    observer.complete();
  }, 1000);
  
  // Cleanup function (teardown logic)
  return () => {
    console.log('Cleanup executed');
  };
});

// Advanced: Random number generator
const randomNumbers$ = new Observable<number>(observer => {
  const interval = setInterval(() => {
    const randomNum = Math.random();
    
    if (randomNum > 0.9) {
      observer.error(new Error('Random number too high!'));
      return;
    }
    
    if (randomNum < 0.1) {
      observer.complete();
      return;
    }
    
    observer.next(randomNum);
  }, 1000);
  
  // Cleanup interval on unsubscription
  return () => {
    clearInterval(interval);
    console.log('Random generator stopped');
  };
});

// Usage
custom$.subscribe({
  next: value => console.log('Custom value:', value),
  error: err => console.error('Custom error:', err),
  complete: () => console.log('Custom complete')
});
```

### defer() - Lazy Creation

Creates an Observable only when subscribed to, allowing for dynamic creation.

```typescript
import { defer, of, timer } from 'rxjs';
import { map } from 'rxjs/operators';

// Lazy creation - Observable created on each subscription
const lazyTime$ = defer(() => {
  const now = new Date();
  console.log('Creating Observable at:', now);
  return of(now);
});

// Each subscription gets a new timestamp
lazyTime$.subscribe(time => console.log('Subscription 1:', time));

setTimeout(() => {
  lazyTime$.subscribe(time => console.log('Subscription 2:', time));
}, 2000);

// Dynamic Observable creation based on conditions
const dynamicObservable$ = defer(() => {
  const random = Math.random();
  
  if (random > 0.5) {
    return of('High value');
  } else {
    return timer(1000).pipe(map(() => 'Low value after delay'));
  }
});

dynamicObservable$.subscribe(value => console.log('Dynamic:', value));
```

### generate() - Stateful Generation

Creates an Observable by generating values using a loop-like structure.

```typescript
import { generate } from 'rxjs';

// Generate Fibonacci sequence
const fibonacci$ = generate(
  [0, 1],                           // Initial state [current, next]
  ([current, next]) => current < 100, // Condition to continue
  ([current, next]) => [next, current + next], // State update
  ([current]) => current            // Value to emit
);

// Marble: 0-1-1-2-3-5-8-13-21-34-55-89|

// Generate even numbers
const evenNumbers$ = generate(
  0,                    // Initial value
  x => x < 20,         // Continue while less than 20
  x => x + 2,          // Increment by 2
  x => x               // Emit the value
);

// Marble: 0-2-4-6-8-10-12-14-16-18|

// Usage
fibonacci$.subscribe(value => console.log('Fibonacci:', value));
evenNumbers$.subscribe(value => console.log('Even:', value));
```

## 📝 Practical Examples

### 1. Real-time Data Simulation

```typescript
import { Observable, interval } from 'rxjs';
import { map, share } from 'rxjs/operators';

// Simulate stock price updates
const stockPrice$ = new Observable<{ symbol: string; price: number; timestamp: Date }>(observer => {
  let basePrice = 100;
  
  const intervalId = setInterval(() => {
    // Simulate price fluctuation
    const change = (Math.random() - 0.5) * 2; // -1 to 1
    basePrice += change;
    
    observer.next({
      symbol: 'AAPL',
      price: Math.round(basePrice * 100) / 100,
      timestamp: new Date()
    });
  }, 1000);
  
  return () => clearInterval(intervalId);
}).pipe(
  share() // Share among multiple subscribers
);

// Multiple subscribers get the same data
stockPrice$.subscribe(stock => console.log('Subscriber 1:', stock));
stockPrice$.subscribe(stock => console.log('Subscriber 2:', stock));
```

### 2. Geolocation Tracking

```typescript
import { Observable } from 'rxjs';

// Geolocation Observable
const geolocation$ = new Observable<GeolocationPosition>(observer => {
  if (!navigator.geolocation) {
    observer.error(new Error('Geolocation not supported'));
    return;
  }
  
  const watchId = navigator.geolocation.watchPosition(
    position => observer.next(position),
    error => observer.error(error),
    { enableHighAccuracy: true, maximumAge: 10000 }
  );
  
  // Cleanup
  return () => {
    navigator.geolocation.clearWatch(watchId);
  };
});

// Usage
geolocation$.subscribe({
  next: position => {
    console.log('Location:', position.coords.latitude, position.coords.longitude);
  },
  error: err => console.error('Geolocation error:', err)
});
```

### 3. File Reader Observable

```typescript
import { Observable } from 'rxjs';

function readFileAsText(file: File): Observable<string> {
  return new Observable<string>(observer => {
    const reader = new FileReader();
    
    reader.onload = () => {
      observer.next(reader.result as string);
      observer.complete();
    };
    
    reader.onerror = () => {
      observer.error(reader.error);
    };
    
    reader.readAsText(file);
    
    // Cleanup (can't actually cancel FileReader)
    return () => {
      // FileReader doesn't have abort in all browsers
      if (reader.abort) {
        reader.abort();
      }
    };
  });
}

// Usage with file input
const fileInput = document.getElementById('fileInput') as HTMLInputElement;
fromEvent(fileInput, 'change').pipe(
  map((event: Event) => (event.target as HTMLInputElement).files![0]),
  switchMap(file => readFileAsText(file))
).subscribe(content => {
  console.log('File content:', content);
});
```

## ⚡ Performance Considerations

### 1. Memory Management

```typescript
// ❌ Memory leak - no cleanup
const badObservable$ = new Observable(observer => {
  setInterval(() => {
    observer.next(Date.now());
  }, 1000);
  // Missing cleanup!
});

// ✅ Proper cleanup
const goodObservable$ = new Observable(observer => {
  const intervalId = setInterval(() => {
    observer.next(Date.now());
  }, 1000);
  
  return () => clearInterval(intervalId); // Cleanup
});
```

### 2. Efficient Creation

```typescript
// ❌ Inefficient - creates new Observable each time
function getNumbers() {
  return from([1, 2, 3, 4, 5]);
}

// ✅ Efficient - reuse Observable
const numbers$ = from([1, 2, 3, 4, 5]);

function getNumbers() {
  return numbers$;
}
```

### 3. Lazy vs Eager Creation

```typescript
// Eager - created immediately
const eagerTime$ = of(new Date());

// Lazy - created when subscribed
const lazyTime$ = defer(() => of(new Date()));

// Different timestamps for multiple subscriptions
console.log('--- Eager ---');
eagerTime$.subscribe(time => console.log('1st:', time));
setTimeout(() => {
  eagerTime$.subscribe(time => console.log('2nd:', time)); // Same time
}, 1000);

console.log('--- Lazy ---');
lazyTime$.subscribe(time => console.log('1st:', time));
setTimeout(() => {
  lazyTime$.subscribe(time => console.log('2nd:', time)); // Different time
}, 1000);
```

## 🧪 Testing Observable Creation

### Unit Testing Custom Observables

```typescript
import { TestScheduler } from 'rxjs/testing';

describe('Custom Observable Creation', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should create observable from values', () => {
    testScheduler.run(({ expectObservable }) => {
      const source$ = of(1, 2, 3);
      const expected = '(123|)';
      
      expectObservable(source$).toBe(expected);
    });
  });

  it('should create observable with timing', () => {
    testScheduler.run(({ expectObservable }) => {
      const source$ = timer(3000).pipe(map(() => 'delayed'));
      const expected = '---a|';
      
      expectObservable(source$).toBe(expected, { a: 'delayed' });
    });
  });
});
```

## 🎯 Best Practices

### ✅ **Do**

1. **Use appropriate creation method** for your data source
2. **Always provide cleanup logic** in custom Observables
3. **Use `defer()` for dynamic creation** when Observable behavior should change per subscription
4. **Handle errors gracefully** in custom Observables
5. **Use `share()` or `shareReplay()`** for expensive operations with multiple subscribers

### ❌ **Don't**

1. **Don't forget cleanup** in custom Observables
2. **Don't create Observables unnecessarily** - reuse when possible
3. **Don't ignore error cases** in custom creation
4. **Don't create infinite Observables** without proper controls
5. **Don't use synchronous creation** for async data sources

## 🎯 Quick Assessment

**Creation Questions:**

1. When would you use `defer()` instead of `of()`?
2. What's the difference between `from()` and `of()` when passing an array?
3. How do you ensure proper cleanup in custom Observables?
4. When should you use `fromEventPattern()` instead of `fromEvent()`?

**Answers:**

1. **`defer()`** when you need lazy creation or dynamic Observable behavior per subscription
2. **`from([1,2,3])`** emits 1,2,3 separately. **`of([1,2,3])`** emits the array as one value
3. **Return a cleanup function** from the Observable constructor
4. **`fromEventPattern()`** for custom event APIs that don't use standard addEventListener/removeEventListener

## 🌟 Key Takeaways

- **Multiple creation methods** serve different purposes and data sources
- **Custom Observables** give complete control but require proper cleanup
- **`defer()`** enables lazy and dynamic Observable creation
- **Proper cleanup** prevents memory leaks in custom Observables
- **Choose the right method** based on your data source and requirements
- **Performance matters** - reuse Observables when possible

## 🚀 Next Steps

Now that you can create Observables from any data source, you're ready to learn about the **Observable Lifecycle & Internal Workings** to understand exactly what happens from creation to completion.

**Next Lesson**: [Observable Lifecycle & Internal Workings](./02-observable-lifecycle.md) 🟡

---

🎉 **Fantastic!** You now know how to create Observables from any data source. This foundational skill will serve you well as you build more complex reactive applications!
