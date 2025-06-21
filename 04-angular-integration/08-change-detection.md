# RxJS & Angular Change Detection

## Learning Objectives
- Understand Angular change detection mechanisms
- Master OnPush strategy with Observables
- Optimize performance with reactive patterns
- Handle zone.js interactions with RxJS
- Implement efficient subscription patterns
- Debug change detection issues

## Introduction to Change Detection

Angular's change detection system monitors component data changes and updates the DOM accordingly. Understanding how RxJS interacts with this system is crucial for building performant applications.

### Change Detection Strategies
- **Default Strategy**: Checks all components on every event
- **OnPush Strategy**: Only checks when inputs change or events are emitted
- **Zone.js**: Patches asynchronous operations to trigger change detection
- **Manual Control**: Direct control over when change detection runs

## Default vs OnPush Strategy

### Default Change Detection

```typescript
// default-component.ts
@Component({
  selector: 'app-default',
  changeDetection: ChangeDetectionStrategy.Default,
  template: `
    <div class="default-component">
      <h3>Default Strategy: {{ title }}</h3>
      <p>Count: {{ count }}</p>
      <p>Random: {{ randomValue }}</p>
      <p>Current Time: {{ currentTime | date:'medium' }}</p>
      
      <button (click)="increment()">Increment</button>
      <button (click)="updateTime()">Update Time</button>
      
      <!-- Child component -->
      <app-child [data]="childData"></app-child>
    </div>
  `
})
export class DefaultComponent implements OnInit {
  title = 'Default Component';
  count = 0;
  randomValue = Math.random();
  currentTime = new Date();
  childData = { value: 'initial' };

  private timer$ = interval(1000);

  ngOnInit() {
    // This will trigger change detection every second
    this.timer$.subscribe(() => {
      this.randomValue = Math.random();
      console.log('Timer tick - Change detection will run');
    });
  }

  increment(): void {
    this.count++;
    console.log('Button clicked - Change detection will run');
  }

  updateTime(): void {
    this.currentTime = new Date();
    // Mutating object reference (won't trigger OnPush children)
    this.childData.value = 'updated at ' + this.currentTime.toISOString();
  }
}
```

### OnPush Change Detection

```typescript
// onpush-component.ts
@Component({
  selector: 'app-onpush',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="onpush-component">
      <h3>OnPush Strategy: {{ title }}</h3>
      <p>Count: {{ count$ | async }}</p>
      <p>Random: {{ randomValue$ | async }}</p>
      <p>Current Time: {{ currentTime$ | async | date:'medium' }}</p>
      
      <button (click)="increment()">Increment</button>
      <button (click)="updateTime()">Update Time</button>
      <button (click)="forceDetection()">Force Detection</button>
      
      <!-- Child component with immutable data -->
      <app-onpush-child [data]="childData$ | async"></app-onpush-child>
    </div>
  `
})
export class OnPushComponent implements OnInit {
  title = 'OnPush Component';

  // Reactive properties
  private countSubject = new BehaviorSubject(0);
  readonly count$ = this.countSubject.asObservable();

  private randomSubject = new BehaviorSubject(Math.random());
  readonly randomValue$ = this.randomSubject.asObservable();

  private timeSubject = new BehaviorSubject(new Date());
  readonly currentTime$ = this.timeSubject.asObservable();

  private childDataSubject = new BehaviorSubject({ value: 'initial', timestamp: Date.now() });
  readonly childData$ = this.childDataSubject.asObservable();

  private timer$ = interval(1000);

  constructor(private cdr: ChangeDetectorRef) {}

  ngOnInit() {
    // AsyncPipe automatically triggers change detection
    this.timer$.subscribe(() => {
      this.randomSubject.next(Math.random());
      console.log('Timer tick - OnPush component will update via AsyncPipe');
    });
  }

  increment(): void {
    const currentCount = this.countSubject.value;
    this.countSubject.next(currentCount + 1);
    console.log('Button clicked - Change detection via event');
  }

  updateTime(): void {
    const now = new Date();
    this.timeSubject.next(now);
    
    // Create new object reference for OnPush children
    this.childDataSubject.next({
      value: 'updated at ' + now.toISOString(),
      timestamp: Date.now()
    });
  }

