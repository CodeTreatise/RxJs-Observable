# RxJS in Microservices Architecture

## Introduction

This lesson explores how RxJS and reactive patterns scale across microservices architectures. We'll cover service communication, distributed state management, event-driven architectures, and cross-service coordination using reactive programming principles.

## Learning Objectives

- Design reactive microservices communication patterns
- Implement distributed event streaming
- Handle cross-service dependencies reactively
- Build resilient inter-service communication
- Manage distributed state with observables
- Create reactive API gateways and service meshes

## Microservices Communication Patterns

### 1. Reactive Service-to-Service Communication

```typescript
// service-communication.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { Observable, Subject, BehaviorSubject, merge, timer, EMPTY } from 'rxjs';
import { 
  switchMap, 
  retry, 
  catchError, 
  shareReplay, 
  throttleTime,
  distinctUntilChanged,
  timeout,
  startWith,
  scan
} from 'rxjs/operators';

export interface ServiceEndpoint {
  name: string;
  url: string;
  version: string;
  health: 'healthy' | 'degraded' | 'unhealthy';
  lastCheck: Date;
}

export interface ServiceCall {
  service: string;
  method: string;
  payload?: any;
  timeout?: number;
  retries?: number;
}

@Injectable({
  providedIn: 'root'
})
export class MicroservicesCommunicationService {
  private serviceRegistry$ = new BehaviorSubject<Map<string, ServiceEndpoint>>(new Map());
  private circuitBreakers$ = new BehaviorSubject<Map<string, CircuitBreakerState>>(new Map());
  private serviceCallEvents$ = new Subject<ServiceCallEvent>();
  
  // Service health monitoring
  serviceHealth$: Observable<ServiceHealthReport> = timer(0, 30000).pipe(
    switchMap(() => this.checkAllServicesHealth()),
    shareReplay(1)
  );
  
  // Circuit breaker status monitoring
  circuitBreakerStatus$: Observable<Map<string, CircuitBreakerState>> = merge(
    this.circuitBreakers$,
    this.serviceCallEvents$.pipe(
      scan((breakers, event) => this.updateCircuitBreakers(breakers, event), new Map())
    )
  ).pipe(
    distinctUntilChanged(),
    shareReplay(1)
  );
  
  constructor(
    private http: HttpClient,
    private loadBalancer: LoadBalancerService,
    private serviceDiscovery: ServiceDiscoveryService,
    private metricsService: MetricsService
  ) {
    this.setupServiceDiscovery();
    this.setupHealthMonitoring();
    this.setupMetricsCollection();
  }
  
  // Main service call method with resilience patterns
  callService<T>(serviceCall: ServiceCall): Observable<T> {
    return this.getServiceEndpoint(serviceCall.service).pipe(
      switchMap(endpoint => {
        // Check circuit breaker
        if (this.isCircuitBreakerOpen(serviceCall.service)) {
          return this.handleCircuitBreakerOpen<T>(serviceCall);
        }
        
        return this.makeServiceCall<T>(endpoint, serviceCall).pipe(
          timeout(serviceCall.timeout || 5000),
          retry({
            count: serviceCall.retries || 3,
            delay: (error, retryCount) => this.calculateRetryDelay(error, retryCount)
          }),
          catchError(error => this.handleServiceCallError<T>(serviceCall, error))
        );
      })
    );
  }
  
  private makeServiceCall<T>(endpoint: ServiceEndpoint, call: ServiceCall): Observable<T> {
    const startTime = Date.now();
    
    return this.http.request<T>(call.method, `${endpoint.url}/${call.service}`, {
      body: call.payload
    }).pipe(
      tap(response => {
        const duration = Date.now() - startTime;
        this.recordServiceCallSuccess(call.service, duration);
      }),
      catchError(error => {
        const duration = Date.now() - startTime;
        this.recordServiceCallFailure(call.service, error, duration);
        throw error;
      })
    );
  }
  
  private getServiceEndpoint(serviceName: string): Observable<ServiceEndpoint> {
    return this.serviceRegistry$.pipe(
      map(registry => registry.get(serviceName)),
      switchMap(endpoint => {
        if (!endpoint) {
          return this.serviceDiscovery.discoverService(serviceName);
        }
        return of(endpoint);
      })
    );
  }
  
  private setupServiceDiscovery() {
    // Discover services and maintain registry
    this.serviceDiscovery.getServiceUpdates().subscribe(update => {
      const currentRegistry = this.serviceRegistry$.value;
      const updatedRegistry = new Map(currentRegistry);
      
      switch (update.type) {
        case 'SERVICE_REGISTERED':
          updatedRegistry.set(update.service.name, update.service);
          break;
        case 'SERVICE_DEREGISTERED':
          updatedRegistry.delete(update.service.name);
          break;
        case 'SERVICE_UPDATED':
          updatedRegistry.set(update.service.name, update.service);
          break;
      }
      
      this.serviceRegistry$.next(updatedRegistry);
    });
  }
  
  private setupHealthMonitoring() {
    // Continuous health checks
    this.serviceHealth$.subscribe(healthReport => {
      this.updateServiceHealth(healthReport);
    });
  }
  
  private checkAllServicesHealth(): Observable<ServiceHealthReport> {
    const services = Array.from(this.serviceRegistry$.value.values());
    
    return combineLatest(
      services.map(service => 
        this.checkServiceHealth(service).pipe(
          catchError(() => of({ ...service, health: 'unhealthy' as const }))
        )
      )
    ).pipe(
      map(healthResults => ({
        timestamp: new Date(),
        services: healthResults,
        overallHealth: this.calculateOverallHealth(healthResults)
      }))
    );
  }
  
  private isCircuitBreakerOpen(serviceName: string): boolean {
    const breaker = this.circuitBreakers$.value.get(serviceName);
    return breaker?.state === 'open';
  }
  
  private handleCircuitBreakerOpen<T>(call: ServiceCall): Observable<T> {
    // Return cached response or fallback
    return this.getFallbackResponse<T>(call).pipe(
      tap(() => {
        this.metricsService.incrementCounter('circuit_breaker_fallback', {
          service: call.service
        });
      })
    );
  }
}
```

### 2. Event-Driven Architecture

