# Backpressure & Buffering Strategies

## Learning Objectives
- Understand backpressure concepts and challenges
- Master buffering strategies for high-volume data streams
- Implement flow control mechanisms
- Handle slow consumers and fast producers
- Build resilient streaming architectures
- Optimize memory usage in high-throughput scenarios

## Introduction to Backpressure

Backpressure occurs when data producers generate information faster than consumers can process it. This is a critical concern in reactive systems where uncontrolled data flow can lead to memory exhaustion, performance degradation, and system instability.

### Core Concepts
- **Producer**: Source of data (API, user input, sensors)
- **Consumer**: Data processor (business logic, UI updates, storage)
- **Buffer**: Temporary storage between producer and consumer
- **Flow Control**: Mechanisms to manage data flow rate
- **Dropping**: Discarding data when buffers are full

### Common Scenarios
- High-frequency sensor data
- Real-time analytics streams
- User interface events (clicks, scrolling)
- Network data transmission
- File processing pipelines

## Understanding Buffer Strategies

### Time-Based Buffering

```typescript
// time-based-buffering.ts
import { Observable, Subject, interval, timer } from 'rxjs';
import { buffer, bufferTime, bufferCount, bufferToggle, bufferWhen } from 'rxjs/operators';

/**
 * Fixed time window buffering
 */
const timeBasedBuffer$ = (source$: Observable<any>, windowMs: number) => {
  return source$.pipe(
    bufferTime(windowMs),
    filter(buffer => buffer.length > 0) // Only emit non-empty buffers
  );
};

// Example: Collecting user interactions every 5 seconds
const userInteractions$ = new Subject<{type: string, timestamp: number}>();

const batchedInteractions$ = timeBasedBuffer$(userInteractions$, 5000).pipe(
  map(interactions => ({
    batch: interactions,
    count: interactions.length,
    timespan: interactions.length > 0 ? 
      interactions[interactions.length - 1].timestamp - interactions[0].timestamp : 0
  }))
);

batchedInteractions$.subscribe(batch => {
  console.log(`Processing ${batch.count} interactions over ${batch.timespan}ms`);
  this.analyticsService.sendBatch(batch.batch);
});

/**
 * Sliding time window buffering
 */
const slidingWindowBuffer$ = <T>(
  source$: Observable<T>, 
  windowMs: number, 
  slideMs: number
) => {
  return timer(0, slideMs).pipe(
    switchMap(() => 
      source$.pipe(
        bufferTime(windowMs),
        take(1)
      )
    ),
    filter(buffer => buffer.length > 0)
  );
};

// Example: Real-time metrics with sliding window
const metricEvents$ = new Subject<{value: number, timestamp: number}>();

const slidingMetrics$ = slidingWindowBuffer$(metricEvents$, 10000, 2000).pipe(
  map(events => ({
    average: events.reduce((sum, evt) => sum + evt.value, 0) / events.length,
    count: events.length,
    min: Math.min(...events.map(evt => evt.value)),
    max: Math.max(...events.map(evt => evt.value))
  }))
);

slidingMetrics$.subscribe(stats => {
  this.updateDashboard(stats);
});

/**
 * Conditional time buffering
 */
const conditionalTimeBuffer$ = <T>(
  source$: Observable<T>,
  condition: (items: T[]) => boolean,
  maxTimeMs: number
) => {
  return source$.pipe(
    scan((acc: {items: T[], startTime: number}, curr: T) => {
      const now = Date.now();
      
      // Reset buffer if max time exceeded
      if (now - acc.startTime > maxTimeMs) {
        return { items: [curr], startTime: now };
      }
      
      return { items: [...acc.items, curr], startTime: acc.startTime };
    }, { items: [], startTime: Date.now() }),
    filter(acc => condition(acc.items) || Date.now() - acc.startTime > maxTimeMs),
    map(acc => acc.items),
    distinctUntilChanged((prev, curr) => prev.length === curr.length)
  );
};

// Example: Buffer until 100 items or 30 seconds
const adaptiveBuffer$ = conditionalTimeBuffer$(
  dataStream$,
  items => items.length >= 100,
  30000
);
```

### Count-Based Buffering

