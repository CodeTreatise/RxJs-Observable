# Lesson 7: RxJS Schedulers & Execution Context

## Learning Objectives
By the end of this lesson, you will:
- Master RxJS Schedulers and their impact on execution timing
- Understand different scheduler types and their use cases
- Control asynchronous execution flow in complex Observable chains
- Optimize performance using appropriate schedulers
- Implement custom schedulers for specialized scenarios

## Prerequisites
- Deep understanding of Observables and operators
- Knowledge of JavaScript event loop and asynchronous execution
- Familiarity with Subjects and multicasting
- Understanding of Angular's change detection cycle

## 1. What are RxJS Schedulers?

### Scheduler Fundamentals
A Scheduler controls when a subscription starts and when notifications are delivered. It's a combination of three components:
1. **A data structure** - prioritized queue of tasks
2. **An execution context** - where and when tasks are executed  
3. **A virtual clock** - provides notion of time

```typescript
import { Observable, asyncScheduler, queueScheduler } from 'rxjs';
import { observeOn, subscribeOn } from 'rxjs/operators';

// Understanding scheduler impact
const observable = new Observable(subscriber => {
  console.log('Observable execution started');
  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);
  subscriber.complete();
  console.log('Observable execution completed');
});

console.log('Before subscription');

// Without scheduler (synchronous)
observable.subscribe(value => console.log('Sync value:', value));

console.log('After synchronous subscription');

// With async scheduler
observable.pipe(
  observeOn(asyncScheduler)
).subscribe(value => console.log('Async value:', value));

console.log('After async subscription setup');

// Output order demonstrates execution timing differences
```

### Scheduler Interface
```typescript
interface SchedulerLike {
  now(): number;                                    // Current time
  schedule<T>(                                      // Schedule a task
    work: (this: SchedulerAction<T>, state?: T) => void,
    delay?: number,
    state?: T
  ): Subscription;
}

interface SchedulerAction<T> extends Subscription {
  schedule(state?: T, delay?: number): Subscription; // Reschedule
}
```

## 2. Built-in Scheduler Types

### 1. null (No Scheduler - Synchronous)
```typescript
import { of } from 'rxjs';

// Default behavior - synchronous execution
const synchronous$ = of(1, 2, 3);

console.log('Before');
synchronous$.subscribe(value => console.log('Value:', value));
console.log('After');

// Output:
// Before
// Value: 1
// Value: 2  
// Value: 3
// After
```

### 2. queueScheduler (Synchronous Queue)
```typescript
import { queueScheduler, of } from 'rxjs';
import { observeOn } from 'rxjs/operators';

// Queue scheduler - synchronous but queued
console.log('Before');

of(1, 2, 3).pipe(
  observeOn(queueScheduler)
).subscribe(value => console.log('Queued:', value));

console.log('After');

// Output:
// Before
// Queued: 1
// Queued: 2
// Queued: 3
// After

// Practical example - preventing stack overflow
function recursiveObservable(n: number): Observable<number> {
  if (n <= 0) {
    return of(n);
  }
  
  return of(n).pipe(
    observeOn(queueScheduler), // Prevents stack overflow
    mergeMap(() => recursiveObservable(n - 1))
  );
}

recursiveObservable(10000).subscribe(); // Won't overflow
```

### 3. asapScheduler (Microtask Queue)
```typescript
import { asapScheduler, of } from 'rxjs';
import { observeOn } from 'rxjs/operators';

// ASAP scheduler uses microtask queue (Promise.resolve().then())
console.log('Start');

setTimeout(() => console.log('Timeout'), 0);

of(1, 2, 3).pipe(
  observeOn(asapScheduler)
).subscribe(value => console.log('ASAP:', value));

Promise.resolve().then(() => console.log('Promise'));

console.log('End');

// Output:
// Start
// End
// ASAP: 1
// ASAP: 2
// ASAP: 3
// Promise
// Timeout
```

