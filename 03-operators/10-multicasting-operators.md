# Multicasting Operators

## Overview

Multicasting operators enable sharing Observable execution among multiple subscribers. By default, Observables are unicast (cold) - each subscription creates a new execution. Multicasting operators convert cold Observables to hot ones, allowing shared execution and resource optimization.

## Key Concepts

### Unicast vs Multicast
- **Unicast (Cold)**: Each subscriber gets its own execution
- **Multicast (Hot)**: All subscribers share the same execution

### ConnectableObservable
A special type of Observable that doesn't start emitting until `connect()` is called, allowing multiple subscriptions before execution begins.

## Core Multicasting Operators

### share()

Shares the source Observable among multiple subscribers using a Subject.

**Marble Diagram:**
```
source:     --1--2--3--4--5--|
share():    --1--2--3--4--5--|
sub1:       --1--2--3--4--5--|
sub2:         --2--3--4--5--|  (subscribes later)
```

**Angular Example:**
```typescript
import { Component, OnInit } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { share, tap } from 'rxjs/operators';

@Component({
  selector: 'app-shared-data',
  template: `
    <div>
      <h3>User Data (Shared Request)</h3>
      <div *ngIf="userData">{{ userData | json }}</div>
      <div *ngIf="userProfile">{{ userProfile | json }}</div>
    </div>
  `
})
export class SharedDataComponent implements OnInit {
  userData: any;
  userProfile: any;

  constructor(private http: HttpClient) {}

  ngOnInit() {
    // Single HTTP request shared among multiple subscribers
    const sharedUserData$ = this.http.get('/api/user').pipe(
      tap(() => console.log('HTTP request executed')), // Only logs once
      share()
    );

    // Multiple subscriptions share the same execution
    sharedUserData$.subscribe(data => this.userData = data);
    sharedUserData$.subscribe(data => this.userProfile = data);
  }
}
```

### shareReplay()

Like `share()` but replays the last N emitted values to new subscribers.

**Marble Diagram:**
```
source:        --1--2--3--4--5--|
shareReplay(2): --1--2--3--4--5--|
sub1:          --1--2--3--4--5--|
sub2:             ^^34--5--|  (gets last 2 values immediately)
```

**Angular Example:**
```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { shareReplay, map } from 'rxjs/operators';
import { Observable } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class ConfigService {
  private config$ = this.http.get<any>('/api/config').pipe(
    shareReplay(1) // Cache the config and replay to new subscribers
  );

  constructor(private http: HttpClient) {}

  getConfig(): Observable<any> {
    return this.config$;
  }

  getFeatureFlag(flag: string): Observable<boolean> {
    return this.config$.pipe(
      map(config => config.features[flag] || false)
    );
  }
}

@Component({
  selector: 'app-feature',
  template: `
    <div *ngIf="featureEnabled$ | async">
      <h3>Premium Feature</h3>
      <p>This feature is enabled!</p>
    </div>
  `
})
export class FeatureComponent {
  featureEnabled$ = this.configService.getFeatureFlag('premiumFeature');

  constructor(private configService: ConfigService) {}
}
```

### multicast()

Multicasts using a specified Subject.

**Basic Example:**
```typescript
import { Subject, interval } from 'rxjs';
import { multicast, take } from 'rxjs/operators';

const source$ = interval(1000).pipe(take(5));
const subject = new Subject();

const multicasted$ = source$.pipe(
  multicast(subject)
);

// Subscribe before connecting
multicasted$.subscribe(x => console.log('Subscriber 1:', x));
multicasted$.subscribe(x => console.log('Subscriber 2:', x));

// Start the execution
multicasted$.connect();
```

### publish()

Shorthand for `multicast(() => new Subject())`.

**Angular Example:**
```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { interval, Subscription } from 'rxjs';
import { publish, take, tap } from 'rxjs/operators';

@Component({
  selector: 'app-published-timer',
  template: `
    <div>
      <h3>Shared Timer</h3>
      <p>Timer 1: {{ timer1 }}</p>
      <p>Timer 2: {{ timer2 }}</p>
      <button (click)="startTimer()">Start</button>
      <button (click)="stopTimer()">Stop</button>
    </div>
  `
})
export class PublishedTimerComponent implements OnInit, OnDestroy {
  timer1 = 0;
  timer2 = 0;
  private connection?: Subscription;
  private published$ = interval(1000).pipe(
    take(10),
    tap(x => console.log('Timer tick:', x)),
    publish()
  );

  ngOnInit() {
    // Subscribe to the published observable
    this.published$.subscribe(x => this.timer1 = x);
    this.published$.subscribe(x => this.timer2 = x * 2);
  }

  startTimer() {
    if (!this.connection) {
      this.connection = this.published$.connect();
    }
  }

  stopTimer() {
    if (this.connection) {
      this.connection.unsubscribe();
      this.connection = undefined;
    }
  }

  ngOnDestroy() {
    this.stopTimer();
  }
}
```