```typescript
// event-driven-microservices.service.ts
export interface DomainEvent {
  id: string;
  type: string;
  aggregateId: string;
  aggregateType: string;
  version: number;
  timestamp: Date;
  data: any;
  metadata?: Record<string, any>;
}

export interface EventStream {
  streamName: string;
  events: Observable<DomainEvent>;
  publish: (event: DomainEvent) => Observable<void>;
}

@Injectable({
  providedIn: 'root'
})
export class EventDrivenMicroservicesService {
  private eventStreams$ = new BehaviorSubject<Map<string, EventStream>>(new Map());
  private eventHandlers$ = new BehaviorSubject<Map<string, EventHandler[]>>(new Map());
  
  // Global event bus
  globalEventBus$: Observable<DomainEvent> = this.eventStreams$.pipe(
    switchMap(streams => 
      merge(...Array.from(streams.values()).map(stream => stream.events))
    ),
    shareReplay(1)
  );
  
  // Event sourcing projection
  projections$: Observable<Map<string, any>> = this.globalEventBus$.pipe(
    scan((projections, event) => this.updateProjections(projections, event), new Map()),
    shareReplay(1)
  );
  
  constructor(
    private websocketService: WebSocketService,
    private messageBroker: MessageBrokerService,
    private eventStore: EventStoreService
  ) {
    this.setupEventStreams();
    this.setupEventHandling();
    this.setupEventSourcing();
  }
  
  // Create or get event stream for a service
  getEventStream(serviceName: string): Observable<EventStream> {
    return this.eventStreams$.pipe(
      map(streams => streams.get(serviceName)),
      switchMap(stream => {
        if (stream) {
          return of(stream);
        }
        return this.createEventStream(serviceName);
      })
    );
  }
  
  private createEventStream(serviceName: string): Observable<EventStream> {
    // WebSocket-based event stream
    const events$ = this.websocketService.connect(`/events/${serviceName}`).pipe(
      map(message => this.parseEvent(message)),
      filter(event => !!event),
      share()
    );
    
    const publish = (event: DomainEvent): Observable<void> => {
      return this.eventStore.append(serviceName, event).pipe(
        switchMap(() => this.messageBroker.publish(`events.${serviceName}`, event)),
        map(() => void 0)
      );
    };
    
    const eventStream: EventStream = {
      streamName: serviceName,
      events: events$,
      publish
    };
    
    // Register stream
    const currentStreams = this.eventStreams$.value;
    const updatedStreams = new Map(currentStreams);
    updatedStreams.set(serviceName, eventStream);
    this.eventStreams$.next(updatedStreams);
    
    return of(eventStream);
  }
  
  // Saga pattern for distributed transactions
  createSaga(sagaDefinition: SagaDefinition): Observable<SagaExecution> {
    return new Observable(subscriber => {
      const sagaExecution: SagaExecution = {
        id: this.generateSagaId(),
        definition: sagaDefinition,
        currentStep: 0,
        state: 'running',
        compensations: [],
        startTime: new Date()
      };
      
      this.executeSagaStep(sagaExecution).subscribe({
        next: (result) => {
          if (result.state === 'completed') {
            subscriber.next(result);
            subscriber.complete();
          } else if (result.state === 'failed') {
            this.executeSagaCompensation(result).subscribe({
              next: (compensatedResult) => subscriber.next(compensatedResult),
              error: (error) => subscriber.error(error),
              complete: () => subscriber.complete()
            });
          } else {
            subscriber.next(result);
            // Continue to next step
            this.executeSagaStep(result).subscribe(subscriber);
          }
        },
        error: (error) => subscriber.error(error)
      });
    });
  }
  
  private executeSagaStep(execution: SagaExecution): Observable<SagaExecution> {
    const step = execution.definition.steps[execution.currentStep];
    if (!step) {
      return of({ ...execution, state: 'completed' });
    }
    
    return step.execute(execution.state).pipe(
      map(result => ({
        ...execution,
        currentStep: execution.currentStep + 1,
        compensations: [...execution.compensations, step.compensate]
      })),
      catchError(error => {
        return of({
          ...execution,
          state: 'failed',
          error: error.message
        });
      })
    );
  }
  
  // Event replay for debugging and recovery
  replayEvents(
    streamName: string, 
    fromTimestamp: Date, 
    toTimestamp?: Date
  ): Observable<DomainEvent[]> {
    return this.eventStore.getEvents(streamName, fromTimestamp, toTimestamp).pipe(
      map(events => events.sort((a, b) => a.timestamp.getTime() - b.timestamp.getTime()))
    );
  }
  
  // Event projection building
  buildProjection<T>(
    streamName: string,
    projectionBuilder: ProjectionBuilder<T>,
    fromVersion?: number
  ): Observable<T> {
    return this.getEventStream(streamName).pipe(
      switchMap(stream => stream.events),
      filter(event => !fromVersion || event.version >= fromVersion),
      scan((projection, event) => projectionBuilder.apply(projection, event), 
            projectionBuilder.empty()),
      distinctUntilChanged()
    );
  }
}
```

### 3. Reactive API Gateway