### 4. asyncScheduler (Macrotask Queue)
```typescript
import { asyncScheduler, of, interval } from 'rxjs';
import { observeOn, take } from 'rxjs/operators';

// Async scheduler uses setTimeout (macrotask queue)
console.log('Start');

of(1, 2, 3).pipe(
  observeOn(asyncScheduler)
).subscribe(value => console.log('Async:', value));

console.log('Synchronous end');

// Delayed execution with asyncScheduler
interval(1000).pipe(
  take(3),
  observeOn(asyncScheduler)
).subscribe(value => console.log('Interval:', value));

// Custom timing with asyncScheduler
asyncScheduler.schedule(() => {
  console.log('Custom scheduled task');
}, 2000);
```

### 5. animationFrameScheduler (Animation Frame)
```typescript
import { animationFrameScheduler, interval } from 'rxjs';
import { observeOn, map } from 'rxjs/operators';

// Animation frame scheduler for smooth animations
const animationData$ = interval(0).pipe(
  observeOn(animationFrameScheduler),
  map(frame => ({
    frame,
    timestamp: performance.now(),
    progress: (frame % 60) / 60 // 60 FPS cycle
  }))
);

// Usage in Angular component for smooth animations
@Component({
  selector: 'app-animation',
  template: `
    <div [style.transform]="'translateX(' + position + 'px)'">
      Animated Element
    </div>
  `
})
export class AnimationComponent implements OnInit, OnDestroy {
  position = 0;
  private destroy$ = new Subject<void>();
  
  ngOnInit() {
    animationData$.pipe(
      takeUntil(this.destroy$)
    ).subscribe(({ progress }) => {
      this.position = progress * 200; // Animate 0-200px
    });
  }
  
  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## 3. TestScheduler for Testing

### Virtual Time Testing
```typescript
import { TestScheduler } from 'rxjs/testing';
import { delay, map } from 'rxjs/operators';

describe('Scheduler Testing', () => {
  let testScheduler: TestScheduler;
  
  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });
  
  it('should test timing-dependent observables', () => {
    testScheduler.run(({ cold, hot, expectObservable, expectSubscriptions }) => {
      const source = cold('  a-b-c|');
      const subs =   '       ^----!';
      const expected = '     a-b-c|';
      
      const result = source.pipe(
        delay(0, testScheduler) // Use test scheduler for delays
      );
      
      expectObservable(result).toBe(expected);
      expectSubscriptions(source.subscriptions).toBe(subs);
    });
  });
  
  it('should test complex timing scenarios', () => {
    testScheduler.run(({ cold, expectObservable }) => {
      const values = { a: 1, b: 2, c: 3, x: 6 };
      
      const source1 = cold('a---b---c|', values);
      const source2 = cold('--a-b-c-|', values);
      
      const result = combineLatest([source1, source2]).pipe(
        map(([x, y]) => x + y)
      );
      
      const expected = '--x-y-z-w|';
      const expectedValues = { x: 2, y: 3, z: 4, w: 5 };
      
      expectObservable(result).toBe(expected, expectedValues);
    });
  });
});
```

### Testing with Virtual Time
```typescript
// Testing time-dependent operations
class TimerService {
  createTimer(interval: number): Observable<number> {
    return timer(0, interval);
  }
  
  createDelayedValue<T>(value: T, delay: number): Observable<T> {
    return of(value).pipe(delay(delay));
  }
}

describe('TimerService', () => {
  let service: TimerService;
  let testScheduler: TestScheduler;
  
  beforeEach(() => {
    service = new TimerService();
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });
  
  it('should create timer with correct intervals', () => {
    testScheduler.run(({ expectObservable }) => {
      const timer$ = service.createTimer(30).pipe(take(4));
      const expected = '0 29ms 1 29ms 2 29ms (3|)';
      
      expectObservable(timer$).toBe(expected);
    });
  });
  
  it('should delay values correctly', () => {
    testScheduler.run(({ expectObservable }) => {
      const delayed$ = service.createDelayedValue('test', 100);
      const expected = '100ms (test|)';
      
      expectObservable(delayed$).toBe(expected);
    });
  });
});
```

## 4. Controlling Execution with Schedulers

### subscribeOn vs observeOn
```typescript
import { of, asyncScheduler, queueScheduler } from 'rxjs';
import { subscribeOn, observeOn, map } from 'rxjs/operators';