```typescript
// count-based-buffering.ts

/**
 * Fixed count buffering with overflow handling
 */
const countBasedBuffer$ = <T>(
  source$: Observable<T>, 
  count: number,
  overflowStrategy: 'drop' | 'slide' | 'error' = 'slide'
) => {
  return source$.pipe(
    scan((buffer: T[], item: T) => {
      const newBuffer = [...buffer, item];
      
      if (newBuffer.length > count) {
        switch (overflowStrategy) {
          case 'drop':
            return buffer; // Don't add new item
          case 'slide':
            return newBuffer.slice(-count); // Keep last N items
          case 'error':
            throw new Error(`Buffer overflow: exceeded ${count} items`);
          default:
            return newBuffer.slice(-count);
        }
      }
      
      return newBuffer;
    }, []),
    filter(buffer => buffer.length === count),
    distinctUntilChanged()
  );
};

/**
 * Dynamic count buffering based on processing speed
 */
class AdaptiveCountBuffer<T> {
  private processingTimes: number[] = [];
  private currentBufferSize: number;
  
  constructor(
    private initialSize: number = 10,
    private minSize: number = 1,
    private maxSize: number = 1000
  ) {
    this.currentBufferSize = initialSize;
  }

  updateBufferSize(processingTime: number): void {
    this.processingTimes.push(processingTime);
    
    // Keep last 10 measurements
    if (this.processingTimes.length > 10) {
      this.processingTimes.shift();
    }

    const avgTime = this.processingTimes.reduce((a, b) => a + b, 0) / this.processingTimes.length;
    
    // Adjust buffer size based on processing performance
    if (avgTime > 1000) { // Slow processing
      this.currentBufferSize = Math.max(this.minSize, this.currentBufferSize - 1);
    } else if (avgTime < 100) { // Fast processing
      this.currentBufferSize = Math.min(this.maxSize, this.currentBufferSize + 1);
    }
  }

  getBufferSize(): number {
    return this.currentBufferSize;
  }
}

const adaptiveCountBuffer$ = <T, R>(
  source$: Observable<T>,
  processor: (batch: T[]) => Observable<R>
): Observable<R> => {
  const adaptiveBuffer = new AdaptiveCountBuffer<T>();
  
  return source$.pipe(
    bufferCount(adaptiveBuffer.getBufferSize()),
    mergeMap(batch => {
      const startTime = Date.now();
      
      return processor(batch).pipe(
        tap(() => {
          const processingTime = Date.now() - startTime;
          adaptiveBuffer.updateBufferSize(processingTime);
        })
      );
    })
  );
};
```

### Conditional and Toggle Buffering

```typescript
// conditional-buffering.ts

/**
 * Buffer based on external signals
 */
const toggleBuffer$ = <T>(
  source$: Observable<T>,
  toggleSignal$: Observable<boolean>
) => {
  return source$.pipe(
    bufferToggle(
      toggleSignal$.pipe(filter(Boolean)), // Start buffering on true
      () => toggleSignal$.pipe(filter(signal => !signal)) // Stop on false
    )
  );
};

// Example: Buffer data during processing windows
const processingWindow$ = new BehaviorSubject<boolean>(false);
const dataStream$ = interval(100).pipe(map(i => `data-${i}`));

const windowedData$ = toggleBuffer$(dataStream$, processingWindow$);

// Control processing windows
setInterval(() => {
  processingWindow$.next(true);
  setTimeout(() => processingWindow$.next(false), 5000);
}, 10000);

/**
 * Conditional buffering based on data content
 */
const contentBasedBuffer$ = <T>(
  source$: Observable<T>,
  shouldFlush: (buffer: T[], newItem: T) => boolean
) => {
  return source$.pipe(
    scan((buffer: T[], item: T) => {
      const newBuffer = [...buffer, item];
      return shouldFlush(buffer, item) ? [] : newBuffer;
    }, []),
    filter(buffer => buffer.length === 0),
    scan((prev: T[], curr: T[]) => {
      // Emit the previous buffer when current becomes empty
      return curr.length === 0 && prev.length > 0 ? prev : curr;
    }, []),
    filter(buffer => buffer.length > 0)
  );
};

// Example: Buffer until specific message type
interface Message {
  type: string;
  data: any;
}

const messageBuffer$ = contentBasedBuffer$(
  messages$,
  (buffer, newMessage) => newMessage.type === 'FLUSH'
);

/**
 * State-based buffering
 */
interface BufferState {
  items: any[];
  lastFlush: number;
  condition: 'time' | 'count' | 'content';
}

const statefulBuffer$ = <T>(
  source$: Observable<T>,
  options: {
    maxCount: number;
    maxTimeMs: number;
    flushCondition?: (item: T) => boolean;
  }
) => {
  return source$.pipe(
    scan((state: BufferState, item: T) => {
      const now = Date.now();
      const newItems = [...state.items, item];
      
      // Check flush conditions
      const shouldFlushByCount = newItems.length >= options.maxCount;
      const shouldFlushByTime = now - state.lastFlush >= options.maxTimeMs;
      const shouldFlushByContent = options.flushCondition?.(item) || false;
      
      if (shouldFlushByCount || shouldFlushByTime || shouldFlushByContent) {
        return {
          items: [],
          lastFlush: now,
          condition: shouldFlushByCount ? 'count' : shouldFlushByTime ? 'time' : 'content'
        };
      }
      
      return { ...state, items: newItems };
    }, { items: [], lastFlush: Date.now(), condition: 'time' } as BufferState),
    filter(state => state.items.length === 0 && state.lastFlush > 0),
    map(state => ({ condition: state.condition, timestamp: state.lastFlush }))
  );
};
```

