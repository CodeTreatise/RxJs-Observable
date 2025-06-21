# RxJS Debugging Streams & Tools

## Learning Objectives
- Master debugging techniques for RxJS observables
- Use browser dev tools effectively for reactive debugging
- Implement custom debugging operators and utilities
- Identify and resolve common observable issues
- Leverage debugging libraries and extensions
- Profile and monitor observable performance

## Prerequisites
- Solid understanding of RxJS operators
- Basic debugging knowledge
- Browser developer tools familiarity
- Angular development experience

---

## Table of Contents
1. [Debugging Fundamentals](#debugging-fundamentals)
2. [Browser Developer Tools](#browser-developer-tools)
3. [Custom Debugging Operators](#custom-debugging-operators)
4. [Logging and Tracing](#logging-and-tracing)
5. [Debugging Tools & Extensions](#debugging-tools--extensions)
6. [Common Debugging Scenarios](#common-debugging-scenarios)
7. [Performance Debugging](#performance-debugging)
8. [Error Debugging](#error-debugging)
9. [Debugging Utilities](#debugging-utilities)
10. [Production Debugging](#production-debugging)

---

## Debugging Fundamentals

### Core Debugging Principles

Understanding observable streams requires systematic debugging approaches:

```typescript
// Basic debugging with tap operator
const debugged$ = source$.pipe(
  tap(value => console.log('Before map:', value)),
  map(x => x * 2),
  tap(value => console.log('After map:', value)),
  filter(x => x > 10),
  tap(value => console.log('After filter:', value))
);
```

### Debugging Observable Lifecycle

```typescript
import { Observable, EMPTY, throwError } from 'rxjs';
import { tap, finalize, catchError } from 'rxjs/operators';

// Comprehensive lifecycle debugging
function debugLifecycle<T>(name: string) {
  return (source: Observable<T>) => {
    console.log(`🟢 [${name}] Observable created`);
    
    return new Observable<T>(subscriber => {
      console.log(`🔵 [${name}] Subscription started`);
      
      const subscription = source.pipe(
        tap({
          next: value => console.log(`📥 [${name}] Next:`, value),
          error: error => console.log(`❌ [${name}] Error:`, error),
          complete: () => console.log(`✅ [${name}] Complete`)
        }),
        finalize(() => console.log(`🔴 [${name}] Finalized`))
      ).subscribe(subscriber);
      
      return () => {
        console.log(`🟡 [${name}] Unsubscribed`);
        subscription.unsubscribe();
      };
    });
  };
}

// Usage example
const source$ = interval(1000).pipe(
  take(3),
  debugLifecycle('Timer')
);

const subscription = source$.subscribe();
setTimeout(() => subscription.unsubscribe(), 2500);
```

### Visual Stream Debugging

```typescript
// Visual debugging with ASCII art
function visualDebug<T>(name: string, symbol: string = '●') {
  let frameCount = 0;
  const startTime = Date.now();
  
  return tap<T>({
    next: (value) => {
      const elapsed = Date.now() - startTime;
      const frame = Math.floor(elapsed / 100);
      const timeline = Array(frame + 1).fill('-').join('');
      console.log(`${name}: ${timeline}${symbol} (${value})`);
      frameCount++;
    },
    complete: () => {
      console.log(`${name}: ${'─'.repeat(frameCount)}|`);
    },
    error: (error) => {
      console.log(`${name}: ${'─'.repeat(frameCount)}# (${error.message})`);
    }
  });
}

// Usage
const stream$ = interval(300).pipe(
  take(5),
  visualDebug('Stream', '●'),
  map(x => x * 2),
  visualDebug('Mapped', '◆')
);
```

---

## Browser Developer Tools

### Console Debugging Strategies

```typescript
// Advanced console debugging
class ConsoleDebugger {
  private static groupDepth = 0;
  
  static logStream<T>(name: string, options: {
    logNext?: boolean;
    logError?: boolean;
    logComplete?: boolean;
    logSubscription?: boolean;
    grouping?: boolean;
  } = {}) {
    const {
      logNext = true,
      logError = true,
      logComplete = true,
      logSubscription = true,
      grouping = false
    } = options;
    
    return (source: Observable<T>) => {
      return new Observable<T>(subscriber => {
        if (logSubscription && grouping) {
          console.group(`🔵 [${name}] Stream Started`);
          this.groupDepth++;
        }
        
        const subscription = source.subscribe({
          next: (value) => {
            if (logNext) {
              console.log(
                `%c📥 [${name}] Next:`, 
                'color: #4CAF50; font-weight: bold',
                value
              );
            }
            subscriber.next(value);
          },
          error: (error) => {
            if (logError) {
              console.error(
                `%c❌ [${name}] Error:`, 
                'color: #F44336; font-weight: bold',
                error
              );
            }
            if (grouping && this.groupDepth > 0) {
              console.groupEnd();
              this.groupDepth--;
            }
            subscriber.error(error);
          },
          complete: () => {
            if (logComplete) {
              console.log(
                `%c✅ [${name}] Complete`, 
                'color: #2196F3; font-weight: bold'
              );
            }
            if (grouping && this.groupDepth > 0) {
              console.groupEnd();
              this.groupDepth--;
            }
            subscriber.complete();
          }
        });
        
        return () => {
          if (logSubscription) {
            console.log(
              `%c🔴 [${name}] Unsubscribed`, 
              'color: #FF9800; font-weight: bold'
            );
          }
          subscription.unsubscribe();
        };
      });
    };
  }
  
  static table<T extends Record<string, any>>(name: string) {
    const data: T[] = [];
    
    return tap<T>({
      next: (value) => {
        data.push(value);
        console.clear();
        console.table(data);
      },
      complete: () => {
        console.log(`%c📊 [${name}] Final Table:`, 'font-weight: bold');
        console.table(data);
      }
    });
  }
}

// Usage examples
const debuggedStream$ = source$.pipe(
  ConsoleDebugger.logStream('API Call', {
    grouping: true,
    logSubscription: true
  }),
  map(transformData),
  ConsoleDebugger.logStream('Transformed'),
  ConsoleDebugger.table('Results')
);
```

### Performance Profiling

```typescript
// Performance debugging utilities
class PerformanceDebugger {
  private static timers = new Map<string, number>();
  private static counters = new Map<string, number>();
  
  static time<T>(label: string) {
    return tap<T>({
      subscribe: () => {
        this.timers.set(label, performance.now());
        console.time(label);
      },
      finalize: () => {
        const startTime = this.timers.get(label);
        if (startTime) {
          const duration = performance.now() - startTime;
          console.timeEnd(label);
          console.log(`⏱️ [${label}] Duration: ${duration.toFixed(2)}ms`);
        }
      }
    });
  }
  
  static count<T>(label: string) {
    return tap<T>({
      next: () => {
        const current = this.counters.get(label) || 0;
        this.counters.set(label, current + 1);
        console.count(label);
      }
    });
  }
  
  static memory<T>(label: string) {
    return tap<T>({
      next: () => {
        if (performance.memory) {
          const memory = performance.memory;
          console.log(`🧠 [${label}] Memory:`, {
            used: `${(memory.usedJSHeapSize / 1024 / 1024).toFixed(2)}MB`,
            total: `${(memory.totalJSHeapSize / 1024 / 1024).toFixed(2)}MB`,
            limit: `${(memory.jsHeapSizeLimit / 1024 / 1024).toFixed(2)}MB`
          });
        }
      }
    });
  }
  
  static profile<T>(label: string) {
    return tap<T>({
      subscribe: () => {
        console.profile(label);
      },
      finalize: () => {
        console.profileEnd(label);
      }
    });
  }
}

// Usage
const stream$ = source$.pipe(
  PerformanceDebugger.time('Processing'),
  PerformanceDebugger.count('Items'),
  map(heavyComputation),
  PerformanceDebugger.memory('After Processing'),
  PerformanceDebugger.profile('Heavy Operation')
);
```

---

## Custom Debugging Operators

### Comprehensive Debug Operator

```typescript
interface DebugOptions<T> {
  name?: string;
  logNext?: boolean;
  logError?: boolean;
  logComplete?: boolean;
  logSubscription?: boolean;
  logUnsubscription?: boolean;
  filter?: (value: T) => boolean;
  transform?: (value: T) => any;
  throttle?: number;
  format?: 'simple' | 'detailed' | 'json';
  color?: string;
}

function debug<T>(options: DebugOptions<T> = {}): MonoTypeOperatorFunction<T> {
  const {
    name = 'Debug',
    logNext = true,
    logError = true,
    logComplete = true,
    logSubscription = true,
    logUnsubscription = true,
    filter = () => true,
    transform = (value) => value,
    throttle = 0,
    format = 'simple',
    color = '#007ACC'
  } = options;
  
  let lastLogTime = 0;
  let subscriptionId = Math.random().toString(36).substr(2, 9);
  
  return (source: Observable<T>) => {
    return new Observable<T>(subscriber => {
      if (logSubscription) {
        console.log(
          `%c🔵 [${name}:${subscriptionId}] Subscribed`,
          `color: ${color}; font-weight: bold`
        );
      }
      
      const subscription = source.subscribe({
        next: (value) => {
          const now = Date.now();
          const shouldLog = logNext && 
                           filter(value) && 
                           (now - lastLogTime >= throttle);
          
          if (shouldLog) {
            lastLogTime = now;
            const displayValue = transform(value);
            
            switch (format) {
              case 'detailed':
                console.group(`%c📥 [${name}] Next`, `color: ${color}`);
                console.log('Value:', displayValue);
                console.log('Type:', typeof value);
                console.log('Timestamp:', new Date().toISOString());
                console.groupEnd();
                break;
              case 'json':
                console.log(
                  `%c📥 [${name}] Next:`,
                  `color: ${color}`,
                  JSON.stringify(displayValue, null, 2)
                );
                break;
              default:
                console.log(
                  `%c📥 [${name}] Next:`,
                  `color: ${color}`,
                  displayValue
                );
            }
          }
          
          subscriber.next(value);
        },
        error: (error) => {
          if (logError) {
            console.error(
              `%c❌ [${name}:${subscriptionId}] Error:`,
              `color: #F44336; font-weight: bold`,
              error
            );
          }
          subscriber.error(error);
        },
        complete: () => {
          if (logComplete) {
            console.log(
              `%c✅ [${name}:${subscriptionId}] Complete`,
              `color: #4CAF50; font-weight: bold`
            );
          }
          subscriber.complete();
        }
      });
      
      return () => {
        if (logUnsubscription) {
          console.log(
            `%c🔴 [${name}:${subscriptionId}] Unsubscribed`,
            `color: #FF9800; font-weight: bold`
          );
        }
        subscription.unsubscribe();
      };
    });
  };
}

// Usage examples
const stream$ = source$.pipe(
  debug({
    name: 'Input',
    format: 'detailed',
    color: '#4CAF50'
  }),
  map(x => x * 2),
  debug({
    name: 'Mapped',
    filter: value => value > 10,
    throttle: 100
  }),
  filter(x => x > 5),
  debug({
    name: 'Filtered',
    transform: value => ({ processed: value, timestamp: Date.now() }),
    format: 'json'
  })
);
```

### Conditional Debugging

```typescript
// Environment-aware debugging
function debugIf<T>(
  condition: boolean | (() => boolean),
  options: DebugOptions<T> = {}
): MonoTypeOperatorFunction<T> {
  return (source: Observable<T>) => {
    const shouldDebug = typeof condition === 'function' ? condition() : condition;
    
    if (shouldDebug) {
      return source.pipe(debug(options));
    }
    
    return source;
  };
}

// Feature flag debugging
function debugFeature<T>(
  feature: string,
  options: DebugOptions<T> = {}
): MonoTypeOperatorFunction<T> {
  return debugIf(() => {
    // Check if feature is enabled in localStorage, environment, etc.
    return localStorage.getItem(`debug-${feature}`) === 'true' ||
           window.location.search.includes(`debug=${feature}`);
  }, { ...options, name: feature });
}

// Usage
const stream$ = source$.pipe(
  debugFeature('user-service', { format: 'detailed' }),
  map(processUser),
  debugIf(environment.production === false, {
    name: 'Development Only',
    color: '#FF6B35'
  })
);
```

---

## Logging and Tracing

### Structured Logging

```typescript
interface LogEntry {
  timestamp: string;
  level: 'debug' | 'info' | 'warn' | 'error';
  stream: string;
  event: 'next' | 'error' | 'complete' | 'subscribe' | 'unsubscribe';
  data?: any;
  metadata?: Record<string, any>;
}

class StreamLogger {
  private logs: LogEntry[] = [];
  private static instance: StreamLogger;
  
  static getInstance(): StreamLogger {
    if (!this.instance) {
      this.instance = new StreamLogger();
    }
    return this.instance;
  }
  
  private log(entry: Omit<LogEntry, 'timestamp'>): void {
    const logEntry: LogEntry = {
      ...entry,
      timestamp: new Date().toISOString()
    };
    
    this.logs.push(logEntry);
    
    // Send to external logging service in production
    if (environment.production) {
      this.sendToLoggingService(logEntry);
    } else {
      this.consoleLog(logEntry);
    }
  }
  
  private consoleLog(entry: LogEntry): void {
    const color = this.getColorForLevel(entry.level);
    const icon = this.getIconForEvent(entry.event);
    
    console.log(
      `%c${icon} [${entry.stream}] ${entry.event}`,
      `color: ${color}; font-weight: bold`,
      entry.data,
      entry.metadata
    );
  }
  
  private sendToLoggingService(entry: LogEntry): void {
    // Implementation for external logging service
    fetch('/api/logs', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(entry)
    }).catch(console.error);
  }
  
  private getColorForLevel(level: string): string {
    const colors = {
      debug: '#9E9E9E',
      info: '#2196F3',
      warn: '#FF9800',
      error: '#F44336'
    };
    return colors[level] || '#000000';
  }
  
  private getIconForEvent(event: string): string {
    const icons = {
      next: '📥',
      error: '❌',
      complete: '✅',
      subscribe: '🔵',
      unsubscribe: '🔴'
    };
    return icons[event] || '📝';
  }
  
  streamLogger<T>(streamName: string, level: LogEntry['level'] = 'debug') {
    return (source: Observable<T>) => {
      return new Observable<T>(subscriber => {
        this.log({
          level,
          stream: streamName,
          event: 'subscribe'
        });
        
        const subscription = source.subscribe({
          next: (value) => {
            this.log({
              level,
              stream: streamName,
              event: 'next',
              data: value,
              metadata: { valueType: typeof value }
            });
            subscriber.next(value);
          },
          error: (error) => {
            this.log({
              level: 'error',
              stream: streamName,
              event: 'error',
              data: error.message,
              metadata: { 
                stack: error.stack,
                name: error.name
              }
            });
            subscriber.error(error);
          },
          complete: () => {
            this.log({
              level,
              stream: streamName,
              event: 'complete'
            });
            subscriber.complete();
          }
        });
        
        return () => {
          this.log({
            level,
            stream: streamName,
            event: 'unsubscribe'
          });
          subscription.unsubscribe();
        };
      });
    };
  }
  
  getLogs(streamName?: string): LogEntry[] {
    if (streamName) {
      return this.logs.filter(log => log.stream === streamName);
    }
    return [...this.logs];
  }
  
  clearLogs(): void {
    this.logs = [];
  }
  
  exportLogs(): string {
    return JSON.stringify(this.logs, null, 2);
  }
}

// Usage
const logger = StreamLogger.getInstance();

const stream$ = source$.pipe(
  logger.streamLogger('UserService', 'info'),
  map(processData),
  logger.streamLogger('ProcessedData', 'debug')
);
```

### Distributed Tracing

```typescript
// Correlation ID tracing
class TraceContext {
  private static correlationId: string | null = null;
  
  static setCorrelationId(id: string): void {
    this.correlationId = id;
  }
  
  static getCorrelationId(): string {
    if (!this.correlationId) {
      this.correlationId = this.generateId();
    }
    return this.correlationId;
  }
  
  static clearCorrelationId(): void {
    this.correlationId = null;
  }
  
  private static generateId(): string {
    return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
}

function traceStream<T>(operationName: string) {
  return (source: Observable<T>) => {
    const correlationId = TraceContext.getCorrelationId();
    const startTime = performance.now();
    
    return new Observable<T>(subscriber => {
      console.log(`🔍 [${correlationId}] ${operationName} started`);
      
      const subscription = source.subscribe({
        next: (value) => {
          console.log(`🔍 [${correlationId}] ${operationName} next:`, value);
          subscriber.next(value);
        },
        error: (error) => {
          const duration = performance.now() - startTime;
          console.error(
            `🔍 [${correlationId}] ${operationName} error after ${duration.toFixed(2)}ms:`,
            error
          );
          subscriber.error(error);
        },
        complete: () => {
          const duration = performance.now() - startTime;
          console.log(
            `🔍 [${correlationId}] ${operationName} completed in ${duration.toFixed(2)}ms`
          );
          subscriber.complete();
        }
      });
      
      return () => subscription.unsubscribe();
    });
  };
}

// Usage
TraceContext.setCorrelationId('user-login-flow-001');

const loginFlow$ = credentials$.pipe(
  traceStream('validate-credentials'),
  switchMap(creds => validateCredentials(creds)),
  traceStream('fetch-user-profile'),
  switchMap(user => fetchUserProfile(user.id)),
  traceStream('update-session'),
  tap(profile => updateSession(profile))
);
```

---

## Debugging Tools & Extensions

### Browser Extensions Integration

```typescript
// Redux DevTools integration for observables
interface DevToolsMessage {
  type: string;
  payload: any;
  timestamp: number;
}

class ObservableDevTools {
  private static isEnabled = typeof window !== 'undefined' && 
                            (window as any).__REDUX_DEVTOOLS_EXTENSION__;
  
  private static devTools = this.isEnabled ? 
    (window as any).__REDUX_DEVTOOLS_EXTENSION__.connect({
      name: 'RxJS Streams'
    }) : null;
  
  static debugStream<T>(streamName: string) {
    if (!this.isEnabled || !this.devTools) {
      return (source: Observable<T>) => source;
    }
    
    return (source: Observable<T>) => {
      return source.pipe(
        tap({
          next: (value) => {
            this.devTools.send({
              type: `${streamName}_NEXT`,
              payload: value,
              timestamp: Date.now()
            } as DevToolsMessage, {
              [streamName]: value
            });
          },
          error: (error) => {
            this.devTools.send({
              type: `${streamName}_ERROR`,
              payload: error.message,
              timestamp: Date.now()
            } as DevToolsMessage, {
              [streamName]: { error: error.message }
            });
          },
          complete: () => {
            this.devTools.send({
              type: `${streamName}_COMPLETE`,
              payload: null,
              timestamp: Date.now()
            } as DevToolsMessage, {
              [streamName]: 'COMPLETED'
            });
          }
        })
      );
    };
  }
}

// Usage
const stream$ = source$.pipe(
  ObservableDevTools.debugStream('UserSearch'),
  debounceTime(300),
  ObservableDevTools.debugStream('Debounced'),
  switchMap(searchApi),
  ObservableDevTools.debugStream('SearchResults')
);
```

### Custom Debugging Panel

```typescript
// Custom debugging panel component
@Component({
  selector: 'app-rx-debug-panel',
  template: `
    <div class="debug-panel" [class.expanded]="isExpanded">
      <div class="debug-header" (click)="toggle()">
        <span>RxJS Debug Panel</span>
        <span class="toggle">{{ isExpanded ? '▼' : '▶' }}</span>
      </div>
      
      <div class="debug-content" *ngIf="isExpanded">
        <div class="controls">
          <button (click)="clearLogs()">Clear</button>
          <button (click)="exportLogs()">Export</button>
          <label>
            <input type="checkbox" [(ngModel)]="autoScroll"> Auto Scroll
          </label>
        </div>
        
        <div class="logs" #logsContainer>
          <div *ngFor="let log of logs$ | async; trackBy: trackByFn"
               class="log-entry"
               [class]="log.level">
            <span class="timestamp">{{ log.timestamp | date:'HH:mm:ss.SSS' }}</span>
            <span class="stream">{{ log.stream }}</span>
            <span class="event">{{ log.event }}</span>
            <span class="data">{{ log.data | json }}</span>
          </div>
        </div>
      </div>
    </div>
  `,
  styles: [`
    .debug-panel {
      position: fixed;
      bottom: 0;
      right: 0;
      width: 400px;
      background: #1e1e1e;
      color: #ffffff;
      font-family: 'Courier New', monospace;
      font-size: 12px;
      z-index: 10000;
      border: 1px solid #333;
    }
    
    .debug-header {
      background: #333;
      padding: 8px 12px;
      cursor: pointer;
      display: flex;
      justify-content: space-between;
      align-items: center;
    }
    
    .debug-content {
      max-height: 300px;
      overflow: hidden;
    }
    
    .controls {
      padding: 8px;
      background: #2d2d2d;
      display: flex;
      gap: 8px;
      align-items: center;
    }
    
    .logs {
      height: 200px;
      overflow-y: auto;
      padding: 4px;
    }
    
    .log-entry {
      padding: 2px 4px;
      border-bottom: 1px solid #333;
      display: grid;
      grid-template-columns: 80px 100px 80px 1fr;
      gap: 8px;
    }
    
    .log-entry.error { background-color: rgba(244, 67, 54, 0.1); }
    .log-entry.warn { background-color: rgba(255, 152, 0, 0.1); }
    .log-entry.info { background-color: rgba(33, 150, 243, 0.1); }
  `]
})
export class RxDebugPanelComponent implements OnInit, AfterViewChecked {
  @ViewChild('logsContainer') logsContainer!: ElementRef;
  
  isExpanded = false;
  autoScroll = true;
  logs$ = StreamLogger.getInstance().getLogs();
  
  private previousLogCount = 0;

  ngOnInit(): void {
    // Only show in development
    if (environment.production) {
      return;
    }
  }

  ngAfterViewChecked(): void {
    if (this.autoScroll && this.logsContainer) {
      const element = this.logsContainer.nativeElement;
      const currentLogCount = this.logs$.length;
      
      if (currentLogCount > this.previousLogCount) {
        element.scrollTop = element.scrollHeight;
        this.previousLogCount = currentLogCount;
      }
    }
  }

  toggle(): void {
    this.isExpanded = !this.isExpanded;
  }

  clearLogs(): void {
    StreamLogger.getInstance().clearLogs();
  }

  exportLogs(): void {
    const logs = StreamLogger.getInstance().exportLogs();
    const blob = new Blob([logs], { type: 'application/json' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `rxjs-logs-${new Date().toISOString()}.json`;
    a.click();
    URL.revokeObjectURL(url);
  }

  trackByFn(index: number, log: LogEntry): string {
    return `${log.timestamp}-${log.stream}-${log.event}`;
  }
}
```

---

## Common Debugging Scenarios

### Debugging Subscription Issues

```typescript
// Debugging multiple subscriptions
function debugSubscriptions<T>(name: string) {
  let subscriptionCount = 0;
  
  return (source: Observable<T>) => {
    return new Observable<T>(subscriber => {
      const subscriptionId = ++subscriptionCount;
      console.log(`🔵 [${name}] Subscription #${subscriptionId} started`);
      console.log(`📊 [${name}] Total active subscriptions: ${subscriptionCount}`);
      
      const subscription = source.subscribe({
        next: (value) => {
          console.log(`📥 [${name}:${subscriptionId}] Next:`, value);
          subscriber.next(value);
        },
        error: (error) => {
          console.error(`❌ [${name}:${subscriptionId}] Error:`, error);
          subscriptionCount--;
          console.log(`📊 [${name}] Total active subscriptions: ${subscriptionCount}`);
          subscriber.error(error);
        },
        complete: () => {
          console.log(`✅ [${name}:${subscriptionId}] Complete`);
          subscriptionCount--;
          console.log(`📊 [${name}] Total active subscriptions: ${subscriptionCount}`);
          subscriber.complete();
        }
      });
      
      return () => {
        console.log(`🔴 [${name}:${subscriptionId}] Unsubscribed`);
        subscriptionCount--;
        console.log(`📊 [${name}] Total active subscriptions: ${subscriptionCount}`);
        subscription.unsubscribe();
      };
    });
  };
}

// Debugging memory leaks
function debugMemoryLeaks<T>(name: string, checkInterval: number = 5000) {
  return (source: Observable<T>) => {
    const checkMemory = () => {
      if (performance.memory) {
        const memory = performance.memory;
        console.log(`🧠 [${name}] Memory Check:`, {
          used: `${(memory.usedJSHeapSize / 1024 / 1024).toFixed(2)}MB`,
          total: `${(memory.totalJSHeapSize / 1024 / 1024).toFixed(2)}MB`
        });
      }
    };
    
    return source.pipe(
      tap({
        subscribe: () => {
          checkMemory();
          const interval = setInterval(checkMemory, checkInterval);
          
          return () => {
            clearInterval(interval);
            checkMemory();
          };
        }
      })
    );
  };
}
```

### Debugging Timing Issues

```typescript
// Debugging race conditions
function debugRace<T>(name: string) {
  return (source: Observable<T>) => {
    const startTime = performance.now();
    
    return source.pipe(
      tap({
        subscribe: () => {
          console.log(`🏁 [${name}] Race started at ${startTime.toFixed(2)}ms`);
        },
        next: (value) => {
          const elapsed = performance.now() - startTime;
          console.log(`🏆 [${name}] Winner at ${elapsed.toFixed(2)}ms:`, value);
        }
      })
    );
  };
}

// Debugging timing dependencies
function debugTiming<T>(name: string) {
  const events: Array<{ event: string; time: number; data?: any }> = [];
  const startTime = performance.now();
  
  return (source: Observable<T>) => {
    return source.pipe(
      tap({
        subscribe: () => {
          events.push({ event: 'subscribe', time: performance.now() - startTime });
          console.log(`⏰ [${name}] Timeline:`, events);
        },
        next: (value) => {
          events.push({ 
            event: 'next', 
            time: performance.now() - startTime, 
            data: value 
          });
          console.log(`⏰ [${name}] Timeline:`, events);
        },
        complete: () => {
          events.push({ event: 'complete', time: performance.now() - startTime });
          console.log(`⏰ [${name}] Final Timeline:`, events);
        }
      })
    );
  };
}
```

### Debugging Operator Chains

```typescript
// Step-by-step operator debugging
function debugPipeline<T>(name: string, steps: string[]) {
  return (source: Observable<T>) => {
    let currentStep = 0;
    
    return source.pipe(
      tap(value => {
        if (currentStep < steps.length) {
          console.log(`🔧 [${name}] Step ${currentStep + 1}: ${steps[currentStep]}`, value);
          currentStep++;
        }
      })
    );
  };
}

// Usage
const pipeline$ = source$.pipe(
  debugPipeline('Data Processing', [
    'Raw input',
    'After validation',
    'After transformation',
    'After filtering',
    'Final output'
  ]),
  validate(),
  debugPipeline('Data Processing', []),
  transform(),
  debugPipeline('Data Processing', []),
  filter(isValid),
  debugPipeline('Data Processing', [])
);
```

---

## Performance Debugging

### Performance Monitoring

```typescript
// Performance metrics collection
class PerformanceMonitor {
  private metrics = new Map<string, {
    count: number;
    totalDuration: number;
    minDuration: number;
    maxDuration: number;
    durations: number[];
  }>();
  
  monitor<T>(operationName: string) {
    return (source: Observable<T>) => {
      return new Observable<T>(subscriber => {
        const startTime = performance.now();
        
        const subscription = source.subscribe({
          next: (value) => {
            const duration = performance.now() - startTime;
            this.recordMetric(operationName, duration);
            subscriber.next(value);
          },
          error: (error) => {
            const duration = performance.now() - startTime;
            this.recordMetric(`${operationName}_ERROR`, duration);
            subscriber.error(error);
          },
          complete: () => {
            const duration = performance.now() - startTime;
            this.recordMetric(operationName, duration);
            subscriber.complete();
          }
        });
        
        return () => subscription.unsubscribe();
      });
    };
  }
  
  private recordMetric(name: string, duration: number): void {
    const existing = this.metrics.get(name) || {
      count: 0,
      totalDuration: 0,
      minDuration: Infinity,
      maxDuration: 0,
      durations: []
    };
    
    existing.count++;
    existing.totalDuration += duration;
    existing.minDuration = Math.min(existing.minDuration, duration);
    existing.maxDuration = Math.max(existing.maxDuration, duration);
    existing.durations.push(duration);
    
    // Keep only last 100 measurements
    if (existing.durations.length > 100) {
      existing.durations = existing.durations.slice(-100);
    }
    
    this.metrics.set(name, existing);
  }
  
  getMetrics(operationName?: string) {
    if (operationName) {
      return this.metrics.get(operationName);
    }
    
    const summary = new Map();
    this.metrics.forEach((metric, name) => {
      summary.set(name, {
        ...metric,
        averageDuration: metric.totalDuration / metric.count,
        percentile95: this.calculatePercentile(metric.durations, 95)
      });
    });
    
    return summary;
  }
  
  private calculatePercentile(durations: number[], percentile: number): number {
    const sorted = [...durations].sort((a, b) => a - b);
    const index = Math.ceil((percentile / 100) * sorted.length) - 1;
    return sorted[index] || 0;
  }
  
  report(): void {
    console.group('🚀 Performance Report');
    
    this.metrics.forEach((metric, name) => {
      const avg = metric.totalDuration / metric.count;
      const p95 = this.calculatePercentile(metric.durations, 95);
      
      console.log(`📊 ${name}:`, {
        count: metric.count,
        average: `${avg.toFixed(2)}ms`,
        min: `${metric.minDuration.toFixed(2)}ms`,
        max: `${metric.maxDuration.toFixed(2)}ms`,
        p95: `${p95.toFixed(2)}ms`
      });
    });
    
    console.groupEnd();
  }
}

// Usage
const monitor = new PerformanceMonitor();

const stream$ = source$.pipe(
  monitor.monitor('HTTP_REQUEST'),
  switchMap(fetchData),
  monitor.monitor('DATA_TRANSFORM'),
  map(transformData),
  monitor.monitor('FILTER_OPERATION'),
  filter(isValid)
);

// Generate performance report
setInterval(() => monitor.report(), 30000);
```

---

## Error Debugging

### Error Tracking and Analysis

```typescript
// Comprehensive error debugging
interface ErrorContext {
  streamName: string;
  operatorStack: string[];
  lastValue?: any;
  subscriptionId: string;
  timestamp: string;
  userAgent: string;
  url: string;
}

class ErrorTracker {
  private errors: Array<{
    error: Error;
    context: ErrorContext;
  }> = [];
  
  trackErrors<T>(streamName: string, operatorName?: string) {
    return (source: Observable<T>) => {
      const subscriptionId = this.generateId();
      const operatorStack: string[] = [];
      let lastValue: any;
      
      if (operatorName) {
        operatorStack.push(operatorName);
      }
      
      return new Observable<T>(subscriber => {
        const subscription = source.subscribe({
          next: (value) => {
            lastValue = value;
            subscriber.next(value);
          },
          error: (error) => {
            const context: ErrorContext = {
              streamName,
              operatorStack: [...operatorStack],
              lastValue,
              subscriptionId,
              timestamp: new Date().toISOString(),
              userAgent: navigator.userAgent,
              url: window.location.href
            };
            
            this.recordError(error, context);
            this.debugError(error, context);
            
            subscriber.error(error);
          },
          complete: () => {
            subscriber.complete();
          }
        });
        
        return () => subscription.unsubscribe();
      });
    };
  }
  
  private recordError(error: Error, context: ErrorContext): void {
    this.errors.push({ error, context });
    
    // Send to error reporting service
    this.reportToService(error, context);
  }
  
  private debugError(error: Error, context: ErrorContext): void {
    console.group(`❌ Error in stream: ${context.streamName}`);
    console.error('Error:', error);
    console.log('Context:', context);
    console.log('Stack trace:', error.stack);
    console.log('Operator stack:', context.operatorStack);
    console.log('Last value:', context.lastValue);
    console.groupEnd();
  }
  
  private reportToService(error: Error, context: ErrorContext): void {
    // Implementation for error reporting service
    if (!environment.production) return;
    
    fetch('/api/errors', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        message: error.message,
        stack: error.stack,
        context
      })
    }).catch(err => console.error('Failed to report error:', err));
  }
  
  private generateId(): string {
    return Math.random().toString(36).substr(2, 9);
  }
  
  getErrors(): Array<{ error: Error; context: ErrorContext }> {
    return [...this.errors];
  }
  
  clearErrors(): void {
    this.errors = [];
  }
}

// Global error tracker instance
const errorTracker = new ErrorTracker();

// Enhanced operators with error tracking
const trackingOperators = {
  map: <T, R>(fn: (value: T) => R, streamName: string = 'unknown') =>
    (source: Observable<T>) =>
      source.pipe(
        errorTracker.trackErrors(streamName, 'map'),
        map(fn)
      ),
  
  filter: <T>(predicate: (value: T) => boolean, streamName: string = 'unknown') =>
    (source: Observable<T>) =>
      source.pipe(
        errorTracker.trackErrors(streamName, 'filter'),
        filter(predicate)
      ),
  
  switchMap: <T, R>(fn: (value: T) => Observable<R>, streamName: string = 'unknown') =>
    (source: Observable<T>) =>
      source.pipe(
        errorTracker.trackErrors(streamName, 'switchMap'),
        switchMap(fn)
      )
};

// Usage
const stream$ = source$.pipe(
  trackingOperators.map(x => x.toUpperCase(), 'UserInput'),
  trackingOperators.filter(x => x.length > 0, 'UserInput'),
  trackingOperators.switchMap(searchApi, 'SearchAPI')
);
```

---

## Debugging Utilities

### Development Helper Functions

```typescript
// Collection of debugging utilities
export const RxDebugUtils = {
  // Log all events with timestamps
  logWithTimestamp<T>(label: string = 'Observable') {
    return tap<T>({
      next: value => console.log(`[${new Date().toISOString()}] ${label} next:`, value),
      error: error => console.error(`[${new Date().toISOString()}] ${label} error:`, error),
      complete: () => console.log(`[${new Date().toISOString()}] ${label} complete`)
    });
  },
  
  // Break execution for debugging
  breakpoint<T>(condition?: (value: T) => boolean) {
    return tap<T>(value => {
      if (!condition || condition(value)) {
        debugger; // This will pause execution in dev tools
      }
    });
  },
  
  // Count emissions
  count<T>(label: string = 'Count') {
    let counter = 0;
    return tap<T>(() => {
      counter++;
      console.log(`${label}: ${counter}`);
    });
  },
  
  // Assert values for testing
  assert<T>(predicate: (value: T) => boolean, message: string = 'Assertion failed') {
    return tap<T>(value => {
      if (!predicate(value)) {
        console.error(message, value);
        throw new Error(`${message}: ${JSON.stringify(value)}`);
      }
    });
  },
  
  // Log stack trace
  stackTrace<T>(label: string = 'Stack') {
    return tap<T>(() => {
      console.log(`${label} stack trace:`, new Error().stack);
    });
  },
  
  // Measure time between emissions
  measureInterval<T>(label: string = 'Interval') {
    let lastTime = performance.now();
    return tap<T>(() => {
      const now = performance.now();
      const interval = now - lastTime;
      console.log(`${label} interval: ${interval.toFixed(2)}ms`);
      lastTime = now;
    });
  },
  
  // Log only changed values
  logChanges<T>(label: string = 'Changes', compareFn?: (a: T, b: T) => boolean) {
    let lastValue: T;
    let isFirst = true;
    
    return tap<T>(value => {
      const hasChanged = isFirst || (compareFn ? 
        !compareFn(lastValue, value) : 
        lastValue !== value);
      
      if (hasChanged) {
        console.log(`${label} changed:`, { from: lastValue, to: value });
        lastValue = value;
        isFirst = false;
      }
    });
  },
  
  // Save values to variable for inspection
  saveToVariable<T>(variableName: string) {
    return tap<T>(value => {
      (window as any)[variableName] = value;
      console.log(`Saved to window.${variableName}:`, value);
    });
  }
};

// Usage examples
const debuggedStream$ = source$.pipe(
  RxDebugUtils.logWithTimestamp('Source'),
  RxDebugUtils.count('Emissions'),
  map(x => x * 2),
  RxDebugUtils.measureInterval('Processing'),
  RxDebugUtils.assert(x => x > 0, 'Value must be positive'),
  RxDebugUtils.logChanges('Results'),
  RxDebugUtils.saveToVariable('lastResult')
);
```

---

## Production Debugging

### Safe Production Debugging

```typescript
// Production-safe debugging utilities
class ProductionDebugger {
  private static isDebugMode(): boolean {
    return !environment.production || 
           localStorage.getItem('enableDebug') === 'true' ||
           window.location.search.includes('debug=true');
  }
  
  static safeDebug<T>(options: {
    name: string;
    level?: 'log' | 'info' | 'warn' | 'error';
    sampleRate?: number; // 0-1, percentage of calls to actually log
    maxLogs?: number; // Maximum number of logs to output
  }) {
    const { name, level = 'log', sampleRate = 1, maxLogs = 100 } = options;
    let logCount = 0;
    
    return (source: Observable<T>) => {
      if (!this.isDebugMode()) {
        return source;
      }
      
      return source.pipe(
        tap({
          next: (value) => {
            if (Math.random() <= sampleRate && logCount < maxLogs) {
              console[level](`🔍 [${name}] Next:`, value);
              logCount++;
            }
          },
          error: (error) => {
            if (logCount < maxLogs) {
              console.error(`🔍 [${name}] Error:`, error);
              logCount++;
            }
          }
        })
      );
    };
  }
  
  static errorOnly<T>(name: string) {
    return (source: Observable<T>) => {
      return source.pipe(
        catchError((error) => {
          // Always log errors, even in production
          console.error(`❌ [${name}] Production Error:`, {
            message: error.message,
            timestamp: new Date().toISOString(),
            userAgent: navigator.userAgent,
            url: window.location.href
          });
          
          // Re-throw the error
          return throwError(() => error);
        })
      );
    };
  }
  
  static performanceOnly<T>(name: string, threshold: number = 1000) {
    return (source: Observable<T>) => {
      const startTime = performance.now();
      
      return source.pipe(
        finalize(() => {
          const duration = performance.now() - startTime;
          if (duration > threshold) {
            console.warn(`⚠️ [${name}] Slow operation: ${duration.toFixed(2)}ms`);
          }
        })
      );
    };
  }
}

// Remote debugging for production issues
class RemoteDebugger {
  private static endpoint = '/api/debug-logs';
  
  static remoteLog<T>(name: string, userId?: string) {
    return (source: Observable<T>) => {
      if (!environment.production) {
        return source;
      }
      
      return source.pipe(
        tap({
          error: (error) => {
            this.sendLog({
              level: 'error',
              name,
              message: error.message,
              stack: error.stack,
              timestamp: new Date().toISOString(),
              userId,
              sessionId: this.getSessionId(),
              metadata: {
                url: window.location.href,
                userAgent: navigator.userAgent
              }
            });
          }
        })
      );
    };
  }
  
  private static sendLog(logData: any): void {
    fetch(this.endpoint, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(logData)
    }).catch(() => {
      // Silently fail in production
    });
  }
  
  private static getSessionId(): string {
    let sessionId = sessionStorage.getItem('debugSessionId');
    if (!sessionId) {
      sessionId = Math.random().toString(36).substr(2, 9);
      sessionStorage.setItem('debugSessionId', sessionId);
    }
    return sessionId;
  }
}

// Usage in production
const productionStream$ = source$.pipe(
  ProductionDebugger.errorOnly('CriticalUserFlow'),
  ProductionDebugger.performanceOnly('APICall', 2000),
  RemoteDebugger.remoteLog('UserAction', currentUserId),
  ProductionDebugger.safeDebug({
    name: 'UserService',
    sampleRate: 0.01, // Log only 1% of calls
    maxLogs: 10
  })
);
```

---

## Summary

This lesson covered comprehensive RxJS debugging techniques and tools:

### Key Debugging Concepts:
1. **Debugging Fundamentals** - Basic principles and lifecycle debugging
2. **Browser Developer Tools** - Console strategies and performance profiling
3. **Custom Debugging Operators** - Building specialized debugging utilities
4. **Logging and Tracing** - Structured logging and distributed tracing
5. **Debugging Tools & Extensions** - Browser extensions and custom panels
6. **Common Debugging Scenarios** - Subscription, timing, and pipeline issues
7. **Performance Debugging** - Monitoring and metrics collection
8. **Error Debugging** - Error tracking and analysis
9. **Debugging Utilities** - Helper functions and development tools
10. **Production Debugging** - Safe production debugging strategies

### Essential Debugging Techniques:
- **Lifecycle monitoring** for subscription management
- **Visual debugging** with ASCII timelines
- **Performance profiling** with metrics collection
- **Error tracking** with context preservation
- **Conditional debugging** for environment-specific logging
- **Remote debugging** for production issue analysis

### Best Practices:
- Use structured logging for better analysis
- Implement conditional debugging for different environments
- Monitor performance metrics continuously
- Track errors with full context
- Create reusable debugging utilities
- Implement safe production debugging
- Use visual techniques for complex timing issues

Effective debugging is crucial for maintaining robust RxJS applications and quickly resolving issues in both development and production environments.

---

## Exercises

1. **Custom Debug Operator**: Create a debug operator that logs marble diagram representations
2. **Performance Monitor**: Build a performance monitoring system for observable chains
3. **Error Tracker**: Implement comprehensive error tracking with context preservation
4. **Debug Panel**: Create a live debugging panel for Angular applications
5. **Production Debugger**: Build a production-safe debugging system with sampling
6. **Memory Leak Detector**: Create utilities to detect and prevent memory leaks
7. **Timing Analyzer**: Build tools to analyze and visualize observable timing

## Additional Resources

- [RxJS Debugging Guide](https://rxjs.dev/guide/debugging)
- [Chrome DevTools for RxJS](https://developers.google.com/web/tools/chrome-devtools)
- [Redux DevTools Extension](https://github.com/reduxjs/redux-devtools)
- [RxJS Debugging Best Practices](https://blog.angular.io/debugging-rxjs-4f0b679cd882)
