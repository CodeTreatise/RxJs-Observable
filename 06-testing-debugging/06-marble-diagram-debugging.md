# Using Marble Diagrams for Debugging

## Table of Contents
1. [Introduction to Marble Diagram Debugging](#introduction)
2. [Visual Debugging Techniques](#visual-debugging)
3. [Interactive Marble Diagram Tools](#interactive-tools)
4. [Debugging Complex Stream Flows](#complex-streams)
5. [Angular-Specific Debugging](#angular-debugging)
6. [Real-Time Stream Visualization](#real-time-viz)
7. [Automated Diagram Generation](#automated-generation)
8. [Best Practices](#best-practices)
9. [Exercises](#exercises)

## Introduction to Marble Diagram Debugging {#introduction}

Marble diagrams are powerful visual tools for understanding and debugging RxJS streams, providing immediate insight into timing, values, and operator behavior.

### Why Marble Diagrams for Debugging?

```typescript
// Instead of this unclear debugging:
console.log('Stream started');
source$.pipe(
  tap(x => console.log('Before map:', x)),
  map(x => x * 2),
  tap(x => console.log('After map:', x)),
  filter(x => x > 10),
  tap(x => console.log('After filter:', x))
).subscribe(x => console.log('Final:', x));

// Use marble diagrams for visual clarity:
/*
Input:   --1--2--3--4--5--6--|
Map(*2): --2--4--6--8--10-12-|
Filter:  -----6--8--10-12-|
Output:  -----6--8--10-12-|
*/
```

### Marble Diagram Debugging Utility

```typescript
interface MarbleFrame {
  time: number;
  type: 'next' | 'error' | 'complete';
  value?: any;
  error?: any;
}

interface MarbleTrace {
  operatorName: string;
  input: MarbleFrame[];
  output: MarbleFrame[];
  duration: number;
}

class MarbleDiagramDebugger {
  private traces: Map<string, MarbleTrace> = new Map();
  private scheduler = new TestScheduler((actual, expected) => {
    console.log('Marble comparison:', { actual, expected });
  });

  debugStream<T>(
    source$: Observable<T>,
    label: string,
    options?: {
      showValues?: boolean;
      timeScale?: number;
      maxFrames?: number;
    }
  ): Observable<T> {
    const frames: MarbleFrame[] = [];
    const startTime = this.scheduler.now();

    return new Observable<T>(observer => {
      const subscription = source$.subscribe({
        next: (value) => {
          frames.push({
            time: this.scheduler.now() - startTime,
            type: 'next',
            value
          });
          
          this.logMarbleFrame(label, 'next', value);
          observer.next(value);
        },
        error: (error) => {
          frames.push({
            time: this.scheduler.now() - startTime,
            type: 'error',
            error
          });
          
          this.logMarbleFrame(label, 'error', error);
          observer.error(error);
        },
        complete: () => {
          frames.push({
            time: this.scheduler.now() - startTime,
            type: 'complete'
          });
          
          this.logMarbleFrame(label, 'complete');
          this.generateMarbleDiagram(label, frames, options);
          observer.complete();
        }
      });

      return () => {
        subscription.unsubscribe();
        this.generateMarbleDiagram(label, frames, options);
      };
    });
  }

  private logMarbleFrame(label: string, type: string, value?: any): void {
    const timestamp = new Date().toISOString().substr(11, 12);
    const valueStr = value !== undefined ? ` (${JSON.stringify(value)})` : '';
    console.log(`🔍 [${timestamp}] ${label}: ${type}${valueStr}`);
  }

  private generateMarbleDiagram(
    label: string,
    frames: MarbleFrame[],
    options?: any
  ): void {
    const diagram = this.framesToMarbleDiagram(frames, options);
    console.group(`📊 Marble Diagram: ${label}`);
    console.log(diagram);
    console.groupEnd();
  }

  private framesToMarbleDiagram(
    frames: MarbleFrame[],
    options?: any
  ): string {
    if (frames.length === 0) return '(empty)';

    const timeScale = options?.timeScale || 10;
    const maxTime = Math.max(...frames.map(f => f.time));
    const diagramLength = Math.ceil(maxTime / timeScale);
    
    let diagram = '';
    let currentPos = 0;

    for (const frame of frames) {
      const targetPos = Math.floor(frame.time / timeScale);
      
      // Add dashes for time gaps
      while (currentPos < targetPos) {
        diagram += '-';
        currentPos++;
      }

      // Add frame symbol
      switch (frame.type) {
        case 'next':
          const valueStr = options?.showValues && frame.value !== undefined
            ? String(frame.value)
            : 'x';
          diagram += valueStr;
          break;
        case 'error':
          diagram += '#';
          break;
        case 'complete':
          diagram += '|';
          break;
      }
      currentPos++;
    }

    return diagram;
  }

  compareStreams<T>(
    expected: string,
    actual$: Observable<T>,
    label: string
  ): void {
    this.scheduler.run(helpers => {
      const { hot, expectObservable } = helpers;
      
      console.group(`🔄 Stream Comparison: ${label}`);
      console.log('Expected:', expected);
      
      expectObservable(actual$).toBe(expected);
      
      console.groupEnd();
    });
  }

  generateInteractiveDiagram(
    streams: { [key: string]: Observable<any> }
  ): void {
    console.group('🎮 Interactive Marble Diagram');
    
    Object.entries(streams).forEach(([name, stream$]) => {
      console.log(`Stream: ${name}`);
      this.debugStream(stream$, name, { showValues: true });
    });
    
    console.groupEnd();
  }
}
```

## Visual Debugging Techniques {#visual-debugging}

### Operator Chain Visualization

```typescript
class OperatorChainVisualizer {
  private debugger = new MarbleDiagramDebugger();

  visualizeChain<T>(
    source$: Observable<T>,
    operators: { name: string; fn: any }[]
  ): Observable<T> {
    console.group('🔗 Operator Chain Visualization');
    
    let current$ = this.debugger.debugStream(source$, 'Source');
    
    operators.forEach((op, index) => {
      current$ = current$.pipe(op.fn);
      current$ = this.debugger.debugStream(
        current$,
        `${index + 1}. ${op.name}`
      );
    });
    
    console.groupEnd();
    return current$;
  }

  createVisualPipeline<T>(source$: Observable<T>): VisualPipeline<T> {
    return new VisualPipeline(source$, this.debugger);
  }
}

class VisualPipeline<T> {
  private current$: Observable<T>;
  private stepCount = 0;

  constructor(
    source$: Observable<T>,
    private debugger: MarbleDiagramDebugger
  ) {
    this.current$ = this.debugger.debugStream(source$, 'Input');
  }

  map<R>(fn: (value: T) => R, label?: string): VisualPipeline<R> {
    this.stepCount++;
    const stepLabel = label || `Step ${this.stepCount}: map`;
    
    const mapped$ = this.current$.pipe(map(fn));
    const debugged$ = this.debugger.debugStream(mapped$, stepLabel);
    
    return new VisualPipeline(debugged$, this.debugger);
  }

  filter(predicate: (value: T) => boolean, label?: string): VisualPipeline<T> {
    this.stepCount++;
    const stepLabel = label || `Step ${this.stepCount}: filter`;
    
    const filtered$ = this.current$.pipe(filter(predicate));
    const debugged$ = this.debugger.debugStream(filtered$, stepLabel);
    
    return new VisualPipeline(debugged$, this.debugger);
  }

  debounceTime(dueTime: number, label?: string): VisualPipeline<T> {
    this.stepCount++;
    const stepLabel = label || `Step ${this.stepCount}: debounceTime(${dueTime})`;
    
    const debounced$ = this.current$.pipe(debounceTime(dueTime));
    const debugged$ = this.debugger.debugStream(debounced$, stepLabel);
    
    return new VisualPipeline(debugged$, this.debugger);
  }

  build(): Observable<T> {
    return this.debugger.debugStream(this.current$, 'Final Output');
  }
}

// Usage Example
const visualizer = new OperatorChainVisualizer();

const searchResults$ = visualizer
  .createVisualPipeline(fromEvent(searchInput, 'input'))
  .map((event: any) => event.target.value, 'Extract Search Term')
  .filter(term => term.length > 2, 'Filter Short Terms')
  .debounceTime(300, 'Debounce User Input')
  .map(term => searchAPI(term), 'API Call')
  .build();
```

### Multi-Stream Debugging

```typescript
class MultiStreamDebugger {
  private debugger = new MarbleDiagramDebugger();

  debugCombination<T1, T2, R>(
    stream1$: Observable<T1>,
    stream2$: Observable<T2>,
    combinator: (v1: T1, v2: T2) => R,
    combinatorName: string
  ): Observable<R> {
    console.group(`🔀 Multi-Stream Debug: ${combinatorName}`);

    const debuggedStream1$ = this.debugger.debugStream(stream1$, 'Stream A');
    const debuggedStream2$ = this.debugger.debugStream(stream2$, 'Stream B');

    let result$: Observable<R>;

    switch (combinatorName) {
      case 'combineLatest':
        result$ = combineLatest([debuggedStream1$, debuggedStream2$]).pipe(
          map(([v1, v2]) => combinator(v1, v2))
        );
        break;
      case 'merge':
        result$ = merge(debuggedStream1$, debuggedStream2$) as any;
        break;
      case 'concat':
        result$ = concat(debuggedStream1$, debuggedStream2$) as any;
        break;
      case 'zip':
        result$ = zip(debuggedStream1$, debuggedStream2$).pipe(
          map(([v1, v2]) => combinator(v1, v2))
        );
        break;
      default:
        throw new Error(`Unknown combinator: ${combinatorName}`);
    }

    const finalResult$ = this.debugger.debugStream(result$, 'Combined Result');
    console.groupEnd();
    
    return finalResult$;
  }

  visualizeRace<T>(...streams: Observable<T>[]): Observable<T> {
    console.group('🏁 Race Visualization');
    
    const debuggedStreams = streams.map((stream$, index) =>
      this.debugger.debugStream(stream$, `Competitor ${index + 1}`)
    );

    const result$ = race(...debuggedStreams);
    const finalResult$ = this.debugger.debugStream(result$, 'Race Winner');
    
    console.groupEnd();
    return finalResult$;
  }
}
```

## Interactive Marble Diagram Tools {#interactive-tools}

### Browser-Based Marble Diagram Viewer

```typescript
class InteractiveMarbleDiagramViewer {
  private container: HTMLElement;
  private streamData: Map<string, any[]> = new Map();

  constructor(containerId: string) {
    this.container = document.getElementById(containerId)!;
    this.setupStyles();
  }

  addStream<T>(
    stream$: Observable<T>,
    label: string,
    color: string = '#007acc'
  ): void {
    const streamElement = this.createStreamElement(label, color);
    this.container.appendChild(streamElement);

    const marbleContainer = streamElement.querySelector('.marble-container')!;
    const values: any[] = [];

    stream$.pipe(
      tap(value => {
        values.push({ value, time: Date.now(), type: 'next' });
        this.addMarble(marbleContainer, value, 'next', color);
      }),
      catchError(error => {
        values.push({ error, time: Date.now(), type: 'error' });
        this.addMarble(marbleContainer, error, 'error', '#ff4444');
        return EMPTY;
      }),
      finalize(() => {
        values.push({ time: Date.now(), type: 'complete' });
        this.addMarble(marbleContainer, null, 'complete', '#44ff44');
        this.streamData.set(label, values);
      })
    ).subscribe();
  }

  private createStreamElement(label: string, color: string): HTMLElement {
    const streamDiv = document.createElement('div');
    streamDiv.className = 'marble-stream';
    streamDiv.innerHTML = `
      <div class="stream-label" style="color: ${color}">${label}</div>
      <div class="marble-container"></div>
      <div class="stream-controls">
        <button onclick="this.parentElement.parentElement.style.display='none'">
          Hide
        </button>
        <button onclick="navigator.clipboard.writeText(this.dataset.values)">
          Copy Values
        </button>
      </div>
    `;
    return streamDiv;
  }

  private addMarble(
    container: Element,
    value: any,
    type: 'next' | 'error' | 'complete',
    color: string
  ): void {
    const marble = document.createElement('div');
    marble.className = `marble marble-${type}`;
    marble.style.backgroundColor = color;
    
    if (type === 'next') {
      marble.textContent = String(value);
      marble.title = `Value: ${JSON.stringify(value)}`;
    } else if (type === 'error') {
      marble.textContent = '✗';
      marble.title = `Error: ${value}`;
    } else {
      marble.textContent = '|';
      marble.title = 'Complete';
    }

    container.appendChild(marble);

    // Animate marble appearance
    marble.style.transform = 'scale(0)';
    setTimeout(() => {
      marble.style.transform = 'scale(1)';
    }, 10);
  }

  private setupStyles(): void {
    const styles = `
      .marble-stream {
        margin: 10px 0;
        padding: 10px;
        border: 1px solid #ddd;
        border-radius: 5px;
        background: #f9f9f9;
      }
      
      .stream-label {
        font-weight: bold;
        margin-bottom: 5px;
      }
      
      .marble-container {
        display: flex;
        align-items: center;
        min-height: 40px;
        background: white;
        border: 1px solid #ccc;
        border-radius: 3px;
        padding: 5px;
        overflow-x: auto;
      }
      
      .marble {
        width: 30px;
        height: 30px;
        border-radius: 50%;
        display: flex;
        align-items: center;
        justify-content: center;
        margin: 0 5px;
        color: white;
        font-weight: bold;
        font-size: 12px;
        transition: transform 0.2s ease;
        cursor: pointer;
      }
      
      .marble-complete {
        border-radius: 0;
        width: 2px;
        height: 30px;
      }
      
      .stream-controls {
        margin-top: 5px;
      }
      
      .stream-controls button {
        margin-right: 5px;
        padding: 2px 8px;
        font-size: 11px;
      }
    `;

    const styleSheet = document.createElement('style');
    styleSheet.textContent = styles;
    document.head.appendChild(styleSheet);
  }

  exportDiagram(): string {
    const diagram = Array.from(this.streamData.entries())
      .map(([label, values]) => {
        const marbleString = values
          .map(v => {
            if (v.type === 'next') return String(v.value);
            if (v.type === 'error') return '#';
            return '|';
          })
          .join('--');
        
        return `${label}: --${marbleString}`;
      })
      .join('\n');

    return diagram;
  }

  clear(): void {
    this.container.innerHTML = '';
    this.streamData.clear();
  }
}
```

### Real-Time Marble Diagram Component

```typescript
@Component({
  selector: 'app-marble-debugger',
  template: `
    <div class="marble-debugger">
      <h3>Real-Time Marble Diagram Debugger</h3>
      
      <div class="controls">
        <button (click)="startDebugging()">Start Debugging</button>
        <button (click)="stopDebugging()">Stop</button>
        <button (click)="clearDiagrams()">Clear</button>
        <button (click)="exportDiagrams()">Export</button>
      </div>

      <div class="stream-selector">
        <label>
          <input type="checkbox" [(ngModel)]="debugOptions.showTimestamps">
          Show Timestamps
        </label>
        <label>
          <input type="checkbox" [(ngModel)]="debugOptions.showValues">
          Show Values
        </label>
        <label>
          Time Scale:
          <input type="range" min="1" max="100" 
                 [(ngModel)]="debugOptions.timeScale">
        </label>
      </div>

      <div id="marble-container" class="marble-container"></div>
    </div>
  `,
  styles: [`
    .marble-debugger {
      padding: 20px;
      border: 1px solid #ddd;
      border-radius: 8px;
      background: #fafafa;
    }
    
    .controls {
      margin-bottom: 15px;
    }
    
    .controls button {
      margin-right: 10px;
      padding: 8px 16px;
      border: none;
      border-radius: 4px;
      background: #007acc;
      color: white;
      cursor: pointer;
    }
    
    .controls button:hover {
      background: #005a9e;
    }
    
    .stream-selector {
      margin-bottom: 15px;
    }
    
    .stream-selector label {
      margin-right: 15px;
      display: inline-flex;
      align-items: center;
    }
    
    .marble-container {
      min-height: 200px;
      border: 1px solid #ccc;
      border-radius: 4px;
      background: white;
      padding: 10px;
    }
  `]
})
export class MarbleDebuggerComponent implements OnInit, OnDestroy {
  debugOptions = {
    showTimestamps: true,
    showValues: true,
    timeScale: 10
  };

  private viewer!: InteractiveMarbleDiagramViewer;
  private debugger = new MarbleDiagramDebugger();
  private isDebugging = false;
  private destroy$ = new Subject<void>();

  constructor(private dataService: DataService) {}

  ngOnInit() {
    this.viewer = new InteractiveMarbleDiagramViewer('marble-container');
  }

  startDebugging() {
    if (this.isDebugging) return;
    
    this.isDebugging = true;
    
    // Debug multiple streams
    this.debugUserActions();
    this.debugApiCalls();
    this.debugFormValidation();
  }

  private debugUserActions() {
    const clicks$ = fromEvent(document, 'click').pipe(
      map(() => 'click'),
      takeUntil(this.destroy$)
    );
    
    this.viewer.addStream(clicks$, 'User Clicks', '#ff6b6b');
  }

  private debugApiCalls() {
    const apiCalls$ = interval(2000).pipe(
      switchMap(() => this.dataService.getData()),
      map(data => `API: ${data?.id || 'error'}`),
      takeUntil(this.destroy$)
    );
    
    this.viewer.addStream(apiCalls$, 'API Calls', '#4ecdc4');
  }

  private debugFormValidation() {
    const validation$ = interval(1000).pipe(
      map(() => Math.random() > 0.7 ? 'valid' : 'invalid'),
      distinctUntilChanged(),
      takeUntil(this.destroy$)
    );
    
    this.viewer.addStream(validation$, 'Form Validation', '#45b7d1');
  }

  stopDebugging() {
    this.isDebugging = false;
    this.destroy$.next();
  }

  clearDiagrams() {
    this.viewer.clear();
  }

  exportDiagrams() {
    const diagram = this.viewer.exportDiagram();
    navigator.clipboard.writeText(diagram).then(() => {
      alert('Marble diagram copied to clipboard!');
    });
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## Debugging Complex Stream Flows {#complex-streams}

### Higher-Order Observable Debugging

```typescript
class HigherOrderStreamDebugger {
  debugFlatMap<T, R>(
    source$: Observable<T>,
    projection: (value: T) => Observable<R>,
    strategy: 'switchMap' | 'mergeMap' | 'concatMap' | 'exhaustMap'
  ): Observable<R> {
    console.group(`🔄 ${strategy} Debug Session`);
    
    let outerIndex = 0;
    let innerStreamCount = 0;

    const debuggedSource$ = source$.pipe(
      tap(value => {
        console.log(`🟦 Outer[${outerIndex++}]:`, value);
      })
    );

    let operator: any;
    switch (strategy) {
      case 'switchMap':
        operator = switchMap;
        break;
      case 'mergeMap':
        operator = mergeMap;
        break;
      case 'concatMap':
        operator = concatMap;
        break;
      case 'exhaustMap':
        operator = exhaustMap;
        break;
    }

    const result$ = debuggedSource$.pipe(
      operator((value: T, index: number) => {
        const innerStreamId = ++innerStreamCount;
        console.log(`🟨 Inner[${innerStreamId}] created from outer[${index}]`);
        
        return projection(value).pipe(
          tap(innerValue => {
            console.log(`🟩 Inner[${innerStreamId}] emitted:`, innerValue);
          }),
          finalize(() => {
            console.log(`🟥 Inner[${innerStreamId}] completed`);
          })
        );
      }),
      tap(finalValue => {
        console.log(`🟪 Final output:`, finalValue);
      }),
      finalize(() => {
        console.log(`📊 Total inner streams created: ${innerStreamCount}`);
        console.groupEnd();
      })
    );

    return result$;
  }

  visualizeStreamTree<T>(
    source$: Observable<T>,
    label: string = 'Stream'
  ): StreamTreeNode<T> {
    return new StreamTreeNode(source$, label);
  }
}

class StreamTreeNode<T> {
  private children: StreamTreeNode<any>[] = [];
  private values: T[] = [];

  constructor(
    private stream$: Observable<T>,
    private label: string
  ) {}

  switchMap<R>(
    projection: (value: T) => Observable<R>,
    childLabel?: string
  ): StreamTreeNode<R> {
    const child$ = this.stream$.pipe(
      switchMap((value, index) => {
        const label = childLabel || `Switch[${index}]`;
        console.log(`🔀 ${this.label} -> ${label}:`, value);
        return projection(value);
      })
    );

    const childNode = new StreamTreeNode(child$, childLabel || 'SwitchMap');
    this.children.push(childNode);
    return childNode;
  }

  mergeMap<R>(
    projection: (value: T) => Observable<R>,
    childLabel?: string
  ): StreamTreeNode<R> {
    const child$ = this.stream$.pipe(
      mergeMap((value, index) => {
        const label = childLabel || `Merge[${index}]`;
        console.log(`⚡ ${this.label} -> ${label}:`, value);
        return projection(value);
      })
    );

    const childNode = new StreamTreeNode(child$, childLabel || 'MergeMap');
    this.children.push(childNode);
    return childNode;
  }

  subscribe(): Subscription {
    return this.stream$.subscribe({
      next: value => {
        this.values.push(value);
        console.log(`📥 ${this.label}:`, value);
      },
      error: error => console.error(`❌ ${this.label}:`, error),
      complete: () => {
        console.log(`✅ ${this.label} completed`);
        this.printTree();
      }
    });
  }

  private printTree(depth: number = 0): void {
    const indent = '  '.repeat(depth);
    console.log(`${indent}${this.label}: [${this.values.join(', ')}]`);
    
    this.children.forEach(child => {
      child.printTree(depth + 1);
    });
  }
}
```

### Error Flow Debugging

```typescript
class ErrorFlowDebugger {
  debugErrorHandling<T>(
    source$: Observable<T>,
    errorHandlers: {
      [key: string]: (error: any) => Observable<T>;
    }
  ): Observable<T> {
    console.group('🚨 Error Flow Debug');
    
    return source$.pipe(
      tap(value => console.log('✅ Success:', value)),
      catchError(error => {
        console.log('❌ Error caught:', error);
        console.log('🔍 Error type:', error.constructor.name);
        console.log('📍 Error message:', error.message);
        
        // Try different error handlers
        for (const [handlerName, handler] of Object.entries(errorHandlers)) {
          console.log(`🛠️ Trying handler: ${handlerName}`);
          try {
            return handler(error).pipe(
              tap(recoveredValue => {
                console.log(`✨ Recovered with ${handlerName}:`, recoveredValue);
              })
            );
          } catch (handlerError) {
            console.log(`❌ Handler ${handlerName} failed:`, handlerError);
          }
        }
        
        console.log('💥 All handlers failed, re-throwing error');
        console.groupEnd();
        return throwError(() => error);
      }),
      finalize(() => {
        console.groupEnd();
      })
    );
  }

  createErrorVisualization(
    streams: Observable<any>[],
    labels: string[]
  ): Observable<any> {
    console.group('🎯 Error Pattern Visualization');
    
    return merge(
      ...streams.map((stream$, index) =>
        stream$.pipe(
          map(value => ({ source: labels[index], value, type: 'success' })),
          catchError(error => 
            of({ source: labels[index], error, type: 'error' })
          )
        )
      )
    ).pipe(
      tap(event => {
        const icon = event.type === 'success' ? '✅' : '❌';
        const data = event.type === 'success' ? event.value : event.error;
        console.log(`${icon} [${event.source}]:`, data);
      }),
      finalize(() => {
        console.groupEnd();
      })
    );
  }
}
```

## Angular-Specific Debugging {#angular-debugging}

### Component Lifecycle Stream Debugging

```typescript
@Component({
  selector: 'app-debug-lifecycle',
  template: `
    <div class="lifecycle-debug">
      <h3>Component Lifecycle Debugging</h3>
      <div id="lifecycle-diagram"></div>
      <button (click)="triggerChange()">Trigger Change</button>
    </div>
  `
})
export class DebugLifecycleComponent implements OnInit, OnDestroy {
  private lifecycleDebugger = new MarbleDiagramDebugger();
  private viewer!: InteractiveMarbleDiagramViewer;
  private changeCounter = 0;

  ngOnInit() {
    this.viewer = new InteractiveMarbleDiagramViewer('lifecycle-diagram');
    this.debugComponentLifecycle();
  }

  private debugComponentLifecycle() {
    // Debug NgOnInit
    const onInit$ = of('OnInit').pipe(delay(0));
    this.viewer.addStream(onInit$, 'NgOnInit', '#4CAF50');

    // Debug change detection
    const changeDetection$ = new Subject<string>();
    this.viewer.addStream(changeDetection$, 'Change Detection', '#FF9800');

    // Debug user interactions
    const interactions$ = new Subject<string>();
    this.viewer.addStream(interactions$, 'User Interactions', '#2196F3');

    // Simulate change detection triggers
    setTimeout(() => changeDetection$.next('CD-1'), 100);
    setTimeout(() => changeDetection$.next('CD-2'), 500);
    setTimeout(() => changeDetection$.next('CD-3'), 1000);
  }

  triggerChange() {
    this.changeCounter++;
    console.log(`🔄 Change Detection Triggered: ${this.changeCounter}`);
  }

  ngOnDestroy() {
    const onDestroy$ = of('OnDestroy').pipe(delay(0));
    this.viewer.addStream(onDestroy$, 'NgOnDestroy', '#F44336');
  }
}
```

### HTTP Request Flow Debugging

```typescript
@Injectable()
export class HttpDebugService {
  private httpDebugger = new MarbleDiagramDebugger();

  debugHttpRequest<T>(
    url: string,
    request$: Observable<T>,
    label?: string
  ): Observable<T> {
    const requestLabel = label || `HTTP: ${url}`;
    
    console.group(`🌐 HTTP Request Debug: ${requestLabel}`);
    
    const startTime = Date.now();
    
    return request$.pipe(
      tap(() => {
        const duration = Date.now() - startTime;
        console.log(`⏱️ Request duration: ${duration}ms`);
      }),
      catchError(error => {
        console.log('❌ HTTP Error:', {
          status: error.status,
          message: error.message,
          url: error.url
        });
        return throwError(() => error);
      }),
      finalize(() => {
        console.groupEnd();
      })
    );
  }

  debugHttpSequence<T>(
    requests: { url: string; request$: Observable<T>; label: string }[]
  ): Observable<T[]> {
    console.group('🔄 HTTP Sequence Debug');
    
    const debuggedRequests = requests.map(({ url, request$, label }) => 
      this.debugHttpRequest(url, request$, label)
    );

    return forkJoin(debuggedRequests).pipe(
      tap(results => {
        console.log('✅ All requests completed:', results);
      }),
      finalize(() => {
        console.groupEnd();
      })
    );
  }
}
```

## Real-Time Stream Visualization {#real-time-viz}

### Live Stream Monitor

```typescript
@Component({
  selector: 'app-stream-monitor',
  template: `
    <div class="stream-monitor">
      <div class="monitor-header">
        <h3>Live Stream Monitor</h3>
        <div class="monitor-controls">
          <button (click)="togglePause()" 
                  [class.active]="isPaused">
            {{ isPaused ? 'Resume' : 'Pause' }}
          </button>
          <button (click)="clearHistory()">Clear</button>
          <span class="stream-count">
            Active Streams: {{ activeStreamCount }}
          </span>
        </div>
      </div>

      <div class="stream-list">
        <div *ngFor="let stream of monitoredStreams" 
             class="stream-item"
             [class.paused]="stream.isPaused">
          <div class="stream-header">
            <span class="stream-name">{{ stream.name }}</span>
            <span class="stream-status">{{ stream.status }}</span>
            <button (click)="pauseStream(stream.id)">
              {{ stream.isPaused ? 'Resume' : 'Pause' }}
            </button>
          </div>
          
          <div class="stream-timeline">
            <div *ngFor="let event of stream.events.slice(-20)" 
                 class="timeline-event"
                 [class]="'event-' + event.type"
                 [title]="event.timestamp + ': ' + event.data">
              {{ event.type === 'value' ? event.data : event.type }}
            </div>
          </div>
        </div>
      </div>
    </div>
  `,
  styles: [`
    .stream-monitor {
      border: 1px solid #ddd;
      border-radius: 8px;
      background: white;
      overflow: hidden;
    }
    
    .monitor-header {
      background: #f5f5f5;
      padding: 15px;
      border-bottom: 1px solid #ddd;
      display: flex;
      justify-content: space-between;
      align-items: center;
    }
    
    .monitor-controls {
      display: flex;
      align-items: center;
      gap: 10px;
    }
    
    .stream-item {
      border-bottom: 1px solid #eee;
      padding: 10px 15px;
    }
    
    .stream-item.paused {
      opacity: 0.6;
      background: #f9f9f9;
    }
    
    .stream-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 10px;
    }
    
    .stream-name {
      font-weight: bold;
      color: #333;
    }
    
    .stream-status {
      padding: 2px 8px;
      border-radius: 12px;
      font-size: 12px;
      background: #e3f2fd;
      color: #1976d2;
    }
    
    .stream-timeline {
      display: flex;
      gap: 2px;
      overflow-x: auto;
      padding: 5px 0;
    }
    
    .timeline-event {
      min-width: 30px;
      height: 30px;
      border-radius: 50%;
      display: flex;
      align-items: center;
      justify-content: center;
      font-size: 10px;
      color: white;
      font-weight: bold;
    }
    
    .event-value { background: #4caf50; }
    .event-error { background: #f44336; }
    .event-complete { background: #ff9800; border-radius: 0; }
  `]
})
export class StreamMonitorComponent implements OnInit, OnDestroy {
  monitoredStreams: MonitoredStream[] = [];
  activeStreamCount = 0;
  isPaused = false;
  
  private streamCounter = 0;
  private destroy$ = new Subject<void>();

  ngOnInit() {
    // Start monitoring some example streams
    this.monitorStream(
      interval(1000).pipe(map(x => x * 2)),
      'Counter Stream'
    );
    
    this.monitorStream(
      timer(0, 2000).pipe(
        switchMap(() => of(Math.random()).pipe(delay(Math.random() * 500)))
      ),
      'Random Data'
    );
  }

  monitorStream<T>(stream$: Observable<T>, name: string): void {
    const streamId = ++this.streamCounter;
    const monitoredStream: MonitoredStream = {
      id: streamId,
      name,
      status: 'active',
      isPaused: false,
      events: []
    };

    this.monitoredStreams.push(monitoredStream);
    this.activeStreamCount++;

    const pauseSubject = new BehaviorSubject(false);
    
    stream$.pipe(
      filter(() => !pauseSubject.value),
      takeUntil(this.destroy$)
    ).subscribe({
      next: (value) => {
        monitoredStream.events.push({
          type: 'value',
          data: String(value),
          timestamp: new Date().toISOString()
        });
        monitoredStream.status = 'active';
      },
      error: (error) => {
        monitoredStream.events.push({
          type: 'error',
          data: error.message,
          timestamp: new Date().toISOString()
        });
        monitoredStream.status = 'error';
        this.activeStreamCount--;
      },
      complete: () => {
        monitoredStream.events.push({
          type: 'complete',
          data: '',
          timestamp: new Date().toISOString()
        });
        monitoredStream.status = 'complete';
        this.activeStreamCount--;
      }
    });

    // Store pause control for this stream
    (monitoredStream as any).pauseControl = pauseSubject;
  }

  pauseStream(streamId: number): void {
    const stream = this.monitoredStreams.find(s => s.id === streamId);
    if (stream && (stream as any).pauseControl) {
      stream.isPaused = !stream.isPaused;
      (stream as any).pauseControl.next(stream.isPaused);
      stream.status = stream.isPaused ? 'paused' : 'active';
    }
  }

  togglePause(): void {
    this.isPaused = !this.isPaused;
    this.monitoredStreams.forEach(stream => {
      if ((stream as any).pauseControl) {
        (stream as any).pauseControl.next(this.isPaused);
        stream.isPaused = this.isPaused;
        stream.status = this.isPaused ? 'paused' : 'active';
      }
    });
  }

  clearHistory(): void {
    this.monitoredStreams.forEach(stream => {
      stream.events = [];
    });
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

interface MonitoredStream {
  id: number;
  name: string;
  status: 'active' | 'paused' | 'error' | 'complete';
  isPaused: boolean;
  events: StreamEvent[];
}

interface StreamEvent {
  type: 'value' | 'error' | 'complete';
  data: string;
  timestamp: string;
}
```

## Automated Diagram Generation {#automated-generation}

### Automated Marble Diagram Generator

```typescript
class AutoMarbleDiagramGenerator {
  generateFromTestData<T>(
    testData: TestStreamData<T>[],
    options?: DiagramOptions
  ): string {
    const maxTime = Math.max(...testData.map(d => d.maxTime || 100));
    const timeScale = options?.timeScale || 10;
    
    let diagram = '';
    
    testData.forEach(({ label, events, color }) => {
      const marbleLine = this.generateMarbleLine(events, maxTime, timeScale);
      const colorCode = color ? ` (${color})` : '';
      diagram += `${label}${colorCode}: ${marbleLine}\n`;
    });
    
    return diagram;
  }

  private generateMarbleLine<T>(
    events: StreamEvent<T>[],
    maxTime: number,
    timeScale: number
  ): string {
    const positions = new Map<number, string>();
    const lineLength = Math.ceil(maxTime / timeScale);
    
    // Place events at their time positions
    events.forEach(event => {
      const position = Math.floor(event.time / timeScale);
      let symbol: string;
      
      switch (event.type) {
        case 'next':
          symbol = this.valueToSymbol(event.value);
          break;
        case 'error':
          symbol = '#';
          break;
        case 'complete':
          symbol = '|';
          break;
        default:
          symbol = '?';
      }
      
      positions.set(position, symbol);
    });
    
    // Build the marble line
    let line = '';
    for (let i = 0; i < lineLength; i++) {
      if (positions.has(i)) {
        line += positions.get(i);
      } else {
        line += '-';
      }
    }
    
    return line;
  }

  private valueToSymbol<T>(value: T): string {
    if (typeof value === 'number') {
      return value.toString();
    }
    if (typeof value === 'string') {
      return value.length === 1 ? value : value.charAt(0);
    }
    if (typeof value === 'boolean') {
      return value ? 'T' : 'F';
    }
    return 'x';
  }

  generateFromObservable<T>(
    observable$: Observable<T>,
    duration: number = 1000,
    label: string = 'Stream'
  ): Promise<string> {
    return new Promise((resolve) => {
      const events: StreamEvent<T>[] = [];
      const startTime = Date.now();
      
      const subscription = observable$.subscribe({
        next: (value) => {
          events.push({
            type: 'next',
            value,
            time: Date.now() - startTime
          });
        },
        error: (error) => {
          events.push({
            type: 'error',
            value: error,
            time: Date.now() - startTime
          });
          
          const diagram = this.generateFromTestData([{
            label,
            events,
            maxTime: Date.now() - startTime
          }]);
          resolve(diagram);
        },
        complete: () => {
          events.push({
            type: 'complete',
            time: Date.now() - startTime
          });
          
          const diagram = this.generateFromTestData([{
            label,
            events,
            maxTime: Date.now() - startTime
          }]);
          resolve(diagram);
        }
      });
      
      // Auto-complete after duration
      setTimeout(() => {
        subscription.unsubscribe();
        const diagram = this.generateFromTestData([{
          label,
          events,
          maxTime: duration
        }]);
        resolve(diagram);
      }, duration);
    });
  }
}

interface TestStreamData<T> {
  label: string;
  events: StreamEvent<T>[];
  maxTime?: number;
  color?: string;
}

interface StreamEvent<T> {
  type: 'next' | 'error' | 'complete';
  value?: T;
  time: number;
}

interface DiagramOptions {
  timeScale?: number;
  showValues?: boolean;
  useColors?: boolean;
}
```

## Best Practices {#best-practices}

### Debugging Best Practices Checklist

```typescript
const MARBLE_DEBUGGING_BEST_PRACTICES = {
  preparation: [
    '✅ Set up marble diagram tools before debugging',
    '✅ Use consistent labeling for streams',
    '✅ Document expected vs actual behavior',
    '✅ Create minimal reproducible examples'
  ],
  
  visualization: [
    '✅ Use colors to distinguish different streams',
    '✅ Include timestamps for timing analysis',
    '✅ Show operator chain progression',
    '✅ Highlight error conditions clearly'
  ],
  
  analysis: [
    '✅ Compare expected vs actual marble diagrams',
    '✅ Focus on timing relationships between streams',
    '✅ Identify backpressure and memory issues',
    '✅ Track subscription lifecycles'
  ],
  
  documentation: [
    '✅ Save marble diagrams for future reference',
    '✅ Document debugging findings',
    '✅ Share insights with team members',
    '✅ Create debugging guides for common issues'
  ]
};
```

### Debugging Workflow Template

```typescript
class DebuggingWorkflow {
  static async debugObservable<T>(
    observable$: Observable<T>,
    context: DebuggingContext
  ): Promise<DebuggingReport> {
    const report: DebuggingReport = {
      context,
      findings: [],
      recommendations: [],
      marbleDiagrams: []
    };

    // 1. Capture marble diagram
    const diagramGenerator = new AutoMarbleDiagramGenerator();
    const diagram = await diagramGenerator.generateFromObservable(
      observable$,
      context.duration,
      context.label
    );
    report.marbleDiagrams.push(diagram);

    // 2. Analyze performance
    const profiler = new ObservableProfiler();
    const profile = await this.profileObservable(observable$, profiler);
    report.findings.push(`Performance: ${profile.averageEmissionTime}ms avg`);

    // 3. Check for memory leaks
    const leakDetector = new MemoryLeakDetectionService();
    const memoryReport = leakDetector.getLeakReport();
    if (memoryReport.totalSubscriptions > 100) {
      report.findings.push('⚠️ High subscription count detected');
      report.recommendations.push('Review subscription management');
    }

    // 4. Validate expected behavior
    if (context.expectedPattern) {
      const matches = this.comparePatterns(diagram, context.expectedPattern);
      if (!matches) {
        report.findings.push('❌ Output doesn\'t match expected pattern');
        report.recommendations.push('Review operator chain logic');
      }
    }

    return report;
  }

  private static async profileObservable<T>(
    observable$: Observable<T>,
    profiler: ObservableProfiler
  ): Promise<any> {
    return new Promise((resolve) => {
      const profiledStream$ = profiler.profile(observable$, {
        label: 'debug-session'
      });
      
      profiledStream$.subscribe({
        complete: () => {
          resolve(profiler.getProfile('debug-session'));
        }
      });
    });
  }

  private static comparePatterns(
    actual: string,
    expected: string
  ): boolean {
    // Simplified pattern matching
    const normalizePattern = (pattern: string) =>
      pattern.replace(/\s+/g, '').replace(/\d+/g, 'x');
    
    return normalizePattern(actual) === normalizePattern(expected);
  }
}

interface DebuggingContext {
  label: string;
  duration: number;
  expectedPattern?: string;
  performanceThresholds?: {
    maxEmissionTime?: number;
    maxMemoryUsage?: number;
  };
}

interface DebuggingReport {
  context: DebuggingContext;
  findings: string[];
  recommendations: string[];
  marbleDiagrams: string[];
}
```

## Exercises {#exercises}

### Exercise 1: Interactive Marble Debugger

Create a comprehensive marble diagram debugger with:
- Real-time stream visualization
- Interactive timeline controls
- Export capabilities
- Performance metrics overlay

### Exercise 2: Complex Stream Flow Analysis

Build a tool that can:
- Visualize higher-order observable patterns
- Show timing relationships between multiple streams
- Identify performance bottlenecks
- Generate automated recommendations

### Exercise 3: Error Flow Visualization

Implement an error debugging system that:
- Traces error propagation through operator chains
- Shows recovery patterns
- Provides error frequency analysis
- Suggests error handling improvements

### Exercise 4: Production Debugging Dashboard

Create a production-ready debugging dashboard that:
- Monitors live Angular applications
- Provides marble diagram views of user interactions
- Tracks performance metrics over time
- Alerts on anomalous patterns

---

**Next Steps:**
- Master advanced marble testing techniques
- Explore production monitoring strategies
- Practice with complex debugging scenarios
- Build custom debugging tools for your specific use cases