### publishReplay()

Shorthand for `multicast(() => new ReplaySubject(bufferSize))`.

**Angular Example:**
```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { publishReplay, refCount } from 'rxjs/operators';
import { Observable } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class CachedDataService {
  private cache$ = this.http.get<any[]>('/api/expensive-data').pipe(
    publishReplay(1), // Cache the last result
    refCount() // Automatically connect/disconnect
  );

  constructor(private http: HttpClient) {}

  getData(): Observable<any[]> {
    return this.cache$;
  }
}
```

### refCount()

Automatically connects when first subscriber subscribes and disconnects when last subscriber unsubscribes.

**Example:**
```typescript
import { interval, ConnectableObservable } from 'rxjs';
import { publish, refCount, take } from 'rxjs/operators';

const source$ = interval(1000).pipe(
  take(5),
  publish(),
  refCount() // Auto-connect/disconnect
) as Observable<number>;

// First subscription triggers connect()
const sub1 = source$.subscribe(x => console.log('Sub1:', x));

setTimeout(() => {
  // Second subscription joins the existing execution
  const sub2 = source$.subscribe(x => console.log('Sub2:', x));
  
  setTimeout(() => {
    sub1.unsubscribe(); // Still connected (sub2 active)
    
    setTimeout(() => {
      sub2.unsubscribe(); // Disconnects (no active subscriptions)
    }, 2000);
  }, 2000);
}, 1000);
```

## Advanced Patterns

### Conditional Sharing

```typescript
import { Component } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { share, shareReplay, switchMap } from 'rxjs/operators';
import { BehaviorSubject, Observable } from 'rxjs';

@Component({
  selector: 'app-conditional-sharing',
  template: `
    <div>
      <button (click)="toggleSharing()">
        {{ isSharing ? 'Disable' : 'Enable' }} Sharing
      </button>
      <div *ngFor="let result of results">{{ result | json }}</div>
    </div>
  `
})
export class ConditionalSharingComponent {
  private isSharing$ = new BehaviorSubject(false);
  results: any[] = [];

  constructor(private http: HttpClient) {}

  getData(): Observable<any> {
    return this.isSharing$.pipe(
      switchMap(shouldShare => {
        const request$ = this.http.get('/api/data');
        return shouldShare ? request$.pipe(shareReplay(1)) : request$;
      })
    );
  }

  loadData() {
    this.getData().subscribe(data => {
      this.results.push(data);
    });
  }

  toggleSharing() {
    this.isSharing$.next(!this.isSharing$.value);
  }

  get isSharing() {
    return this.isSharing$.value;
  }
}
```

### Resource Management with Multicasting

```typescript
import { Injectable, OnDestroy } from '@angular/core';
import { WebSocketSubject } from 'rxjs/webSocket';
import { share, retry, finalize } from 'rxjs/operators';
import { Observable, Subject } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class WebSocketService implements OnDestroy {
  private socket$?: WebSocketSubject<any>;
  private messages$?: Observable<any>;
  private destroy$ = new Subject<void>();

  connect(url: string): Observable<any> {
    if (!this.messages$) {
      this.socket$ = new WebSocketSubject(url);
      
      this.messages$ = this.socket$.pipe(
        retry({ delay: 5000 }), // Retry connection after 5s
        share(), // Share connection among multiple subscribers
        finalize(() => {
          console.log('WebSocket connection closed');
          this.cleanup();
        })
      );
    }
    
    return this.messages$;
  }

  send(message: any) {
    if (this.socket$) {
      this.socket$.next(message);
    }
  }

  private cleanup() {
    this.socket$ = undefined;
    this.messages$ = undefined;
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
    if (this.socket$) {
      this.socket$.complete();
    }
  }
}
```

## Performance Considerations

### Memory Management

```typescript
import { Component, OnDestroy } from '@angular/core';
import { interval, Subscription } from 'rxjs';
import { shareReplay, take, finalize } from 'rxjs/operators';

@Component({
  selector: 'app-memory-efficient',
  template: `<div>Memory Efficient Sharing</div>`
})
export class MemoryEfficientComponent implements OnDestroy {
  private subscriptions = new Subscription();

  ngOnInit() {
    // Good: Limited replay buffer
    const efficientShare$ = interval(1000).pipe(
      take(100),
      shareReplay(1), // Only replay last value
      finalize(() => console.log('Stream completed'))
    );

    // Avoid: Unlimited replay buffer
    // const inefficientShare$ = interval(1000).pipe(
    //   shareReplay() // Replays ALL values - memory leak!
    // );

    this.subscriptions.add(
      efficientShare$.subscribe(x => console.log('Value:', x))
    );
  }

  ngOnDestroy() {
    this.subscriptions.unsubscribe();
  }
}
```