```typescript
// reactive-api-gateway.service.ts
export interface RouteConfig {
  path: string;
  method: string;
  targetService: string;
  targetPath: string;
  middleware: MiddlewareFunction[];
  rateLimit?: RateLimitConfig;
  cache?: CacheConfig;
  authentication?: AuthConfig;
}

export interface RequestContext {
  requestId: string;
  originalRequest: HttpRequest<any>;
  route: RouteConfig;
  user?: User;
  startTime: Date;
  metadata: Record<string, any>;
}

@Injectable({
  providedIn: 'root'
})
export class ReactiveApiGatewayService {
  private routes$ = new BehaviorSubject<RouteConfig[]>([]);
  private requestMetrics$ = new Subject<RequestMetric>();
  private rateLimiters$ = new BehaviorSubject<Map<string, RateLimiter>>(new Map());
  
  // Request processing pipeline
  processRequest$: Observable<HttpResponse<any>> = this.incomingRequests$.pipe(
    mergeMap(request => this.processRequestPipeline(request), 10), // Concurrency limit
    shareReplay(1)
  );
  
  // Real-time metrics
  metrics$: Observable<GatewayMetrics> = this.requestMetrics$.pipe(
    buffer(timer(5000)), // 5-second windows
    map(metrics => this.aggregateMetrics(metrics)),
    shareReplay(1)
  );
  
  constructor(
    private http: HttpClient,
    private authService: AuthenticationService,
    private cacheService: CacheService,
    private loadBalancer: LoadBalancerService,
    private circuitBreaker: CircuitBreakerService
  ) {
    this.setupRouteConfiguration();
    this.setupRateLimiting();
    this.setupMetricsCollection();
  }
  
  private processRequestPipeline(request: HttpRequest<any>): Observable<HttpResponse<any>> {
    const requestId = this.generateRequestId();
    const startTime = new Date();
    
    return this.matchRoute(request).pipe(
      switchMap(route => {
        const context: RequestContext = {
          requestId,
          originalRequest: request,
          route,
          startTime,
          metadata: {}
        };
        
        return this.executeMiddlewarePipeline(context).pipe(
          switchMap(ctx => this.forwardToService(ctx)),
          catchError(error => this.handleGatewayError(context, error))
        );
      }),
      tap(response => this.recordRequestMetrics(requestId, request, response, startTime)),
      catchError(error => this.handleGlobalError(request, error))
    );
  }
  
  private executeMiddlewarePipeline(context: RequestContext): Observable<RequestContext> {
    return from(context.route.middleware).pipe(
      concatMap(middleware => middleware(context)),
      scan((ctx, updatedCtx) => ({ ...ctx, ...updatedCtx }), context),
      last()
    );
  }
  
  private forwardToService(context: RequestContext): Observable<HttpResponse<any>> {
    const { route, originalRequest } = context;
    
    // Check cache first
    if (route.cache) {
      const cacheKey = this.generateCacheKey(context);
      const cached$ = this.cacheService.get(cacheKey);
      
      if (cached$) {
        return cached$.pipe(
          switchMap(cachedResponse => {
            if (cachedResponse && this.isCacheValid(cachedResponse, route.cache!)) {
              return of(cachedResponse);
            }
            return this.makeServiceCall(context).pipe(
              tap(response => this.cacheService.set(cacheKey, response, route.cache!.ttl))
            );
          })
        );
      }
    }
    
    return this.makeServiceCall(context);
  }
  
  private makeServiceCall(context: RequestContext): Observable<HttpResponse<any>> {
    const { route, originalRequest } = context;
    
    return this.loadBalancer.getServiceInstance(route.targetService).pipe(
      switchMap(serviceInstance => {
        const targetUrl = `${serviceInstance.url}${route.targetPath}`;
        
        // Check circuit breaker
        if (this.circuitBreaker.isOpen(route.targetService)) {
          return this.getFallbackResponse(context);
        }
        
        // Forward request
        const forwardedRequest = originalRequest.clone({
          url: targetUrl,
          setHeaders: {
            'X-Request-ID': context.requestId,
            'X-Forwarded-For': this.getClientIP(originalRequest),
            'X-Gateway-Time': context.startTime.toISOString()
          }
        });
        
        return this.http.request(forwardedRequest).pipe(
          timeout(30000), // 30-second timeout
          retry({
            count: 2,
            delay: error => timer(1000) // 1-second delay between retries
          }),
          catchError(error => this.handleServiceError(context, error))
        );
      })
    );
  }
  
  // Rate limiting middleware
  createRateLimitingMiddleware(config: RateLimitConfig): MiddlewareFunction {
    return (context: RequestContext): Observable<RequestContext> => {
      const identifier = this.getRateLimitIdentifier(context, config);
      const limiter = this.getRateLimiter(identifier, config);
      
      return limiter.checkLimit().pipe(
        map(allowed => {
          if (!allowed) {
            throw new HttpErrorResponse({
              status: 429,
              statusText: 'Too Many Requests',
              error: 'Rate limit exceeded'
            });
          }
          return context;
        })
      );
    };
  }
  
  // Authentication middleware
  createAuthenticationMiddleware(config: AuthConfig): MiddlewareFunction {
    return (context: RequestContext): Observable<RequestContext> => {
      const token = this.extractAuthToken(context.originalRequest);
      
      if (!token && config.required) {
        throw new HttpErrorResponse({
          status: 401,
          statusText: 'Unauthorized',
          error: 'Authentication required'
        });
      }
      
      if (token) {
        return this.authService.validateToken(token).pipe(
          map(user => ({
            ...context,
            user,
            metadata: { ...context.metadata, authenticated: true }
          })),
          catchError(error => {
            if (config.required) {
              throw new HttpErrorResponse({
                status: 401,
                statusText: 'Unauthorized',
                error: 'Invalid authentication token'
              });
            }
            return of(context);
          })
        );
      }
      
      return of(context);
    };
  }
  
  // Request transformation middleware
  createTransformationMiddleware(transformer: RequestTransformer): MiddlewareFunction {
    return (context: RequestContext): Observable<RequestContext> => {
      return transformer.transform(context.originalRequest).pipe(
        map(transformedRequest => ({
          ...context,
          originalRequest: transformedRequest
        }))
      );
    };
  }
}
```

## Service Mesh Integration

### 1. Service Discovery and Load Balancing