  forceDetection(): void {
    // Manual change detection trigger
    this.cdr.detectChanges();
    console.log('Manual change detection triggered');
  }
}
```

## AsyncPipe and Change Detection

### AsyncPipe Internals

```typescript
// custom-async.pipe.ts
@Pipe({
  name: 'customAsync',
  pure: false // Impure pipe to check on every cycle
})
export class CustomAsyncPipe implements OnDestroy, PipeTransform {
  private subscription: Subscription | null = null;
  private latestValue: any = null;
  private latestReturnedValue: any = null;

  constructor(private cdr: ChangeDetectorRef) {}

  transform(obj: Observable<any> | Promise<any> | null | undefined): any {
    if (!obj) {
      if (this.subscription) {
        this._dispose();
        return this.latestValue = null;
      }
      return null;
    }

    if (!this.subscription) {
      this._subscribe(obj);
      return this.latestValue;
    }

    if (obj !== this.latestReturnedValue) {
      this._dispose();
      return this.transform(obj);
    }

    return this.latestValue;
  }

  private _subscribe(obj: Observable<any> | Promise<any>): void {
    this.latestReturnedValue = obj;
    
    if (obj instanceof Observable) {
      this.subscription = obj.subscribe({
        next: (value) => this._updateLatestValue(value),
        error: (error) => { throw error; }
      });
    } else {
      // Handle Promise
      Promise.resolve(obj).then(
        (value) => this._updateLatestValue(value),
        (error) => { throw error; }
      );
    }
  }

  private _updateLatestValue(value: any): void {
    this.latestValue = value;
    // Trigger change detection
    this.cdr.markForCheck();
  }

  private _dispose(): void {
    if (this.subscription) {
      this.subscription.unsubscribe();
      this.subscription = null;
    }
    this.latestReturnedValue = null;
    this.latestValue = null;
  }

  ngOnDestroy(): void {
    this._dispose();
  }
}
```

### Optimized Subscription Patterns

```typescript
// optimized-subscriptions.component.ts
@Component({
  selector: 'app-optimized',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="optimized-component">
      <!-- Single subscription with combineLatest -->
      <div *ngIf="viewModel$ | async as vm">
        <h3>{{ vm.title }}</h3>
        <p>User: {{ vm.user.name }}</p>
        <p>Posts: {{ vm.posts.length }}</p>
        <p>Loading: {{ vm.loading }}</p>
        
        <div class="posts">
          <div *ngFor="let post of vm.posts; trackBy: trackByPostId" 
               class="post-item">
            {{ post.title }}
          </div>
        </div>
      </div>

      <!-- Separate observables for different update frequencies -->
      <div class="status">
        <span>Status: {{ connectionStatus$ | async }}</span>
        <span>Time: {{ currentTime$ | async | date:'short' }}</span>
      </div>
    </div>
  `
})
export class OptimizedComponent implements OnInit {
  
  // Combined view model to reduce template subscriptions
  readonly viewModel$ = combineLatest([
    this.userService.currentUser$,
    this.postService.posts$,
    this.loadingService.loading$
  ]).pipe(
    map(([user, posts, loading]) => ({
      title: `${user?.name}'s Dashboard`,
      user: user || { name: 'Anonymous' },
      posts: posts || [],
      loading
    })),
    // Cache the latest value to prevent unnecessary recalculations
    shareReplay(1)
  );

  // Separate observables for different concerns
  readonly connectionStatus$ = this.connectionService.status$.pipe(
    distinctUntilChanged(),
    shareReplay(1)
  );

  readonly currentTime$ = interval(1000).pipe(
    map(() => new Date()),
    shareReplay(1)
  );

  // Memoized trackBy function
  readonly trackByPostId = (index: number, post: Post): string => post.id;

  constructor(
    private userService: UserService,
    private postService: PostService,
    private loadingService: LoadingService,
    private connectionService: ConnectionService
  ) {}

  ngOnInit() {
    // Preload data
    this.postService.loadPosts().subscribe();
  }
}
```

## Zone.js and RxJS Integration

### Understanding Zone.js Patches

```typescript
// zone-interaction.service.ts
@Injectable({
  providedIn: 'root'
})
export class ZoneInteractionService {
  
  constructor(private ngZone: NgZone) {}

  // Running outside Angular zone
  createHighFrequencyTimer(): Observable<number> {
    return new Observable<number>(subscriber => {
      let count = 0;
      
      // Run outside zone to avoid triggering change detection
      this.ngZone.runOutsideAngular(() => {
        const intervalId = setInterval(() => {
          count++;
          subscriber.next(count);
        }, 10); // Very high frequency
        
        return () => clearInterval(intervalId);
      });
    });
  }