## Advanced Flow Control Mechanisms

### Throttling and Debouncing Strategies

```typescript
// flow-control.ts

/**
 * Adaptive throttling based on system load
 */
class AdaptiveThrottler {
  private currentInterval = 1000; // Start with 1 second
  private readonly minInterval = 100;
  private readonly maxInterval = 10000;
  private systemLoadSamples: number[] = [];

  updateInterval(systemLoad: number): void {
    this.systemLoadSamples.push(systemLoad);
    
    if (this.systemLoadSamples.length > 5) {
      this.systemLoadSamples.shift();
    }

    const avgLoad = this.systemLoadSamples.reduce((a, b) => a + b, 0) / this.systemLoadSamples.length;
    
    if (avgLoad > 0.8) { // High load
      this.currentInterval = Math.min(this.maxInterval, this.currentInterval * 1.5);
    } else if (avgLoad < 0.3) { // Low load
      this.currentInterval = Math.max(this.minInterval, this.currentInterval * 0.8);
    }
  }

  getCurrentInterval(): number {
    return this.currentInterval;
  }
}

const adaptiveThrottle$ = <T>(
  source$: Observable<T>,
  getSystemLoad: () => number
): Observable<T> => {
  const throttler = new AdaptiveThrottler();
  
  return source$.pipe(
    throttle(() => {
      const systemLoad = getSystemLoad();
      throttler.updateInterval(systemLoad);
      return timer(throttler.getCurrentInterval());
    })
  );
};

/**
 * Intelligent debouncing with burst detection
 */
const smartDebounce$ = <T>(
  source$: Observable<T>,
  normalDelay: number = 300,
  burstDelay: number = 100,
  burstThreshold: number = 5
) => {
  return source$.pipe(
    scan((acc: {items: T[], timestamps: number[]}, item: T) => {
      const now = Date.now();
      const recentTimestamps = acc.timestamps.filter(ts => now - ts < 1000);
      
      return {
        items: [...acc.items, item],
        timestamps: [...recentTimestamps, now]
      };
    }, {items: [], timestamps: []}),
    debounce(acc => {
      const isBurst = acc.timestamps.length >= burstThreshold;
      return timer(isBurst ? burstDelay : normalDelay);
    }),
    map(acc => acc.items),
    filter(items => items.length > 0)
  );
};

/**
 * Pressure-sensitive flow control
 */
interface PressureMetrics {
  queueSize: number;
  processingRate: number;
  errorRate: number;
}

const pressureSensitiveFlow$ = <T, R>(
  source$: Observable<T>,
  processor: (item: T) => Observable<R>,
  getPressureMetrics: () => PressureMetrics
): Observable<R> => {
  return source$.pipe(
    mergeMap(item => {
      const metrics = getPressureMetrics();
      
      // Adjust concurrency based on pressure
      let concurrency: number;
      if (metrics.queueSize > 1000 || metrics.errorRate > 0.1) {
        concurrency = 1; // Serialize under pressure
      } else if (metrics.queueSize < 100 && metrics.errorRate < 0.01) {
        concurrency = 10; // Parallelize when stable
      } else {
        concurrency = 3; // Moderate concurrency
      }
      
      return processor(item);
    }, (item, result, index) => result, 1) // Dynamic concurrency would need custom implementation
  );
};
```

