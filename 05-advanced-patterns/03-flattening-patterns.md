# Flattening Patterns

## Learning Objectives
- Master advanced flattening techniques and patterns
- Implement conditional and dynamic flattening strategies
- Build complex data flow architectures
- Handle multiple observable sources efficiently
- Optimize flattening for performance and memory usage
- Debug and troubleshoot flattening issues

## Introduction to Advanced Flattening

While basic flattening operators (mergeMap, switchMap, concatMap, exhaustMap) handle most scenarios, complex applications often require sophisticated flattening patterns that combine multiple strategies, handle conditional logic, and optimize performance.

### Advanced Flattening Concepts
- **Conditional Flattening**: Different strategies based on data
- **Hierarchical Flattening**: Multi-level observable nesting
- **Selective Flattening**: Filtering and routing observables
- **Adaptive Flattening**: Dynamic strategy selection
- **Composite Flattening**: Combining multiple flattening approaches

## Conditional Flattening Patterns

### Strategy Selection Based on Data

```typescript
// conditional-flattening.ts
import { Observable, of, EMPTY, throwError } from 'rxjs';
import { mergeMap, switchMap, concatMap, exhaustMap } from 'rxjs/operators';

interface DataRequest {
  type: 'immediate' | 'cancellable' | 'sequential' | 'rate-limited';
  payload: any;
  priority: number;
}

/**
 * Dynamic flattening strategy based on request type
 */
const processRequests$ = (requests$: Observable<DataRequest>) => {
  return requests$.pipe(
    mergeMap(request => {
      switch (request.type) {
        case 'immediate':
          // Use mergeMap for parallel processing
          return of(request).pipe(
            mergeMap(req => this.processImmediate(req))
          );
        
        case 'cancellable':
          // Use switchMap for cancellable operations
          return of(request).pipe(
            switchMap(req => this.processCancellable(req))
          );
        
        case 'sequential':
          // Use concatMap for ordered processing
          return of(request).pipe(
            concatMap(req => this.processSequential(req))
          );
        
        case 'rate-limited':
          // Use exhaustMap to prevent flooding
          return of(request).pipe(
            exhaustMap(req => this.processRateLimited(req))
          );
        
        default:
          return EMPTY;
      }
    })
  );
};

/**
 * Priority-based flattening
 */
interface PriorityTask {
  id: string;
  priority: 'high' | 'medium' | 'low';
  operation: () => Observable<any>;
}

const processByPriority$ = (tasks$: Observable<PriorityTask>) => {
  const highPriority$ = tasks$.pipe(
    filter(task => task.priority === 'high'),
    switchMap(task => task.operation()) // Cancel previous high priority tasks
  );

  const mediumPriority$ = tasks$.pipe(
    filter(task => task.priority === 'medium'),
    mergeMap(task => task.operation(), 2) // Limit concurrency
  );

  const lowPriority$ = tasks$.pipe(
    filter(task => task.priority === 'low'),
    concatMap(task => task.operation()) // Process sequentially
  );

  return merge(highPriority$, mediumPriority$, lowPriority$);
};

// Usage
const taskStream$ = new Subject<PriorityTask>();

processByPriority$(taskStream$).subscribe(result => {
  console.log('Task completed:', result);
});
```

### Content-Based Routing