  // Running inside Angular zone when needed
  createControlledTimer(): Observable<number> {
    return this.createHighFrequencyTimer().pipe(
      // Only emit every 100 ticks
      filter(count => count % 100 === 0),
      // Re-enter Angular zone for change detection
      observeOn(asyncScheduler)
    );
  }

  // Manual zone control
  processDataOutsideZone<T>(source$: Observable<T>): Observable<T> {
    return new Observable<T>(subscriber => {
      this.ngZone.runOutsideAngular(() => {
        source$.subscribe({
          next: value => {
            // Process outside zone
            const processed = this.heavyProcessing(value);
            
            // Re-enter zone for UI updates
            this.ngZone.run(() => {
              subscriber.next(processed);
            });
          },
          error: err => subscriber.error(err),
          complete: () => subscriber.complete()
        });
      });
    });
  }

  private heavyProcessing<T>(data: T): T {
    // Simulate heavy computation
    return data;
  }
}
```

### Custom Scheduler for Zone Control

```typescript
// zone-scheduler.ts
class ZoneScheduler implements SchedulerLike {
  constructor(private ngZone: NgZone, private runInZone: boolean = true) {}

  now(): number {
    return Date.now();
  }

  schedule<T>(
    work: (this: SchedulerAction<T>, state?: T) => void,
    delay: number = 0,
    state?: T
  ): Subscription {
    const action = new ZoneAction(work, this.ngZone, this.runInZone);
    return action.schedule(state, delay);
  }
}

class ZoneAction<T> extends AsyncAction<T> {
  constructor(
    protected work: (this: SchedulerAction<T>, state?: T) => void,
    private ngZone: NgZone,
    private runInZone: boolean
  ) {
    super(null as any, work);
  }

  protected requestAsyncId(scheduler: any, id?: any, delay: number = 0): any {
    if (this.runInZone) {
      return this.ngZone.run(() => super.requestAsyncId(scheduler, id, delay));
    } else {
      return this.ngZone.runOutsideAngular(() => 
        super.requestAsyncId(scheduler, id, delay)
      );
    }
  }
}

// Usage
const insideZoneScheduler = new ZoneScheduler(this.ngZone, true);
const outsideZoneScheduler = new ZoneScheduler(this.ngZone, false);

