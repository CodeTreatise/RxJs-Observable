# Advanced Reactive Programming Patterns

## Learning Objectives
By the end of this lesson, you will be able to:
- Master advanced reactive programming patterns and architectures
- Implement complex state management with reactive streams
- Apply reactive patterns to solve real-world business problems
- Design event-driven architectures using RxJS
- Optimize reactive systems for performance and maintainability

## Table of Contents
1. [Reactive System Architecture](#reactive-system-architecture)
2. [Event Sourcing Patterns](#event-sourcing-patterns)
3. [Command Query Responsibility Segregation (CQRS)](#command-query-responsibility-segregation-cqrs)
4. [Reactive State Management](#reactive-state-management)
5. [Event-Driven Architecture](#event-driven-architecture)
6. [Reactive Microservices](#reactive-microservices)
7. [Actor Pattern with RxJS](#actor-pattern-with-rxjs)
8. [Saga Pattern Implementation](#saga-pattern-implementation)
9. [Angular Examples](#angular-examples)
10. [Best Practices](#best-practices)
11. [Testing Strategies](#testing-strategies)
12. [Exercises](#exercises)

## Reactive System Architecture

### Reactive System Principles

The reactive manifesto defines four key principles:

```typescript
interface ReactiveSystem {
  responsive: boolean;    // System responds in a timely manner
  resilient: boolean;     // System stays responsive in face of failure
  elastic: boolean;       // System stays responsive under varying workload
  message_driven: boolean; // System relies on asynchronous message-passing
}
```

### Message-Driven Architecture

```typescript
import { Subject, Observable, BehaviorSubject, merge } from 'rxjs';
import { filter, map, tap, catchError, retry, takeUntil } from 'rxjs/operators';

// Message types for type safety
interface Message {
  id: string;
  type: string;
  payload: any;
  timestamp: number;
  source: string;
}

interface Command extends Message {
  type: 'COMMAND';
}

interface Event extends Message {
  type: 'EVENT';
}

interface Query extends Message {
  type: 'QUERY';
}

// Message Bus Implementation
class MessageBus {
  private messageStream$ = new Subject<Message>();
  private destroyed$ = new Subject<void>();

  // Publish messages
  publish(message: Message): void {
    this.messageStream$.next({
      ...message,
      timestamp: Date.now(),
      id: message.id || this.generateId()
    });
  }

  // Subscribe to specific message types
  subscribe<T extends Message>(
    messageType: string,
    predicate?: (message: T) => boolean
  ): Observable<T> {
    return this.messageStream$.pipe(
      filter((message): message is T => message.type === messageType),
      filter(predicate || (() => true)),
      takeUntil(this.destroyed$)
    );
  }

  // Subscribe to all messages
  subscribeToAll(): Observable<Message> {
    return this.messageStream$.pipe(
      takeUntil(this.destroyed$)
    );
  }

  destroy(): void {
    this.destroyed$.next();
    this.destroyed$.complete();
  }

  private generateId(): string {
    return Math.random().toString(36).substr(2, 9);
  }
}
```

### Marble Diagram: Message Flow
```
Commands:    --C1----C2------C3------>
Events:      ----E1----E2--E3--E4---->
Queries:     --------Q1----Q2------Q3->

MessageBus:  --C1-E1-C2-Q1-E2-E3-C3-E4-Q2-Q3->
```

## Event Sourcing Patterns

### Event Store Implementation

```typescript
interface DomainEvent {
  aggregateId: string;
  eventType: string;
  eventData: any;
  version: number;
  timestamp: Date;
  metadata?: any;
}

class EventStore {
  private events$ = new BehaviorSubject<DomainEvent[]>([]);
  private eventStream$ = new Subject<DomainEvent>();

  // Append events to store
  appendEvents(aggregateId: string, events: DomainEvent[]): Observable<void> {
    return new Observable(observer => {
      try {
        const currentEvents = this.events$.value;
        const newEvents = events.map((event, index) => ({
          ...event,
          aggregateId,
          version: this.getNextVersion(aggregateId) + index,
          timestamp: new Date()
        }));

        // Update store
        this.events$.next([...currentEvents, ...newEvents]);
        
        // Notify subscribers
        newEvents.forEach(event => this.eventStream$.next(event));
        
        observer.next();
        observer.complete();
      } catch (error) {
        observer.error(error);
      }
    });
  }

  // Get events for aggregate
  getEvents(aggregateId: string, fromVersion = 0): Observable<DomainEvent[]> {
    return this.events$.pipe(
      map(events => events.filter(e => 
        e.aggregateId === aggregateId && e.version >= fromVersion
      ))
    );
  }

  // Get event stream
  getEventStream(eventType?: string): Observable<DomainEvent> {
    return eventType 
      ? this.eventStream$.pipe(filter(e => e.eventType === eventType))
      : this.eventStream$.asObservable();
  }

  private getNextVersion(aggregateId: string): number {
    const events = this.events$.value.filter(e => e.aggregateId === aggregateId);
    return events.length > 0 ? Math.max(...events.map(e => e.version)) + 1 : 1;
  }
}

// Aggregate Root Base Class
abstract class AggregateRoot {
  protected id: string;
  protected version = 0;
  private uncommittedEvents: DomainEvent[] = [];

  constructor(id: string) {
    this.id = id;
  }

  // Apply event to aggregate
  protected applyEvent(event: DomainEvent): void {
    this.uncommittedEvents.push(event);
    this.applyEventToState(event);
    this.version++;
  }

  // Get uncommitted events
  getUncommittedEvents(): DomainEvent[] {
    return [...this.uncommittedEvents];
  }

  // Mark events as committed
  markEventsAsCommitted(): void {
    this.uncommittedEvents = [];
  }

  // Load from history
  loadFromHistory(events: DomainEvent[]): void {
    events.forEach(event => {
      this.applyEventToState(event);
      this.version = Math.max(this.version, event.version);
    });
  }

  protected abstract applyEventToState(event: DomainEvent): void;
}
```

### Example: User Aggregate

```typescript
// User Events
interface UserCreated {
  eventType: 'UserCreated';
  eventData: {
    name: string;
    email: string;
  };
}

interface UserEmailChanged {
  eventType: 'UserEmailChanged';
  eventData: {
    newEmail: string;
    oldEmail: string;
  };
}

// User Aggregate
class User extends AggregateRoot {
  private name: string;
  private email: string;
  private isActive: boolean;

  static create(id: string, name: string, email: string): User {
    const user = new User(id);
    user.applyEvent({
      aggregateId: id,
      eventType: 'UserCreated',
      eventData: { name, email },
      version: 0,
      timestamp: new Date()
    });
    return user;
  }

  changeEmail(newEmail: string): void {
    if (this.email === newEmail) return;
    
    this.applyEvent({
      aggregateId: this.id,
      eventType: 'UserEmailChanged',
      eventData: { newEmail, oldEmail: this.email },
      version: 0,
      timestamp: new Date()
    });
  }

  protected applyEventToState(event: DomainEvent): void {
    switch (event.eventType) {
      case 'UserCreated':
        this.name = event.eventData.name;
        this.email = event.eventData.email;
        this.isActive = true;
        break;
      case 'UserEmailChanged':
        this.email = event.eventData.newEmail;
        break;
    }
  }

  // Getters
  getName(): string { return this.name; }
  getEmail(): string { return this.email; }
  getIsActive(): boolean { return this.isActive; }
}
```

## Command Query Responsibility Segregation (CQRS)

### CQRS Implementation

```typescript
// Command Side
interface Command {
  type: string;
  aggregateId: string;
  payload: any;
}

interface CommandHandler<T extends Command> {
  handle(command: T): Observable<void>;
}

class CommandBus {
  private handlers = new Map<string, CommandHandler<any>>();

  register<T extends Command>(
    commandType: string, 
    handler: CommandHandler<T>
  ): void {
    this.handlers.set(commandType, handler);
  }

  execute<T extends Command>(command: T): Observable<void> {
    const handler = this.handlers.get(command.type);
    if (!handler) {
      return throwError(new Error(`No handler for command: ${command.type}`));
    }
    return handler.handle(command);
  }
}

// Query Side
interface Query {
  type: string;
  parameters: any;
}

interface QueryHandler<T extends Query, R> {
  handle(query: T): Observable<R>;
}

class QueryBus {
  private handlers = new Map<string, QueryHandler<any, any>>();

  register<T extends Query, R>(
    queryType: string, 
    handler: QueryHandler<T, R>
  ): void {
    this.handlers.set(queryType, handler);
  }

  execute<T extends Query, R>(query: T): Observable<R> {
    const handler = this.handlers.get(query.type);
    if (!handler) {
      return throwError(new Error(`No handler for query: ${query.type}`));
    }
    return handler.handle(query);
  }
}

// Example: User Commands and Queries
interface CreateUserCommand extends Command {
  type: 'CreateUser';
  payload: {
    name: string;
    email: string;
  };
}

interface GetUserQuery extends Query {
  type: 'GetUser';
  parameters: {
    userId: string;
  };
}

class CreateUserCommandHandler implements CommandHandler<CreateUserCommand> {
  constructor(
    private eventStore: EventStore,
    private userRepository: Repository<User>
  ) {}

  handle(command: CreateUserCommand): Observable<void> {
    return new Observable(observer => {
      try {
        const user = User.create(
          command.aggregateId, 
          command.payload.name, 
          command.payload.email
        );
        
        const events = user.getUncommittedEvents();
        
        this.eventStore.appendEvents(command.aggregateId, events)
          .subscribe({
            next: () => {
              user.markEventsAsCommitted();
              observer.next();
              observer.complete();
            },
            error: err => observer.error(err)
          });
      } catch (error) {
        observer.error(error);
      }
    });
  }
}
```

### Marble Diagram: CQRS Flow
```
Commands:    --C1----C2------C3------>
             |       |       |
EventStore:  --E1----E2------E3------>
             |       |       |
ReadModel:   --R1----R2------R3------>
             |       |       |
Queries:     --------Q1------Q2------>
```

## Reactive State Management

### Advanced State Pattern

```typescript
interface AppState {
  user: UserState;
  products: ProductState;
  cart: CartState;
}

interface StateSlice<T> {
  select<K extends keyof T>(key: K): Observable<T[K]>;
  update(updater: (state: T) => T): void;
  reset(): void;
}

class ReactiveStore<T> implements StateSlice<T> {
  private state$ = new BehaviorSubject<T>(this.initialState);
  private actions$ = new Subject<Action>();

  constructor(
    private initialState: T,
    private reducer: (state: T, action: Action) => T
  ) {
    // Set up action processing
    this.actions$.pipe(
      scan((state, action) => this.reducer(state, action), this.initialState)
    ).subscribe(this.state$);
  }

  // State selection with memoization
  select<K extends keyof T>(key: K): Observable<T[K]> {
    return this.state$.pipe(
      map(state => state[key]),
      distinctUntilChanged()
    );
  }

  // Complex selectors
  selectMany<R>(selector: (state: T) => R): Observable<R> {
    return this.state$.pipe(
      map(selector),
      distinctUntilChanged()
    );
  }

  // Update state
  update(updater: (state: T) => T): void {
    const currentState = this.state$.value;
    const newState = updater(currentState);
    this.state$.next(newState);
  }

  // Dispatch actions
  dispatch(action: Action): void {
    this.actions$.next(action);
  }

  // Reset to initial state
  reset(): void {
    this.state$.next(this.initialState);
  }

  // Get current state
  getState(): T {
    return this.state$.value;
  }

  // State stream
  getState$(): Observable<T> {
    return this.state$.asObservable();
  }
}

// Advanced selectors with combining streams
class StateSelector<T> {
  constructor(private store: ReactiveStore<T>) {}

  // Combine multiple state slices
  combine<K1 extends keyof T, K2 extends keyof T, R>(
    key1: K1,
    key2: K2,
    combiner: (val1: T[K1], val2: T[K2]) => R
  ): Observable<R> {
    return combineLatest([
      this.store.select(key1),
      this.store.select(key2)
    ]).pipe(
      map(([val1, val2]) => combiner(val1, val2)),
      distinctUntilChanged()
    );
  }

  // Derived state with caching
  derived<R>(
    selector: (state: T) => R,
    cacheSize = 1
  ): Observable<R> {
    return this.store.getState$().pipe(
      map(selector),
      distinctUntilChanged(),
      shareReplay(cacheSize)
    );
  }
}
```

## Event-Driven Architecture

### Event Bus with Hierarchical Events

```typescript
interface HierarchicalEvent {
  namespace: string;
  type: string;
  payload: any;
  timestamp: number;
  source: string;
  correlationId?: string;
  causationId?: string;
}

class HierarchicalEventBus {
  private eventStream$ = new Subject<HierarchicalEvent>();
  private namespaces = new Map<string, Subject<HierarchicalEvent>>();

  // Publish event
  publish(event: Omit<HierarchicalEvent, 'timestamp'>): void {
    const fullEvent: HierarchicalEvent = {
      ...event,
      timestamp: Date.now()
    };

    this.eventStream$.next(fullEvent);
    this.getNamespaceStream(event.namespace).next(fullEvent);
  }

  // Subscribe to specific namespace
  subscribe(namespace: string): Observable<HierarchicalEvent> {
    return this.getNamespaceStream(namespace).asObservable();
  }

  // Subscribe with pattern matching
  subscribePattern(pattern: string): Observable<HierarchicalEvent> {
    const regex = new RegExp(pattern.replace('*', '.*'));
    return this.eventStream$.pipe(
      filter(event => regex.test(`${event.namespace}.${event.type}`))
    );
  }

  // Subscribe to event correlation
  subscribeCorrelation(correlationId: string): Observable<HierarchicalEvent[]> {
    return this.eventStream$.pipe(
      filter(event => event.correlationId === correlationId),
      scan((events, event) => [...events, event], [] as HierarchicalEvent[])
    );
  }

  private getNamespaceStream(namespace: string): Subject<HierarchicalEvent> {
    if (!this.namespaces.has(namespace)) {
      this.namespaces.set(namespace, new Subject<HierarchicalEvent>());
    }
    return this.namespaces.get(namespace)!;
  }
}

// Event Saga Coordinator
class SagaCoordinator {
  private sagas = new Map<string, SagaDefinition>();

  constructor(private eventBus: HierarchicalEventBus) {}

  // Register saga
  registerSaga(definition: SagaDefinition): void {
    this.sagas.set(definition.name, definition);
    
    // Start saga when trigger event occurs
    this.eventBus.subscribePattern(definition.triggerPattern)
      .subscribe(event => this.startSaga(definition, event));
  }

  private startSaga(definition: SagaDefinition, triggerEvent: HierarchicalEvent): void {
    const sagaId = this.generateSagaId();
    const correlationId = triggerEvent.correlationId || sagaId;

    // Execute saga steps
    definition.steps.reduce((stream, step) => 
      stream.pipe(
        switchMap(prevEvent => this.executeStep(step, prevEvent, correlationId))
      ),
      of(triggerEvent)
    ).subscribe({
      next: () => console.log(`Saga ${definition.name} completed`),
      error: err => this.handleSagaError(definition, err, correlationId)
    });
  }

  private executeStep(
    step: SagaStep, 
    prevEvent: HierarchicalEvent, 
    correlationId: string
  ): Observable<HierarchicalEvent> {
    return step.execute(prevEvent).pipe(
      tap(event => {
        this.eventBus.publish({
          ...event,
          correlationId,
          causationId: prevEvent.correlationId
        });
      })
    );
  }

  private handleSagaError(
    definition: SagaDefinition, 
    error: any, 
    correlationId: string
  ): void {
    if (definition.compensations) {
      this.executeCompensations(definition.compensations, correlationId);
    }
  }

  private executeCompensations(
    compensations: SagaStep[], 
    correlationId: string
  ): void {
    // Execute compensations in reverse order
    compensations.reverse().forEach((compensation, index) => {
      setTimeout(() => {
        compensation.execute({} as HierarchicalEvent).subscribe();
      }, index * 1000);
    });
  }

  private generateSagaId(): string {
    return `saga_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}

interface SagaDefinition {
  name: string;
  triggerPattern: string;
  steps: SagaStep[];
  compensations?: SagaStep[];
}

interface SagaStep {
  execute(event: HierarchicalEvent): Observable<HierarchicalEvent>;
}
```

## Actor Pattern with RxJS

### Actor Implementation

```typescript
interface ActorMessage {
  type: string;
  payload: any;
  sender?: string;
  correlationId?: string;
}

abstract class Actor {
  protected mailbox$ = new Subject<ActorMessage>();
  protected destroyed$ = new Subject<void>();
  protected children = new Map<string, Actor>();

  constructor(protected name: string, protected parent?: Actor) {
    this.setupMessageProcessing();
  }

  // Send message to actor
  send(message: ActorMessage): void {
    this.mailbox$.next(message);
  }

  // Create child actor
  protected createChild<T extends Actor>(
    name: string, 
    actorClass: new (name: string, parent: Actor) => T
  ): T {
    const child = new actorClass(name, this);
    this.children.set(name, child);
    return child;
  }

  // Send message to child
  protected sendToChild(childName: string, message: ActorMessage): void {
    const child = this.children.get(childName);
    if (child) {
      child.send(message);
    }
  }

  // Broadcast to all children
  protected broadcast(message: ActorMessage): void {
    this.children.forEach(child => child.send(message));
  }

  // Stop actor
  stop(): void {
    this.children.forEach(child => child.stop());
    this.destroyed$.next();
    this.destroyed$.complete();
  }

  private setupMessageProcessing(): void {
    this.mailbox$.pipe(
      takeUntil(this.destroyed$),
      catchError(error => {
        this.handleError(error);
        return EMPTY;
      })
    ).subscribe(message => this.receive(message));
  }

  protected abstract receive(message: ActorMessage): void;
  protected abstract handleError(error: any): void;
}

// Example: User Session Actor
class UserSessionActor extends Actor {
  private sessionData: any = {};
  private sessionTimeout?: any;

  protected receive(message: ActorMessage): void {
    switch (message.type) {
      case 'LOGIN':
        this.handleLogin(message.payload);
        break;
      case 'LOGOUT':
        this.handleLogout();
        break;
      case 'UPDATE_DATA':
        this.handleUpdateData(message.payload);
        break;
      case 'SESSION_TIMEOUT':
        this.handleSessionTimeout();
        break;
      default:
        console.warn(`Unknown message type: ${message.type}`);
    }
  }

  private handleLogin(credentials: any): void {
    this.sessionData = { ...credentials, loginTime: Date.now() };
    this.resetSessionTimeout();
    
    // Notify parent of successful login
    if (this.parent) {
      this.parent.send({
        type: 'USER_LOGGED_IN',
        payload: { userId: this.name, sessionData: this.sessionData }
      });
    }
  }

  private handleLogout(): void {
    if (this.sessionTimeout) {
      clearTimeout(this.sessionTimeout);
    }
    
    if (this.parent) {
      this.parent.send({
        type: 'USER_LOGGED_OUT',
        payload: { userId: this.name }
      });
    }
    
    this.stop();
  }

  private handleUpdateData(data: any): void {
    this.sessionData = { ...this.sessionData, ...data };
    this.resetSessionTimeout();
  }

  private handleSessionTimeout(): void {
    this.send({ type: 'LOGOUT', payload: {} });
  }

  private resetSessionTimeout(): void {
    if (this.sessionTimeout) {
      clearTimeout(this.sessionTimeout);
    }
    
    this.sessionTimeout = setTimeout(() => {
      this.send({ type: 'SESSION_TIMEOUT', payload: {} });
    }, 30 * 60 * 1000); // 30 minutes
  }

  protected handleError(error: any): void {
    console.error(`Error in UserSessionActor ${this.name}:`, error);
    
    if (this.parent) {
      this.parent.send({
        type: 'CHILD_ERROR',
        payload: { childName: this.name, error }
      });
    }
  }
}

// Actor System Supervisor
class ActorSupervisor extends Actor {
  private userSessions = new Map<string, UserSessionActor>();

  protected receive(message: ActorMessage): void {
    switch (message.type) {
      case 'CREATE_SESSION':
        this.createUserSession(message.payload.userId);
        break;
      case 'USER_LOGGED_OUT':
        this.removeUserSession(message.payload.userId);
        break;
      case 'CHILD_ERROR':
        this.handleChildError(message.payload);
        break;
      default:
        console.warn(`Unknown message type: ${message.type}`);
    }
  }

  private createUserSession(userId: string): void {
    if (!this.userSessions.has(userId)) {
      const session = this.createChild(userId, UserSessionActor);
      this.userSessions.set(userId, session);
    }
  }

  private removeUserSession(userId: string): void {
    this.userSessions.delete(userId);
  }

  private handleChildError(payload: any): void {
    console.error(`Child actor error:`, payload);
    
    // Restart child actor if needed
    if (this.userSessions.has(payload.childName)) {
      this.userSessions.get(payload.childName)?.stop();
      this.createUserSession(payload.childName);
    }
  }

  protected handleError(error: any): void {
    console.error(`Error in ActorSupervisor:`, error);
  }
}
```

## Angular Examples

### Reactive Component Architecture

```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Observable, Subject, BehaviorSubject, combineLatest } from 'rxjs';
import { map, takeUntil, startWith, distinctUntilChanged } from 'rxjs/operators';

@Component({
  selector: 'app-reactive-dashboard',
  template: `
    <div class="dashboard">
      <div class="metrics" *ngIf="viewModel$ | async as vm">
        <div class="metric-card" *ngFor="let metric of vm.metrics">
          <h3>{{ metric.title }}</h3>
          <span [class]="'value ' + metric.trend">{{ metric.value }}</span>
        </div>
      </div>
      
      <div class="filters">
        <select (change)="filterChange$.next($event.target.value)">
          <option value="all">All</option>
          <option value="active">Active</option>
          <option value="inactive">Inactive</option>
        </select>
        
        <input type="text" 
               placeholder="Search..."
               (input)="searchChange$.next($event.target.value)">
      </div>
      
      <div class="data-grid" *ngIf="filteredData$ | async as data">
        <div class="data-row" *ngFor="let item of data">
          {{ item.name }} - {{ item.status }}
        </div>
      </div>
    </div>
  `
})
export class ReactiveDashboardComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  
  // Input streams
  private filterChange$ = new Subject<string>();
  private searchChange$ = new Subject<string>();
  private refreshTrigger$ = new Subject<void>();
  
  // Data streams
  private rawData$ = new BehaviorSubject<any[]>([]);
  
  // Derived streams
  filteredData$: Observable<any[]>;
  viewModel$: Observable<DashboardViewModel>;
  
  constructor(
    private dataService: DashboardDataService,
    private metricsService: MetricsService
  ) {
    this.setupStreams();
  }
  
  ngOnInit(): void {
    // Trigger initial data load
    this.refreshTrigger$.next();
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
  
  private setupStreams(): void {
    // Data loading with refresh capability
    const dataStream$ = this.refreshTrigger$.pipe(
      switchMap(() => this.dataService.getData()),
      tap(data => this.rawData$.next(data)),
      takeUntil(this.destroy$)
    );
    
    // Filter stream with debouncing
    const filter$ = this.filterChange$.pipe(
      startWith('all'),
      distinctUntilChanged(),
      takeUntil(this.destroy$)
    );
    
    // Search stream with debouncing
    const search$ = this.searchChange$.pipe(
      startWith(''),
      debounceTime(300),
      distinctUntilChanged(),
      takeUntil(this.destroy$)
    );
    
    // Filtered data combining filter and search
    this.filteredData$ = combineLatest([
      this.rawData$,
      filter$,
      search$
    ]).pipe(
      map(([data, filter, search]) => this.filterAndSearch(data, filter, search)),
      takeUntil(this.destroy$)
    );
    
    // Metrics calculation
    const metrics$ = this.rawData$.pipe(
      map(data => this.metricsService.calculateMetrics(data)),
      takeUntil(this.destroy$)
    );
    
    // Combined view model
    this.viewModel$ = combineLatest([
      metrics$,
      this.filteredData$
    ]).pipe(
      map(([metrics, filteredData]) => ({
        metrics,
        dataCount: filteredData.length,
        lastUpdated: new Date()
      })),
      takeUntil(this.destroy$)
    );
    
    // Subscribe to data stream to start the flow
    dataStream$.subscribe();
  }
  
  private filterAndSearch(data: any[], filter: string, search: string): any[] {
    return data
      .filter(item => filter === 'all' || item.status === filter)
      .filter(item => 
        search === '' || 
        item.name.toLowerCase().includes(search.toLowerCase())
      );
  }
}

interface DashboardViewModel {
  metrics: MetricData[];
  dataCount: number;
  lastUpdated: Date;
}
```

### Reactive Service with Caching

```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, BehaviorSubject, timer, EMPTY } from 'rxjs';
import { 
  map, 
  shareReplay, 
  switchMap, 
  catchError, 
  retry, 
  tap,
  filter,
  takeUntil
} from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class ReactiveDataService {
  private cache = new Map<string, Observable<any>>();
  private refreshTriggers = new Map<string, Subject<void>>();
  private destroy$ = new Subject<void>();
  
  constructor(private http: HttpClient) {}
  
  // Get data with intelligent caching
  getData<T>(endpoint: string, cacheKey?: string): Observable<T> {
    const key = cacheKey || endpoint;
    
    if (!this.cache.has(key)) {
      this.cache.set(key, this.createDataStream<T>(endpoint, key));
    }
    
    return this.cache.get(key)!;
  }
  
  // Refresh specific cache entry
  refresh(cacheKey: string): void {
    if (this.refreshTriggers.has(cacheKey)) {
      this.refreshTriggers.get(cacheKey)!.next();
    }
  }
  
  // Clear cache
  clearCache(cacheKey?: string): void {
    if (cacheKey) {
      this.cache.delete(cacheKey);
      this.refreshTriggers.delete(cacheKey);
    } else {
      this.cache.clear();
      this.refreshTriggers.clear();
    }
  }
  
  private createDataStream<T>(endpoint: string, cacheKey: string): Observable<T> {
    // Create refresh trigger for this cache key
    const refreshTrigger$ = new Subject<void>();
    this.refreshTriggers.set(cacheKey, refreshTrigger$);
    
    // Auto-refresh every 5 minutes
    const autoRefresh$ = timer(0, 5 * 60 * 1000);
    
    // Combined refresh stream
    const refresh$ = merge(refreshTrigger$, autoRefresh$);
    
    return refresh$.pipe(
      switchMap(() => this.http.get<T>(endpoint).pipe(
        retry(3),
        catchError(error => {
          console.error(`Error fetching ${endpoint}:`, error);
          return EMPTY;
        })
      )),
      shareReplay(1),
      takeUntil(this.destroy$)
    );
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## Best Practices

### 1. Pattern Selection Guidelines

```typescript
// Use different patterns based on requirements
class PatternSelector {
  static selectPattern(requirements: Requirements): RecommendedPattern {
    if (requirements.needsEventSourcing) {
      return RecommendedPattern.EVENT_SOURCING;
    }
    
    if (requirements.needsComplexWorkflows) {
      return RecommendedPattern.SAGA;
    }
    
    if (requirements.needsDistributedProcessing) {
      return RecommendedPattern.ACTOR;
    }
    
    if (requirements.needsSeparateReadWrite) {
      return RecommendedPattern.CQRS;
    }
    
    return RecommendedPattern.SIMPLE_REACTIVE;
  }
}
```

### 2. Error Handling Strategies

```typescript
// Comprehensive error handling for reactive patterns
const errorHandlingStrategy = (error: any, context: string) => {
  // Log error with context
  console.error(`Error in ${context}:`, error);
  
  // Determine retry strategy
  if (error.status === 429) { // Rate limited
    return timer(1000).pipe(switchMap(() => throwError(error)));
  }
  
  if (error.status >= 500) { // Server error
    return timer(5000).pipe(switchMap(() => throwError(error)));
  }
  
  // Client error - don't retry
  return throwError(error);
};
```

### 3. Memory Management

```typescript
// Proper cleanup for reactive patterns
class ReactivePatternManager {
  private subscriptions = new Map<string, Subscription>();
  private subjects = new Map<string, Subject<any>>();
  
  registerSubscription(key: string, subscription: Subscription): void {
    this.subscriptions.set(key, subscription);
  }
  
  registerSubject(key: string, subject: Subject<any>): void {
    this.subjects.set(key, subject);
  }
  
  cleanup(): void {
    // Unsubscribe all
    this.subscriptions.forEach(sub => sub.unsubscribe());
    this.subscriptions.clear();
    
    // Complete all subjects
    this.subjects.forEach(subject => {
      subject.complete();
    });
    this.subjects.clear();
  }
}
```

## Testing Strategies

### Testing Reactive Patterns

```typescript
import { TestScheduler } from 'rxjs/testing';

describe('Reactive Patterns', () => {
  let testScheduler: TestScheduler;
  
  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });
  
  it('should handle event sourcing correctly', () => {
    testScheduler.run(({ cold, hot, expectObservable }) => {
      const events$ = hot('  -a-b-c-|', {
        a: { type: 'UserCreated', data: { name: 'John' } },
        b: { type: 'EmailChanged', data: { email: 'john@test.com' } },
        c: { type: 'UserDeleted', data: {} }
      });
      
      const eventStore = new EventStore();
      const result$ = events$.pipe(
        tap(event => eventStore.appendEvent(event)),
        switchMap(() => eventStore.getEvents())
      );
      
      expectObservable(result$).toBe('-a-b-c-|', {
        a: [{ type: 'UserCreated', data: { name: 'John' } }],
        b: [
          { type: 'UserCreated', data: { name: 'John' } },
          { type: 'EmailChanged', data: { email: 'john@test.com' } }
        ],
        c: [
          { type: 'UserCreated', data: { name: 'John' } },
          { type: 'EmailChanged', data: { email: 'john@test.com' } },
          { type: 'UserDeleted', data: {} }
        ]
      });
    });
  });
  
  it('should handle CQRS pattern correctly', () => {
    testScheduler.run(({ cold, hot, expectObservable }) => {
      const commands$ = hot('-a-b-|', {
        a: { type: 'CreateUser', payload: { name: 'John' } },
        b: { type: 'UpdateUser', payload: { name: 'Jane' } }
      });
      
      const commandBus = new CommandBus();
      const result$ = commands$.pipe(
        switchMap(command => commandBus.execute(command))
      );
      
      expectObservable(result$).toBe('-a-b-|');
    });
  });
});
```

## Exercises

### Exercise 1: Implement Event Sourcing
Create a complete event sourcing system for a shopping cart:

```typescript
// TODO: Implement the following:
// 1. CartAggregate with events: ItemAdded, ItemRemoved, CartCleared
// 2. EventStore with persistence
// 3. Command handlers for cart operations
// 4. Query handlers for cart projections

class ShoppingCart extends AggregateRoot {
  // Your implementation here
}

// Test your implementation
const cart = ShoppingCart.create('cart-1');
cart.addItem('product-1', 2);
cart.removeItem('product-1', 1);

// Verify events are generated correctly
console.log(cart.getUncommittedEvents());
```

### Exercise 2: Build a Saga System
Implement a distributed transaction using the Saga pattern:

```typescript
// TODO: Implement an order processing saga that:
// 1. Reserves inventory
// 2. Processes payment
// 3. Ships order
// 4. Handles compensations if any step fails

class OrderProcessingSaga {
  // Your implementation here
}
```

### Exercise 3: Create an Actor System
Build an actor-based chat system:

```typescript
// TODO: Implement:
// 1. ChatRoomActor that manages messages and users
// 2. UserActor that handles user sessions
// 3. MessageBrokerActor that routes messages
// 4. Proper error handling and actor supervision

class ChatRoomActor extends Actor {
  // Your implementation here
}
```

## Summary

In this lesson, we explored advanced reactive programming patterns that enable building sophisticated, scalable, and maintainable applications:

### Key Patterns Covered:
1. **Event Sourcing** - Storing state as a sequence of events
2. **CQRS** - Separating command and query responsibilities
3. **Saga Pattern** - Managing distributed transactions
4. **Actor Model** - Concurrent computation with isolated actors
5. **Event-Driven Architecture** - Loose coupling through events

### Best Practices:
- Choose the right pattern for your requirements
- Implement proper error handling and recovery
- Ensure memory management and cleanup
- Use marble testing for verification
- Design for scalability and maintainability

### Next Steps:
- Practice implementing these patterns in real projects
- Explore reactive microservices architecture
- Study advanced error handling and resilience patterns
- Learn about reactive system monitoring and observability

These patterns form the foundation for building reactive, event-driven applications that can handle complex business requirements while maintaining high performance and reliability.