```typescript
// content-routing.ts

interface Message {
  type: 'user' | 'system' | 'error' | 'analytics';
  content: any;
  timestamp: Date;
}

/**
 * Route messages to different processing pipelines
 */
const routeMessages$ = (messages$: Observable<Message>) => {
  const userMessages$ = messages$.pipe(
    filter(msg => msg.type === 'user'),
    switchMap(msg => this.processUserMessage(msg))
  );

  const systemMessages$ = messages$.pipe(
    filter(msg => msg.type === 'system'),
    concatMap(msg => this.processSystemMessage(msg)) // Maintain order
  );

  const errorMessages$ = messages$.pipe(
    filter(msg => msg.type === 'error'),
    mergeMap(msg => this.processErrorMessage(msg).pipe(
      retry(3),
      catchError(error => {
        console.error('Failed to process error message:', error);
        return of({ type: 'PROCESSING_FAILED', original: msg });
      })
    ))
  );

  const analyticsMessages$ = messages$.pipe(
    filter(msg => msg.type === 'analytics'),
    bufferTime(5000), // Batch analytics messages
    filter(batch => batch.length > 0),
    mergeMap(batch => this.processAnalyticsBatch(batch))
  );

  return merge(
    userMessages$,
    systemMessages$,
    errorMessages$,
    analyticsMessages$
  );
};

/**
 * Dynamic routing based on content analysis
 */
const intelligentRouting$ = (data$: Observable<any>) => {
  return data$.pipe(
    mergeMap(data => {
      // Analyze data to determine processing strategy
      const analysis = this.analyzeData(data);
      
      if (analysis.isLargeDataset) {
        // Process large datasets in chunks
        return this.chunkProcessor(data).pipe(
          concatMap(chunk => this.processChunk(chunk))
        );
      } else if (analysis.isRealTime) {
        // Real-time data needs immediate processing
        return this.processRealTime(data);
      } else if (analysis.requiresValidation) {
        // Validate before processing
        return this.validateData(data).pipe(
          switchMap(validatedData => this.processValidated(validatedData))
        );
      } else {
        // Standard processing
        return this.processStandard(data);
      }
    })
  );
};
```

## Hierarchical Flattening Patterns

### Multi-Level Observable Nesting

```typescript
// hierarchical-flattening.ts

interface Organization {
  id: string;
  departments: string[];
}

interface Department {
  id: string;
  teams: string[];
}

interface Team {
  id: string;
  members: string[];
}

interface Employee {
  id: string;
  name: string;
  role: string;
}

/**
 * Deep hierarchical data loading with proper flattening
 */
const loadOrganizationHierarchy$ = (orgId: string) => {
  return this.orgService.getOrganization(orgId).pipe(
    switchMap(org => 
      // Load all departments in parallel
      combineLatest(
        org.departments.map(deptId => 
          this.deptService.getDepartment(deptId).pipe(
            switchMap(dept => 
              // Load teams for each department in parallel
              combineLatest(
                dept.teams.map(teamId =>
                  this.teamService.getTeam(teamId).pipe(
                    switchMap(team =>
                      // Load employees for each team in parallel
                      combineLatest(
                        team.members.map(memberId =>
                          this.employeeService.getEmployee(memberId)
                        )
                      ).pipe(
                        map(employees => ({ ...team, employees }))
                      )
                    )
                  )
                )
              ).pipe(
                map(teams => ({ ...dept, teams }))
              )
            )
          )
        )
      ).pipe(
        map(departments => ({ ...org, departments }))
      )
    )
  );
};

/**
 * Optimized hierarchical loading with batching
 */
const optimizedHierarchyLoad$ = (orgId: string) => {
  return this.orgService.getOrganization(orgId).pipe(
    switchMap(org => {
      // Collect all IDs first
      const allDeptIds = org.departments;
      
      return this.deptService.getDepartmentsBatch(allDeptIds).pipe(
        switchMap(departments => {
          const allTeamIds = departments.flatMap(dept => dept.teams);
          
          return this.teamService.getTeamsBatch(allTeamIds).pipe(
            switchMap(teams => {
              const allEmployeeIds = teams.flatMap(team => team.members);
              
              return this.employeeService.getEmployeesBatch(allEmployeeIds).pipe(
                map(employees => {
                  // Reconstruct hierarchy
                  const employeeMap = new Map(employees.map(emp => [emp.id, emp]));
                  const teamMap = new Map(teams.map(team => [team.id, {
                    ...team,
                    employees: team.members.map(id => employeeMap.get(id)).filter(Boolean)
                  }]));
                  const deptMap = new Map(departments.map(dept => [dept.id, {
                    ...dept,
                    teams: dept.teams.map(id => teamMap.get(id)).filter(Boolean)
                  }]));
                  
                  return {
                    ...org,
                    departments: org.departments.map(id => deptMap.get(id)).filter(Boolean)
                  };
                })
              );
            })
          );
        })
      );
    })
  );
};
```