// Process outside zone, emit inside zone
source$.pipe(
  observeOn(outsideZoneScheduler), // Heavy processing
  map(data => this.processData(data)),
  observeOn(insideZoneScheduler)   // UI updates
);
```

## Manual Change Detection Control

### Selective Change Detection

```typescript
// selective-detection.component.ts
@Component({
  selector: 'app-selective',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="selective-component">
      <div class="high-frequency" #highFreq>
        <h4>High Frequency Updates</h4>
        <p>Value: {{ highFrequencyValue }}</p>
        <p>Update Count: {{ highFrequencyCount }}</p>
      </div>

      <div class="low-frequency">
        <h4>Low Frequency Updates</h4>
        <p>Data: {{ lowFrequencyData$ | async | json }}</p>
      </div>

      <div class="manual-control">
        <button (click)="startHighFrequency()">Start High Frequency</button>
        <button (click)="stopHighFrequency()">Stop High Frequency</button>
        <button (click)="detectChanges()">Detect Changes</button>
      </div>
    </div>
  `
})
export class SelectiveDetectionComponent implements OnInit, OnDestroy {
  @ViewChild('highFreq', { read: ElementRef }) highFreqElement: ElementRef;

  highFrequencyValue = 0;
  highFrequencyCount = 0;
  
  readonly lowFrequencyData$ = interval(2000).pipe(
    map(i => ({ timestamp: new Date(), value: i }))
  );

  private highFrequencySubscription: Subscription | null = null;
  private destroy$ = new Subject<void>();

  constructor(
    private cdr: ChangeDetectorRef,
    private ngZone: NgZone,
    private renderer: Renderer2
  ) {}

  ngOnInit() {
    // Low frequency updates that trigger change detection
    this.lowFrequencyData$
      .pipe(takeUntil(this.destroy$))
      .subscribe(() => {
        console.log('Low frequency update - change detection triggered');
      });
  }

  startHighFrequency(): void {
    if (this.highFrequencySubscription) {
      return;
    }

    // High frequency updates outside Angular zone
    this.highFrequencySubscription = this.ngZone.runOutsideAngular(() => {
      return interval(16).pipe( // ~60fps
        takeUntil(this.destroy$)
      ).subscribe(value => {
        this.highFrequencyValue = value;
        this.highFrequencyCount++;
        
        // Direct DOM manipulation to avoid change detection
        if (this.highFreqElement) {
          const valueEl = this.highFreqElement.nativeElement.querySelector('p:first-of-type');
          const countEl = this.highFreqElement.nativeElement.querySelector('p:last-of-type');
          
          if (valueEl) {
            this.renderer.setProperty(valueEl, 'textContent', `Value: ${value}`);
          }
          if (countEl) {
            this.renderer.setProperty(countEl, 'textContent', `Update Count: ${this.highFrequencyCount}`);
          }
        }

        // Trigger change detection occasionally
        if (value % 60 === 0) { // Once per second
          this.ngZone.run(() => {
            this.cdr.detectChanges();
            console.log('Periodic change detection triggered');
          });
        }
      });
    });
  }

  stopHighFrequency(): void {
    if (this.highFrequencySubscription) {
      this.highFrequencySubscription.unsubscribe();
      this.highFrequencySubscription = null;
    }
  }

  detectChanges(): void {
    this.cdr.detectChanges();
    console.log('Manual change detection triggered');
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
    this.stopHighFrequency();
  }
}
```

### Detached Change Detection

```typescript
// detached-component.ts
@Component({
  selector: 'app-detached',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="detached-component">
      <h4>Detached Component</h4>
      <p>Status: {{ status }}</p>
      <p>Updates: {{ updateCount }}</p>
      
      <button (click)="attach()">Attach</button>
      <button (click)="detach()">Detach</button>
      <button (click)="reattach()">Reattach</button>
    </div>
  `
})
export class DetachedComponent implements OnInit, OnDestroy {
  status = 'attached';
  updateCount = 0;

  private timer$ = interval(1000);
  private destroy$ = new Subject<void>();

  constructor(private cdr: ChangeDetectorRef) {}

  ngOnInit() {
    this.timer$
      .pipe(takeUntil(this.destroy$))
      .subscribe(() => {
        this.updateCount++;
        this.status = `Updated at ${new Date().toLocaleTimeString()}`;
        
        // Only detect changes if attached
        if (!this.cdr.checkNoChanges) {
          this.cdr.detectChanges();
        }
      });
  }

  attach(): void {
    this.cdr.reattach();
    this.status = 'attached';
    this.cdr.detectChanges();
    console.log('Component attached to change detection');
  }

  detach(): void {
    this.cdr.detach();
    this.status = 'detached';
    console.log('Component detached from change detection');
  }

  reattach(): void {
    this.cdr.reattach();
    this.status = 'reattached';
    this.cdr.detectChanges();
    console.log('Component reattached to change detection');
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## Performance Optimization Strategies

### Immutable State Updates

```typescript
// immutable-state.component.ts
interface AppState {
  users: User[];
  selectedUserId: string | null;
  loading: boolean;
}

@Component({
  selector: 'app-immutable-state',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="immutable-state">
      <div *ngIf="state$ | async as state">
        <div class="user-list">
          <div *ngFor="let user of state.users; trackBy: trackByUserId"
               [class.selected]="user.id === state.selectedUserId"
               (click)="selectUser(user.id)">
            {{ user.name }}
          </div>
        </div>
        
        <div *ngIf="state.loading" class="loading">Loading...</div>
      </div>
    </div>
  `
})
export class ImmutableStateComponent {
  private stateSubject = new BehaviorSubject<AppState>({
    users: [],
    selectedUserId: null,
    loading: false
  });

  readonly state$ = this.stateSubject.asObservable();

  readonly trackByUserId = (index: number, user: User): string => user.id;

  constructor(private userService: UserService) {
    this.loadUsers();
  }

  private loadUsers(): void {
    this.updateState({ loading: true });
    
    this.userService.getUsers().subscribe(users => {
      this.updateState({ users, loading: false });
    });
  }

  selectUser(userId: string): void {
    this.updateState({ selectedUserId: userId });
  }

