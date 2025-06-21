# Observable Caching & Sharing Patterns

## Learning Objectives
By the end of this lesson, you will be able to:
- Master advanced caching strategies for Observable streams
- Implement intelligent cache invalidation and refresh mechanisms
- Design multi-level caching architectures with RxJS
- Apply advanced sharing patterns for resource optimization
- Build cache-aware reactive applications with optimal performance
- Handle cache coherency and consistency in distributed systems

## Table of Contents
1. [Advanced Caching Fundamentals](#advanced-caching-fundamentals)
2. [Multi-Level Cache Architecture](#multi-level-cache-architecture)
3. [Intelligent Cache Invalidation](#intelligent-cache-invalidation)
4. [Adaptive Caching Strategies](#adaptive-caching-strategies)
5. [Cache Coherency Patterns](#cache-coherency-patterns)
6. [Distributed Caching with RxJS](#distributed-caching-with-rxjs)
7. [Memory-Aware Caching](#memory-aware-caching)
8. [Cache Warming and Prefetching](#cache-warming-and-prefetching)
9. [Angular Caching Integration](#angular-caching-integration)
10. [Performance Monitoring](#performance-monitoring)
11. [Testing Caching Strategies](#testing-caching-strategies)
12. [Exercises](#exercises)

## Advanced Caching Fundamentals

### Smart Cache Implementation

```typescript
import { 
  Observable, BehaviorSubject, Subject, timer, merge, EMPTY, throwError,
  combineLatest, of, from, interval 
} from 'rxjs';
import { 
  map, switchMap, tap, catchError, shareReplay, refCount, 
  distinctUntilChanged, filter, take, takeUntil, startWith,
  debounceTime, throttleTime, scan, finalize
} from 'rxjs/operators';

// Advanced cache entry with metadata
interface CacheEntry<T> {
  value: T;
  timestamp: number;
  ttl: number;
  accessCount: number;
  lastAccessed: number;
  version: number;
  tags: string[];
  metadata: CacheMetadata;
}

interface CacheMetadata {
  source: string;
  priority: number;
  size: number;
  dependencies: string[];
  validator?: (value: any) => boolean;
}

interface CacheConfig {
  maxSize: number;
  defaultTtl: number;
  evictionPolicy: 'LRU' | 'LFU' | 'FIFO' | 'TTL';
  enableCompression: boolean;
  enableMetrics: boolean;
  backgroundRefresh: boolean;
  staleWhileRevalidate: boolean;
}

// Sophisticated Observable Cache
class AdvancedObservableCache<K, V> {
  private cache = new Map<K, CacheEntry<V>>();
  private refreshStreams = new Map<K, Observable<V>>();
  private metrics = new CacheMetrics();
  private evictionSubject = new Subject<{ key: K; reason: string }>();
  
  constructor(
    private config: CacheConfig,
    private refreshFactory: (key: K) => Observable<V>
  ) {
    this.setupBackgroundTasks();
  }

  // Get value with advanced caching logic
  get(key: K, options?: CacheGetOptions): Observable<V> {
    const entry = this.cache.get(key);
    const now = Date.now();

    // Cache hit with valid entry
    if (entry && this.isValid(entry, now)) {
      this.updateAccessInfo(entry, now);
      this.metrics.recordHit();
      
      // Background refresh if enabled and entry is stale
      if (this.config.staleWhileRevalidate && this.isStale(entry, now)) {
        this.backgroundRefresh(key);
      }
      
      return of(entry.value);
    }

    // Cache miss or invalid entry
    this.metrics.recordMiss();
    
    // Return stale data while revalidating if configured
    if (entry && this.config.staleWhileRevalidate) {
      this.backgroundRefresh(key);
      return of(entry.value);
    }

    // Fetch fresh data
    return this.fetchAndCache(key, options);
  }

  // Set value with advanced options
  set(
    key: K, 
    value: V, 
    options?: CacheSetOptions
  ): void {
    const now = Date.now();
    const entry: CacheEntry<V> = {
      value,
      timestamp: now,
      ttl: options?.ttl || this.config.defaultTtl,
      accessCount: 1,
      lastAccessed: now,
      version: options?.version || 1,
      tags: options?.tags || [],
      metadata: {
        source: options?.source || 'manual',
        priority: options?.priority || 0,
        size: this.calculateSize(value),
        dependencies: options?.dependencies || [],
        validator: options?.validator
      }
    };

    // Check capacity and evict if necessary
    this.ensureCapacity();
    
    this.cache.set(key, entry);
    this.metrics.recordSet();
  }

  // Invalidate by key
  invalidate(key: K): void {
    if (this.cache.has(key)) {
      this.cache.delete(key);
      this.evictionSubject.next({ key, reason: 'manual_invalidation' });
      this.metrics.recordEviction();
    }
  }

  // Invalidate by tags
  invalidateByTags(tags: string[]): void {
    const keysToInvalidate: K[] = [];
    
    this.cache.forEach((entry, key) => {
      if (entry.tags.some(tag => tags.includes(tag))) {
        keysToInvalidate.push(key);
      }
    });

    keysToInvalidate.forEach(key => this.invalidate(key));
  }

  // Bulk operations
  getMultiple(keys: K[]): Observable<Map<K, V>> {
    const results = new Map<K, V>();
    const missingKeys: K[] = [];

    // Check cache for each key
    keys.forEach(key => {
      const entry = this.cache.get(key);
      if (entry && this.isValid(entry, Date.now())) {
        results.set(key, entry.value);
        this.updateAccessInfo(entry, Date.now());
      } else {
        missingKeys.push(key);
      }
    });

    if (missingKeys.length === 0) {
      return of(results);
    }

    // Fetch missing keys in parallel
    const fetchObservables = missingKeys.map(key =>
      this.fetchAndCache(key).pipe(
        map(value => ({ key, value })),
        catchError(() => of(null))
      )
    );

    return combineLatest(fetchObservables).pipe(
      map(fetchResults => {
        fetchResults.forEach(result => {
          if (result) {
            results.set(result.key, result.value);
          }
        });
        return results;
      })
    );
  }

  // Cache warming
  warm(keys: K[]): Observable<void> {
    const warmObservables = keys.map(key => 
      this.fetchAndCache(key).pipe(
        catchError(() => of(null))
      )
    );

    return combineLatest(warmObservables).pipe(
      map(() => void 0)
    );
  }

  // Get cache statistics
  getStatistics(): CacheStatistics {
    return {
      size: this.cache.size,
      hitRate: this.metrics.getHitRate(),
      missRate: this.metrics.getMissRate(),
      evictionCount: this.metrics.getEvictionCount(),
      memoryUsage: this.calculateTotalSize(),
      oldestEntry: this.getOldestEntryAge(),
      averageAge: this.getAverageEntryAge()
    };
  }

  // Private methods
  private isValid(entry: CacheEntry<V>, now: number): boolean {
    const isNotExpired = (now - entry.timestamp) < entry.ttl;
    const isValidByValidator = !entry.metadata.validator || 
                               entry.metadata.validator(entry.value);
    return isNotExpired && isValidByValidator;
  }

  private isStale(entry: CacheEntry<V>, now: number): boolean {
    const staleThreshold = entry.ttl * 0.8; // 80% of TTL
    return (now - entry.timestamp) > staleThreshold;
  }

  private updateAccessInfo(entry: CacheEntry<V>, now: number): void {
    entry.accessCount++;
    entry.lastAccessed = now;
  }

  private fetchAndCache(key: K, options?: CacheGetOptions): Observable<V> {
    // Prevent duplicate requests
    if (this.refreshStreams.has(key)) {
      return this.refreshStreams.get(key)!;
    }

    const refresh$ = this.refreshFactory(key).pipe(
      tap(value => {
        this.set(key, value, {
          ttl: options?.ttl,
          tags: options?.tags,
          priority: options?.priority
        });
      }),
      finalize(() => {
        this.refreshStreams.delete(key);
      }),
      shareReplay(1)
    );

    this.refreshStreams.set(key, refresh$);
    return refresh$;
  }

  private backgroundRefresh(key: K): void {
    if (!this.refreshStreams.has(key)) {
      this.fetchAndCache(key).subscribe();
    }
  }

  private ensureCapacity(): void {
    if (this.cache.size >= this.config.maxSize) {
      this.evictEntries(1);
    }
  }

  private evictEntries(count: number): void {
    const entriesToEvict = this.selectEntriesForEviction(count);
    
    entriesToEvict.forEach(([key, entry]) => {
      this.cache.delete(key);
      this.evictionSubject.next({ key, reason: this.config.evictionPolicy });
      this.metrics.recordEviction();
    });
  }

  private selectEntriesForEviction(count: number): [K, CacheEntry<V>][] {
    const entries = Array.from(this.cache.entries());
    
    switch (this.config.evictionPolicy) {
      case 'LRU':
        return entries
          .sort(([, a], [, b]) => a.lastAccessed - b.lastAccessed)
          .slice(0, count);
      
      case 'LFU':
        return entries
          .sort(([, a], [, b]) => a.accessCount - b.accessCount)
          .slice(0, count);
      
      case 'FIFO':
        return entries
          .sort(([, a], [, b]) => a.timestamp - b.timestamp)
          .slice(0, count);
      
      case 'TTL':
        return entries
          .filter(([, entry]) => !this.isValid(entry, Date.now()))
          .slice(0, count);
      
      default:
        return entries.slice(0, count);
    }
  }

  private calculateSize(value: V): number {
    return JSON.stringify(value).length;
  }

  private calculateTotalSize(): number {
    let totalSize = 0;
    this.cache.forEach(entry => {
      totalSize += entry.metadata.size;
    });
    return totalSize;
  }

  private getOldestEntryAge(): number {
    let oldest = 0;
    const now = Date.now();
    
    this.cache.forEach(entry => {
      const age = now - entry.timestamp;
      if (age > oldest) {
        oldest = age;
      }
    });
    
    return oldest;
  }

  private getAverageEntryAge(): number {
    if (this.cache.size === 0) return 0;
    
    let totalAge = 0;
    const now = Date.now();
    
    this.cache.forEach(entry => {
      totalAge += (now - entry.timestamp);
    });
    
    return totalAge / this.cache.size;
  }

  private setupBackgroundTasks(): void {
    // Cleanup expired entries every minute
    interval(60000).subscribe(() => {
      this.cleanupExpiredEntries();
    });

    // Log metrics every 5 minutes if enabled
    if (this.config.enableMetrics) {
      interval(300000).subscribe(() => {
        console.log('Cache Statistics:', this.getStatistics());
      });
    }
  }

  private cleanupExpiredEntries(): void {
    const now = Date.now();
    const expiredKeys: K[] = [];

    this.cache.forEach((entry, key) => {
      if (!this.isValid(entry, now)) {
        expiredKeys.push(key);
      }
    });

    expiredKeys.forEach(key => {
      this.cache.delete(key);
      this.evictionSubject.next({ key, reason: 'expired' });
    });
  }
}

// Cache options interfaces
interface CacheGetOptions {
  ttl?: number;
  tags?: string[];
  priority?: number;
  forceRefresh?: boolean;
}

interface CacheSetOptions {
  ttl?: number;
  tags?: string[];
  priority?: number;
  version?: number;
  source?: string;
  dependencies?: string[];
  validator?: (value: any) => boolean;
}

interface CacheStatistics {
  size: number;
  hitRate: number;
  missRate: number;
  evictionCount: number;
  memoryUsage: number;
  oldestEntry: number;
  averageAge: number;
}

// Cache metrics tracking
class CacheMetrics {
  private hits = 0;
  private misses = 0;
  private sets = 0;
  private evictions = 0;

  recordHit(): void { this.hits++; }
  recordMiss(): void { this.misses++; }
  recordSet(): void { this.sets++; }
  recordEviction(): void { this.evictions++; }

  getHitRate(): number {
    const total = this.hits + this.misses;
    return total === 0 ? 0 : this.hits / total;
  }

  getMissRate(): number {
    const total = this.hits + this.misses;
    return total === 0 ? 0 : this.misses / total;
  }

  getEvictionCount(): number { return this.evictions; }

  reset(): void {
    this.hits = 0;
    this.misses = 0;
    this.sets = 0;
    this.evictions = 0;
  }
}
```

### Marble Diagram: Cache Hit/Miss Patterns
```
Requests:    --R1--R1--R2--R1--R3--R2-->
Cache:       -----H---M----H---M---H--->  (H=Hit, M=Miss)
Network:     --N1-------N2----N3------->  (N=Network call)
Response:    --D1--D1--D2--D1--D3--D2-->  (D=Data)
```

## Multi-Level Cache Architecture

### Hierarchical Caching System

```typescript
// Multi-level cache with L1 (memory) and L2 (storage)
class MultiLevelCache<K, V> {
  private l1Cache: AdvancedObservableCache<K, V>;
  private l2Cache: StorageCache<K, V>;
  private l3Cache?: NetworkCache<K, V>;

  constructor(
    private config: MultiLevelCacheConfig,
    private dataSource: (key: K) => Observable<V>
  ) {
    this.setupCacheLevels();
  }

  get(key: K): Observable<V> {
    // Try L1 first (fastest)
    return this.l1Cache.get(key).pipe(
      catchError(() => 
        // Try L2 on L1 miss
        this.l2Cache.get(key).pipe(
          tap(value => {
            // Promote to L1
            this.l1Cache.set(key, value);
          }),
          catchError(() => 
            // Try L3 on L2 miss (if available)
            this.l3Cache ? 
              this.l3Cache.get(key).pipe(
                tap(value => {
                  // Promote to L2 and L1
                  this.l2Cache.set(key, value);
                  this.l1Cache.set(key, value);
                })
              ) :
              // Fallback to data source
              this.dataSource(key).pipe(
                tap(value => {
                  // Cache in all levels
                  this.l1Cache.set(key, value);
                  this.l2Cache.set(key, value);
                  if (this.l3Cache) {
                    this.l3Cache.set(key, value);
                  }
                })
              )
          )
        )
      )
    );
  }

  set(key: K, value: V, level: CacheLevel = 'all'): void {
    switch (level) {
      case 'l1':
        this.l1Cache.set(key, value);
        break;
      case 'l2':
        this.l2Cache.set(key, value);
        break;
      case 'l3':
        if (this.l3Cache) {
          this.l3Cache.set(key, value);
        }
        break;
      case 'all':
        this.l1Cache.set(key, value);
        this.l2Cache.set(key, value);
        if (this.l3Cache) {
          this.l3Cache.set(key, value);
        }
        break;
    }
  }

  invalidate(key: K, level: CacheLevel = 'all'): void {
    switch (level) {
      case 'l1':
        this.l1Cache.invalidate(key);
        break;
      case 'l2':
        this.l2Cache.invalidate(key);
        break;
      case 'l3':
        if (this.l3Cache) {
          this.l3Cache.invalidate(key);
        }
        break;
      case 'all':
        this.l1Cache.invalidate(key);
        this.l2Cache.invalidate(key);
        if (this.l3Cache) {
          this.l3Cache.invalidate(key);
        }
        break;
    }
  }

  // Cache coherency across levels
  synchronizeLevels(): Observable<void> {
    // Implement cache synchronization logic
    return new Observable<void>(observer => {
      // Sync L2 to L1 for frequently accessed items
      // Evict from L1 items that are expired in L2
      // Update L3 with changes from L1/L2
      observer.next();
      observer.complete();
    });
  }

  private setupCacheLevels(): void {
    // L1: Fast in-memory cache
    this.l1Cache = new AdvancedObservableCache<K, V>(
      {
        maxSize: this.config.l1MaxSize,
        defaultTtl: this.config.l1Ttl,
        evictionPolicy: 'LRU',
        enableCompression: false,
        enableMetrics: true,
        backgroundRefresh: true,
        staleWhileRevalidate: true
      },
      key => this.dataSource(key)
    );

    // L2: Persistent storage cache
    this.l2Cache = new StorageCache<K, V>(
      this.config.l2Config,
      key => this.dataSource(key)
    );

    // L3: Network/CDN cache (optional)
    if (this.config.l3Config) {
      this.l3Cache = new NetworkCache<K, V>(
        this.config.l3Config,
        key => this.dataSource(key)
      );
    }
  }
}

// Storage-based cache implementation
class StorageCache<K, V> {
  constructor(
    private config: StorageCacheConfig,
    private refreshFactory: (key: K) => Observable<V>
  ) {}

  get(key: K): Observable<V> {
    return new Observable<V>(observer => {
      try {
        const stored = localStorage.getItem(this.getStorageKey(key));
        if (stored) {
          const entry = JSON.parse(stored);
          if (this.isValid(entry)) {
            observer.next(entry.value);
            observer.complete();
            return;
          }
        }
        observer.error(new Error('Cache miss'));
      } catch (error) {
        observer.error(error);
      }
    });
  }

  set(key: K, value: V): void {
    const entry = {
      value,
      timestamp: Date.now(),
      ttl: this.config.defaultTtl
    };
    
    try {
      localStorage.setItem(
        this.getStorageKey(key),
        JSON.stringify(entry)
      );
    } catch (error) {
      console.warn('Storage cache set failed:', error);
    }
  }

  invalidate(key: K): void {
    try {
      localStorage.removeItem(this.getStorageKey(key));
    } catch (error) {
      console.warn('Storage cache invalidate failed:', error);
    }
  }

  private getStorageKey(key: K): string {
    return `${this.config.prefix}_${JSON.stringify(key)}`;
  }

  private isValid(entry: any): boolean {
    const now = Date.now();
    return (now - entry.timestamp) < entry.ttl;
  }
}

// Network/CDN cache implementation
class NetworkCache<K, V> {
  constructor(
    private config: NetworkCacheConfig,
    private refreshFactory: (key: K) => Observable<V>
  ) {}

  get(key: K): Observable<V> {
    const url = `${this.config.baseUrl}/${this.encodeKey(key)}`;
    
    return new Observable<V>(observer => {
      fetch(url, {
        headers: {
          'Cache-Control': `max-age=${this.config.maxAge}`
        }
      })
        .then(response => {
          if (response.ok) {
            return response.json();
          }
          throw new Error(`Network cache miss: ${response.status}`);
        })
        .then(data => {
          observer.next(data);
          observer.complete();
        })
        .catch(error => observer.error(error));
    });
  }

  set(key: K, value: V): void {
    const url = `${this.config.baseUrl}/${this.encodeKey(key)}`;
    
    fetch(url, {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/json',
        'Cache-Control': `max-age=${this.config.maxAge}`
      },
      body: JSON.stringify(value)
    }).catch(error => {
      console.warn('Network cache set failed:', error);
    });
  }

  invalidate(key: K): void {
    const url = `${this.config.baseUrl}/${this.encodeKey(key)}`;
    
    fetch(url, {
      method: 'DELETE'
    }).catch(error => {
      console.warn('Network cache invalidate failed:', error);
    });
  }

  private encodeKey(key: K): string {
    return btoa(JSON.stringify(key));
  }
}

// Configuration interfaces
interface MultiLevelCacheConfig {
  l1MaxSize: number;
  l1Ttl: number;
  l2Config: StorageCacheConfig;
  l3Config?: NetworkCacheConfig;
}

interface StorageCacheConfig {
  prefix: string;
  defaultTtl: number;
  maxSize: number;
}

interface NetworkCacheConfig {
  baseUrl: string;
  maxAge: number;
  timeout: number;
}

type CacheLevel = 'l1' | 'l2' | 'l3' | 'all';
```

## Intelligent Cache Invalidation

### Advanced Invalidation Strategies

```typescript
// Cache invalidation orchestrator
class CacheInvalidationOrchestrator<K> {
  private invalidationRules = new Map<string, InvalidationRule<K>>();
  private dependencyGraph = new Map<K, Set<K>>();
  private invalidationStream$ = new Subject<InvalidationEvent<K>>();

  constructor(private cache: AdvancedObservableCache<K, any>) {
    this.setupInvalidationHandling();
  }

  // Register invalidation rule
  addRule(name: string, rule: InvalidationRule<K>): void {
    this.invalidationRules.set(name, rule);
  }

  // Add dependency relationship
  addDependency(dependent: K, dependency: K): void {
    if (!this.dependencyGraph.has(dependent)) {
      this.dependencyGraph.set(dependent, new Set());
    }
    this.dependencyGraph.get(dependent)!.add(dependency);
  }

  // Trigger smart invalidation
  invalidate(key: K, reason: InvalidationReason): Observable<InvalidationResult> {
    return new Observable<InvalidationResult>(observer => {
      const startTime = Date.now();
      const invalidatedKeys = new Set<K>();
      
      try {
        // Direct invalidation
        this.cache.invalidate(key);
        invalidatedKeys.add(key);

        // Cascade invalidation based on dependencies
        this.cascadeInvalidation(key, invalidatedKeys);

        // Apply invalidation rules
        this.applyInvalidationRules(key, reason, invalidatedKeys);

        // Emit invalidation event
        this.invalidationStream$.next({
          key,
          reason,
          timestamp: Date.now(),
          affectedKeys: Array.from(invalidatedKeys)
        });

        const result: InvalidationResult = {
          success: true,
          invalidatedKeys: Array.from(invalidatedKeys),
          duration: Date.now() - startTime
        };

        observer.next(result);
        observer.complete();
      } catch (error) {
        observer.error(error);
      }
    });
  }

  // Batch invalidation
  invalidateMultiple(
    keys: K[], 
    reason: InvalidationReason
  ): Observable<InvalidationResult> {
    const invalidationObservables = keys.map(key => 
      this.invalidate(key, reason)
    );

    return combineLatest(invalidationObservables).pipe(
      map(results => ({
        success: results.every(r => r.success),
        invalidatedKeys: results.flatMap(r => r.invalidatedKeys),
        duration: Math.max(...results.map(r => r.duration))
      }))
    );
  }

  // Time-based invalidation
  scheduleInvalidation(
    key: K, 
    delay: number, 
    reason: InvalidationReason
  ): Observable<void> {
    return timer(delay).pipe(
      switchMap(() => this.invalidate(key, reason)),
      map(() => void 0)
    );
  }

  // Conditional invalidation
  invalidateWhen<T>(
    condition$: Observable<T>,
    predicate: (value: T) => boolean,
    keySelector: (value: T) => K[],
    reason: InvalidationReason
  ): Observable<void> {
    return condition$.pipe(
      filter(predicate),
      switchMap(value => {
        const keys = keySelector(value);
        return this.invalidateMultiple(keys, reason);
      }),
      map(() => void 0)
    );
  }

  // Get invalidation events stream
  getInvalidationEvents(): Observable<InvalidationEvent<K>> {
    return this.invalidationStream$.asObservable();
  }

  private cascadeInvalidation(key: K, invalidated: Set<K>): void {
    // Find all keys that depend on the invalidated key
    this.dependencyGraph.forEach((dependencies, dependent) => {
      if (dependencies.has(key) && !invalidated.has(dependent)) {
        this.cache.invalidate(dependent);
        invalidated.add(dependent);
        
        // Recursive cascade
        this.cascadeInvalidation(dependent, invalidated);
      }
    });
  }

  private applyInvalidationRules(
    key: K, 
    reason: InvalidationReason, 
    invalidated: Set<K>
  ): void {
    this.invalidationRules.forEach(rule => {
      if (rule.condition(key, reason)) {
        const keysToInvalidate = rule.action(key, reason);
        keysToInvalidate.forEach(k => {
          if (!invalidated.has(k)) {
            this.cache.invalidate(k);
            invalidated.add(k);
          }
        });
      }
    });
  }

  private setupInvalidationHandling(): void {
    // Auto-cleanup of dependency graph
    this.invalidationStream$.pipe(
      debounceTime(5000) // Batch cleanup every 5 seconds
    ).subscribe(event => {
      this.cleanupDependencyGraph(event.affectedKeys);
    });
  }

  private cleanupDependencyGraph(invalidatedKeys: K[]): void {
    invalidatedKeys.forEach(key => {
      // Remove invalidated keys from dependency relationships
      this.dependencyGraph.delete(key);
      this.dependencyGraph.forEach(dependencies => {
        dependencies.delete(key);
      });
    });
  }
}

// Invalidation interfaces
interface InvalidationRule<K> {
  condition: (key: K, reason: InvalidationReason) => boolean;
  action: (key: K, reason: InvalidationReason) => K[];
}

interface InvalidationEvent<K> {
  key: K;
  reason: InvalidationReason;
  timestamp: number;
  affectedKeys: K[];
}

interface InvalidationResult {
  success: boolean;
  invalidatedKeys: any[];
  duration: number;
}

type InvalidationReason = 
  | 'expired'
  | 'manual'
  | 'dependency_changed'
  | 'data_updated'
  | 'rule_triggered'
  | 'capacity_exceeded';

// Example: E-commerce cache invalidation
class EcommerceCacheInvalidator {
  private orchestrator: CacheInvalidationOrchestrator<string>;

  constructor(cache: AdvancedObservableCache<string, any>) {
    this.orchestrator = new CacheInvalidationOrchestrator(cache);
    this.setupEcommerceRules();
  }

  private setupEcommerceRules(): void {
    // Product update invalidates related caches
    this.orchestrator.addRule('product_update', {
      condition: (key, reason) => 
        key.startsWith('product:') && reason === 'data_updated',
      action: (key) => {
        const productId = key.split(':')[1];
        return [
          `product_list:category:${productId}`,
          `recommendations:${productId}`,
          `reviews:${productId}`,
          `inventory:${productId}`
        ];
      }
    });

    // Price change invalidates pricing caches
    this.orchestrator.addRule('price_change', {
      condition: (key, reason) => 
        key.startsWith('price:') && reason === 'data_updated',
      action: (key) => {
        const productId = key.split(':')[1];
        return [
          `product:${productId}`,
          `cart:*`, // All carts
          `price_comparison:${productId}`
        ];
      }
    });

    // Inventory change invalidates availability
    this.orchestrator.addRule('inventory_change', {
      condition: (key, reason) => 
        key.startsWith('inventory:') && reason === 'data_updated',
      action: (key) => {
        const productId = key.split(':')[1];
        return [
          `product:${productId}`,
          `product_list:*`,
          `search_results:*`
        ];
      }
    });

    // User preference change invalidates personalized content
    this.orchestrator.addRule('user_preference_change', {
      condition: (key, reason) => 
        key.startsWith('user_preferences:') && reason === 'data_updated',
      action: (key) => {
        const userId = key.split(':')[1];
        return [
          `recommendations:user:${userId}`,
          `personalized_products:${userId}`,
          `user_dashboard:${userId}`
        ];
      }
    });
  }

  // Public methods for specific invalidations
  invalidateProduct(productId: string): Observable<InvalidationResult> {
    return this.orchestrator.invalidate(`product:${productId}`, 'data_updated');
  }

  invalidateUserData(userId: string): Observable<InvalidationResult> {
    return this.orchestrator.invalidateMultiple([
      `user:${userId}`,
      `user_preferences:${userId}`,
      `order_history:${userId}`
    ], 'data_updated');
  }

  invalidateCategory(categoryId: string): Observable<InvalidationResult> {
    return this.orchestrator.invalidate(`category:${categoryId}`, 'data_updated');
  }
}
```

## Adaptive Caching Strategies

### Dynamic Cache Optimization

```typescript
// Adaptive cache strategy that learns from usage patterns
class AdaptiveCacheStrategy<K, V> {
  private usagePatterns = new Map<K, UsagePattern>();
  private strategyMetrics = new Map<string, StrategyMetrics>();
  private currentStrategy: CacheStrategy = 'balanced';
  private adaptationInterval = 300000; // 5 minutes

  constructor(
    private cache: AdvancedObservableCache<K, V>,
    private config: AdaptiveCacheConfig
  ) {
    this.setupAdaptation();
  }

  // Get with adaptive strategy
  get(key: K): Observable<V> {
    this.recordAccess(key);
    
    const pattern = this.usagePatterns.get(key);
    const strategy = this.selectStrategy(key, pattern);
    
    return this.cache.get(key, {
      ttl: this.calculateTtl(pattern, strategy),
      priority: this.calculatePriority(pattern, strategy)
    });
  }

  // Set with adaptive parameters
  set(key: K, value: V): void {
    const pattern = this.usagePatterns.get(key);
    const strategy = this.selectStrategy(key, pattern);
    
    this.cache.set(key, value, {
      ttl: this.calculateTtl(pattern, strategy),
      priority: this.calculatePriority(pattern, strategy),
      tags: this.generateTags(key, pattern)
    });
  }

  // Analyze and adapt strategy
  adapt(): Observable<AdaptationResult> {
    return new Observable<AdaptationResult>(observer => {
      const analysis = this.analyzeUsagePatterns();
      const recommendations = this.generateRecommendations(analysis);
      const newStrategy = this.selectOptimalStrategy(recommendations);
      
      if (newStrategy !== this.currentStrategy) {
        this.applyStrategy(newStrategy);
        
        observer.next({
          previousStrategy: this.currentStrategy,
          newStrategy,
          improvements: recommendations,
          timestamp: Date.now()
        });
        
        this.currentStrategy = newStrategy;
      }
      
      observer.complete();
    });
  }

  private recordAccess(key: K): void {
    const now = Date.now();
    const pattern = this.usagePatterns.get(key) || {
      accessCount: 0,
      firstAccess: now,
      lastAccess: now,
      accessHistory: [],
      avgInterval: 0,
      peakHours: [],
      accessPattern: 'unknown'
    };

    pattern.accessCount++;
    pattern.lastAccess = now;
    pattern.accessHistory.push(now);

    // Keep only recent history
    const oneHourAgo = now - 3600000;
    pattern.accessHistory = pattern.accessHistory.filter(time => time > oneHourAgo);

    // Calculate average interval
    if (pattern.accessHistory.length > 1) {
      const intervals = pattern.accessHistory
        .slice(1)
        .map((time, i) => time - pattern.accessHistory[i]);
      pattern.avgInterval = intervals.reduce((a, b) => a + b, 0) / intervals.length;
    }

    // Determine access pattern
    pattern.accessPattern = this.classifyAccessPattern(pattern);

    this.usagePatterns.set(key, pattern);
  }

  private classifyAccessPattern(pattern: UsagePattern): AccessPattern {
    const { accessHistory, avgInterval } = pattern;
    
    if (accessHistory.length < 3) return 'unknown';
    
    if (avgInterval < 60000) return 'frequent'; // < 1 minute
    if (avgInterval < 300000) return 'regular'; // < 5 minutes
    if (avgInterval < 3600000) return 'occasional'; // < 1 hour
    return 'rare';
  }

  private selectStrategy(key: K, pattern?: UsagePattern): CacheStrategy {
    if (!pattern) return this.currentStrategy;

    switch (pattern.accessPattern) {
      case 'frequent':
        return 'aggressive'; // High TTL, low eviction priority
      case 'regular':
        return 'balanced';
      case 'occasional':
        return 'conservative';
      case 'rare':
        return 'minimal'; // Low TTL, high eviction priority
      default:
        return this.currentStrategy;
    }
  }

  private calculateTtl(pattern?: UsagePattern, strategy?: CacheStrategy): number {
    const baseTtl = this.config.baseTtl;
    
    if (!pattern || !strategy) return baseTtl;

    const multipliers: Record<CacheStrategy, number> = {
      aggressive: 4.0,
      balanced: 1.0,
      conservative: 0.5,
      minimal: 0.25
    };

    // Adjust based on access frequency
    let frequencyMultiplier = 1.0;
    switch (pattern.accessPattern) {
      case 'frequent':
        frequencyMultiplier = 2.0;
        break;
      case 'regular':
        frequencyMultiplier = 1.0;
        break;
      case 'occasional':
        frequencyMultiplier = 0.7;
        break;
      case 'rare':
        frequencyMultiplier = 0.3;
        break;
    }

    return baseTtl * multipliers[strategy] * frequencyMultiplier;
  }

  private calculatePriority(pattern?: UsagePattern, strategy?: CacheStrategy): number {
    if (!pattern) return 0;

    // Higher priority = less likely to be evicted
    let priority = pattern.accessCount;

    // Recent access increases priority
    const timeSinceLastAccess = Date.now() - pattern.lastAccess;
    if (timeSinceLastAccess < 300000) { // 5 minutes
      priority += 100;
    }

    // Frequent access pattern increases priority
    if (pattern.accessPattern === 'frequent') {
      priority += 200;
    }

    return priority;
  }

  private generateTags(key: K, pattern?: UsagePattern): string[] {
    const tags: string[] = [];
    
    if (pattern) {
      tags.push(`pattern:${pattern.accessPattern}`);
      tags.push(`count:${Math.floor(pattern.accessCount / 10) * 10}`);
    }

    // Add key-based tags
    const keyStr = String(key);
    if (keyStr.includes(':')) {
      const parts = keyStr.split(':');
      tags.push(`type:${parts[0]}`);
    }

    return tags;
  }

  private analyzeUsagePatterns(): UsageAnalysis {
    const patterns = Array.from(this.usagePatterns.values());
    
    return {
      totalKeys: patterns.length,
      frequentKeys: patterns.filter(p => p.accessPattern === 'frequent').length,
      regularKeys: patterns.filter(p => p.accessPattern === 'regular').length,
      occasionalKeys: patterns.filter(p => p.accessPattern === 'occasional').length,
      rareKeys: patterns.filter(p => p.accessPattern === 'rare').length,
      averageAccessCount: patterns.reduce((sum, p) => sum + p.accessCount, 0) / patterns.length,
      memoryEfficiency: this.calculateMemoryEfficiency(),
      hitRate: this.cache.getStatistics().hitRate
    };
  }

  private generateRecommendations(analysis: UsageAnalysis): CacheRecommendation[] {
    const recommendations: CacheRecommendation[] = [];

    // Memory efficiency recommendations
    if (analysis.memoryEfficiency < 0.7) {
      recommendations.push({
        type: 'memory_optimization',
        description: 'Reduce TTL for rare keys to improve memory efficiency',
        impact: 'medium',
        action: 'reduce_rare_ttl'
      });
    }

    // Hit rate recommendations
    if (analysis.hitRate < 0.8) {
      recommendations.push({
        type: 'hit_rate_optimization',
        description: 'Increase TTL for frequent keys to improve hit rate',
        impact: 'high',
        action: 'increase_frequent_ttl'
      });
    }

    // Access pattern recommendations
    const frequentRatio = analysis.frequentKeys / analysis.totalKeys;
    if (frequentRatio > 0.3) {
      recommendations.push({
        type: 'strategy_optimization',
        description: 'Consider aggressive caching strategy for better performance',
        impact: 'high',
        action: 'use_aggressive_strategy'
      });
    }

    return recommendations;
  }

  private selectOptimalStrategy(recommendations: CacheRecommendation[]): CacheStrategy {
    const hasHighImpactRecommendations = recommendations.some(r => r.impact === 'high');
    
    if (hasHighImpactRecommendations) {
      const aggressiveRecommendation = recommendations.find(r => 
        r.action === 'use_aggressive_strategy'
      );
      
      if (aggressiveRecommendation) return 'aggressive';
    }

    const memoryRecommendation = recommendations.find(r => 
      r.type === 'memory_optimization'
    );
    
    if (memoryRecommendation) return 'conservative';

    return 'balanced';
  }

  private applyStrategy(strategy: CacheStrategy): void {
    // Apply strategy-specific configurations
    const configs: Record<CacheStrategy, Partial<CacheConfig>> = {
      aggressive: {
        defaultTtl: this.config.baseTtl * 4,
        maxSize: this.config.maxSize * 0.8,
        evictionPolicy: 'LFU'
      },
      balanced: {
        defaultTtl: this.config.baseTtl,
        maxSize: this.config.maxSize,
        evictionPolicy: 'LRU'
      },
      conservative: {
        defaultTtl: this.config.baseTtl * 0.5,
        maxSize: this.config.maxSize * 1.2,
        evictionPolicy: 'TTL'
      },
      minimal: {
        defaultTtl: this.config.baseTtl * 0.25,
        maxSize: this.config.maxSize * 0.6,
        evictionPolicy: 'FIFO'
      }
    };

    // Apply configuration (would require cache reconfiguration)
    console.log(`Applying ${strategy} cache strategy:`, configs[strategy]);
  }

  private calculateMemoryEfficiency(): number {
    const stats = this.cache.getStatistics();
    // Simplified efficiency calculation
    return stats.hitRate * (1 - stats.size / this.config.maxSize);
  }

  private setupAdaptation(): void {
    interval(this.adaptationInterval).subscribe(() => {
      this.adapt().subscribe(result => {
        console.log('Cache adaptation completed:', result);
      });
    });
  }
}

// Interfaces for adaptive caching
interface UsagePattern {
  accessCount: number;
  firstAccess: number;
  lastAccess: number;
  accessHistory: number[];
  avgInterval: number;
  peakHours: number[];
  accessPattern: AccessPattern;
}

interface UsageAnalysis {
  totalKeys: number;
  frequentKeys: number;
  regularKeys: number;
  occasionalKeys: number;
  rareKeys: number;
  averageAccessCount: number;
  memoryEfficiency: number;
  hitRate: number;
}

interface CacheRecommendation {
  type: string;
  description: string;
  impact: 'low' | 'medium' | 'high';
  action: string;
}

interface AdaptationResult {
  previousStrategy: CacheStrategy;
  newStrategy: CacheStrategy;
  improvements: CacheRecommendation[];
  timestamp: number;
}

interface AdaptiveCacheConfig {
  baseTtl: number;
  maxSize: number;
  adaptationEnabled: boolean;
  learningPeriod: number;
}

type AccessPattern = 'frequent' | 'regular' | 'occasional' | 'rare' | 'unknown';
type CacheStrategy = 'aggressive' | 'balanced' | 'conservative' | 'minimal';
```

## Angular Caching Integration

### Angular Service with Advanced Caching

```typescript
import { Injectable, OnDestroy } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable, Subject, BehaviorSubject } from 'rxjs';
import { map, tap, catchError, shareReplay, takeUntil } from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class AdvancedCacheService implements OnDestroy {
  private destroy$ = new Subject<void>();
  private cache: MultiLevelCache<string, any>;
  private invalidator: CacheInvalidationOrchestrator<string>;
  private adaptiveStrategy: AdaptiveCacheStrategy<string, any>;

  // Real-time cache metrics
  private cacheMetrics$ = new BehaviorSubject<CacheStatistics>({
    size: 0,
    hitRate: 0,
    missRate: 0,
    evictionCount: 0,
    memoryUsage: 0,
    oldestEntry: 0,
    averageAge: 0
  });

  constructor(private http: HttpClient) {
    this.initializeCache();
    this.setupMetricsUpdates();
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }

  // Generic cached HTTP GET
  getCached<T>(
    url: string, 
    params?: HttpParams,
    cacheOptions?: CacheGetOptions
  ): Observable<T> {
    const cacheKey = this.generateCacheKey(url, params);
    
    return this.cache.get(cacheKey).pipe(
      catchError(() => 
        this.http.get<T>(url, { params }).pipe(
          tap(data => this.cache.set(cacheKey, data)),
          shareReplay(1)
        )
      )
    );
  }

  // Cached POST with invalidation
  postWithInvalidation<T>(
    url: string, 
    body: any,
    invalidationTags?: string[]
  ): Observable<T> {
    return this.http.post<T>(url, body).pipe(
      tap(() => {
        if (invalidationTags) {
          this.invalidator.invalidateByTags(invalidationTags);
        }
      })
    );
  }

  // Smart prefetching based on user behavior
  prefetch(urls: string[], priority: 'low' | 'medium' | 'high' = 'medium'): Observable<void> {
    const prefetchObservables = urls.map(url => 
      this.getCached(url, undefined, { priority: this.getPriorityValue(priority) })
        .pipe(catchError(() => of(null)))
    );

    return combineLatest(prefetchObservables).pipe(
      map(() => void 0)
    );
  }

  // Cache warming for critical data
  warmCache(endpoints: CacheWarmingEndpoint[]): Observable<void> {
    const warmingObservables = endpoints.map(endpoint =>
      this.getCached(endpoint.url, endpoint.params, {
        ttl: endpoint.ttl,
        tags: endpoint.tags,
        priority: endpoint.priority
      }).pipe(
        catchError(error => {
          console.warn(`Cache warming failed for ${endpoint.url}:`, error);
          return of(null);
        })
      )
    );

    return combineLatest(warmingObservables).pipe(
      map(() => void 0)
    );
  }

  // Real-time cache metrics
  getCacheMetrics(): Observable<CacheStatistics> {
    return this.cacheMetrics$.asObservable();
  }

  // Manual cache management
  invalidatePattern(pattern: string): void {
    // Implementation would depend on cache key patterns
    console.log(`Invalidating cache pattern: ${pattern}`);
  }

  clearCache(): void {
    // Clear all cache levels
    this.cache.invalidate('*', 'all');
    this.updateMetrics();
  }

  // Cache health check
  getHealthStatus(): Observable<CacheHealthStatus> {
    return new Observable<CacheHealthStatus>(observer => {
      const stats = this.cache.getStatistics();
      
      const health: CacheHealthStatus = {
        status: this.determineHealthStatus(stats),
        hitRate: stats.hitRate,
        memoryUsage: stats.memoryUsage,
        errorRate: this.calculateErrorRate(),
        recommendations: this.generateHealthRecommendations(stats),
        lastCheck: Date.now()
      };

      observer.next(health);
      observer.complete();
    });
  }

  private initializeCache(): void {
    // Initialize multi-level cache
    this.cache = new MultiLevelCache<string, any>(
      {
        l1MaxSize: 1000,
        l1Ttl: 300000, // 5 minutes
        l2Config: {
          prefix: 'app_cache',
          defaultTtl: 1800000, // 30 minutes
          maxSize: 5000
        }
      },
      key => this.fetchFromNetwork(key)
    );

    // Initialize invalidation orchestrator
    this.invalidator = new CacheInvalidationOrchestrator(this.cache.l1Cache);
    this.setupInvalidationRules();

    // Initialize adaptive strategy
    this.adaptiveStrategy = new AdaptiveCacheStrategy(this.cache.l1Cache, {
      baseTtl: 300000,
      maxSize: 1000,
      adaptationEnabled: true,
      learningPeriod: 3600000 // 1 hour
    });
  }

  private setupInvalidationRules(): void {
    // User data invalidation
    this.invalidator.addRule('user_data_update', {
      condition: (key, reason) => 
        key.startsWith('user:') && reason === 'data_updated',
      action: (key) => {
        const userId = key.split(':')[1];
        return [
          `profile:${userId}`,
          `preferences:${userId}`,
          `dashboard:${userId}`
        ];
      }
    });

    // Content invalidation
    this.invalidator.addRule('content_update', {
      condition: (key, reason) => 
        key.startsWith('content:') && reason === 'data_updated',
      action: (key) => {
        return ['content_list:*', 'featured_content:*'];
      }
    });
  }

  private setupMetricsUpdates(): void {
    interval(30000).pipe( // Update every 30 seconds
      takeUntil(this.destroy$)
    ).subscribe(() => {
      this.updateMetrics();
    });
  }

  private updateMetrics(): void {
    const stats = this.cache.getStatistics();
    this.cacheMetrics$.next(stats);
  }

  private generateCacheKey(url: string, params?: HttpParams): string {
    const paramString = params ? params.toString() : '';
    return `${url}${paramString ? '?' + paramString : ''}`;
  }

  private fetchFromNetwork(key: string): Observable<any> {
    // Extract URL from cache key for network fetch
    const [url, queryString] = key.split('?');
    const params = queryString ? new HttpParams({ fromString: queryString }) : undefined;
    
    return this.http.get(url, { params });
  }

  private getPriorityValue(priority: 'low' | 'medium' | 'high'): number {
    const priorities = { low: 1, medium: 5, high: 10 };
    return priorities[priority];
  }

  private determineHealthStatus(stats: CacheStatistics): 'healthy' | 'warning' | 'critical' {
    if (stats.hitRate > 0.8 && stats.memoryUsage < 0.9) return 'healthy';
    if (stats.hitRate > 0.6 && stats.memoryUsage < 0.95) return 'warning';
    return 'critical';
  }

  private calculateErrorRate(): number {
    // Simplified error rate calculation
    return 0.02; // 2% error rate
  }

  private generateHealthRecommendations(stats: CacheStatistics): string[] {
    const recommendations: string[] = [];

    if (stats.hitRate < 0.6) {
      recommendations.push('Consider increasing cache TTL for frequently accessed data');
    }

    if (stats.memoryUsage > 0.9) {
      recommendations.push('Cache memory usage is high, consider reducing cache size or TTL');
    }

    if (stats.evictionCount > 100) {
      recommendations.push('High eviction rate detected, consider optimizing cache size');
    }

    return recommendations;
  }
}

// Interfaces for Angular integration
interface CacheWarmingEndpoint {
  url: string;
  params?: HttpParams;
  ttl?: number;
  tags?: string[];
  priority?: number;
}

interface CacheHealthStatus {
  status: 'healthy' | 'warning' | 'critical';
  hitRate: number;
  memoryUsage: number;
  errorRate: number;
  recommendations: string[];
  lastCheck: number;
}

// Cache configuration component
@Component({
  selector: 'app-cache-dashboard',
  template: `
    <div class="cache-dashboard">
      <div class="metrics-section">
        <h3>Cache Metrics</h3>
        <div class="metric-cards" *ngIf="metrics$ | async as metrics">
          <div class="metric-card">
            <span class="metric-label">Hit Rate</span>
            <span class="metric-value">{{ (metrics.hitRate * 100) | number:'1.1-1' }}%</span>
          </div>
          <div class="metric-card">
            <span class="metric-label">Cache Size</span>
            <span class="metric-value">{{ metrics.size }}</span>
          </div>
          <div class="metric-card">
            <span class="metric-label">Memory Usage</span>
            <span class="metric-value">{{ (metrics.memoryUsage / 1024 / 1024) | number:'1.1-1' }} MB</span>
          </div>
        </div>
      </div>

      <div class="health-section">
        <h3>Cache Health</h3>
        <div class="health-status" *ngIf="health$ | async as health">
          <div class="status-indicator" [class]="health.status">
            {{ health.status }}
          </div>
          <ul class="recommendations">
            <li *ngFor="let rec of health.recommendations">{{ rec }}</li>
          </ul>
        </div>
      </div>

      <div class="actions-section">
        <h3>Cache Actions</h3>
        <button (click)="clearCache()">Clear Cache</button>
        <button (click)="warmCache()">Warm Critical Data</button>
        <button (click)="runHealthCheck()">Health Check</button>
      </div>
    </div>
  `,
  styles: [`
    .cache-dashboard {
      padding: 20px;
    }
    .metric-cards {
      display: flex;
      gap: 15px;
    }
    .metric-card {
      padding: 15px;
      border: 1px solid #ddd;
      border-radius: 8px;
      display: flex;
      flex-direction: column;
    }
    .metric-label {
      font-size: 12px;
      color: #666;
    }
    .metric-value {
      font-size: 24px;
      font-weight: bold;
      color: #333;
    }
    .status-indicator {
      padding: 8px 16px;
      border-radius: 4px;
      font-weight: bold;
    }
    .status-indicator.healthy { background: #d4edda; color: #155724; }
    .status-indicator.warning { background: #fff3cd; color: #856404; }
    .status-indicator.critical { background: #f8d7da; color: #721c24; }
  `]
})
export class CacheDashboardComponent implements OnInit {
  metrics$: Observable<CacheStatistics>;
  health$: Observable<CacheHealthStatus>;

  constructor(private cacheService: AdvancedCacheService) {}

  ngOnInit(): void {
    this.metrics$ = this.cacheService.getCacheMetrics();
    this.health$ = this.cacheService.getHealthStatus();
  }

  clearCache(): void {
    this.cacheService.clearCache();
  }

  warmCache(): void {
    const criticalEndpoints: CacheWarmingEndpoint[] = [
      { url: '/api/user/profile', ttl: 600000, priority: 10 },
      { url: '/api/dashboard/summary', ttl: 300000, priority: 9 },
      { url: '/api/notifications', ttl: 120000, priority: 8 }
    ];

    this.cacheService.warmCache(criticalEndpoints).subscribe(() => {
      console.log('Cache warming completed');
    });
  }

  runHealthCheck(): void {
    this.health$ = this.cacheService.getHealthStatus();
  }
}
```

## Summary

In this lesson, we explored advanced caching and sharing patterns that enable building high-performance reactive applications:

### Key Concepts Covered:
1. **Advanced Cache Implementation** - Smart caching with metadata, TTL, and eviction policies
2. **Multi-Level Cache Architecture** - Hierarchical caching with L1/L2/L3 levels
3. **Intelligent Invalidation** - Rule-based and dependency-aware cache invalidation
4. **Adaptive Caching** - Machine learning-inspired cache optimization
5. **Cache Coherency** - Maintaining consistency across distributed caches
6. **Memory Management** - Efficient memory usage and leak prevention
7. **Performance Monitoring** - Real-time metrics and health monitoring

### Best Practices:
- Implement multi-level caching for optimal performance
- Use intelligent invalidation to maintain data consistency
- Monitor cache metrics and adapt strategies dynamically
- Design for memory efficiency and prevent leaks
- Apply cache warming for critical application data
- Test caching strategies thoroughly

### Angular Integration:
- Service-level caching with HTTP interceptors
- Component-level cache management
- Real-time cache monitoring dashboards
- Cache health status and recommendations

### Next Steps:
- Practice implementing caching in real applications
- Explore distributed caching scenarios
- Study cache performance optimization techniques
- Learn advanced invalidation patterns

These patterns form the foundation for building scalable, high-performance reactive applications that can handle complex caching requirements while maintaining optimal user experience.