### Dropping and Sampling Strategies

```typescript
// dropping-strategies.ts

/**
 * Intelligent dropping based on priority
 */
interface PriorityItem<T> {
  data: T;
  priority: number;
  timestamp: number;
}

const priorityDrop$ = <T>(
  source$: Observable<PriorityItem<T>>,
  maxBufferSize: number = 1000
): Observable<PriorityItem<T>> => {
  return source$.pipe(
    scan((buffer: PriorityItem<T>[], item: PriorityItem<T>) => {
      const newBuffer = [...buffer, item];
      
      if (newBuffer.length > maxBufferSize) {
        // Sort by priority (higher number = higher priority) and timestamp
        newBuffer.sort((a, b) => {
          if (a.priority !== b.priority) {
            return b.priority - a.priority;
          }
          return b.timestamp - a.timestamp; // Newer items first for same priority
        });
        
        // Keep only the highest priority items
        return newBuffer.slice(0, maxBufferSize);
      }
      
      return newBuffer;
    }, []),
    mergeMap(buffer => from(buffer))
  );
};

/**
 * Reservoir sampling for large streams
 */
const reservoirSample$ = <T>(
  source$: Observable<T>,
  sampleSize: number
): Observable<T[]> => {
  return source$.pipe(
    scan((acc: {reservoir: T[], count: number}, item: T) => {
      const newCount = acc.count + 1;
      
      if (acc.reservoir.length < sampleSize) {
        // Fill reservoir
        return {
          reservoir: [...acc.reservoir, item],
          count: newCount
        };
      } else {
        // Replace random item with probability sampleSize/count
        const randomIndex = Math.floor(Math.random() * newCount);
        if (randomIndex < sampleSize) {
          const newReservoir = [...acc.reservoir];
          newReservoir[randomIndex] = item;
          return {
            reservoir: newReservoir,
            count: newCount
          };
        }
        return { ...acc, count: newCount };
      }
    }, {reservoir: [], count: 0}),
    map(acc => acc.reservoir),
    distinctUntilChanged((prev, curr) => prev.length === curr.length)
  );
};

/**
 * Temporal sampling with decay
 */
const temporalSample$ = <T>(
  source$: Observable<T>,
  sampleRate: number = 0.1, // 10% sampling rate
  decayFactor: number = 0.95
): Observable<T> => {
  let currentRate = sampleRate;
  
  return source$.pipe(
    filter(() => {
      const shouldSample = Math.random() < currentRate;
      
      // Decay the sampling rate over time to reduce load
      currentRate *= decayFactor;
      
      // Reset rate periodically
      if (currentRate < sampleRate * 0.1) {
        currentRate = sampleRate;
      }
      
      return shouldSample;
    })
  );
};

/**
 * Load-based dropping
 */
const loadBasedDrop$ = <T>(
  source$: Observable<T>,
  getSystemLoad: () => number,
  dropThreshold: number = 0.8
): Observable<T> => {
  return source$.pipe(
    filter(() => {
      const systemLoad = getSystemLoad();
      
      if (systemLoad > dropThreshold) {
        // Drop items based on load level
        const dropProbability = (systemLoad - dropThreshold) / (1 - dropThreshold);
        return Math.random() > dropProbability;
      }
      
      return true; // Don't drop when load is acceptable
    })
  );
};
```

## Memory-Efficient Streaming

### Bounded Buffer Implementation