### Tree Structure Flattening

```typescript
// tree-flattening.ts

interface TreeNode {
  id: string;
  children?: string[];
  data: any;
}

/**
 * Flatten tree structure with depth-first traversal
 */
const flattenTreeDepthFirst$ = (rootId: string): Observable<TreeNode[]> => {
  const traverse = (nodeId: string): Observable<TreeNode[]> => {
    return this.nodeService.getNode(nodeId).pipe(
      switchMap(node => {
        if (!node.children || node.children.length === 0) {
          return of([node]);
        }
        
        return combineLatest(
          node.children.map(childId => traverse(childId))
        ).pipe(
          map(childResults => [node, ...childResults.flat()])
        );
      })
    );
  };
  
  return traverse(rootId);
};

/**
 * Breadth-first tree flattening with level tracking
 */
const flattenTreeBreadthFirst$ = (rootId: string): Observable<{node: TreeNode, level: number}[]> => {
  return new Observable(subscriber => {
    const queue: {id: string, level: number}[] = [{id: rootId, level: 0}];
    const results: {node: TreeNode, level: number}[] = [];
    
    const processNext = () => {
      if (queue.length === 0) {
        subscriber.next(results);
        subscriber.complete();
        return;
      }
      
      const {id, level} = queue.shift()!;
      
      this.nodeService.getNode(id).subscribe({
        next: node => {
          results.push({node, level});
          
          if (node.children) {
            queue.push(...node.children.map(childId => ({id: childId, level: level + 1})));
          }
          
          processNext();
        },
        error: err => subscriber.error(err)
      });
    };
    
    processNext();
  });
};

/**
 * Lazy tree loading with on-demand expansion
 */
const lazyTreeLoader$ = (nodeId: string, maxDepth: number = 3) => {
  const loadLevel = (id: string, currentDepth: number): Observable<TreeNode> => {
    return this.nodeService.getNode(id).pipe(
      switchMap(node => {
        if (currentDepth >= maxDepth || !node.children) {
          return of(node);
        }
        
        // Load children lazily
        return combineLatest(
          node.children.map(childId => 
            loadLevel(childId, currentDepth + 1)
          )
        ).pipe(
          map(children => ({
            ...node,
            expandedChildren: children
          }))
        );
      })
    );
  };
  
  return loadLevel(nodeId, 0);
};
```

## Performance-Optimized Flattening

### Concurrency Control and Batching