```typescript
// service-mesh.service.ts
export interface ServiceInstance {
  id: string;
  name: string;
  version: string;
  address: string;
  port: number;
  health: HealthStatus;
  metadata: Record<string, string>;
  weight: number;
  lastSeen: Date;
}

export interface LoadBalancingStrategy {
  name: string;
  selectInstance: (instances: ServiceInstance[]) => ServiceInstance;
}

@Injectable({
  providedIn: 'root'
})
export class ServiceMeshService {
  private serviceRegistry$ = new BehaviorSubject<Map<string, ServiceInstance[]>>(new Map());
  private loadBalancingStrategies$ = new BehaviorSubject<Map<string, LoadBalancingStrategy>>(new Map());
  private serviceHealth$ = new Subject<ServiceHealthUpdate>();
  
  // Active service monitoring
  activeServices$: Observable<ServiceInventory> = this.serviceRegistry$.pipe(
    map(registry => this.buildServiceInventory(registry)),
    shareReplay(1)
  );
  
  // Service topology
  serviceTopology$: Observable<ServiceTopology> = this.serviceHealth$.pipe(
    scan((topology, update) => this.updateTopology(topology, update), new ServiceTopology()),
    shareReplay(1)
  );
  
  constructor(
    private consulService: ConsulService,
    private eurekaService: EurekaService,
    private kubernetesService: KubernetesService
  ) {
    this.setupServiceDiscovery();
    this.setupHealthMonitoring();
    this.setupLoadBalancingStrategies();
  }
  
  // Get service instance with load balancing
  getServiceInstance(serviceName: string, strategy?: string): Observable<ServiceInstance> {
    return combineLatest([
      this.serviceRegistry$,
      this.loadBalancingStrategies$
    ]).pipe(
      map(([registry, strategies]) => {
        const instances = registry.get(serviceName) || [];
        const healthyInstances = instances.filter(instance => instance.health === 'healthy');
        
        if (healthyInstances.length === 0) {
          throw new Error(`No healthy instances available for service: ${serviceName}`);
        }
        
        const strategyName = strategy || 'round-robin';
        const loadBalancer = strategies.get(strategyName);
        
        if (!loadBalancer) {
          throw new Error(`Unknown load balancing strategy: ${strategyName}`);
        }
        
        return loadBalancer.selectInstance(healthyInstances);
      })
    );
  }
  
  // Register service instance
  registerService(instance: ServiceInstance): Observable<void> {
    return this.consulService.registerService(instance).pipe(
      tap(() => this.updateLocalRegistry(instance)),
      map(() => void 0)
    );
  }
  
  // Service discovery from multiple sources
  private setupServiceDiscovery() {
    // Consul discovery
    const consulServices$ = this.consulService.watchServices().pipe(
      map(services => ({ source: 'consul', services }))
    );
    
    // Kubernetes discovery
    const k8sServices$ = this.kubernetesService.watchServices().pipe(
      map(services => ({ source: 'kubernetes', services }))
    );
    
    // Eureka discovery
    const eurekaServices$ = this.eurekaService.watchServices().pipe(
      map(services => ({ source: 'eureka', services }))
    );
    
    // Merge all discovery sources
    merge(consulServices$, k8sServices$, eurekaServices$).subscribe(
      ({ source, services }) => {
        this.updateServiceRegistry(services, source);
      }
    );
  }
  
  private setupLoadBalancingStrategies() {
    const strategies = new Map<string, LoadBalancingStrategy>();
    
    // Round Robin
    let roundRobinIndex = 0;
    strategies.set('round-robin', {
      name: 'round-robin',
      selectInstance: (instances) => {
        const instance = instances[roundRobinIndex % instances.length];
        roundRobinIndex++;
        return instance;
      }
    });
    
    // Weighted Round Robin
    strategies.set('weighted-round-robin', {
      name: 'weighted-round-robin',
      selectInstance: (instances) => {
        const totalWeight = instances.reduce((sum, instance) => sum + instance.weight, 0);
        const random = Math.random() * totalWeight;
        let currentWeight = 0;
        
        for (const instance of instances) {
          currentWeight += instance.weight;
          if (random <= currentWeight) {
            return instance;
          }
        }
        
        return instances[0]; // Fallback
      }
    });
    
    // Least Connections
    strategies.set('least-connections', {
      name: 'least-connections',
      selectInstance: (instances) => {
        return instances.reduce((least, current) => 
          (current.metadata.activeConnections || 0) < (least.metadata.activeConnections || 0) 
            ? current 
            : least
        );
      }
    });
    
    // Random
    strategies.set('random', {
      name: 'random',
      selectInstance: (instances) => {
        const randomIndex = Math.floor(Math.random() * instances.length);
        return instances[randomIndex];
      }
    });
    
    this.loadBalancingStrategies$.next(strategies);
  }
}
```

### 2. Distributed Tracing

```typescript
// distributed-tracing.service.ts
export interface TraceSpan {
  traceId: string;
  spanId: string;
  parentSpanId?: string;
  operationName: string;
  startTime: Date;
  endTime?: Date;
  duration?: number;
  tags: Record<string, any>;
  logs: TraceLog[];
  status: 'ok' | 'error' | 'timeout';
}

export interface TraceContext {
  traceId: string;
  spanId: string;
  baggage?: Record<string, string>;
}

@Injectable({
  providedIn: 'root'
})
export class DistributedTracingService {
  private activeSpans$ = new BehaviorSubject<Map<string, TraceSpan>>(new Map());
  private traceContext$ = new BehaviorSubject<TraceContext | null>(null);
  
  // HTTP interceptor for automatic tracing
  createTracingInterceptor(): HttpInterceptor {
    return {
      intercept: (req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> => {
        const traceContext = this.getCurrentTraceContext();
        const span = this.startSpan(`HTTP ${req.method} ${req.url}`, traceContext);
        
        // Add tracing headers
        const tracedRequest = req.clone({
          setHeaders: {
            'X-Trace-ID': span.traceId,
            'X-Span-ID': span.spanId,
            'X-Parent-Span-ID': span.parentSpanId || ''
          }
        });
        
        return next.handle(tracedRequest).pipe(
          tap({
            next: (event) => {
              if (event instanceof HttpResponse) {
                this.addSpanTag(span.spanId, 'http.status_code', event.status);
                this.finishSpan(span.spanId);
              }
            },
            error: (error) => {
              this.addSpanTag(span.spanId, 'error', true);
              this.addSpanTag(span.spanId, 'error.message', error.message);
              this.finishSpan(span.spanId, 'error');
            }
          })
        );
      }
    };
  }
  
  // Start a new trace span
  startSpan(operationName: string, parentContext?: TraceContext): TraceSpan {
    const traceId = parentContext?.traceId || this.generateTraceId();
    const spanId = this.generateSpanId();
    
    const span: TraceSpan = {
      traceId,
      spanId,
      parentSpanId: parentContext?.spanId,
      operationName,
      startTime: new Date(),
      tags: {},
      logs: [],
      status: 'ok'
    };
    
    // Update active spans
    const activeSpans = this.activeSpans$.value;
    activeSpans.set(spanId, span);
    this.activeSpans$.next(activeSpans);
    
    return span;
  }
  
  // Finish a span
  finishSpan(spanId: string, status: 'ok' | 'error' | 'timeout' = 'ok'): void {
    const activeSpans = this.activeSpans$.value;
    const span = activeSpans.get(spanId);
    
    if (span) {
      span.endTime = new Date();
      span.duration = span.endTime.getTime() - span.startTime.getTime();
      span.status = status;
      
      // Send to tracing backend
      this.sendSpanToBackend(span);
      
      // Remove from active spans
      activeSpans.delete(spanId);
      this.activeSpans$.next(activeSpans);
    }
  }
  
  // Add tag to span
  addSpanTag(spanId: string, key: string, value: any): void {
    const activeSpans = this.activeSpans$.value;
    const span = activeSpans.get(spanId);
    
    if (span) {
      span.tags[key] = value;
      this.activeSpans$.next(activeSpans);
    }
  }
  
  // Service call tracing decorator
  traceServiceCall<T>(
    serviceName: string,
    operationName: string,
    serviceCall: () => Observable<T>
  ): Observable<T> {
    return new Observable(subscriber => {
      const span = this.startSpan(`${serviceName}.${operationName}`);
      
      this.addSpanTag(span.spanId, 'service.name', serviceName);
      this.addSpanTag(span.spanId, 'operation.name', operationName);
      
      const subscription = serviceCall().subscribe({
        next: (value) => {
          subscriber.next(value);
        },
        error: (error) => {
          this.addSpanTag(span.spanId, 'error', true);
          this.addSpanTag(span.spanId, 'error.message', error.message);
          this.finishSpan(span.spanId, 'error');
          subscriber.error(error);
        },
        complete: () => {
          this.finishSpan(span.spanId);
          subscriber.complete();
        }
      });
      
      return () => {
        subscription.unsubscribe();
        this.finishSpan(span.spanId, 'timeout');
      };
    });
  }
  
  private sendSpanToBackend(span: TraceSpan): void {
    // Send to Jaeger, Zipkin, or other tracing backend
    this.http.post('/api/traces', span).subscribe({
      error: (error) => console.error('Failed to send trace span:', error)
    });
  }
}
```