```typescript
// bounded-buffer.ts

class BoundedBuffer<T> {
  private buffer: T[] = [];
  private readonly subject = new Subject<T[]>();

  constructor(
    private readonly maxSize: number,
    private readonly overflowStrategy: 'dropOldest' | 'dropNewest' | 'error' = 'dropOldest'
  ) {}

  add(item: T): void {
    if (this.buffer.length >= this.maxSize) {
      switch (this.overflowStrategy) {
        case 'dropOldest':
          this.buffer.shift();
          break;
        case 'dropNewest':
          return; // Don't add the new item
        case 'error':
          throw new Error(`Buffer overflow: cannot add item to full buffer (size: ${this.maxSize})`);
      }
    }

    this.buffer.push(item);
  }

  flush(): T[] {
    const items = [...this.buffer];
    this.buffer.length = 0;
    this.subject.next(items);
    return items;
  }

  size(): number {
    return this.buffer.length;
  }

  asObservable(): Observable<T[]> {
    return this.subject.asObservable();
  }
}

/**
 * Streaming processor with bounded memory
 */
const boundedStreamProcessor$ = <T, R>(
  source$: Observable<T>,
  processor: (batch: T[]) => Observable<R>,
  options: {
    maxBufferSize: number;
    flushInterval: number;
    overflowStrategy?: 'dropOldest' | 'dropNewest' | 'error';
  }
): Observable<R> => {
  const buffer = new BoundedBuffer<T>(
    options.maxBufferSize, 
    options.overflowStrategy
  );

  // Auto-flush timer
  const flushTimer$ = timer(0, options.flushInterval).pipe(
    map(() => buffer.flush()),
    filter(batch => batch.length > 0)
  );

  // Source subscription
  const sourceSubscription = source$.subscribe(item => {
    try {
      buffer.add(item);
    } catch (error) {
      console.error('Buffer overflow:', error);
    }
  });

  return merge(
    flushTimer$,
    buffer.asObservable()
  ).pipe(
    mergeMap(batch => processor(batch)),
    finalize(() => sourceSubscription.unsubscribe())
  );
};

/**
 * Circular buffer for fixed memory usage
 */
class CircularBuffer<T> {
  private buffer: (T | undefined)[];
  private head = 0;
  private tail = 0;
  private full = false;

  constructor(private readonly capacity: number) {
    this.buffer = new Array(capacity);
  }

  push(item: T): T | undefined {
    const removed = this.full ? this.buffer[this.tail] : undefined;
    
    this.buffer[this.head] = item;
    this.head = (this.head + 1) % this.capacity;
    
    if (this.full) {
      this.tail = (this.tail + 1) % this.capacity;
    }
    
    this.full = this.head === this.tail;
    
    return removed;
  }

  toArray(): T[] {
    if (!this.full) {
      return this.buffer.slice(0, this.head).filter(item => item !== undefined) as T[];
    }
    
    const result: T[] = [];
    let index = this.tail;
    
    do {
      if (this.buffer[index] !== undefined) {
        result.push(this.buffer[index] as T);
      }
      index = (index + 1) % this.capacity;
    } while (index !== this.tail);
    
    return result;
  }

  size(): number {
    if (this.full) return this.capacity;
    return this.head >= this.tail ? this.head - this.tail : this.capacity - this.tail + this.head;
  }

  isFull(): boolean {
    return this.full;
  }
}

const circularBufferStream$ = <T>(
  source$: Observable<T>,
  bufferSize: number
): Observable<T[]> => {
  const circularBuffer = new CircularBuffer<T>(bufferSize);
  
  return source$.pipe(
    map(item => {
      circularBuffer.push(item);
      return circularBuffer.toArray();
    }),
    distinctUntilChanged((prev, curr) => prev.length === curr.length)
  );
};
```

## Real-World Applications

### High-Frequency Trading Data

```typescript
// hft-data-processing.ts

interface TradeData {
  symbol: string;
  price: number;
  volume: number;
  timestamp: number;
  exchange: string;
}

interface MarketData {
  symbol: string;
  bid: number;
  ask: number;
  volume: number;
  timestamp: number;
}

/**
 * High-frequency trading data processor with backpressure handling
 */
const processTradingData$ = (
  tradeStream$: Observable<TradeData>,
  marketStream$: Observable<MarketData>
) => {
  // Buffer trades by symbol to batch processing
  const tradesBySymbol$ = tradeStream$.pipe(
    groupBy(trade => trade.symbol),
    mergeMap(group$ => 
      group$.pipe(
        bufferTime(100), // 100ms windows
        filter(trades => trades.length > 0),
        map(trades => ({
          symbol: group$.key,
          trades,
          avgPrice: trades.reduce((sum, t) => sum + t.price, 0) / trades.length,
          totalVolume: trades.reduce((sum, t) => sum + t.volume, 0)
        }))
      )
    )
  );

  // Sample market data to reduce load
  const sampledMarketData$ = marketStream$.pipe(
    groupBy(data => data.symbol),
    mergeMap(group$ =>
      group$.pipe(
        throttleTime(50), // Max one update per 50ms per symbol
        distinctUntilChanged((prev, curr) => 
          Math.abs(prev.bid - curr.bid) < 0.01 && 
          Math.abs(prev.ask - curr.ask) < 0.01
        )
      )
    )
  );

  return combineLatest([
    tradesBySymbol$,
    sampledMarketData$
  ]).pipe(
    filter(([trades, market]) => trades.symbol === market.symbol),
    map(([trades, market]) => ({
      symbol: trades.symbol,
      trades: trades.trades,
      marketData: market,
      spread: market.ask - market.bid,
      priceDeviation: Math.abs(trades.avgPrice - (market.bid + market.ask) / 2)
    }))
  );
};
```