// subscribeOn - controls when subscription logic executes
// observeOn - controls when emissions are delivered

const source$ = of(1, 2, 3);

console.log('=== subscribeOn Example ===');
source$.pipe(
  subscribeOn(asyncScheduler), // Subscription happens asynchronously
  map(x => {
    console.log('Map operator:', x);
    return x * 2;
  })
).subscribe(value => console.log('Final value:', value));

console.log('After subscribeOn setup');

// vs

console.log('=== observeOn Example ===');
source$.pipe(
  map(x => {
    console.log('Map operator (sync):', x);
    return x * 2;
  }),
  observeOn(asyncScheduler) // Emissions delivered asynchronously
).subscribe(value => console.log('Final value (async):', value));

console.log('After observeOn setup');
```

### Complex Scheduling Scenarios
```typescript
// Real-world example: API calls with proper scheduling
@Injectable({ providedIn: 'root' })
export class ApiService {
  constructor(private http: HttpClient) {}
  
  // CPU-intensive data processing with proper scheduling
  processLargeDataset(data: any[]): Observable<ProcessedData[]> {
    return of(data).pipe(
      // Use async scheduler to prevent blocking UI
      observeOn(asyncScheduler),
      map(items => {
        console.log('Processing started...');
        return items.map(item => this.expensiveProcessing(item));
      }),
      // Switch back to synchronous for final result
      observeOn(queueScheduler)
    );
  }
  
  // Batched API calls with controlled timing
  batchApiCalls<T>(
    requests: Observable<T>[],
    concurrency: number = 3,
    delayBetweenBatches: number = 1000
  ): Observable<T[]> {
    
    return from(requests).pipe(
      // Group into batches
      bufferCount(concurrency),
      // Add delay between batches using asyncScheduler
      concatMap((batch, index) => {
        const batchResult$ = forkJoin(batch);
        
        return index === 0 
          ? batchResult$
          : timer(delayBetweenBatches, asyncScheduler).pipe(
              mergeMap(() => batchResult$)
            );
      }),
      // Flatten all results
      scan((acc, batch) => [...acc, ...batch], [] as T[])
    );
  }
  
  private expensiveProcessing(item: any): ProcessedData {
    // Simulate CPU-intensive work
    const start = performance.now();
    let result = item;
    
    for (let i = 0; i < 1000000; i++) {
      result = { ...result, processed: true };
    }
    
    console.log(`Processing took: ${performance.now() - start}ms`);
    return result;
  }
}
```

## 5. Angular-Specific Scheduler Considerations

### Zone.js and Schedulers
```typescript
import { NgZone } from '@angular/core';

@Component({
  selector: 'app-zone-aware',
  template: `
    <div>Counter: {{ counter }}</div>
    <div>Zone Counter: {{ zoneCounter }}</div>
  `
})
export class ZoneAwareComponent implements OnInit {
  counter = 0;
  zoneCounter = 0;
  
  constructor(private ngZone: NgZone) {}
  
  ngOnInit() {
    // This runs outside Angular zone - no change detection
    this.ngZone.runOutsideAngular(() => {
      interval(1000).subscribe(() => {
        this.counter++; // Won't trigger change detection
      });
    });
    
    // This runs inside Angular zone - triggers change detection
    interval(1000).pipe(
      observeOn(asyncScheduler)
    ).subscribe(() => {
      this.zoneCounter++; // Will trigger change detection
    });
    
    // Manual zone control
    interval(2000).pipe(
      observeOn(asyncScheduler)
    ).subscribe(() => {
      this.ngZone.run(() => {
        this.counter = this.zoneCounter; // Sync counters
      });
    });
  }
}
```

### Custom Zone-aware Scheduler
```typescript
// Custom scheduler that always runs in Angular zone
class AngularScheduler implements SchedulerLike {
  constructor(
    private ngZone: NgZone,
    private baseScheduler: SchedulerLike = asyncScheduler
  ) {}
  