```typescript
// performance-flattening.ts

/**
 * Controlled concurrency with queue management
 */
const controlledConcurrency$ = <T, R>(
  source$: Observable<T>,
  processor: (item: T) => Observable<R>,
  maxConcurrent: number = 3,
  queueSize: number = 100
): Observable<R> => {
  return new Observable<R>(subscriber => {
    const activeStreams = new Set<Subscription>();
    const queue: T[] = [];
    let isCompleted = false;
    let hasErrored = false;

    const processNext = () => {
      while (activeStreams.size < maxConcurrent && queue.length > 0) {
        const item = queue.shift()!;
        
        const subscription = processor(item).subscribe({
          next: result => subscriber.next(result),
          error: error => {
            if (!hasErrored) {
              hasErrored = true;
              activeStreams.forEach(sub => sub.unsubscribe());
              subscriber.error(error);
            }
          },
          complete: () => {
            activeStreams.delete(subscription);
            
            if (isCompleted && activeStreams.size === 0 && queue.length === 0) {
              subscriber.complete();
            } else {
              processNext();
            }
          }
        });
        
        activeStreams.add(subscription);
      }
    };

    const sourceSubscription = source$.subscribe({
      next: item => {
        if (queue.length < queueSize) {
          queue.push(item);
          processNext();
        } else {
          console.warn('Queue full, dropping item:', item);
        }
      },
      error: error => {
        if (!hasErrored) {
          hasErrored = true;
          activeStreams.forEach(sub => sub.unsubscribe());
          subscriber.error(error);
        }
      },
      complete: () => {
        isCompleted = true;
        if (activeStreams.size === 0 && queue.length === 0) {
          subscriber.complete();
        }
      }
    });

    return () => {
      sourceSubscription.unsubscribe();
      activeStreams.forEach(sub => sub.unsubscribe());
    };
  });
};

/**
 * Adaptive batching based on system resources
 */
class AdaptiveBatcher<T> {
  private batchSize = 10;
  private readonly minBatchSize = 1;
  private readonly maxBatchSize = 100;
  private processingTimes: number[] = [];

  adjustBatchSize(processingTime: number): void {
    this.processingTimes.push(processingTime);
    
    // Keep only last 10 measurements
    if (this.processingTimes.length > 10) {
      this.processingTimes.shift();
    }

    const averageTime = this.processingTimes.reduce((a, b) => a + b, 0) / this.processingTimes.length;
    
    if (averageTime > 1000) { // If processing takes > 1 second
      this.batchSize = Math.max(this.minBatchSize, this.batchSize - 1);
    } else if (averageTime < 100) { // If processing is fast
      this.batchSize = Math.min(this.maxBatchSize, this.batchSize + 1);
    }
  }

  getBatchSize(): number {
    return this.batchSize;
  }
}

const adaptiveBatchProcessor$ = <T, R>(
  source$: Observable<T>,
  processor: (batch: T[]) => Observable<R>
): Observable<R> => {
  const batcher = new AdaptiveBatcher<T>();
  
  return source$.pipe(
    bufferCount(batcher.getBatchSize()),
    mergeMap(batch => {
      const startTime = Date.now();
      
      return processor(batch).pipe(
        tap(() => {
          const processingTime = Date.now() - startTime;
          batcher.adjustBatchSize(processingTime);
        })
      );
    })
  );
};
```

### Memory-Efficient Flattening

```typescript
// memory-efficient.ts

/**
 * Streaming flattening for large datasets
 */
const streamingFlatten$ = <T, R>(
  source$: Observable<T>,
  processor: (item: T) => Observable<R>,
  bufferSize: number = 1000
): Observable<R> => {
  return source$.pipe(
    // Use a small buffer to prevent memory buildup
    bufferCount(bufferSize),
    concatMap(batch => 
      from(batch).pipe(
        mergeMap(item => processor(item), 5) // Limit concurrency
      )
    )
  );
};

/**
 * Windowed processing with cleanup
 */
const windowedProcessor$ = <T, R>(
  source$: Observable<T>,
  windowSize: number,
  processor: (window: T[]) => Observable<R>
): Observable<R> => {
  return source$.pipe(
    scan((acc, curr) => {
      acc.push(curr);
      if (acc.length > windowSize) {
        acc.shift(); // Remove oldest item
      }
      return acc;
    }, [] as T[]),
    filter(window => window.length === windowSize),
    distinctUntilChanged((prev, curr) => 
      // Only process if window actually changed
      prev.length === curr.length && 
      prev.every((item, index) => item === curr[index])
    ),
    switchMap(window => processor([...window])) // Copy to prevent mutations
  );
};

/**
 * Resource-aware processing
 */
interface SystemResources {
  availableMemory: number;
  cpuUsage: number;
}

const resourceAwareProcessor$ = <T, R>(
  source$: Observable<T>,
  processor: (items: T[]) => Observable<R>,
  getSystemResources: () => SystemResources
): Observable<R> => {
  return source$.pipe(
    bufferTime(1000),
    filter(batch => batch.length > 0),
    mergeMap(batch => {
      const resources = getSystemResources();
      
      // Adjust processing based on available resources
      let batchSize: number;
      if (resources.availableMemory < 0.2 || resources.cpuUsage > 0.8) {
        batchSize = Math.max(1, Math.floor(batch.length / 4)); // Process in smaller chunks
      } else if (resources.availableMemory > 0.8 && resources.cpuUsage < 0.3) {
        batchSize = batch.length; // Process all at once
      } else {
        batchSize = Math.floor(batch.length / 2); // Process half
      }
      
      // Split batch based on available resources
      const chunks: T[][] = [];
      for (let i = 0; i < batch.length; i += batchSize) {
        chunks.push(batch.slice(i, i + batchSize));
      }
      
      return from(chunks).pipe(
        concatMap(chunk => processor(chunk))
      );
    })
  );
};
```