## Distributed State Management

### 1. Event Sourcing Across Services

```typescript
// distributed-event-sourcing.service.ts
export interface DistributedEvent {
  id: string;
  streamId: string;
  eventType: string;
  aggregateId: string;
  aggregateType: string;
  version: number;
  timestamp: Date;
  data: any;
  metadata: {
    correlationId?: string;
    causationId?: string;
    userId?: string;
    serviceId: string;
  };
}

export interface EventStoreConnection {
  streamName: string;
  connection: Observable<DistributedEvent>;
  append: (event: DistributedEvent) => Observable<void>;
}

@Injectable({
  providedIn: 'root'
})
export class DistributedEventSourcingService {
  private eventConnections$ = new BehaviorSubject<Map<string, EventStoreConnection>>(new Map());
  private eventProjections$ = new BehaviorSubject<Map<string, any>>(new Map());
  private sagaManagers$ = new BehaviorSubject<Map<string, SagaManager>>(new Map());
  
  // Global event stream across all services
  globalEventStream$: Observable<DistributedEvent> = this.eventConnections$.pipe(
    switchMap(connections => 
      merge(...Array.from(connections.values()).map(conn => conn.connection))
    ),
    shareReplay(1)
  );
  
  // Cross-service projections
  crossServiceProjections$: Observable<Map<string, any>> = this.globalEventStream$.pipe(
    scan((projections, event) => this.updateCrossServiceProjections(projections, event), new Map()),
    shareReplay(1)
  );
  
  constructor(
    private eventStoreService: EventStoreService,
    private messageBroker: MessageBrokerService,
    private sagaOrchestrator: SagaOrchestratorService
  ) {
    this.setupEventStreams();
    this.setupSagaManagement();
    this.setupProjectionUpdates();
  }
  
  // Connect to an event stream
  connectToEventStream(streamName: string): Observable<EventStoreConnection> {
    const existingConnection = this.eventConnections$.value.get(streamName);
    if (existingConnection) {
      return of(existingConnection);
    }
    
    return this.createEventStreamConnection(streamName).pipe(
      tap(connection => {
        const connections = this.eventConnections$.value;
        connections.set(streamName, connection);
        this.eventConnections$.next(connections);
      })
    );
  }
  
  private createEventStreamConnection(streamName: string): Observable<EventStoreConnection> {
    // WebSocket connection to event store
    const eventStream$ = this.eventStoreService.subscribeToStream(streamName).pipe(
      map(event => this.enrichEventWithMetadata(event)),
      share()
    );
    
    const appendEvent = (event: DistributedEvent): Observable<void> => {
      return this.eventStoreService.appendToStream(streamName, event).pipe(
        switchMap(() => this.messageBroker.publish(`events.${streamName}`, event)),
        map(() => void 0)
      );
    };
    
    return of({
      streamName,
      connection: eventStream$,
      append: appendEvent
    });
  }
  
  // Distributed saga coordination
  startDistributedSaga(sagaType: string, initialEvent: DistributedEvent): Observable<SagaExecution> {
    const sagaId = this.generateSagaId();
    const sagaManager = this.sagaManagers$.value.get(sagaType);
    
    if (!sagaManager) {
      throw new Error(`No saga manager found for type: ${sagaType}`);
    }
    
    return sagaManager.startSaga(sagaId, initialEvent).pipe(
      switchMap(saga => this.monitorSagaExecution(saga))
    );
  }
  
  private monitorSagaExecution(saga: SagaExecution): Observable<SagaExecution> {
    return this.globalEventStream$.pipe(
      filter(event => event.metadata.correlationId === saga.correlationId),
      scan((currentSaga, event) => this.updateSagaState(currentSaga, event), saga),
      takeWhile(sagaState => sagaState.status !== 'completed' && sagaState.status !== 'failed'),
      share()
    );
  }
  
  // Cross-service read models
  buildCrossServiceReadModel<T>(
    readModelId: string,
    eventTypes: string[],
    projectionFunction: (current: T, event: DistributedEvent) => T,
    initialState: T
  ): Observable<T> {
    return this.globalEventStream$.pipe(
      filter(event => eventTypes.includes(event.eventType)),
      scan((readModel, event) => projectionFunction(readModel, event), initialState),
      distinctUntilChanged(),
      shareReplay(1)
    );
  }
  
  // Event replay for service recovery
  replayEventsForService(
    serviceId: string,
    fromTimestamp: Date,
    eventHandler: (event: DistributedEvent) => Observable<void>
  ): Observable<void> {
    return this.eventStoreService.getEventsForService(serviceId, fromTimestamp).pipe(
      mergeMap(events => from(events)),
      concatMap(event => eventHandler(event)),
      map(() => void 0)
    );
  }
}
```

### 2. Distributed CQRS Implementation