### Share vs ShareReplay Decision Matrix

| Use Case | Operator | Reason |
|----------|----------|---------|
| HTTP Requests | `shareReplay(1)` | Cache response for late subscribers |
| Real-time Data | `share()` | Don't need historical values |
| Configuration | `shareReplay(1)` | Cache config for app lifetime |
| User Events | `share()` | Only current subscribers need events |
| WebSocket | `share()` | Real-time, no replay needed |

## Best Practices

### 1. Choose the Right Multicasting Operator

```typescript
// For HTTP requests - cache the result
const httpRequest$ = this.http.get('/api/data').pipe(
  shareReplay(1)
);

// For real-time events - share current emissions only
const realTimeData$ = this.websocket.connect().pipe(
  share()
);

// For expensive computations - share and cache
const expensiveComputation$ = this.computeExpensiveData().pipe(
  shareReplay(1)
);
```

### 2. Manage Subscriptions Properly

```typescript
@Component({
  selector: 'app-proper-sharing',
  template: `<div>Proper Sharing Component</div>`
})
export class ProperSharingComponent implements OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit() {
    const shared$ = this.expensiveOperation().pipe(
      shareReplay(1),
      takeUntil(this.destroy$)
    );

    // Multiple subscriptions to shared stream
    shared$.subscribe(data => this.handleData(data));
    shared$.subscribe(data => this.logData(data));
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### 3. Error Handling in Multicasted Streams

```typescript
const resilientShared$ = this.http.get('/api/data').pipe(
  retry(3),
  catchError(error => {
    console.error('Request failed:', error);
    return of(null); // Provide fallback
  }),
  shareReplay(1)
);
```

## Common Pitfalls

### 1. Memory Leaks with shareReplay()

```typescript
// ❌ Bad: Unlimited replay buffer
const memoryLeak$ = interval(1000).pipe(
  shareReplay() // Keeps ALL emitted values in memory!
);

// ✅ Good: Limited replay buffer
const memoryEfficient$ = interval(1000).pipe(
  shareReplay(1) // Only keeps last value
);
```

### 2. Unexpected Behavior with Late Subscribers

```typescript
// ❌ Issue: Late subscriber misses current value
const shared$ = interval(1000).pipe(share());

shared$.subscribe(x => console.log('Early:', x));

setTimeout(() => {
  shared$.subscribe(x => console.log('Late:', x)); // Misses initial values
}, 3000);

// ✅ Solution: Use shareReplay for late subscribers
const replayShared$ = interval(1000).pipe(shareReplay(1));
```

### 3. Not Handling ConnectableObservable Properly

```typescript
// ❌ Bad: Forgot to connect
const published$ = interval(1000).pipe(publish());
published$.subscribe(x => console.log(x)); // Nothing happens!

// ✅ Good: Remember to connect
const connected$ = interval(1000).pipe(publish());
connected$.subscribe(x => console.log(x));
connected$.connect(); // Now it works
```

## Exercises

### Exercise 1: Implement a Shared Configuration Service

Create a service that loads configuration once and shares it among all components:

```typescript
@Injectable({
  providedIn: 'root'
})
export class SharedConfigService {
  // TODO: Implement shared configuration loading
  // Requirements:
  // 1. Load config from HTTP endpoint
  // 2. Share among all subscribers
  // 3. Cache the result
  // 4. Handle errors gracefully
}
```

### Exercise 2: Real-time Data Sharing

Implement a component that shares real-time stock prices among multiple views:

```typescript
@Component({
  selector: 'app-stock-dashboard',
  template: `
    <!-- TODO: Display stock prices from shared stream -->
    <!-- Multiple components should show same data -->
    <!-- No duplicate WebSocket connections -->
  `
})
export class StockDashboardComponent {
  // TODO: Implement shared real-time data
}
```

### Exercise 3: Resource-Efficient Multicasting

Create a pattern that automatically manages connections based on subscriber count:

```typescript
class SmartMulticastService {
  // TODO: Implement smart multicasting
  // Requirements:
  // 1. Connect when first subscriber joins
  // 2. Disconnect when last subscriber leaves
  // 3. Handle reconnection logic
  // 4. Provide connection status
}
```

## Summary

Multicasting operators are essential for:
- **Resource Optimization**: Share expensive operations among multiple subscribers
- **Performance**: Avoid duplicate HTTP requests, WebSocket connections, etc.
- **Consistency**: Ensure all subscribers see the same data
- **Memory Management**: Control replay buffer size to prevent memory leaks

Key operators:
- `share()`: Basic sharing without replay
- `shareReplay(n)`: Sharing with replay buffer
- `publish()` + `connect()`: Manual connection control
- `refCount()`: Automatic connection management

Choose the right operator based on your use case: use `shareReplay()` for cacheable data like HTTP responses, and `share()` for real-time streams that don't need replay.