  now(): number {
    return this.baseScheduler.now();
  }
  
  schedule<T>(
    work: (this: SchedulerAction<T>, state?: T) => void,
    delay: number = 0,
    state?: T
  ): Subscription {
    
    return this.baseScheduler.schedule(function(this: SchedulerAction<T>, innerState?: T) {
      // Ensure work runs inside Angular zone
      return this.ngZone.run(() => {
        return work.call(this, innerState);
      });
    }, delay, state);
  }
}

// Usage
@Injectable({ providedIn: 'root' })
export class SchedulerService {
  private angularScheduler: AngularScheduler;
  
  constructor(private ngZone: NgZone) {
    this.angularScheduler = new AngularScheduler(ngZone);
  }
  
  createZoneAwareObservable<T>(source: Observable<T>): Observable<T> {
    return source.pipe(
      observeOn(this.angularScheduler)
    );
  }
}
```

## 6. Performance Optimization with Schedulers

### Preventing UI Blocking
```typescript
// Service for CPU-intensive operations
@Injectable({ providedIn: 'root' })
export class HeavyComputationService {
  
  // Break heavy computation into chunks
  processLargeArray<T, R>(
    items: T[],
    processor: (item: T) => R,
    chunkSize: number = 100
  ): Observable<R[]> {
    
    return from(this.chunkArray(items, chunkSize)).pipe(
      // Process each chunk asynchronously
      concatMap((chunk, index) => {
        return of(chunk).pipe(
          // Use async scheduler to yield control
          observeOn(asyncScheduler),
          map(chunkItems => chunkItems.map(processor)),
          tap(() => console.log(`Processed chunk ${index + 1}`))
        );
      }),
      // Accumulate results
      scan((acc, chunkResults) => [...acc, ...chunkResults], [] as R[]),
      // Return final result
      last()
    );
  }
  
  // Debounced heavy computation
  createDebouncedProcessor<T, R>(
    processor: (item: T) => R,
    debounceMs: number = 300
  ): (input$: Observable<T>) => Observable<R> {
    
    return (input$: Observable<T>) => {
      return input$.pipe(
        debounceTime(debounceMs, asyncScheduler),
        observeOn(asyncScheduler), // Process asynchronously
        map(processor)
      );
    };
  }
  
  private chunkArray<T>(array: T[], chunkSize: number): T[][] {
    const chunks: T[][] = [];
    for (let i = 0; i < array.length; i += chunkSize) {
      chunks.push(array.slice(i, i + chunkSize));
    }
    return chunks;
  }
}

// Usage in component
@Component({
  selector: 'app-heavy-computation',
  template: `
    <button (click)="processData()">Process Large Dataset</button>
    <div *ngIf="processing">Processing... {{ progress }}%</div>
    <div *ngIf="results">Results: {{ results.length }} items</div>
  `
})
export class HeavyComputationComponent {
  processing = false;
  progress = 0;
  results: any[] | null = null;
  
  constructor(private heavyService: HeavyComputationService) {}
  
  processData() {
    const largeDataset = Array.from({ length: 10000 }, (_, i) => ({ id: i, value: Math.random() }));
    
    this.processing = true;
    this.progress = 0;
    
    this.heavyService.processLargeArray(
      largeDataset,
      item => ({ ...item, processed: item.value * 2 }),
      100 // Process 100 items at a time
    ).subscribe({
      next: results => {
        this.results = results;
        this.processing = false;
        this.progress = 100;
      },
      error: error => {
        console.error('Processing error:', error);
        this.processing = false;
      }
    });
  }
}
```

### Animation Optimization
```typescript
// Smooth animation service using animationFrameScheduler
@Injectable({ providedIn: 'root' })
export class AnimationService {
  