```typescript
// distributed-cqrs.service.ts
export interface Command {
  id: string;
  type: string;
  aggregateId: string;
  payload: any;
  metadata: {
    userId: string;
    timestamp: Date;
    correlationId: string;
  };
}

export interface Query {
  id: string;
  type: string;
  parameters: any;
  metadata: {
    userId?: string;
    timestamp: Date;
  };
}

export interface ReadModel {
  id: string;
  version: number;
  data: any;
  lastUpdated: Date;
}

@Injectable({
  providedIn: 'root'
})
export class DistributedCQRSService {
  private commandHandlers$ = new BehaviorSubject<Map<string, CommandHandler>>(new Map());
  private queryHandlers$ = new BehaviorSubject<Map<string, QueryHandler>>(new Map());
  private readModels$ = new BehaviorSubject<Map<string, ReadModel>>(new Map());
  
  // Command processing pipeline
  commandPipeline$: Observable<CommandResult> = this.incomingCommands$.pipe(
    mergeMap(command => this.processCommand(command), 5), // Limit concurrency
    shareReplay(1)
  );
  
  // Query processing pipeline
  queryPipeline$: Observable<QueryResult> = this.incomingQueries$.pipe(
    mergeMap(query => this.processQuery(query), 10), // Higher concurrency for reads
    shareReplay(1)
  );
  
  constructor(
    private commandBus: CommandBusService,
    private queryBus: QueryBusService,
    private eventSourcing: DistributedEventSourcingService,
    private readModelStore: ReadModelStoreService
  ) {
    this.setupCommandHandling();
    this.setupQueryHandling();
    this.setupReadModelUpdates();
  }
  
  // Send command to appropriate service
  sendCommand(command: Command): Observable<CommandResult> {
    // Validate command
    const validationResult = this.validateCommand(command);
    if (!validationResult.isValid) {
      return throwError(() => new Error(validationResult.error));
    }
    
    // Route to appropriate service
    const targetService = this.getCommandTargetService(command.type);
    
    return this.commandBus.send(targetService, command).pipe(
      timeout(30000), // 30-second timeout
      retry({
        count: 2,
        delay: error => timer(1000)
      }),
      catchError(error => this.handleCommandError(command, error))
    );
  }
  
  // Execute query against read models
  executeQuery<T>(query: Query): Observable<T> {
    const queryHandler = this.queryHandlers$.value.get(query.type);
    
    if (!queryHandler) {
      return throwError(() => new Error(`No query handler found for: ${query.type}`));
    }
    
    return queryHandler.handle(query).pipe(
      timeout(10000), // 10-second timeout for queries
      catchError(error => this.handleQueryError(query, error))
    );
  }
  
  private processCommand(command: Command): Observable<CommandResult> {
    const handler = this.commandHandlers$.value.get(command.type);
    
    if (!handler) {
      return of({
        commandId: command.id,
        success: false,
        error: `No handler found for command type: ${command.type}`
      });
    }
    
    return handler.handle(command).pipe(
      map(result => ({
        commandId: command.id,
        success: true,
        result
      })),
      catchError(error => of({
        commandId: command.id,
        success: false,
        error: error.message
      }))
    );
  }
  
  private processQuery(query: Query): Observable<QueryResult> {
    const handler = this.queryHandlers$.value.get(query.type);
    
    if (!handler) {
      return of({
        queryId: query.id,
        success: false,
        error: `No handler found for query type: ${query.type}`
      });
    }
    
    return handler.handle(query).pipe(
      map(result => ({
        queryId: query.id,
        success: true,
        data: result
      })),
      catchError(error => of({
        queryId: query.id,
        success: false,
        error: error.message
      }))
    );
  }
  
  // Read model synchronization across services
  synchronizeReadModels(): Observable<SyncResult> {
    return this.eventSourcing.globalEventStream$.pipe(
      buffer(timer(5000)), // Batch events every 5 seconds
      filter(events => events.length > 0),
      mergeMap(events => this.updateReadModelsFromEvents(events)),
      map(updates => ({
        timestamp: new Date(),
        updatedModels: updates.length,
        success: true
      })),
      catchError(error => of({
        timestamp: new Date(),
        updatedModels: 0,
        success: false,
        error: error.message
      }))
    );
  }
  
  private updateReadModelsFromEvents(events: DistributedEvent[]): Observable<ReadModelUpdate[]> {
    return from(events).pipe(
      groupBy(event => event.aggregateType),
      mergeMap(groupedEvents => 
        groupedEvents.pipe(
          toArray(),
          mergeMap(typeEvents => this.updateReadModelsForAggregate(typeEvents))
        )
      ),
      toArray()
    );
  }
  
  // Eventual consistency monitoring
  monitorEventualConsistency(): Observable<ConsistencyReport> {
    return interval(30000).pipe( // Check every 30 seconds
      switchMap(() => this.checkConsistencyAcrossServices()),
      shareReplay(1)
    );
  }
  
  private checkConsistencyAcrossServices(): Observable<ConsistencyReport> {
    return combineLatest([
      this.getServiceVersions(),
      this.getReadModelVersions()
    ]).pipe(
      map(([serviceVersions, readModelVersions]) => {
        const inconsistencies = this.findInconsistencies(serviceVersions, readModelVersions);
        
        return {
          timestamp: new Date(),
          isConsistent: inconsistencies.length === 0,
          inconsistencies,
          lagTime: this.calculateMaxLagTime(serviceVersions, readModelVersions)
        };
      })
    );
  }
}
```

## Performance and Scalability

### 1. Reactive Caching Strategies