  private updateState(partial: Partial<AppState>): void {
    const currentState = this.stateSubject.value;
    // Create new state object for immutability
    const newState = { ...currentState, ...partial };
    this.stateSubject.next(newState);
  }
}
```

### Memoized Computations

```typescript
// memoized-component.ts
@Component({
  selector: 'app-memoized',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="memoized-component">
      <div *ngIf="viewModel$ | async as vm">
        <h4>Expensive Computation Result</h4>
        <p>Sum: {{ vm.sum }}</p>
        <p>Average: {{ vm.average }}</p>
        <p>Computation Time: {{ vm.computationTime }}ms</p>
        
        <div class="numbers">
          <span *ngFor="let num of vm.numbers">{{ num }}</span>
        </div>
      </div>

      <button (click)="addRandomNumber()">Add Random Number</button>
      <button (click)="clearNumbers()">Clear Numbers</button>
    </div>
  `
})
export class MemoizedComponent {
  private numbersSubject = new BehaviorSubject<number[]>([1, 2, 3, 4, 5]);

  readonly viewModel$ = this.numbersSubject.pipe(
    map(numbers => this.computeExpensiveStats(numbers)),
    shareReplay(1) // Memoize the result
  );

  private memoizedComputation = memoize(this.expensiveComputation.bind(this));

  addRandomNumber(): void {
    const current = this.numbersSubject.value;
    const newNumber = Math.floor(Math.random() * 100);
    this.numbersSubject.next([...current, newNumber]);
  }

  clearNumbers(): void {
    this.numbersSubject.next([]);
  }

  private computeExpensiveStats(numbers: number[]): any {
    const startTime = performance.now();
    
    // Use memoized computation
    const result = this.memoizedComputation(numbers);
    
    const endTime = performance.now();
    
    return {
      ...result,
      computationTime: endTime - startTime
    };
  }

  private expensiveComputation(numbers: number[]): any {
    // Simulate expensive computation
    let sum = 0;
    for (let i = 0; i < numbers.length; i++) {
      for (let j = 0; j < 1000; j++) {
        sum += numbers[i];
      }
    }
    
    return {
      numbers,
      sum: numbers.reduce((a, b) => a + b, 0),
      average: numbers.length > 0 ? sum / numbers.length / 1000 : 0
    };
  }
}
```

## Debugging Change Detection

### Change Detection Profiler

```typescript
// change-detection-profiler.service.ts
@Injectable({
  providedIn: 'root'
})
export class ChangeDetectionProfiler {
  private profiles = new Map<string, number[]>();

  profile<T>(name: string, fn: () => T): T {
    const start = performance.now();
    const result = fn();
    const end = performance.now();
    
    if (!this.profiles.has(name)) {
      this.profiles.set(name, []);
    }
    
    this.profiles.get(name)!.push(end - start);
    
    return result;
  }

  getProfileStats(name: string): any {
    const times = this.profiles.get(name) || [];
    
    if (times.length === 0) {
      return null;
    }

    const sum = times.reduce((a, b) => a + b, 0);
    const avg = sum / times.length;
    const min = Math.min(...times);
    const max = Math.max(...times);

    return { count: times.length, avg, min, max, total: sum };
  }

  logAllProfiles(): void {
    console.group('Change Detection Profiles');
    for (const [name, times] of this.profiles) {
      const stats = this.getProfileStats(name);
      console.log(`${name}:`, stats);
    }
    console.groupEnd();
  }

  clearProfiles(): void {
    this.profiles.clear();
  }
}

// Usage in component
@Component({
  selector: 'app-profiled',
  template: `...`
})
export class ProfiledComponent {
  constructor(private profiler: ChangeDetectionProfiler) {}

  expensiveOperation(): any {
    return this.profiler.profile('expensiveOperation', () => {
      // Expensive computation
      let result = 0;
      for (let i = 0; i < 1000000; i++) {
        result += Math.random();
      }
      return result;
    });
  }
}
```

### Debug Utilities

```typescript
// debug-change-detection.directive.ts
@Directive({
  selector: '[debugCD]'
})
export class DebugChangeDetectionDirective implements DoCheck {
  @Input() debugCD: string = '';

  private previousValue: any;

  ngDoCheck(): void {
    if (this.debugCD && this.previousValue !== this.debugCD) {
      console.log(`[DebugCD] ${this.debugCD} changed from`, this.previousValue, 'to', this.debugCD);
      this.previousValue = this.debugCD;
    }
  }
}

// Usage
// <div [debugCD]="someValue">Content</div>
```

## Testing Change Detection

### Testing OnPush Components

```typescript
// onpush-component.spec.ts
describe('OnPushComponent', () => {
  let component: OnPushComponent;
  let fixture: ComponentFixture<OnPushComponent>;

  beforeEach(() => {
    TestBed.configureTestingModule({
      declarations: [OnPushComponent],
      changeDetection: ChangeDetectionStrategy.OnPush
    });

    fixture = TestBed.createComponent(OnPushComponent);
    component = fixture.componentInstance;
  });

  it('should not update without change detection trigger', () => {
    component.title = 'New Title';
    
    // No change detection
    expect(fixture.nativeElement.textContent).not.toContain('New Title');
    
    // Trigger change detection
    fixture.detectChanges();
    expect(fixture.nativeElement.textContent).toContain('New Title');
  });

  it('should update with observable changes', fakeAsync(() => {
    const testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });

    testScheduler.run(({ cold, expectObservable }) => {
      const values$ = cold('a-b-c', {
        a: 'value1',
        b: 'value2', 
        c: 'value3'
      });

      component.value$ = values$;
      fixture.detectChanges();

      tick(1);
      expect(fixture.nativeElement.textContent).toContain('value1');

      tick(1);
      expect(fixture.nativeElement.textContent).toContain('value2');

      tick(1);
      expect(fixture.nativeElement.textContent).toContain('value3');
    });
  }));
});
```

## Best Practices

### 1. OnPush Strategy
- Use OnPush for performance-critical components
- Ensure immutable data flow
- Use AsyncPipe for reactive updates
- Implement proper trackBy functions

### 2. Observable Patterns
- Prefer reactive patterns over imperative updates
- Use shareReplay for expensive computations
- Combine related observables to reduce subscriptions
- Use distinctUntilChanged to prevent unnecessary updates

### 3. Zone.js Management
- Run expensive operations outside Angular zone
- Use observeOn to control scheduler context
- Be mindful of third-party libraries and zone patches
- Use runOutsideAngular for high-frequency events

### 4. Manual Control
- Use ChangeDetectorRef judiciously
- Detach components for complex animations
- Profile change detection performance
- Implement selective update strategies

### 5. Testing
- Test change detection behavior explicitly
- Use TestScheduler for timing-dependent tests
- Mock ChangeDetectorRef in unit tests
- Verify OnPush component behavior

## Common Pitfalls

### 1. Mutating Objects
```typescript
// ❌ Bad: Mutating object (won't trigger OnPush)
updateUser() {
  this.user.name = 'New Name'; // Mutation
}

// ✅ Good: Creating new object
updateUser() {
  this.user = { ...this.user, name: 'New Name' };
}
```

### 2. Excessive Subscriptions
```typescript
// ❌ Bad: Multiple subscriptions in template
template: `
  <div>{{ data$ | async }}</div>
  <div>{{ (data$ | async)?.property }}</div>
  <div>{{ (data$ | async)?.method() }}</div>
`

// ✅ Good: Single subscription
template: `
  <div *ngIf="data$ | async as data">
    <div>{{ data }}</div>
    <div>{{ data.property }}</div>
    <div>{{ data.method() }}</div>
  </div>
`
```

### 3. Zone.js Issues
```typescript
// ❌ Bad: Blocking the main thread
heavyComputation() {
  // Expensive sync operation
  for (let i = 0; i < 1000000; i++) {
    // computation
  }
}

// ✅ Good: Running outside zone
heavyComputation() {
  this.ngZone.runOutsideAngular(() => {
    // Expensive sync operation
    for (let i = 0; i < 1000000; i++) {
      // computation
    }
    
    this.ngZone.run(() => {
      // Update UI
    });
  });
}
```

## Summary

Understanding the interaction between RxJS and Angular's change detection is crucial for building performant applications. Key takeaways:

- Use OnPush strategy with reactive patterns for optimal performance
- Leverage AsyncPipe for automatic change detection with observables
- Control zone.js behavior for expensive operations
- Implement immutable data flow for predictable updates
- Use manual change detection control when needed
- Profile and debug change detection issues systematically
- Test change detection behavior in components
- Combine observables to reduce template subscriptions
- Use proper trackBy functions for ngFor optimization
- Avoid common pitfalls like object mutations and excessive subscriptions

These patterns enable building highly performant Angular applications that scale well with complex reactive data flows.