  createSmoothAnimation(
    duration: number,
    easing: (t: number) => number = t => t
  ): Observable<number> {
    
    const startTime = performance.now();
    
    return interval(0, animationFrameScheduler).pipe(
      map(() => {
        const elapsed = performance.now() - startTime;
        const progress = Math.min(elapsed / duration, 1);
        return easing(progress);
      }),
      takeWhile(progress => progress < 1, true) // Include final value
    );
  }
  
  // Easing functions
  static easeInOut(t: number): number {
    return t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t;
  }
  
  static easeIn(t: number): number {
    return t * t;
  }
  
  static easeOut(t: number): number {
    return t * (2 - t);
  }
  
  // Animate value from start to end
  animateValue(
    start: number,
    end: number,
    duration: number,
    easing?: (t: number) => number
  ): Observable<number> {
    
    return this.createSmoothAnimation(duration, easing).pipe(
      map(progress => start + (end - start) * progress)
    );
  }
  
  // Animate CSS property
  animateProperty(
    element: HTMLElement,
    property: string,
    start: number,
    end: number,
    duration: number,
    unit: string = 'px'
  ): Observable<number> {
    
    return this.animateValue(start, end, duration).pipe(
      tap(value => {
        element.style.setProperty(property, `${value}${unit}`);
      })
    );
  }
}

// Usage in directive
@Directive({
  selector: '[appSmoothMove]'
})
export class SmoothMoveDirective implements OnInit, OnDestroy {
  @Input() targetX = 0;
  @Input() targetY = 0;
  @Input() duration = 1000;
  
  private destroy$ = new Subject<void>();
  
  constructor(
    private el: ElementRef<HTMLElement>,
    private animationService: AnimationService
  ) {}
  