```typescript
// distributed-caching.service.ts
export interface CacheEntry<T> {
  key: string;
  value: T;
  timestamp: Date;
  ttl: number;
  tags: string[];
  dependencies: string[];
}

export interface CacheInvalidationEvent {
  type: 'invalidate' | 'refresh';
  keys: string[];
  tags: string[];
  reason: string;
}

@Injectable({
  providedIn: 'root'
})
export class DistributedCachingService {
  private localCache$ = new BehaviorSubject<Map<string, CacheEntry<any>>>(new Map());
  private invalidationEvents$ = new Subject<CacheInvalidationEvent>();
  
  // Cache statistics
  cacheStats$: Observable<CacheStatistics> = interval(10000).pipe(
    map(() => this.calculateCacheStatistics()),
    shareReplay(1)
  );
  
  // Distributed cache synchronization
  cacheSyncEvents$: Observable<CacheSyncEvent> = this.messageBroker.subscribe('cache.sync').pipe(
    map(event => this.parseCacheSyncEvent(event)),
    shareReplay(1)
  );
  
  constructor(
    private redisService: RedisService,
    private messageBroker: MessageBrokerService,
    private compressionService: CompressionService
  ) {
    this.setupCacheInvalidation();
    this.setupDistributedSync();
    this.setupCacheEviction();
  }
  
  // Get with automatic population
  get<T>(
    key: string,
    factory?: () => Observable<T>,
    options?: CacheOptions
  ): Observable<T | null> {
    return this.getFromLocalCache<T>(key).pipe(
      switchMap(cached => {
        if (cached && this.isCacheValid(cached)) {
          return of(cached.value);
        }
        
        // Try distributed cache
        return this.getFromDistributedCache<T>(key).pipe(
          switchMap(distributedCached => {
            if (distributedCached && this.isCacheValid(distributedCached)) {
              // Update local cache
              this.setLocalCache(key, distributedCached);
              return of(distributedCached.value);
            }
            
            // Use factory if provided
            if (factory) {
              return factory().pipe(
                tap(value => {
                  if (value !== null && value !== undefined) {
                    this.set(key, value, options);
                  }
                })
              );
            }
            
            return of(null);
          })
        );
      })
    );
  }
  
  // Set with distributed propagation
  set<T>(key: string, value: T, options?: CacheOptions): Observable<void> {
    const entry: CacheEntry<T> = {
      key,
      value,
      timestamp: new Date(),
      ttl: options?.ttl || 300000, // 5 minutes default
      tags: options?.tags || [],
      dependencies: options?.dependencies || []
    };
    
    return this.setLocalCache(key, entry).pipe(
      switchMap(() => this.setDistributedCache(key, entry)),
      switchMap(() => this.notifyOtherServices(key, 'set')),
      map(() => void 0)
    );
  }
  
  // Cache-aside pattern with reactive updates
  cacheAside<T>(
    key: string,
    dataSource: () => Observable<T>,
    options?: CacheOptions
  ): Observable<T> {
    return this.get<T>(key).pipe(
      switchMap(cached => {
        if (cached !== null) {
          // Schedule background refresh if near expiry
          this.scheduleBackgroundRefresh(key, dataSource, options);
          return of(cached);
        }
        
        // Load from data source
        return dataSource().pipe(
          tap(data => this.set(key, data, options))
        );
      })
    );
  }
  
  // Write-through caching
  writeThrough<T>(
    key: string,
    value: T,
    dataStore: (value: T) => Observable<T>,
    options?: CacheOptions
  ): Observable<T> {
    return dataStore(value).pipe(
      switchMap(storedValue => 
        this.set(key, storedValue, options).pipe(
          map(() => storedValue)
        )
      )
    );
  }
  
  // Write-behind caching with batching
  writeBehind<T>(
    key: string,
    value: T,
    options?: CacheOptions
  ): Observable<void> {
    // Immediately update cache
    return this.set(key, value, options).pipe(
      tap(() => {
        // Queue for background persistence
        this.queueForPersistence(key, value);
      })
    );
  }
  
  // Intelligent prefetching
  prefetch(keys: string[], factory: (key: string) => Observable<any>): Observable<void> {
    return from(keys).pipe(
      mergeMap(key => 
        this.get(key).pipe(
          switchMap(cached => {
            if (cached === null) {
              return factory(key).pipe(
                switchMap(value => this.set(key, value))
              );
            }
            return of(void 0);
          }),
          catchError(error => {
            console.warn(`Prefetch failed for key ${key}:`, error);
            return of(void 0);
          })
        ),
        3 // Limit concurrency
      ),
      toArray(),
      map(() => void 0)
    );
  }
  
  // Tag-based invalidation
  invalidateByTags(tags: string[]): Observable<void> {
    return this.getKeysByTags(tags).pipe(
      switchMap(keys => this.invalidateKeys(keys)),
      tap(() => {
        this.invalidationEvents$.next({
          type: 'invalidate',
          keys: [],
          tags,
          reason: 'tag-based invalidation'
        });
      })
    );
  }
  
  // Dependency-based invalidation
  invalidateByDependencies(dependencies: string[]): Observable<void> {
    return this.getKeysByDependencies(dependencies).pipe(
      switchMap(keys => this.invalidateKeys(keys))
    );
  }
  
  private scheduleBackgroundRefresh<T>(
    key: string,
    dataSource: () => Observable<T>,
    options?: CacheOptions
  ): void {
    const entry = this.localCache$.value.get(key);
    if (!entry) return;
    
    const timeToExpiry = (entry.timestamp.getTime() + entry.ttl) - Date.now();
    const refreshTime = timeToExpiry * 0.8; // Refresh at 80% of TTL
    
    if (refreshTime > 0) {
      timer(refreshTime).pipe(
        switchMap(() => dataSource()),
        take(1)
      ).subscribe(
        data => this.set(key, data, options),
        error => console.warn(`Background refresh failed for key ${key}:`, error)
      );
    }
  }
}
```

## Monitoring and Observability

### 1. Distributed Metrics Collection

