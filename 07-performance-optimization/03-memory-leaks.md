# Identifying & Preventing Memory Leaks

## Table of Contents
1. [Introduction to Memory Leaks in RxJS](#introduction)
2. [Types of Memory Leaks](#types-of-leaks)
3. [Detection Techniques](#detection)
4. [Prevention Strategies](#prevention)
5. [Memory Leak Testing](#testing)
6. [Monitoring & Alerting](#monitoring)
7. [Angular-Specific Leak Patterns](#angular-patterns)
8. [Recovery & Cleanup Strategies](#cleanup)
9. [Tools & Libraries](#tools)
10. [Exercises](#exercises)

## Introduction to Memory Leaks in RxJS {#introduction}

Memory leaks in RxJS applications occur when observables, subscriptions, or related objects are not properly cleaned up, leading to memory accumulation over time.

### Understanding Memory Leaks

```typescript
interface MemoryLeakProfile {
  type: 'subscription' | 'closure' | 'dom' | 'circular';
  severity: 'low' | 'medium' | 'high' | 'critical';
  growthRate: number; // MB per hour
  detectionDifficulty: 'easy' | 'medium' | 'hard';
  commonScenarios: string[];
}

// Memory leak impact assessment
const MEMORY_LEAK_TYPES: Record<string, MemoryLeakProfile> = {
  unmanagedSubscription: {
    type: 'subscription',
    severity: 'high',
    growthRate: 5,
    detectionDifficulty: 'easy',
    commonScenarios: [
      'Component destruction without unsubscription',
      'Service subscriptions without cleanup',
      'Event listener subscriptions'
    ]
  },
  
  retainedClosures: {
    type: 'closure',
    severity: 'medium',
    growthRate: 2,
    detectionDifficulty: 'hard',
    commonScenarios: [
      'Operators capturing large objects',
      'Callback functions with references',
      'Cached observables holding data'
    ]
  },
  
  domReferences: {
    type: 'dom',
    severity: 'medium',
    growthRate: 3,
    detectionDifficulty: 'medium',
    commonScenarios: [
      'Element references in observables',
      'Event listeners on destroyed elements',
      'Template reference variables'
    ]
  },
  
  circularReferences: {
    type: 'circular',
    severity: 'critical',
    growthRate: 10,
    detectionDifficulty: 'hard',
    commonScenarios: [
      'Parent-child component communication',
      'Service dependency cycles',
      'Recursive observable patterns'
    ]
  }
};
```

### Memory Leak Lifecycle

```typescript
class MemoryLeakAnalyzer {
  // Track memory leak progression
  analyzeLeakProgression(measurements: MemoryMeasurement[]): LeakAnalysis {
    const growthRate = this.calculateGrowthRate(measurements);
    const patterns = this.identifyPatterns(measurements);
    const severity = this.assessSeverity(growthRate, patterns);
    
    return {
      growthRate,
      patterns,
      severity,
      timeToOOM: this.estimateTimeToOutOfMemory(growthRate),
      recommendations: this.generateRecommendations(patterns)
    };
  }

  private calculateGrowthRate(measurements: MemoryMeasurement[]): number {
    if (measurements.length < 2) return 0;
    
    const first = measurements[0];
    const last = measurements[measurements.length - 1];
    const timeDiff = (last.timestamp - first.timestamp) / (1000 * 60 * 60); // hours
    const memoryDiff = last.heapUsed - first.heapUsed;
    
    return memoryDiff / timeDiff; // MB per hour
  }

  private identifyPatterns(measurements: MemoryMeasurement[]): LeakPattern[] {
    const patterns: LeakPattern[] = [];
    
    // Detect consistent growth pattern
    if (this.isConsistentGrowth(measurements)) {
      patterns.push({
        type: 'consistent_growth',
        confidence: 0.9,
        description: 'Memory consistently increasing over time'
      });
    }
    
    // Detect spike patterns
    if (this.hasMemorySpikes(measurements)) {
      patterns.push({
        type: 'memory_spikes',
        confidence: 0.8,
        description: 'Sudden memory increases followed by partial cleanup'
      });
    }
    
    return patterns;
  }
}

interface MemoryMeasurement {
  timestamp: number;
  heapUsed: number;
  heapTotal: number;
  external: number;
  rss?: number;
}

interface LeakPattern {
  type: string;
  confidence: number;
  description: string;
}

interface LeakAnalysis {
  growthRate: number;
  patterns: LeakPattern[];
  severity: 'low' | 'medium' | 'high' | 'critical';
  timeToOOM: number; // hours until out of memory
  recommendations: string[];
}
```

## Types of Memory Leaks {#types-of-leaks}

### 1. Subscription Memory Leaks

```typescript
// ❌ PROBLEM: Unmanaged subscriptions
@Component({
  selector: 'app-leaky-subscriptions',
  template: `<div>Leaky component</div>`
})
export class LeakySubscriptionsComponent implements OnInit {
  data: any[] = [];

  constructor(
    private dataService: DataService,
    private router: Router
  ) {}

  ngOnInit() {
    // ❌ HTTP subscription never cleaned up
    this.dataService.getData().subscribe(data => {
      this.data = data;
    });

    // ❌ Router subscription continues after component destruction
    this.router.events.subscribe(event => {
      if (event instanceof NavigationEnd) {
        console.log('Navigation:', event.url);
      }
    });

    // ❌ Interval subscription runs forever
    interval(1000).subscribe(tick => {
      this.updateTimestamp(tick);
    });

    // ❌ WebSocket subscription never closed
    this.dataService.getWebSocketData().subscribe(message => {
      this.processMessage(message);
    });
  }

  private updateTimestamp(tick: number): void {
    // This method and its closure keep growing in memory
    const timestamp = new Date();
    console.log(`Tick ${tick} at ${timestamp}`);
  }

  private processMessage(message: any): void {
    // Processing logic that retains references
  }
}

// ✅ SOLUTION: Proper subscription management
@Component({
  selector: 'app-clean-subscriptions',
  template: `<div>Clean component</div>`
})
export class CleanSubscriptionsComponent implements OnInit, OnDestroy {
  data: any[] = [];
  private destroy$ = new Subject<void>();
  private subscriptions = new CompositeSubscription();

  constructor(
    private dataService: DataService,
    private router: Router
  ) {}

  ngOnInit() {
    // ✅ HTTP subscription with automatic cleanup
    this.dataService.getData().pipe(
      takeUntil(this.destroy$)
    ).subscribe(data => {
      this.data = data;
    });

    // ✅ Router subscription with cleanup
    this.router.events.pipe(
      filter(event => event instanceof NavigationEnd),
      takeUntil(this.destroy$)
    ).subscribe(event => {
      console.log('Navigation:', (event as NavigationEnd).url);
    });

    // ✅ Interval subscription with cleanup
    interval(1000).pipe(
      takeUntil(this.destroy$)
    ).subscribe(tick => {
      this.updateTimestamp(tick);
    });

    // ✅ WebSocket subscription with proper cleanup
    this.subscriptions.add(
      this.dataService.getWebSocketData().subscribe(message => {
        this.processMessage(message);
      })
    );
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
    this.subscriptions.unsubscribe();
  }

  private updateTimestamp(tick: number): void {
    const timestamp = new Date();
    console.log(`Tick ${tick} at ${timestamp}`);
  }

  private processMessage(message: any): void {
    // Processing logic
  }
}

// ✅ Advanced subscription management utility
class CompositeSubscription {
  private subscriptions: Subscription[] = [];

  add(subscription: Subscription): void {
    this.subscriptions.push(subscription);
  }

  unsubscribe(): void {
    this.subscriptions.forEach(sub => {
      if (!sub.closed) {
        sub.unsubscribe();
      }
    });
    this.subscriptions = [];
  }

  get count(): number {
    return this.subscriptions.filter(sub => !sub.closed).length;
  }
}
```

### 2. Closure Memory Leaks

```typescript
// ❌ PROBLEM: Closures retaining large objects
class ClosureLeakExample {
  processLargeDataset(dataset: LargeObject[]): Observable<ProcessedData[]> {
    // ❌ Closure captures entire dataset
    return from(dataset).pipe(
      map(item => {
        // This closure keeps reference to entire dataset
        return this.processItem(item, dataset);
      }),
      toArray()
    );
  }

  createObservableWithClosure(largeConfig: ConfigObject): Observable<any> {
    // ❌ Observer function captures large config object
    return new Observable(observer => {
      const processData = (data: any) => {
        // Entire largeConfig remains in memory
        if (largeConfig.enableFeature) {
          observer.next(this.transformData(data, largeConfig));
        }
      };
      
      return this.dataSource.subscribe(processData);
    });
  }
}

// ✅ SOLUTION: Minimize closure captures
class OptimizedClosureExample {
  processLargeDataset(dataset: LargeObject[]): Observable<ProcessedData[]> {
    // ✅ Extract only needed values from dataset
    const datasetSize = dataset.length;
    const processingConfig = this.extractProcessingConfig(dataset);
    
    return from(dataset).pipe(
      map(item => {
        // Closure only captures minimal data
        return this.processItem(item, { size: datasetSize, config: processingConfig });
      }),
      toArray()
    );
  }

  createObservableWithClosure(largeConfig: ConfigObject): Observable<any> {
    // ✅ Extract only needed configuration
    const enableFeature = largeConfig.enableFeature;
    const transformConfig = {
      option1: largeConfig.transformation.option1,
      option2: largeConfig.transformation.option2
    };
    
    return new Observable(observer => {
      const processData = (data: any) => {
        // Closure captures minimal data
        if (enableFeature) {
          observer.next(this.transformData(data, transformConfig));
        }
      };
      
      return this.dataSource.subscribe(processData);
    });
  }

  private extractProcessingConfig(dataset: LargeObject[]): ProcessingConfig {
    // Extract minimal configuration needed for processing
    return {
      averageSize: dataset.reduce((sum, item) => sum + item.size, 0) / dataset.length,
      maxComplexity: Math.max(...dataset.map(item => item.complexity))
    };
  }
}
```

### 3. DOM Reference Leaks

```typescript
// ❌ PROBLEM: DOM references in observables
@Component({
  selector: 'app-dom-leak',
  template: `
    <div #container>
      <button #actionButton (click)="setupObservables()">Setup</button>
      <div *ngFor="let item of items" #itemElement>{{ item }}</div>
    </div>
  `
})
export class DomLeakComponent implements OnInit, AfterViewInit {
  @ViewChild('container') container!: ElementRef;
  @ViewChild('actionButton') actionButton!: ElementRef;
  @ViewChildren('itemElement') itemElements!: QueryList<ElementRef>;

  items = ['Item 1', 'Item 2', 'Item 3'];

  ngAfterViewInit() {
    // ❌ Observable retains reference to DOM elements
    this.setupDomObservables();
  }

  setupDomObservables() {
    // ❌ Element references prevent garbage collection
    fromEvent(this.actionButton.nativeElement, 'click').subscribe(() => {
      // This subscription keeps reference to button element
      this.animateContainer(this.container.nativeElement);
    });

    // ❌ QueryList elements remain in memory
    this.itemElements.forEach(elementRef => {
      fromEvent(elementRef.nativeElement, 'mouseover').subscribe(() => {
        // Each element reference is retained
        this.highlightElement(elementRef.nativeElement);
      });
    });
  }

  private animateContainer(element: HTMLElement): void {
    // Animation logic
  }

  private highlightElement(element: HTMLElement): void {
    // Highlighting logic
  }
}

// ✅ SOLUTION: Minimize DOM references and use proper cleanup
@Component({
  selector: 'app-clean-dom',
  template: `
    <div #container>
      <button (click)="handleClick()">Action</button>
      <div *ngFor="let item of items; trackBy: trackByItem" 
           (mouseover)="handleMouseOver($event)">{{ item }}</div>
    </div>
  `
})
export class CleanDomComponent implements OnInit, OnDestroy, AfterViewInit {
  @ViewChild('container') container!: ElementRef;
  
  items = ['Item 1', 'Item 2', 'Item 3'];
  private destroy$ = new Subject<void>();

  ngAfterViewInit() {
    // ✅ Use event delegation to minimize element references
    this.setupEfficientEventHandling();
  }

  private setupEfficientEventHandling(): void {
    // ✅ Single event listener on container with delegation
    fromEvent(this.container.nativeElement, 'click').pipe(
      takeUntil(this.destroy$)
    ).subscribe((event: Event) => {
      const target = event.target as HTMLElement;
      if (target.tagName === 'BUTTON') {
        this.handleClick();
      }
    });

    // ✅ Use CSS for hover effects instead of JS listeners
    // No DOM references retained in JavaScript
  }

  handleClick(): void {
    // Handle click without retaining element reference
    const element = this.container.nativeElement;
    this.animateContainer(element);
    // Element reference is local and will be garbage collected
  }

  handleMouseOver(event: MouseEvent): void {
    // Handle mouseover with event target
    const element = event.target as HTMLElement;
    this.highlightElement(element);
    // No retained references
  }

  trackByItem(index: number, item: string): string {
    return item;
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }

  private animateContainer(element: HTMLElement): void {
    // Animation logic
  }

  private highlightElement(element: HTMLElement): void {
    // Highlighting logic
  }
}
```

### 4. Circular Reference Leaks

```typescript
// ❌ PROBLEM: Circular references between components/services
@Injectable()
export class LeakyParentService {
  private children: LeakyChildService[] = [];

  addChild(child: LeakyChildService): void {
    this.children.push(child);
    // ❌ Child keeps reference to parent, parent keeps reference to child
    child.setParent(this);
  }

  removeChild(child: LeakyChildService): void {
    const index = this.children.indexOf(child);
    if (index > -1) {
      // ❌ Only removes from array, doesn't clear parent reference in child
      this.children.splice(index, 1);
    }
  }
}

@Injectable()
export class LeakyChildService {
  private parent: LeakyParentService | null = null;

  setParent(parent: LeakyParentService): void {
    this.parent = parent;
  }

  ngOnDestroy() {
    // ❌ Doesn't clear parent reference
    // Circular reference prevents garbage collection
  }
}

// ✅ SOLUTION: Break circular references with WeakRef or proper cleanup
@Injectable()
export class CleanParentService implements OnDestroy {
  private children = new Set<CleanChildService>();
  private childDestroySubscriptions = new Map<CleanChildService, Subscription>();

  addChild(child: CleanChildService): void {
    this.children.add(child);
    
    // ✅ Subscribe to child destruction to auto-cleanup
    const destroySubscription = child.onDestroy$.subscribe(() => {
      this.removeChild(child);
    });
    
    this.childDestroySubscriptions.set(child, destroySubscription);
    child.setParent(this);
  }

  removeChild(child: CleanChildService): void {
    // ✅ Properly break circular reference
    if (this.children.has(child)) {
      this.children.delete(child);
      child.clearParent();
      
      // Clean up destroy subscription
      const subscription = this.childDestroySubscriptions.get(child);
      if (subscription) {
        subscription.unsubscribe();
        this.childDestroySubscriptions.delete(child);
      }
    }
  }

  ngOnDestroy() {
    // ✅ Clear all references
    this.children.forEach(child => child.clearParent());
    this.children.clear();
    
    this.childDestroySubscriptions.forEach(sub => sub.unsubscribe());
    this.childDestroySubscriptions.clear();
  }
}

@Injectable()
export class CleanChildService implements OnDestroy {
  private parent: WeakRef<CleanParentService> | null = null;
  onDestroy$ = new Subject<void>();

  setParent(parent: CleanParentService): void {
    // ✅ Use WeakRef to prevent circular reference
    this.parent = new WeakRef(parent);
  }

  clearParent(): void {
    this.parent = null;
  }

  getParent(): CleanParentService | null {
    return this.parent?.deref() || null;
  }

  ngOnDestroy() {
    this.onDestroy$.next();
    this.onDestroy$.complete();
    this.clearParent();
  }
}
```

## Detection Techniques {#detection}

### Browser DevTools Memory Profiling

```typescript
@Injectable({
  providedIn: 'root'
})
export class MemoryProfiler {
  private measurements: MemoryMeasurement[] = [];
  private isProfileActive = false;

  startProfiling(intervalMs: number = 1000): void {
    if (this.isProfileActive) return;
    
    this.isProfileActive = true;
    this.measurements = [];
    
    const measureMemory = () => {
      if (!this.isProfileActive) return;
      
      const measurement = this.captureMemorySnapshot();
      this.measurements.push(measurement);
      
      // Log warnings for significant increases
      if (this.measurements.length > 1) {
        this.analyzeMemoryTrend();
      }
      
      setTimeout(measureMemory, intervalMs);
    };
    
    measureMemory();
  }

  stopProfiling(): MemoryProfileReport {
    this.isProfileActive = false;
    return this.generateReport();
  }

  private captureMemorySnapshot(): MemoryMeasurement {
    const memory = (performance as any).memory || {
      usedJSHeapSize: 0,
      totalJSHeapSize: 0,
      jsHeapSizeLimit: 0
    };

    return {
      timestamp: Date.now(),
      heapUsed: memory.usedJSHeapSize / 1024 / 1024, // MB
      heapTotal: memory.totalJSHeapSize / 1024 / 1024, // MB
      heapLimit: memory.jsHeapSizeLimit / 1024 / 1024, // MB
      external: 0 // Would need Node.js process.memoryUsage() for this
    };
  }

  private analyzeMemoryTrend(): void {
    const recent = this.measurements.slice(-5); // Last 5 measurements
    const growthRate = this.calculateGrowthRate(recent);
    
    if (growthRate > 5) { // More than 5MB/hour growth
      console.warn('🚨 High memory growth detected:', {
        growthRate: `${growthRate.toFixed(2)} MB/hour`,
        currentUsage: `${recent[recent.length - 1].heapUsed.toFixed(2)} MB`,
        trend: this.measurements.length > 10 ? 'increasing' : 'unknown'
      });
    }
  }

  private calculateGrowthRate(measurements: MemoryMeasurement[]): number {
    if (measurements.length < 2) return 0;
    
    const first = measurements[0];
    const last = measurements[measurements.length - 1];
    const timeDiff = (last.timestamp - first.timestamp) / (1000 * 60 * 60); // hours
    const memoryDiff = last.heapUsed - first.heapUsed;
    
    return timeDiff > 0 ? memoryDiff / timeDiff : 0;
  }

  private generateReport(): MemoryProfileReport {
    return {
      duration: this.measurements.length > 0 ? 
        this.measurements[this.measurements.length - 1].timestamp - this.measurements[0].timestamp : 0,
      measurementCount: this.measurements.length,
      initialMemory: this.measurements[0]?.heapUsed || 0,
      finalMemory: this.measurements[this.measurements.length - 1]?.heapUsed || 0,
      peakMemory: Math.max(...this.measurements.map(m => m.heapUsed)),
      averageGrowthRate: this.calculateGrowthRate(this.measurements),
      leakSuspected: this.calculateGrowthRate(this.measurements) > 2,
      recommendations: this.generateRecommendations()
    };
  }

  private generateRecommendations(): string[] {
    const recommendations: string[] = [];
    const growthRate = this.calculateGrowthRate(this.measurements);
    
    if (growthRate > 10) {
      recommendations.push('Critical: Memory leak suspected - check for unmanaged subscriptions');
    } else if (growthRate > 5) {
      recommendations.push('Warning: High memory growth - review subscription management');
    }
    
    if (this.measurements.length > 0) {
      const peakMemory = Math.max(...this.measurements.map(m => m.heapUsed));
      if (peakMemory > 100) {
        recommendations.push('High memory usage detected - consider implementing pagination or virtual scrolling');
      }
    }
    
    return recommendations;
  }
}

interface MemoryProfileReport {
  duration: number;
  measurementCount: number;
  initialMemory: number;
  finalMemory: number;
  peakMemory: number;
  averageGrowthRate: number;
  leakSuspected: boolean;
  recommendations: string[];
}
```

### Automated Leak Detection

```typescript
@Injectable({
  providedIn: 'root'
})
export class AutomatedLeakDetector {
  private subscriptionTracker = new Map<string, SubscriptionTrackingInfo>();
  private leakThresholds = {
    maxSubscriptions: 100,
    maxAge: 300000, // 5 minutes
    maxGrowthRate: 5 // subscriptions per minute
  };

  trackSubscription(id: string, source: string, context?: any): void {
    const info: SubscriptionTrackingInfo = {
      id,
      source,
      createdAt: Date.now(),
      context: context ? this.serializeContext(context) : undefined,
      stackTrace: new Error().stack || 'No stack trace available'
    };
    
    this.subscriptionTracker.set(id, info);
    this.checkForLeaks();
  }

  untrackSubscription(id: string): void {
    this.subscriptionTracker.delete(id);
  }

  private checkForLeaks(): void {
    const now = Date.now();
    const activeSubscriptions = Array.from(this.subscriptionTracker.values());
    
    // Check for too many subscriptions
    if (activeSubscriptions.length > this.leakThresholds.maxSubscriptions) {
      this.reportLeak('SUBSCRIPTION_COUNT', {
        count: activeSubscriptions.length,
        threshold: this.leakThresholds.maxSubscriptions
      });
    }
    
    // Check for old subscriptions
    const oldSubscriptions = activeSubscriptions.filter(
      sub => (now - sub.createdAt) > this.leakThresholds.maxAge
    );
    
    if (oldSubscriptions.length > 0) {
      this.reportLeak('OLD_SUBSCRIPTIONS', {
        count: oldSubscriptions.length,
        oldestAge: Math.max(...oldSubscriptions.map(sub => now - sub.createdAt)),
        sources: oldSubscriptions.map(sub => sub.source)
      });
    }
    
    // Check growth rate
    this.checkGrowthRate();
  }

  private checkGrowthRate(): void {
    const now = Date.now();
    const recentSubscriptions = Array.from(this.subscriptionTracker.values())
      .filter(sub => (now - sub.createdAt) < 60000); // Last minute
    
    if (recentSubscriptions.length > this.leakThresholds.maxGrowthRate) {
      this.reportLeak('HIGH_GROWTH_RATE', {
        recentCount: recentSubscriptions.length,
        threshold: this.leakThresholds.maxGrowthRate,
        sources: [...new Set(recentSubscriptions.map(sub => sub.source))]
      });
    }
  }

  private reportLeak(type: string, details: any): void {
    console.error(`🚨 Memory Leak Detected: ${type}`, details);
    
    // In production, send to monitoring service
    this.sendToMonitoringService({
      type,
      details,
      timestamp: Date.now(),
      userAgent: navigator.userAgent,
      url: window.location.href
    });
  }

  private serializeContext(context: any): string {
    try {
      return JSON.stringify(context, null, 2);
    } catch {
      return context.toString();
    }
  }

  private sendToMonitoringService(leakReport: any): void {
    // Implementation would send to external monitoring service
    // For example: Sentry, LogRocket, custom analytics
  }

  getLeakReport(): LeakDetectionReport {
    const activeSubscriptions = Array.from(this.subscriptionTracker.values());
    const now = Date.now();
    
    return {
      totalSubscriptions: activeSubscriptions.length,
      oldSubscriptions: activeSubscriptions.filter(
        sub => (now - sub.createdAt) > this.leakThresholds.maxAge
      ).length,
      subscriptionsBySource: this.groupBySource(activeSubscriptions),
      suspiciousPatterns: this.identifySuspiciousPatterns(activeSubscriptions),
      recommendations: this.generateLeakRecommendations(activeSubscriptions)
    };
  }

  private groupBySource(subscriptions: SubscriptionTrackingInfo[]): Record<string, number> {
    return subscriptions.reduce((acc, sub) => {
      acc[sub.source] = (acc[sub.source] || 0) + 1;
      return acc;
    }, {} as Record<string, number>);
  }

  private identifySuspiciousPatterns(subscriptions: SubscriptionTrackingInfo[]): string[] {
    const patterns: string[] = [];
    const sourceGroups = this.groupBySource(subscriptions);
    
    // Look for sources with too many subscriptions
    Object.entries(sourceGroups).forEach(([source, count]) => {
      if (count > 10) {
        patterns.push(`High subscription count from ${source}: ${count}`);
      }
    });
    
    // Look for subscriptions created in tight loops
    const creationTimes = subscriptions.map(sub => sub.createdAt).sort();
    let rapidCreations = 0;
    for (let i = 1; i < creationTimes.length; i++) {
      if (creationTimes[i] - creationTimes[i-1] < 100) { // Less than 100ms apart
        rapidCreations++;
      }
    }
    
    if (rapidCreations > 10) {
      patterns.push(`Rapid subscription creation detected: ${rapidCreations} subscriptions created within 100ms of each other`);
    }
    
    return patterns;
  }

  private generateLeakRecommendations(subscriptions: SubscriptionTrackingInfo[]): string[] {
    const recommendations: string[] = [];
    const sourceGroups = this.groupBySource(subscriptions);
    
    // Recommendations based on subscription patterns
    if (subscriptions.length > 50) {
      recommendations.push('Consider using async pipe instead of manual subscriptions');
      recommendations.push('Implement subscription pooling for similar observables');
    }
    
    // Find sources with most subscriptions
    const topSources = Object.entries(sourceGroups)
      .sort(([,a], [,b]) => b - a)
      .slice(0, 3);
    
    topSources.forEach(([source, count]) => {
      if (count > 5) {
        recommendations.push(`Review subscription management in ${source} (${count} active subscriptions)`);
      }
    });
    
    return recommendations;
  }
}

interface SubscriptionTrackingInfo {
  id: string;
  source: string;
  createdAt: number;
  context?: string;
  stackTrace: string;
}

interface LeakDetectionReport {
  totalSubscriptions: number;
  oldSubscriptions: number;
  subscriptionsBySource: Record<string, number>;
  suspiciousPatterns: string[];
  recommendations: string[];
}
```

## Prevention Strategies {#prevention}

### Automatic Subscription Management

```typescript
// ✅ Base component with automatic subscription management
export abstract class AutoCleanupComponent implements OnDestroy {
  protected destroy$ = new Subject<void>();
  private subscriptionTracker = inject(AutomatedLeakDetector);
  private subscriptions: Subscription[] = [];

  ngOnDestroy(): void {
    // Automatic cleanup
    this.destroy$.next();
    this.destroy$.complete();
    
    // Unsubscribe all tracked subscriptions
    this.subscriptions.forEach(sub => {
      if (!sub.closed) {
        sub.unsubscribe();
      }
    });
    
    this.subscriptions = [];
  }

  // Helper method for automatic subscription tracking
  protected manageSubscription<T>(
    observable$: Observable<T>,
    observer: Partial<Observer<T>> | ((value: T) => void),
    context?: string
  ): void {
    const subscription = observable$.pipe(
      takeUntil(this.destroy$)
    ).subscribe(
      typeof observer === 'function' ? observer : observer
    );

    this.subscriptions.push(subscription);
    
    // Track for leak detection
    this.subscriptionTracker.trackSubscription(
      this.generateSubscriptionId(),
      this.constructor.name,
      context
    );
  }

  // Helper for conditional subscriptions
  protected manageConditionalSubscription<T>(
    condition$: Observable<boolean>,
    observable$: Observable<T>,
    observer: Partial<Observer<T>> | ((value: T) => void),
    context?: string
  ): void {
    this.manageSubscription(
      condition$.pipe(
        switchMap(condition => condition ? observable$ : EMPTY)
      ),
      observer,
      `conditional:${context}`
    );
  }

  private generateSubscriptionId(): string {
    return `${this.constructor.name}_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}

// Usage example
@Component({
  selector: 'app-managed',
  template: `<div>Automatically managed component</div>`
})
export class ManagedComponent extends AutoCleanupComponent implements OnInit {
  data: any[] = [];

  constructor(private dataService: DataService) {
    super();
  }

  ngOnInit() {
    // ✅ Automatically managed subscription
    this.manageSubscription(
      this.dataService.getData(),
      data => this.data = data,
      'initial-data-load'
    );

    // ✅ Conditional subscription
    this.manageConditionalSubscription(
      this.dataService.hasPermission$,
      this.dataService.getSensitiveData(),
      sensitiveData => this.processSensitiveData(sensitiveData),
      'sensitive-data-load'
    );
  }

  private processSensitiveData(data: any): void {
    // Process sensitive data
  }
}
```

### Memory-Efficient Observable Patterns

```typescript
class MemoryEfficientObservables {
  // ✅ Efficient data sharing with automatic cleanup
  createSharedObservable<T>(
    source$: Observable<T>,
    cacheKey: string,
    ttl: number = 300000 // 5 minutes
  ): Observable<T> {
    return new Observable<T>(observer => {
      const cached = this.getCachedObservable<T>(cacheKey);
      
      if (cached && this.isCacheValid(cacheKey, ttl)) {
        return cached.subscribe(observer);
      }
      
      const shared$ = source$.pipe(
        shareReplay(1),
        finalize(() => {
          // Clean up cache after TTL
          setTimeout(() => {
            this.clearCache(cacheKey);
          }, ttl);
        })
      );
      
      this.setCachedObservable(cacheKey, shared$);
      return shared$.subscribe(observer);
    });
  }

  // ✅ Memory-efficient infinite scroll
  createInfiniteScroll<T>(
    loadPage: (page: number) => Observable<T[]>,
    options: {
      pageSize?: number;
      maxItemsInMemory?: number;
      preloadPages?: number;
    } = {}
  ): Observable<InfiniteScrollResult<T>> {
    const { pageSize = 20, maxItemsInMemory = 1000, preloadPages = 2 } = options;
    
    return new Observable<InfiniteScrollResult<T>>(observer => {
      let allItems: T[] = [];
      let currentPage = 0;
      let isLoading = false;
      let hasMoreData = true;

      const loadNextPage = () => {
        if (isLoading || !hasMoreData) return;
        
        isLoading = true;
        
        loadPage(currentPage).subscribe({
          next: (newItems) => {
            if (newItems.length === 0) {
              hasMoreData = false;
            } else {
              allItems = [...allItems, ...newItems];
              
              // Memory management: keep only recent items
              if (allItems.length > maxItemsInMemory) {
                const itemsToRemove = allItems.length - maxItemsInMemory;
                allItems = allItems.slice(itemsToRemove);
              }
              
              currentPage++;
            }
            
            observer.next({
              items: allItems,
              currentPage,
              hasMoreData,
              totalLoaded: allItems.length
            });
            
            isLoading = false;
          },
          error: (error) => {
            isLoading = false;
            observer.error(error);
          }
        });
      };

      // Expose load function
      const result = {
        loadMore: loadNextPage,
        reset: () => {
          allItems = [];
          currentPage = 0;
          hasMoreData = true;
          isLoading = false;
        }
      };

      // Load initial page
      loadNextPage();

      return () => {
        // Cleanup
        allItems = [];
      };
    });
  }

  // ✅ Efficient stream composition with backpressure handling
  createBackpressureHandledStream<T>(
    source$: Observable<T>,
    processingFn: (item: T) => Observable<any>,
    options: {
      maxConcurrent?: number;
      bufferSize?: number;
      strategy?: 'drop' | 'latest' | 'buffer';
    } = {}
  ): Observable<any> {
    const { maxConcurrent = 3, bufferSize = 100, strategy = 'buffer' } = options;
    
    switch (strategy) {
      case 'drop':
        return source$.pipe(
          mergeMap(item => processingFn(item), maxConcurrent)
        );
        
      case 'latest':
        return source$.pipe(
          switchMap(item => processingFn(item))
        );
        
      case 'buffer':
      default:
        return source$.pipe(
          bufferCount(bufferSize),
          concatMap(batch => 
            from(batch).pipe(
              mergeMap(item => processingFn(item), maxConcurrent)
            )
          )
        );
    }
  }

  private cache = new Map<string, { observable: Observable<any>; timestamp: number }>();

  private getCachedObservable<T>(key: string): Observable<T> | null {
    return this.cache.get(key)?.observable || null;
  }

  private setCachedObservable<T>(key: string, observable: Observable<T>): void {
    this.cache.set(key, { observable, timestamp: Date.now() });
  }

  private isCacheValid(key: string, ttl: number): boolean {
    const cached = this.cache.get(key);
    return cached ? (Date.now() - cached.timestamp) < ttl : false;
  }

  private clearCache(key: string): void {
    this.cache.delete(key);
  }
}

interface InfiniteScrollResult<T> {
  items: T[];
  currentPage: number;
  hasMoreData: boolean;
  totalLoaded: number;
}
```

## Memory Leak Testing {#testing}

### Unit Testing for Memory Leaks

```typescript
describe('Memory Leak Prevention', () => {
  let component: TestComponent;
  let fixture: ComponentFixture<TestComponent>;
  let leakDetector: AutomatedLeakDetector;
  let memoryProfiler: MemoryProfiler;

  beforeEach(() => {
    TestBed.configureTestingModule({
      declarations: [TestComponent],
      providers: [AutomatedLeakDetector, MemoryProfiler]
    });

    fixture = TestBed.createComponent(TestComponent);
    component = fixture.componentInstance;
    leakDetector = TestBed.inject(AutomatedLeakDetector);
    memoryProfiler = TestBed.inject(MemoryProfiler);
  });

  it('should not leak subscriptions on component destruction', () => {
    // Arrange
    const initialSubscriptionCount = leakDetector.getLeakReport().totalSubscriptions;
    
    // Act
    component.ngOnInit();
    fixture.detectChanges();
    
    const afterInitCount = leakDetector.getLeakReport().totalSubscriptions;
    expect(afterInitCount).toBeGreaterThan(initialSubscriptionCount);
    
    // Destroy component
    component.ngOnDestroy();
    fixture.destroy();
    
    // Assert
    const finalCount = leakDetector.getLeakReport().totalSubscriptions;
    expect(finalCount).toBe(initialSubscriptionCount);
  });

  it('should handle memory efficiently during stress test', fakeAsync(() => {
    // Arrange
    memoryProfiler.startProfiling(100); // Profile every 100ms
    const stressOperations = 1000;
    
    // Act - Simulate stress conditions
    for (let i = 0; i < stressOperations; i++) {
      component.performOperation();
      tick(10);
    }
    
    tick(5000); // Let memory settle
    
    // Assert
    const report = memoryProfiler.stopProfiling();
    expect(report.leakSuspected).toBe(false);
    expect(report.averageGrowthRate).toBeLessThan(5); // Less than 5MB/hour
  }));

  it('should clean up circular references', () => {
    // Test for circular reference cleanup
    const parentService = TestBed.inject(CleanParentService);
    const childService = TestBed.inject(CleanChildService);
    
    // Create circular reference
    parentService.addChild(childService);
    expect(childService.getParent()).toBe(parentService);
    
    // Destroy child
    childService.ngOnDestroy();
    
    // Verify cleanup
    expect(childService.getParent()).toBeNull();
  });
});

// Integration test for memory leaks
describe('Memory Leak Integration Tests', () => {
  let app: AppComponent;
  let leakDetector: AutomatedLeakDetector;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [AppModule],
      providers: [
        { provide: AutomatedLeakDetector, useClass: MockLeakDetector }
      ]
    }).compileComponents();

    const fixture = TestBed.createComponent(AppComponent);
    app = fixture.componentInstance;
    leakDetector = TestBed.inject(AutomatedLeakDetector);
  });

  it('should not accumulate memory during navigation', fakeAsync(() => {
    const router = TestBed.inject(Router);
    const initialMemory = getMemoryUsage();
    
    // Navigate through multiple routes
    const routes = ['/home', '/products', '/profile', '/settings', '/home'];
    
    routes.forEach(route => {
      router.navigate([route]);
      tick(1000);
      fixture.detectChanges();
    });
    
    // Force garbage collection
    forceGarbageCollection();
    tick(2000);
    
    const finalMemory = getMemoryUsage();
    const memoryGrowth = finalMemory - initialMemory;
    
    // Memory growth should be minimal (less than 10MB)
    expect(memoryGrowth).toBeLessThan(10 * 1024 * 1024);
  }));
});

// Test utilities
function getMemoryUsage(): number {
  const memory = (performance as any).memory;
  return memory ? memory.usedJSHeapSize : 0;
}

function forceGarbageCollection(): void {
  // In test environment, simulate garbage collection
  if ((window as any).gc) {
    (window as any).gc();
  }
}

class MockLeakDetector extends AutomatedLeakDetector {
  // Override methods for testing
  trackSubscription(id: string, source: string, context?: any): void {
    super.trackSubscription(id, source, context);
    // Additional test-specific tracking
  }
}
```

## Monitoring & Alerting {#monitoring}

### Production Memory Monitoring

```typescript
@Injectable({
  providedIn: 'root'
})
export class ProductionMemoryMonitor {
  private readonly MEMORY_THRESHOLDS = {
    warning: 100, // MB
    critical: 200, // MB
    leak: 5 // MB growth per hour
  };

  private monitoringInterval?: number;
  private baselineMemory?: number;
  private measurements: MemoryMeasurement[] = [];

  startMonitoring(): void {
    if (this.monitoringInterval) return;
    
    this.baselineMemory = this.getCurrentMemoryUsage();
    this.monitoringInterval = window.setInterval(() => {
      this.checkMemoryUsage();
    }, 60000); // Check every minute
  }

  stopMonitoring(): void {
    if (this.monitoringInterval) {
      clearInterval(this.monitoringInterval);
      this.monitoringInterval = undefined;
    }
  }

  private checkMemoryUsage(): void {
    const currentMemory = this.getCurrentMemoryUsage();
    const measurement: MemoryMeasurement = {
      timestamp: Date.now(),
      heapUsed: currentMemory,
      heapTotal: this.getTotalMemoryUsage(),
      external: 0
    };

    this.measurements.push(measurement);
    
    // Keep only last hour of measurements
    const oneHourAgo = Date.now() - (60 * 60 * 1000);
    this.measurements = this.measurements.filter(m => m.timestamp > oneHourAgo);

    this.analyzeMemoryTrend(measurement);
  }

  private analyzeMemoryTrend(current: MemoryMeasurement): void {
    // Check absolute thresholds
    if (current.heapUsed > this.MEMORY_THRESHOLDS.critical) {
      this.sendAlert('CRITICAL_MEMORY_USAGE', {
        current: current.heapUsed,
        threshold: this.MEMORY_THRESHOLDS.critical,
        severity: 'critical'
      });
    } else if (current.heapUsed > this.MEMORY_THRESHOLDS.warning) {
      this.sendAlert('HIGH_MEMORY_USAGE', {
        current: current.heapUsed,
        threshold: this.MEMORY_THRESHOLDS.warning,
        severity: 'warning'
      });
    }

    // Check growth rate
    if (this.measurements.length >= 5) {
      const growthRate = this.calculateGrowthRate(this.measurements.slice(-5));
      if (growthRate > this.MEMORY_THRESHOLDS.leak) {
        this.sendAlert('MEMORY_LEAK_DETECTED', {
          growthRate,
          threshold: this.MEMORY_THRESHOLDS.leak,
          severity: 'critical',
          measurements: this.measurements.slice(-10)
        });
      }
    }
  }

  private getCurrentMemoryUsage(): number {
    const memory = (performance as any).memory;
    return memory ? memory.usedJSHeapSize / 1024 / 1024 : 0; // Convert to MB
  }

  private getTotalMemoryUsage(): number {
    const memory = (performance as any).memory;
    return memory ? memory.totalJSHeapSize / 1024 / 1024 : 0;
  }

  private calculateGrowthRate(measurements: MemoryMeasurement[]): number {
    if (measurements.length < 2) return 0;
    
    const first = measurements[0];
    const last = measurements[measurements.length - 1];
    const timeDiff = (last.timestamp - first.timestamp) / (1000 * 60 * 60); // hours
    const memoryDiff = last.heapUsed - first.heapUsed;
    
    return timeDiff > 0 ? memoryDiff / timeDiff : 0;
  }

  private sendAlert(type: string, details: any): void {
    console.error(`🚨 Memory Alert: ${type}`, details);
    
    // Send to monitoring service
    this.sendToMonitoringService({
      type,
      details,
      timestamp: Date.now(),
      userAgent: navigator.userAgent,
      url: window.location.href,
      sessionId: this.getSessionId()
    });

    // Show user notification for critical issues
    if (details.severity === 'critical') {
      this.showUserNotification(type, details);
    }
  }

  private sendToMonitoringService(alert: any): void {
    // Implementation would send to external service
    // Examples: Sentry, DataDog, New Relic, custom endpoint
    fetch('/api/monitoring/memory-alert', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(alert)
    }).catch(error => {
      console.error('Failed to send memory alert:', error);
    });
  }

  private showUserNotification(type: string, details: any): void {
    // Show user-friendly notification
    if ('Notification' in window && Notification.permission === 'granted') {
      new Notification('Application Performance Issue', {
        body: 'High memory usage detected. Please save your work and refresh the page.',
        icon: '/assets/warning-icon.png'
      });
    }
  }

  private getSessionId(): string {
    return sessionStorage.getItem('sessionId') || 'unknown';
  }
}

// Initialize monitoring in app module
@NgModule({
  // ... other configuration
})
export class AppModule {
  constructor(private memoryMonitor: ProductionMemoryMonitor) {
    // Start monitoring in production only
    if (environment.production) {
      this.memoryMonitor.startMonitoring();
    }
  }
}
```

## Recovery & Cleanup Strategies {#cleanup}

### Emergency Memory Cleanup

```typescript
@Injectable({
  providedIn: 'root'
})
export class EmergencyMemoryCleanup {
  private readonly CLEANUP_STRATEGIES = [
    { name: 'clearCaches', priority: 1, impact: 'low' },
    { name: 'unsubscribeOldSubscriptions', priority: 2, impact: 'medium' },
    { name: 'clearComponentCache', priority: 3, impact: 'medium' },
    { name: 'forceGarbageCollection', priority: 4, impact: 'high' },
    { name: 'reloadApplication', priority: 5, impact: 'critical' }
  ];

  private isCleanupInProgress = false;

  async performEmergencyCleanup(severity: 'low' | 'medium' | 'high' | 'critical'): Promise<CleanupResult> {
    if (this.isCleanupInProgress) {
      return { success: false, message: 'Cleanup already in progress' };
    }

    this.isCleanupInProgress = true;
    const startMemory = this.getCurrentMemoryUsage();
    const strategies = this.getStrategiesForSeverity(severity);
    const results: CleanupStrategyResult[] = [];

    try {
      for (const strategy of strategies) {
        const memoryBefore = this.getCurrentMemoryUsage();
        
        try {
          await this.executeStrategy(strategy.name);
          
          const memoryAfter = this.getCurrentMemoryUsage();
          const memoryFreed = memoryBefore - memoryAfter;
          
          results.push({
            strategy: strategy.name,
            success: true,
            memoryFreed,
            impact: strategy.impact
          });

          // If we've freed enough memory, stop here
          if (memoryFreed > 50 && severity !== 'critical') {
            break;
          }
        } catch (error) {
          results.push({
            strategy: strategy.name,
            success: false,
            error: error.message,
            impact: strategy.impact
          });
        }
      }

      const finalMemory = this.getCurrentMemoryUsage();
      const totalMemoryFreed = startMemory - finalMemory;

      return {
        success: true,
        memoryFreed: totalMemoryFreed,
        strategies: results,
        message: `Cleanup completed. Freed ${totalMemoryFreed.toFixed(2)} MB`
      };
    } finally {
      this.isCleanupInProgress = false;
    }
  }

  private getStrategiesForSeverity(severity: string) {
    const maxPriority = {
      'low': 2,
      'medium': 3,
      'high': 4,
      'critical': 5
    }[severity] || 2;

    return this.CLEANUP_STRATEGIES.filter(s => s.priority <= maxPriority);
  }

  private async executeStrategy(strategyName: string): Promise<void> {
    switch (strategyName) {
      case 'clearCaches':
        await this.clearCaches();
        break;
      case 'unsubscribeOldSubscriptions':
        await this.unsubscribeOldSubscriptions();
        break;
      case 'clearComponentCache':
        await this.clearComponentCache();
        break;
      case 'forceGarbageCollection':
        await this.forceGarbageCollection();
        break;
      case 'reloadApplication':
        await this.reloadApplication();
        break;
      default:
        throw new Error(`Unknown cleanup strategy: ${strategyName}`);
    }
  }

  private async clearCaches(): Promise<void> {
    // Clear HTTP cache
    if ('caches' in window) {
      const cacheNames = await caches.keys();
      await Promise.all(cacheNames.map(name => caches.delete(name)));
    }

    // Clear application-specific caches
    const cacheService = this.getCacheService();
    if (cacheService) {
      cacheService.clearAll();
    }
  }

  private async unsubscribeOldSubscriptions(): Promise<void> {
    const leakDetector = this.getLeakDetector();
    if (leakDetector) {
      const report = leakDetector.getLeakReport();
      // Force cleanup of old subscriptions
      // Implementation would vary based on leak detector
    }
  }

  private async clearComponentCache(): Promise<void> {
    // Clear Angular component cache if available
    // This would depend on custom caching implementation
  }

  private async forceGarbageCollection(): Promise<void> {
    // Request garbage collection if available
    if ((window as any).gc) {
      (window as any).gc();
    }
    
    // Wait for GC to complete
    await new Promise(resolve => setTimeout(resolve, 1000));
  }

  private async reloadApplication(): Promise<void> {
    // Save critical state before reload
    this.saveApplicationState();
    
    // Reload the page
    window.location.reload();
  }

  private getCurrentMemoryUsage(): number {
    const memory = (performance as any).memory;
    return memory ? memory.usedJSHeapSize / 1024 / 1024 : 0;
  }

  private getCacheService(): any {
    // Get application cache service
    return null; // Implementation would inject actual cache service
  }

  private getLeakDetector(): AutomatedLeakDetector | null {
    // Get leak detector service
    return null; // Implementation would inject actual leak detector
  }

  private saveApplicationState(): void {
    // Save critical application state to localStorage
    // before emergency reload
  }
}

interface CleanupResult {
  success: boolean;
  memoryFreed?: number;
  strategies?: CleanupStrategyResult[];
  message: string;
}

interface CleanupStrategyResult {
  strategy: string;
  success: boolean;
  memoryFreed?: number;
  error?: string;
  impact: string;
}
```

## Tools & Libraries {#tools}

### Recommended Memory Monitoring Tools

```typescript
// Integration with popular monitoring libraries
export class MemoryMonitoringIntegration {
  
  // Sentry integration
  static setupSentryMemoryMonitoring(): void {
    if (typeof window !== 'undefined' && (window as any).Sentry) {
      const Sentry = (window as any).Sentry;
      
      // Add memory context to all events
      Sentry.configureScope((scope: any) => {
        setInterval(() => {
          const memory = (performance as any).memory;
          if (memory) {
            scope.setContext('memory', {
              usedJSHeapSize: memory.usedJSHeapSize,
              totalJSHeapSize: memory.totalJSHeapSize,
              jsHeapSizeLimit: memory.jsHeapSizeLimit
            });
          }
        }, 30000); // Update every 30 seconds
      });
    }
  }

  // LogRocket integration
  static setupLogRocketMemoryMonitoring(): void {
    if (typeof window !== 'undefined' && (window as any).LogRocket) {
      const LogRocket = (window as any).LogRocket;
      
      setInterval(() => {
        const memory = (performance as any).memory;
        if (memory && memory.usedJSHeapSize > 100 * 1024 * 1024) { // 100MB
          LogRocket.track('High Memory Usage', {
            memoryUsage: memory.usedJSHeapSize / 1024 / 1024,
            totalMemory: memory.totalJSHeapSize / 1024 / 1024
          });
        }
      }, 60000); // Check every minute
    }
  }

  // Custom analytics integration
  static setupCustomAnalytics(analyticsEndpoint: string): void {
    const memoryProfiler = new MemoryProfiler();
    memoryProfiler.startProfiling(60000); // Profile every minute
    
    setInterval(() => {
      const report = memoryProfiler.stopProfiling();
      memoryProfiler.startProfiling(60000); // Restart profiling
      
      if (report.leakSuspected) {
        fetch(analyticsEndpoint, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            type: 'memory_leak_detected',
            report,
            timestamp: Date.now(),
            userAgent: navigator.userAgent,
            url: window.location.href
          })
        }).catch(console.error);
      }
    }, 300000); // Report every 5 minutes
  }
}

// Chrome DevTools Performance API integration
export class DevToolsMemoryProfiler {
  static async captureHeapSnapshot(): Promise<any> {
    if (!('memory' in performance)) {
      throw new Error('Memory API not available');
    }

    // This would integrate with Chrome DevTools Protocol
    // In a real implementation, you'd use puppeteer or similar
    return {
      timestamp: Date.now(),
      heapSize: (performance as any).memory.usedJSHeapSize,
      // Additional heap analysis would go here
    };
  }

  static measureMemoryDuringTest(testFn: () => Promise<void>): Promise<MemoryTestResult> {
    return new Promise(async (resolve) => {
      const startMemory = (performance as any).memory?.usedJSHeapSize || 0;
      const startTime = performance.now();
      
      await testFn();
      
      // Wait for potential garbage collection
      setTimeout(() => {
        const endMemory = (performance as any).memory?.usedJSHeapSize || 0;
        const endTime = performance.now();
        
        resolve({
          startMemory: startMemory / 1024 / 1024, // MB
          endMemory: endMemory / 1024 / 1024, // MB
          memoryDelta: (endMemory - startMemory) / 1024 / 1024, // MB
          duration: endTime - startTime, // ms
          leakSuspected: (endMemory - startMemory) > 10 * 1024 * 1024 // 10MB threshold
        });
      }, 2000);
    });
  }
}

interface MemoryTestResult {
  startMemory: number;
  endMemory: number;
  memoryDelta: number;
  duration: number;
  leakSuspected: boolean;
}
```

## Exercises {#exercises}

### Exercise 1: Memory Leak Detection System

Build a comprehensive memory leak detection system that:
- Automatically tracks all RxJS subscriptions
- Identifies patterns indicating potential leaks
- Provides actionable recommendations
- Integrates with existing monitoring tools

### Exercise 2: Automated Memory Testing

Create an automated testing framework that:
- Tests components for memory leaks
- Simulates user interactions over time
- Measures memory usage patterns
- Generates detailed leak reports

### Exercise 3: Emergency Cleanup System

Implement an emergency memory cleanup system that:
- Detects critical memory situations
- Automatically performs cleanup operations
- Preserves user state during cleanup
- Provides user feedback during the process

### Exercise 4: Production Memory Monitoring

Build a production-ready memory monitoring solution that:
- Continuously monitors memory usage
- Sends alerts when thresholds are exceeded
- Provides historical memory usage data
- Integrates with popular monitoring platforms

---

**Next Steps:**
- Explore RxJS bundle size optimization techniques
- Learn about browser compatibility considerations
- Master advanced debugging strategies for complex memory issues
- Implement comprehensive monitoring in production applications