  ngOnInit() {
    const element = this.el.nativeElement;
    const rect = element.getBoundingClientRect();
    const startX = rect.left;
    const startY = rect.top;
    
    // Animate X position
    this.animationService.animateProperty(
      element,
      'left',
      startX,
      this.targetX,
      this.duration
    ).pipe(
      takeUntil(this.destroy$)
    ).subscribe();
    
    // Animate Y position
    this.animationService.animateProperty(
      element,
      'top',
      startY,
      this.targetY,
      this.duration
    ).pipe(
      takeUntil(this.destroy$)
    ).subscribe();
  }
  
  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## 7. Custom Scheduler Implementation

### Building a Custom Scheduler
```typescript
// Custom scheduler with priority queue
interface PriorityTask<T> {
  work: (this: SchedulerAction<T>, state?: T) => void;
  state?: T;
  delay: number;
  priority: number;
}

class PriorityScheduler implements SchedulerLike {
  private taskQueue: PriorityTask<any>[] = [];
  private isRunning = false;
  
  now(): number {
    return Date.now();
  }
  
  schedule<T>(
    work: (this: SchedulerAction<T>, state?: T) => void,
    delay: number = 0,
    state?: T
  ): Subscription {
    
    const priority = delay; // Use delay as priority (lower = higher priority)
    const task: PriorityTask<T> = { work, state, delay, priority };
    
    this.addTask(task);
    this.scheduleExecution();
    
    return new Subscription(() => {
      this.removeTask(task);
    });
  }
  
  private addTask<T>(task: PriorityTask<T>): void {
    // Insert task in priority order
    let insertIndex = this.taskQueue.length;
    
    for (let i = 0; i < this.taskQueue.length; i++) {
      if (task.priority < this.taskQueue[i].priority) {
        insertIndex = i;
        break;
      }
    }
    
    this.taskQueue.splice(insertIndex, 0, task);
  }
  
  private removeTask<T>(task: PriorityTask<T>): void {
    const index = this.taskQueue.indexOf(task);
    if (index >= 0) {
      this.taskQueue.splice(index, 1);
    }
  }
  
  private scheduleExecution(): void {
    if (this.isRunning || this.taskQueue.length === 0) return;
    
    this.isRunning = true;
    
    setTimeout(() => {
      this.executeTasks();
    }, 0);
  }
  
  private executeTasks(): void {
    const now = this.now();
    
    while (this.taskQueue.length > 0) {
      const task = this.taskQueue.shift()!;
      
      try {
        // Create mock scheduler action
        const action = {
          schedule: (state?: any, delay?: number) => {
            return this.schedule(task.work, delay, state);
          }
        } as SchedulerAction<any>;
        
        task.work.call(action, task.state);
      } catch (error) {
        console.error('Error executing scheduled task:', error);
      }
    }
    
    this.isRunning = false;
    
    // Check if more tasks were added during execution
    if (this.taskQueue.length > 0) {
      this.scheduleExecution();
    }
  }
}

// Usage
const priorityScheduler = new PriorityScheduler();

// High priority (delay: 1)
priorityScheduler.schedule(() => console.log('High priority'), 1);

// Low priority (delay: 10)  
priorityScheduler.schedule(() => console.log('Low priority'), 10);

// Medium priority (delay: 5)
priorityScheduler.schedule(() => console.log('Medium priority'), 5);

// Output: High priority, Medium priority, Low priority
```

### Rate-Limited Scheduler
```typescript
// Scheduler that limits execution rate
class RateLimitedScheduler implements SchedulerLike {
  private lastExecution = 0;
  private minInterval: number;
  
  constructor(maxPerSecond: number) {
    this.minInterval = 1000 / maxPerSecond;
  }
  
  now(): number {
    return Date.now();
  }
  
  schedule<T>(
    work: (this: SchedulerAction<T>, state?: T) => void,
    delay: number = 0,
    state?: T
  ): Subscription {
    
    const now = this.now();
    const timeSinceLastExecution = now - this.lastExecution;
    const minDelay = Math.max(0, this.minInterval - timeSinceLastExecution);
    const actualDelay = Math.max(delay, minDelay);
    
    const timeoutId = setTimeout(() => {
      this.lastExecution = this.now();
      
      const action = {
        schedule: (state?: any, delay?: number) => {
          return this.schedule(work, delay, state);
        }
      } as SchedulerAction<T>;
      
      work.call(action, state);
    }, actualDelay);
    
    return new Subscription(() => {
      clearTimeout(timeoutId);
    });
  }
}

// Usage for API rate limiting
@Injectable({ providedIn: 'root' })
export class RateLimitedApiService {
  private rateLimitedScheduler = new RateLimitedScheduler(5); // 5 requests per second
  
  makeApiCall(url: string): Observable<any> {
    return new Observable(subscriber => {
      const subscription = this.rateLimitedScheduler.schedule(() => {
        // Make actual API call
        fetch(url)
          .then(response => response.json())
          .then(data => {
            subscriber.next(data);
            subscriber.complete();
          })
          .catch(error => subscriber.error(error));
      });
      
      return () => subscription.unsubscribe();
    });
  }
}
```

## 8. Debugging and Monitoring Schedulers

### Scheduler Performance Monitor
```typescript
// Monitor scheduler performance
class SchedulerMonitor {
  private executionTimes: number[] = [];
  private taskCounts: number[] = [];
  
  wrapScheduler(scheduler: SchedulerLike, name: string): SchedulerLike {
    return {
      now: () => scheduler.now(),
      
      schedule: <T>(
        work: (this: SchedulerAction<T>, state?: T) => void,
        delay: number = 0,
        state?: T
      ) => {
        const startTime = performance.now();
        let taskCount = 0;
        
        const wrappedWork = function(this: SchedulerAction<T>, innerState?: T) {
          taskCount++;
          const taskStart = performance.now();
          
          try {
            const result = work.call(this, innerState);
            const taskEnd = performance.now();
            
            console.log(`${name} - Task ${taskCount} took: ${taskEnd - taskStart}ms`);
            return result;
          } catch (error) {
            console.error(`${name} - Task ${taskCount} error:`, error);
            throw error;
          }
        };
        
        const subscription = scheduler.schedule(wrappedWork, delay, state);
        
        const endTime = performance.now();
        this.executionTimes.push(endTime - startTime);
        this.taskCounts.push(taskCount);
        
        return subscription;
      }
    };
  }
  
  getStats() {
    return {
      averageExecutionTime: this.executionTimes.reduce((a, b) => a + b, 0) / this.executionTimes.length,
      totalTasks: this.taskCounts.reduce((a, b) => a + b, 0),
      executionCount: this.executionTimes.length
    };
  }
}

// Usage
const monitor = new SchedulerMonitor();
const monitoredAsyncScheduler = monitor.wrapScheduler(asyncScheduler, 'AsyncScheduler');

// Use monitored scheduler
of(1, 2, 3).pipe(
  observeOn(monitoredAsyncScheduler)
).subscribe(value => console.log('Value:', value));

// Get performance stats
setTimeout(() => {
  console.log('Scheduler stats:', monitor.getStats());
}, 1000);
```

## 9. Best Practices

### 1. Choose the Right Scheduler
```typescript
// Decision matrix for scheduler selection
const schedulerGuide = {
  'synchronous-operations': null,              // No scheduler
  'prevent-stack-overflow': queueScheduler,   // Queue scheduler
  'high-priority-microtasks': asapScheduler,  // ASAP scheduler  
  'normal-async-operations': asyncScheduler,  // Async scheduler
  'smooth-animations': animationFrameScheduler, // Animation frame
  'testing': TestScheduler                    // Test scheduler
};

// Implementation examples
function chooseScheduler(context: string): SchedulerLike | null {
  switch (context) {
    case 'cpu-intensive':
      return asyncScheduler; // Prevent UI blocking
    case 'animation':
      return animationFrameScheduler; // Smooth 60fps
    case 'immediate':
      return asapScheduler; // High priority
    case 'bulk-processing':
      return queueScheduler; // Prevent stack overflow
    default:
      return null; // Synchronous
  }
}
```

### 2. Avoid Common Pitfalls
```typescript
// ❌ BAD: Blocking the UI with synchronous operations
of(heavyData).pipe(
  map(data => expensiveOperation(data)) // Blocks UI
).subscribe();

// ✅ GOOD: Use scheduler to prevent blocking
of(heavyData).pipe(
  observeOn(asyncScheduler),
  map(data => expensiveOperation(data)) // Non-blocking
).subscribe();

// ❌ BAD: Incorrect scheduler for animations
interval(16).pipe( // Trying to achieve 60fps
  map(frame => calculateAnimationFrame(frame))
).subscribe();

// ✅ GOOD: Use animationFrameScheduler for animations
interval(0, animationFrameScheduler).pipe(
  map(frame => calculateAnimationFrame(frame))
).subscribe();
```

### 3. Testing with Schedulers
```typescript
// Always use TestScheduler for time-dependent tests
describe('Time-dependent operations', () => {
  let testScheduler: TestScheduler;
  
  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });
  
  it('should handle delays correctly', () => {
    testScheduler.run(({ cold, expectObservable }) => {
      const source = cold('a-b-c|');
      const result = source.pipe(delay(10, testScheduler));
      const expected = '10ms a-b-c|';
      
      expectObservable(result).toBe(expected);
    });
  });
});
```

## Key Takeaways

1. **Schedulers control when and where Observable operations execute**
2. **Choose schedulers based on use case**: sync, async, animation, testing
3. **Use asyncScheduler to prevent UI blocking** for CPU-intensive operations
4. **Use animationFrameScheduler for smooth animations** at 60fps
5. **TestScheduler is essential** for testing time-dependent code
6. **subscribeOn controls subscription timing**, observeOn controls emission timing
7. **Consider Angular's zone.js** when working with schedulers in Angular apps
8. **Monitor scheduler performance** in production applications

## Next Steps
In the next lesson, we'll explore how operators work internally, building on our understanding of schedulers and execution contexts to see how RxJS operators transform and control Observable streams.