### IoT Sensor Data Processing

```typescript
// iot-sensor-processing.ts

interface SensorReading {
  sensorId: string;
  value: number;
  timestamp: number;
  quality: 'good' | 'poor' | 'bad';
}

/**
 * IoT sensor data processing with adaptive sampling
 */
const processIoTData$ = (sensorStream$: Observable<SensorReading>) => {
  // Adaptive sampling based on value changes
  const adaptiveSample$ = sensorStream$.pipe(
    groupBy(reading => reading.sensorId),
    mergeMap(group$ => {
      let lastSignificantValue: number | null = null;
      
      return group$.pipe(
        filter(reading => {
          // Always include good quality readings
          if (reading.quality === 'good') {
            // Check for significant change
            if (lastSignificantValue === null || 
                Math.abs(reading.value - lastSignificantValue) > 0.1) {
              lastSignificantValue = reading.value;
              return true;
            }
          }
          
          // Drop poor/bad quality readings unless they're critical
          return reading.quality === 'good' && Math.random() < 0.1; // Sample 10% of non-significant readings
        })
      );
    })
  );

  // Aggregate readings into time windows
  const aggregatedData$ = adaptiveSample$.pipe(
    groupBy(reading => reading.sensorId),
    mergeMap(group$ =>
      group$.pipe(
        bufferTime(60000), // 1-minute windows
        filter(readings => readings.length > 0),
        map(readings => ({
          sensorId: group$.key,
          timeWindow: {
            start: Math.min(...readings.map(r => r.timestamp)),
            end: Math.max(...readings.map(r => r.timestamp))
          },
          statistics: {
            avg: readings.reduce((sum, r) => sum + r.value, 0) / readings.length,
            min: Math.min(...readings.map(r => r.value)),
            max: Math.max(...readings.map(r => r.value)),
            count: readings.length,
            qualityScore: readings.filter(r => r.quality === 'good').length / readings.length
          }
        }))
      )
    )
  );

  return aggregatedData$;
};
```

## Testing Backpressure Scenarios

### Load Testing with Observables