## Error Handling in Complex Flattening

### Resilient Flattening Patterns

```typescript
// resilient-flattening.ts

/**
 * Fault-tolerant flattening with fallback strategies
 */
const resilientFlatten$ = <T, R>(
  source$: Observable<T>,
  primaryProcessor: (item: T) => Observable<R>,
  fallbackProcessor?: (item: T) => Observable<R>
): Observable<R> => {
  return source$.pipe(
    mergeMap(item => 
      primaryProcessor(item).pipe(
        retry({
          count: 3,
          delay: (error, retryCount) => {
            console.log(`Retry ${retryCount} for item:`, item);
            return timer(Math.pow(2, retryCount) * 1000);
          }
        }),
        catchError(error => {
          console.error('Primary processor failed:', error);
          
          if (fallbackProcessor) {
            return fallbackProcessor(item).pipe(
              catchError(fallbackError => {
                console.error('Fallback processor also failed:', fallbackError);
                return of(null as any); // Or throw, depending on requirements
              })
            );
          }
          
          return of(null as any);
        })
      )
    ),
    filter(result => result !== null)
  );
};

/**
 * Circuit breaker pattern for flattening
 */
class CircuitBreakerState {
  private failures = 0;
  private lastFailureTime = 0;
  private state: 'closed' | 'open' | 'half-open' = 'closed';
  
  constructor(
    private readonly failureThreshold = 5,
    private readonly recoveryTimeoutMs = 30000
  ) {}

  canExecute(): boolean {
    if (this.state === 'closed') return true;
    
    if (this.state === 'open') {
      if (Date.now() - this.lastFailureTime > this.recoveryTimeoutMs) {
        this.state = 'half-open';
        return true;
      }
      return false;
    }
    
    // half-open state
    return true;
  }

  onSuccess(): void {
    this.failures = 0;
    this.state = 'closed';
  }

  onFailure(): void {
    this.failures++;
    this.lastFailureTime = Date.now();
    
    if (this.failures >= this.failureThreshold) {
      this.state = 'open';
    }
  }

  getState(): string {
    return this.state;
  }
}

const circuitBreakerFlatten$ = <T, R>(
  source$: Observable<T>,
  processor: (item: T) => Observable<R>
): Observable<R> => {
  const circuitBreaker = new CircuitBreakerState();
  
  return source$.pipe(
    mergeMap(item => {
      if (!circuitBreaker.canExecute()) {
        console.warn('Circuit breaker is open, skipping item:', item);
        return EMPTY;
      }
      
      return processor(item).pipe(
        tap(() => circuitBreaker.onSuccess()),
        catchError(error => {
          circuitBreaker.onFailure();
          console.error(`Processing failed (circuit breaker: ${circuitBreaker.getState()}):`, error);
          return EMPTY;
        })
      );
    })
  );
};
```

## Testing Flattening Patterns

### Comprehensive Testing Strategies