```typescript
// distributed-metrics.service.ts
export interface MetricData {
  name: string;
  value: number;
  timestamp: Date;
  tags: Record<string, string>;
  type: 'counter' | 'gauge' | 'histogram' | 'timer';
}

export interface ServiceMetrics {
  serviceId: string;
  metrics: MetricData[];
  systemMetrics: SystemMetrics;
  customMetrics: Record<string, any>;
}

@Injectable({
  providedIn: 'root'
})
export class DistributedMetricsService {
  private metricsBuffer$ = new BehaviorSubject<MetricData[]>([]);
  private serviceMetrics$ = new Subject<ServiceMetrics>();
  
  // Aggregated metrics across services
  aggregatedMetrics$: Observable<AggregatedMetrics> = this.serviceMetrics$.pipe(
    buffer(timer(10000)), // 10-second windows
    map(metricsArray => this.aggregateMetrics(metricsArray)),
    shareReplay(1)
  );
  
  // Real-time dashboards
  realTimeDashboard$: Observable<DashboardData> = interval(5000).pipe(
    switchMap(() => this.buildDashboardData()),
    shareReplay(1)
  );
  
  constructor(
    private prometheusService: PrometheusService,
    private grafanaService: GrafanaService,
    private alertingService: AlertingService
  ) {
    this.setupMetricsCollection();
    this.setupAlerting();
    this.setupMetricsForwarding();
  }
  
  // RxJS operator for automatic metrics collection
  collectMetrics<T>(
    metricName: string,
    tags?: Record<string, string>
  ): MonoTypeOperatorFunction<T> {
    return (source: Observable<T>) => {
      return new Observable(subscriber => {
        const startTime = Date.now();
        let itemCount = 0;
        
        return source.subscribe({
          next: (value) => {
            itemCount++;
            this.incrementCounter(`${metricName}.items`, tags);
            subscriber.next(value);
          },
          error: (error) => {
            const duration = Date.now() - startTime;
            this.recordTimer(`${metricName}.duration`, duration, { ...tags, status: 'error' });
            this.incrementCounter(`${metricName}.errors`, tags);
            subscriber.error(error);
          },
          complete: () => {
            const duration = Date.now() - startTime;
            this.recordTimer(`${metricName}.duration`, duration, { ...tags, status: 'success' });
            this.setGauge(`${metricName}.total_items`, itemCount, tags);
            subscriber.complete();
          }
        });
      });
    };
  }
  
  // HTTP request metrics interceptor
  createMetricsInterceptor(): HttpInterceptor {
    return {
      intercept: (req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> => {
        const startTime = Date.now();
        const tags = {
          method: req.method,
          url: req.url,
          service: this.extractServiceName(req.url)
        };
        
        this.incrementCounter('http.requests.total', tags);
        
        return next.handle(req).pipe(
          tap({
            next: (event) => {
              if (event instanceof HttpResponse) {
                const duration = Date.now() - startTime;
                this.recordTimer('http.request.duration', duration, {
                  ...tags,
                  status_code: event.status.toString()
                });
                this.incrementCounter('http.responses.total', {
                  ...tags,
                  status_code: event.status.toString()
                });
              }
            },
            error: (error) => {
              const duration = Date.now() - startTime;
              this.recordTimer('http.request.duration', duration, {
                ...tags,
                status_code: error.status?.toString() || 'unknown'
              });
              this.incrementCounter('http.errors.total', {
                ...tags,
                error_type: error.name
              });
            }
          })
        );
      }
    };
  }
  
  // Stream performance monitoring
  monitorStreamPerformance<T>(
    streamName: string,
    source: Observable<T>
  ): Observable<T> {
    return source.pipe(
      this.collectMetrics(`stream.${streamName}`),
      tap({
        next: () => this.recordGauge(`stream.${streamName}.active_subscribers`, 1),
        error: () => this.incrementCounter(`stream.${streamName}.errors`),
        complete: () => this.recordGauge(`stream.${streamName}.active_subscribers`, 0)
      })
    );
  }
  
  // Circuit breaker metrics
  monitorCircuitBreaker(
    serviceName: string,
    circuitBreaker: Observable<CircuitBreakerState>
  ): Observable<CircuitBreakerState> {
    return circuitBreaker.pipe(
      tap(state => {
        this.recordGauge('circuit_breaker.state', state.state === 'open' ? 1 : 0, {
          service: serviceName
        });
        this.recordGauge('circuit_breaker.failure_count', state.failureCount, {
          service: serviceName
        });
        this.recordGauge('circuit_breaker.success_count', state.successCount, {
          service: serviceName
        });
      })
    );
  }
  
  // SLA monitoring
  monitorSLA(
    serviceName: string,
    slaThresholds: SLAThresholds
  ): Observable<SLAReport> {
    return interval(60000).pipe( // Check every minute
      switchMap(() => this.calculateSLA(serviceName, slaThresholds)),
      tap(slaReport => {
        this.recordGauge('sla.availability', slaReport.availability, {
          service: serviceName
        });
        this.recordGauge('sla.response_time_p99', slaReport.responseTimeP99, {
          service: serviceName
        });
        this.recordGauge('sla.error_rate', slaReport.errorRate, {
          service: serviceName
        });
        
        // Trigger alerts if SLA is breached
        if (slaReport.availability < slaThresholds.availability ||
            slaReport.responseTimeP99 > slaThresholds.responseTime ||
            slaReport.errorRate > slaThresholds.errorRate) {
          this.alertingService.triggerSLAAlert(serviceName, slaReport);
        }
      })
    );
  }
  
  // Resource utilization monitoring
  monitorResourceUtilization(): Observable<ResourceUtilization> {
    return interval(30000).pipe( // Check every 30 seconds
      map(() => ({
        timestamp: new Date(),
        cpu: this.getCPUUsage(),
        memory: this.getMemoryUsage(),
        disk: this.getDiskUsage(),
        network: this.getNetworkUsage()
      })),
      tap(utilization => {
        this.recordGauge('system.cpu.usage', utilization.cpu);
        this.recordGauge('system.memory.usage', utilization.memory);
        this.recordGauge('system.disk.usage', utilization.disk);
        this.recordGauge('system.network.usage', utilization.network);
      })
    );
  }
}
```

## Best Practices & Patterns

### 1. Microservices Design Principles

```typescript
// Service Autonomy
class UserService {
  // ✅ Self-contained with own data store
  private userRepository: UserRepository;
  
  // ✅ Publishes events for other services
  getUserEvents(): Observable<UserEvent> {
    return this.eventStream$;
  }
  
  // ✅ Minimal dependencies on other services
  validateUser(userId: string): Observable<boolean> {
    return this.userRepository.exists(userId);
  }
}

// ✅ Event-driven communication
class OrderService {
  constructor(private eventBus: EventBusService) {
    // Subscribe to user events
    this.eventBus.subscribe('user.created').subscribe(event => {
      this.handleUserCreated(event);
    });
  }
}
```

### 2. Error Handling Strategies

```typescript
// Bulkhead pattern
class ResilientMicroservice {
  // Separate thread pools for different operations
  private criticalOperations$ = new Subject().pipe(
    mergeMap(op => this.executeCritical(op), 2) // Limited concurrency
  );
  
  private nonCriticalOperations$ = new Subject().pipe(
    mergeMap(op => this.executeNonCritical(op), 10) // Higher concurrency
  );
  
  // Circuit breaker for external dependencies
  private externalApiCall$ = this.api$.pipe(
    retryWhen(errors => 
      errors.pipe(
        scan((retryCount, error) => {
          if (retryCount >= 3) throw error;
          return retryCount + 1;
        }, 0),
        delay(1000)
      )
    )
  );
}
```

## Summary

In this comprehensive lesson, we've covered:

- ✅ Reactive microservices communication patterns
- ✅ Event-driven architecture implementation
- ✅ Service mesh integration with RxJS
- ✅ Distributed state management strategies
- ✅ Reactive API gateway patterns
- ✅ Performance monitoring and observability
- ✅ Distributed tracing and debugging
- ✅ Resilience patterns and error handling

These patterns enable building scalable, resilient microservices architectures using reactive programming principles with RxJS.

## Next Steps

This completes Module 8: Real-World Applications & Patterns. Next, we'll begin **Module 9: RxJS Ecosystem & Future**, exploring the broader RxJS ecosystem, best practices, and upcoming features in reactive programming.