```typescript
// backpressure-testing.spec.ts

describe('Backpressure Handling', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should handle high-frequency data with buffering', () => {
    testScheduler.run(({ cold, hot, expectObservable }) => {
      // Simulate high-frequency source
      const source = hot('abcdefghijklmnop|');
      
      // Buffer every 3 items
      const buffered = source.pipe(bufferCount(3));
      
      const expected = '---x---y---z---(w|)';
      const values = {
        x: ['a', 'b', 'c'],
        y: ['d', 'e', 'f'],
        z: ['g', 'h', 'i'],
        w: ['j', 'k', 'l']
      };

      expectObservable(buffered).toBe(expected, values);
    });
  });

  it('should drop items under high load', () => {
    testScheduler.run(({ cold, hot, expectObservable }) => {
      let dropRate = 0;
      
      const source = hot('abcdefghij|');
      const dropped = source.pipe(
        filter(() => {
          dropRate += 0.1;
          return Math.random() > dropRate; // Increasing drop rate
        })
      );

      // Exact output depends on random values, so test structure
      expectObservable(dropped).toBeDefined();
    });
  });

  it('should handle buffer overflow gracefully', fakeAsync(() => {
    const buffer = new BoundedBuffer<number>(3, 'dropOldest');
    
    // Fill buffer beyond capacity
    for (let i = 0; i < 5; i++) {
      buffer.add(i);
    }
    
    expect(buffer.size()).toBe(3);
    expect(buffer.flush()).toEqual([2, 3, 4]); // Oldest items dropped
  }));
});

/**
 * Performance testing utilities
 */
class BackpressureTestHarness {
  private metricsSubject = new Subject<{
    timestamp: number;
    queueSize: number;
    processingRate: number;
    dropRate: number;
  }>();

  readonly metrics$ = this.metricsSubject.asObservable();

  createHighVolumeStream(itemsPerSecond: number, durationMs: number): Observable<number> {
    const intervalMs = 1000 / itemsPerSecond;
    const totalItems = Math.floor(durationMs / intervalMs);
    
    return interval(intervalMs).pipe(
      take(totalItems),
      tap(() => this.updateMetrics())
    );
  }

  private updateMetrics(): void {
    // Simulate metrics collection
    this.metricsSubject.next({
      timestamp: Date.now(),
      queueSize: Math.floor(Math.random() * 1000),
      processingRate: Math.random() * 100,
      dropRate: Math.random() * 0.1
    });
  }

  testBackpressureHandling(
    processor: (stream$: Observable<number>) => Observable<any>
  ): Observable<any> {
    const highVolumeStream$ = this.createHighVolumeStream(1000, 10000); // 1000 items/sec for 10 seconds
    return processor(highVolumeStream$);
  }
}

// Usage
const testHarness = new BackpressureTestHarness();

const testResult$ = testHarness.testBackpressureHandling(stream$ =>
  stream$.pipe(
    bufferTime(100),
    mergeMap(batch => of(batch.length), 3) // Process with limited concurrency
  )
);

testResult$.subscribe(batchSize => {
  console.log('Processed batch of size:', batchSize);
});
```

## Best Practices

### 1. Buffer Strategy Selection
- Use time-based buffering for real-time processing
- Use count-based buffering for batch operations
- Implement adaptive buffering for variable loads
- Consider memory constraints in buffer sizing

### 2. Flow Control Implementation
- Monitor system resources and adjust flow rates
- Implement circuit breakers for overload protection
- Use intelligent dropping strategies for non-critical data
- Provide backpressure feedback to upstream producers

### 3. Memory Management
- Use bounded buffers to prevent memory exhaustion
- Implement circular buffers for fixed memory usage
- Monitor buffer sizes and processing rates
- Clean up resources promptly

### 4. Performance Optimization
- Profile buffer performance under various loads
- Use sampling for high-frequency, low-value data
- Implement priority-based processing for critical data
- Consider parallel processing for CPU-intensive operations

### 5. Error Handling
- Handle buffer overflow gracefully
- Implement retry mechanisms for transient failures
- Provide fallback strategies for system overload
- Log and monitor error rates

## Common Pitfalls

### 1. Memory Leaks
```typescript
// ❌ Bad: Unbounded buffer growth
source$.pipe(
  buffer(NEVER) // Never flushes - memory leak!
);

// ✅ Good: Bounded buffer with timeout
source$.pipe(
  buffer(timer(5000)) // Flush every 5 seconds
);
```

### 2. Inefficient Buffering
```typescript
// ❌ Bad: Too small buffers causing overhead
source$.pipe(
  bufferCount(1) // No batching benefit
);

// ✅ Good: Appropriate buffer size
source$.pipe(
  bufferCount(100) // Efficient batch size
);
```

### 3. Ignoring Backpressure
```typescript
// ❌ Bad: No flow control
fastProducer$.pipe(
  mergeMap(item => slowProcessor(item)) // Queue builds up
);

// ✅ Good: Controlled concurrency
fastProducer$.pipe(
  mergeMap(item => slowProcessor(item), 3) // Limit concurrent processing
);
```

## Summary

Effective backpressure and buffering strategies are essential for building resilient reactive systems. Key takeaways:

- Choose appropriate buffering strategies based on data characteristics and system requirements
- Implement intelligent flow control mechanisms to prevent system overload
- Use bounded buffers and memory-efficient streaming for large-scale applications
- Monitor system metrics and adapt processing strategies dynamically
- Test backpressure scenarios thoroughly with realistic load patterns
- Handle buffer overflow and system overload gracefully
- Provide feedback mechanisms between producers and consumers
- Consider both performance and memory usage in buffer design

These patterns enable building robust streaming applications that can handle variable loads and maintain system stability under pressure.