```typescript
// flattening-tests.spec.ts
import { TestScheduler } from 'rxjs/testing';

describe('Advanced Flattening Patterns', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  describe('Conditional Flattening', () => {
    it('should use different strategies based on data type', () => {
      testScheduler.run(({ cold, hot, expectObservable }) => {
        const requests = hot('a-b-c-|', {
          a: { type: 'immediate', data: 1 },
          b: { type: 'cancellable', data: 2 },
          c: { type: 'sequential', data: 3 }
        });

        const mockProcessors = {
          immediate: cold('--x|'),
          cancellable: cold('---y|'),
          sequential: cold('----z|')
        };

        const expected = '--x-y----z|';

        const result = requests.pipe(
          mergeMap(req => {
            switch (req.type) {
              case 'immediate': return mockProcessors.immediate;
              case 'cancellable': return mockProcessors.cancellable;
              case 'sequential': return mockProcessors.sequential;
              default: return EMPTY;
            }
          })
        );

        expectObservable(result).toBe(expected, { x: 'x', y: 'y', z: 'z' });
      });
    });
  });

  describe('Hierarchical Flattening', () => {
    it('should handle nested data loading correctly', () => {
      testScheduler.run(({ cold, hot, expectObservable }) => {
        const rootRequest = hot('a|');
        const childRequests = {
          a: cold('--b|', { b: ['c', 'd'] })
        };
        const leafRequests = {
          c: cold('  --x|'),
          d: cold('  ---y|')
        };

        const expected = '-----(xy)|';

        const result = rootRequest.pipe(
          switchMap(key => 
            childRequests[key].pipe(
              switchMap(children => 
                combineLatest(
                  children.map(child => leafRequests[child])
                )
              )
            )
          )
        );

        expectObservable(result).toBe(expected, { x: 'x', y: 'y' });
      });
    });
  });

  describe('Performance Optimization', () => {
    it('should respect concurrency limits', () => {
      testScheduler.run(({ cold, hot, expectObservable }) => {
        const source = hot('abc|');
        const processor = cold('--x|');
        
        // With concurrency limit of 2
        const expected = '--x-x-x|';

        const result = source.pipe(
          mergeMap(() => processor, 2)
        );

        expectObservable(result).toBe(expected, { x: 'x' });
      });
    });
  });
});
```

## Best Practices

### 1. Strategy Selection
- Analyze data flow requirements before choosing flattening strategy
- Consider combining multiple strategies for complex scenarios
- Use conditional flattening for dynamic requirements
- Profile performance with different approaches

### 2. Error Handling
- Implement proper error boundaries in nested flattening
- Use circuit breaker patterns for external dependencies
- Provide fallback strategies for critical operations
- Log errors with sufficient context for debugging

### 3. Performance
- Monitor memory usage in long-running flattening operations
- Implement backpressure handling for high-volume streams
- Use adaptive batching for variable load scenarios
- Consider resource constraints in processing decisions

### 4. Testing
- Test different data flow scenarios comprehensively
- Use marble testing for complex timing scenarios
- Mock external dependencies for isolated testing
- Verify error handling and recovery behaviors

### 5. Monitoring
- Implement metrics for flattening performance
- Monitor subscription counts and memory usage
- Track error rates and recovery success
- Use distributed tracing for complex hierarchies

## Summary

Advanced flattening patterns enable sophisticated reactive architectures that handle complex data flows efficiently. Key takeaways:

- Choose flattening strategies based on data characteristics and requirements
- Implement conditional and adaptive flattening for dynamic scenarios
- Optimize performance with concurrency control and batching
- Handle errors gracefully with circuit breakers and fallbacks
- Use hierarchical patterns for nested data structures
- Test thoroughly with comprehensive scenarios
- Monitor performance and resource usage in production

These patterns form the foundation for building scalable, resilient reactive applications that can handle complex real-world data flow requirements.
