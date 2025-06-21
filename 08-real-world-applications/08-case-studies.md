# Real-World Case Studies & Examples

## Introduction

This lesson examines how major applications and platforms implement complex reactive patterns using RxJS and observables. We'll analyze architectural decisions, performance optimizations, and design patterns from real-world applications like Gmail, Slack, Trello, Netflix, and more.

## Learning Objectives

- Analyze real-world reactive architectures
- Understand scalable observable patterns
- Learn from production implementations
- Study performance optimization strategies
- Examine error handling in complex systems
- Understand state management at scale

## Case Study 1: Gmail - Email Client Architecture

### Overview
Gmail's web client handles millions of real-time events, complex state synchronization, and sophisticated user interactions using reactive patterns.

### Core Architecture Patterns

#### 1. Email Stream Management

```typescript
// gmail-email.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable, merge, combineLatest, timer } from 'rxjs';
import { map, switchMap, shareReplay, retry, distinctUntilChanged, throttleTime } from 'rxjs/operators';

export interface EmailThread {
  id: string;
  subject: string;
  participants: string[];
  messages: EmailMessage[];
  labels: string[];
  lastActivity: Date;
  unreadCount: number;
}

export interface EmailMessage {
  id: string;
  threadId: string;
  sender: string;
  recipients: string[];
  subject: string;
  body: string;
  timestamp: Date;
  isRead: boolean;
  attachments: Attachment[];
}

@Injectable({
  providedIn: 'root'
})
export class GmailEmailService {
  private refreshTrigger$ = new BehaviorSubject<void>(undefined);
  private searchQuery$ = new BehaviorSubject<string>('');
  private selectedLabel$ = new BehaviorSubject<string>('inbox');
  private pageSize$ = new BehaviorSubject<number>(50);
  
  // Real-time sync every 30 seconds
  private autoRefresh$ = timer(0, 30000);
  
  // Combined refresh triggers
  private refresh$ = merge(
    this.refreshTrigger$,
    this.autoRefresh$
  );
  
  // Main email threads stream
  emailThreads$: Observable<EmailThread[]> = combineLatest([
    this.refresh$,
    this.searchQuery$.pipe(distinctUntilChanged()),
    this.selectedLabel$.pipe(distinctUntilChanged()),
    this.pageSize$
  ]).pipe(
    throttleTime(500), // Prevent excessive API calls
    switchMap(([, query, label, pageSize]) => 
      this.fetchEmailThreads(query, label, pageSize)
    ),
    retry({ count: 3, delay: 1000 }),
    shareReplay(1)
  );
  
  // Unread count stream
  unreadCount$: Observable<number> = this.emailThreads$.pipe(
    map(threads => threads.reduce((count, thread) => count + thread.unreadCount, 0)),
    distinctUntilChanged(),
    shareReplay(1)
  );
  
  // Real-time updates via WebSocket
  realtimeUpdates$: Observable<any> = this.websocketService.connect().pipe(
    map(event => this.parseRealtimeEvent(event)),
    shareReplay(1)
  );
  
  constructor(
    private http: HttpClient,
    private websocketService: WebSocketService,
    private cacheService: CacheService
  ) {
    this.setupRealtimeSync();
  }
  
  private setupRealtimeSync() {
    // Apply real-time updates to local cache
    this.realtimeUpdates$.subscribe(update => {
      switch (update.type) {
        case 'NEW_EMAIL':
          this.handleNewEmail(update.data);
          break;
        case 'EMAIL_READ':
          this.handleEmailRead(update.data);
          break;
        case 'EMAIL_DELETED':
          this.handleEmailDeleted(update.data);
          break;
        case 'LABEL_CHANGED':
          this.handleLabelChange(update.data);
          break;
      }
    });
  }
  
  private fetchEmailThreads(query: string, label: string, pageSize: number): Observable<EmailThread[]> {
    const cacheKey = `emails_${query}_${label}_${pageSize}`;
    
    return this.cacheService.get(cacheKey).pipe(
      switchMap(cached => {
        if (cached && this.isCacheValid(cached.timestamp)) {
          return of(cached.data);
        }
        
        return this.http.get<EmailThread[]>('/api/emails', {
          params: { query, label, pageSize: pageSize.toString() }
        }).pipe(
          tap(data => this.cacheService.set(cacheKey, data, 300000)), // 5 min cache
          catchError(error => {
            console.error('Failed to fetch emails:', error);
            return cached ? of(cached.data) : throwError(error);
          })
        );
      })
    );
  }
  
  // Optimistic updates for better UX
  markAsRead(emailId: string): Observable<void> {
    // Immediately update local state
    this.optimisticallyUpdateEmailState(emailId, { isRead: true });
    
    return this.http.patch<void>(`/api/emails/${emailId}/read`, {}).pipe(
      catchError(error => {
        // Revert optimistic update on error
        this.optimisticallyUpdateEmailState(emailId, { isRead: false });
        return throwError(error);
      })
    );
  }
  
  // Advanced search with debouncing
  search(query: string): void {
    this.searchQuery$.next(query);
  }
  
  // Pagination handling
  loadMore(): void {
    const currentSize = this.pageSize$.value;
    this.pageSize$.next(currentSize + 50);
  }
  
  // Manual refresh trigger
  refresh(): void {
    this.refreshTrigger$.next();
  }
}
```

#### 2. Conversation Threading

```typescript
// conversation-thread.service.ts
@Injectable({
  providedIn: 'root'
})
export class ConversationThreadService {
  private expandedThreads$ = new BehaviorSubject<Set<string>>(new Set());
  
  // Smart conversation expansion
  getThreadMessages(threadId: string): Observable<EmailMessage[]> {
    return this.expandedThreads$.pipe(
      switchMap(expanded => {
        if (!expanded.has(threadId)) {
          // Load summary view (latest message only)
          return this.emailService.getLatestMessage(threadId).pipe(
            map(message => [message])
          );
        }
        
        // Load full conversation
        return this.emailService.getThreadMessages(threadId).pipe(
          map(messages => this.organizeConversation(messages))
        );
      }),
      shareReplay(1)
    );
  }
  
  expandThread(threadId: string): void {
    const current = this.expandedThreads$.value;
    const updated = new Set(current);
    updated.add(threadId);
    this.expandedThreads$.next(updated);
  }
  
  collapseThread(threadId: string): void {
    const current = this.expandedThreads$.value;
    const updated = new Set(current);
    updated.delete(threadId);
    this.expandedThreads$.next(updated);
  }
  
  private organizeConversation(messages: EmailMessage[]): EmailMessage[] {
    return messages
      .sort((a, b) => a.timestamp.getTime() - b.timestamp.getTime())
      .map((message, index) => ({
        ...message,
        isExpanded: index === messages.length - 1 || !message.isRead
      }));
  }
}
```

### Key Patterns Used

1. **Reactive State Management**: BehaviorSubjects for application state
2. **Smart Caching**: Cache with TTL and fallback strategies
3. **Optimistic Updates**: Immediate UI updates with error rollback
4. **Real-time Sync**: WebSocket integration with local state
5. **Performance Optimization**: Throttling, debouncing, and shareReplay

## Case Study 2: Slack - Real-Time Messaging Platform

### Overview
Slack handles real-time messaging, presence updates, and complex channel management using sophisticated observable patterns.

### Core Architecture Patterns

#### 1. Message Stream Architecture

```typescript
// slack-messaging.service.ts
@Injectable({
  providedIn: 'root'
})
export class SlackMessagingService {
  private currentChannel$ = new BehaviorSubject<string | null>(null);
  private connectionStatus$ = new BehaviorSubject<'connected' | 'disconnected' | 'reconnecting'>('disconnected');
  
  // Message buffer for offline scenarios
  private messageBuffer: Message[] = [];
  
  // Real-time message stream
  messages$: Observable<Message[]> = this.currentChannel$.pipe(
    filter(channelId => !!channelId),
    switchMap(channelId => this.getChannelMessages(channelId!)),
    shareReplay(1)
  );
  
  // Typing indicators
  typingIndicators$: Observable<TypingIndicator[]> = this.currentChannel$.pipe(
    filter(channelId => !!channelId),
    switchMap(channelId => this.getTypingIndicators(channelId!)),
    shareReplay(1)
  );
  
  // Presence updates
  userPresence$: Observable<Map<string, UserPresence>> = this.websocketService.connect().pipe(
    filter(event => event.type === 'presence_change'),
    scan((presence, event) => {
      const updated = new Map(presence);
      updated.set(event.userId, event.presence);
      return updated;
    }, new Map()),
    shareReplay(1)
  );
  
  constructor(
    private websocketService: SlackWebSocketService,
    private http: HttpClient,
    private offlineService: OfflineService
  ) {
    this.setupRealtimeMessaging();
    this.setupOfflineHandling();
  }
  
  private setupRealtimeMessaging() {
    // Handle incoming messages
    this.websocketService.connect().pipe(
      filter(event => event.type === 'message'),
      map(event => this.parseMessage(event.data))
    ).subscribe(message => {
      this.addMessageToStream(message);
    });
    
    // Handle message updates/edits
    this.websocketService.connect().pipe(
      filter(event => event.type === 'message_changed'),
      map(event => event.data)
    ).subscribe(update => {
      this.updateMessageInStream(update);
    });
    
    // Handle message deletions
    this.websocketService.connect().pipe(
      filter(event => event.type === 'message_deleted')
    ).subscribe(deletion => {
      this.removeMessageFromStream(deletion.messageId);
    });
  }
  
  private setupOfflineHandling() {
    // Queue messages when offline
    this.offlineService.isOnline$.pipe(
      distinctUntilChanged(),
      filter(isOnline => isOnline),
      switchMap(() => this.syncOfflineMessages())
    ).subscribe();
  }
  
  private getChannelMessages(channelId: string): Observable<Message[]> {
    // Combine historical messages with real-time updates
    return merge(
      this.loadHistoricalMessages(channelId),
      this.getRealtimeMessages(channelId)
    ).pipe(
      scan((messages, newMessage) => {
        if (Array.isArray(newMessage)) {
          return newMessage; // Historical load
        }
        return this.insertMessageInOrder(messages, newMessage);
      }, [] as Message[]),
      shareReplay(1)
    );
  }
  
  private getRealtimeMessages(channelId: string): Observable<Message> {
    return this.websocketService.connect().pipe(
      filter(event => 
        event.type === 'message' && 
        event.data.channel === channelId
      ),
      map(event => this.parseMessage(event.data))
    );
  }
  
  sendMessage(channelId: string, content: string, parentId?: string): Observable<Message> {
    const tempId = this.generateTempId();
    const optimisticMessage: Message = {
      id: tempId,
      channelId,
      content,
      sender: this.getCurrentUser(),
      timestamp: new Date(),
      parentId,
      isOptimistic: true
    };
    
    // Add optimistic message immediately
    this.addMessageToStream(optimisticMessage);
    
    return this.http.post<Message>('/api/messages', {
      channelId,
      content,
      parentId
    }).pipe(
      tap(actualMessage => {
        // Replace optimistic message with real one
        this.replaceOptimisticMessage(tempId, actualMessage);
      }),
      catchError(error => {
        // Remove failed optimistic message
        this.removeMessageFromStream(tempId);
        return throwError(error);
      })
    );
  }
  
  // Thread replies handling
  getThreadReplies(parentId: string): Observable<Message[]> {
    return this.messages$.pipe(
      map(messages => messages.filter(msg => msg.parentId === parentId)),
      distinctUntilChanged((a, b) => a.length === b.length)
    );
  }
  
  // Typing indicators
  startTyping(channelId: string): void {
    this.websocketService.send({
      type: 'typing_start',
      channel: channelId
    });
    
    // Auto-stop typing after 3 seconds
    timer(3000).subscribe(() => {
      this.stopTyping(channelId);
    });
  }
  
  stopTyping(channelId: string): void {
    this.websocketService.send({
      type: 'typing_stop',
      channel: channelId
    });
  }
}
```

#### 2. Channel Management

```typescript
// slack-channels.service.ts
@Injectable({
  providedIn: 'root'
})
export class SlackChannelsService {
  private channels$ = new BehaviorSubject<Channel[]>([]);
  private unreadCounts$ = new BehaviorSubject<Map<string, number>>(new Map());
  
  // Smart channel list with unread counts
  channelsWithUnread$: Observable<ChannelWithUnread[]> = combineLatest([
    this.channels$,
    this.unreadCounts$
  ]).pipe(
    map(([channels, unreadCounts]) => 
      channels.map(channel => ({
        ...channel,
        unreadCount: unreadCounts.get(channel.id) || 0,
        hasUnread: (unreadCounts.get(channel.id) || 0) > 0
      }))
    ),
    shareReplay(1)
  );
  
  // Prioritized channel list
  prioritizedChannels$: Observable<ChannelWithUnread[]> = this.channelsWithUnread$.pipe(
    map(channels => this.prioritizeChannels(channels)),
    shareReplay(1)
  );
  
  constructor(
    private websocketService: SlackWebSocketService,
    private messagingService: SlackMessagingService
  ) {
    this.setupUnreadTracking();
  }
  
  private setupUnreadTracking() {
    // Track unread counts from message stream
    this.messagingService.messages$.subscribe(messages => {
      this.updateUnreadCounts(messages);
    });
    
    // Handle read receipts
    this.websocketService.connect().pipe(
      filter(event => event.type === 'channel_marked')
    ).subscribe(event => {
      this.markChannelAsRead(event.data.channel, event.data.timestamp);
    });
  }
  
  private prioritizeChannels(channels: ChannelWithUnread[]): ChannelWithUnread[] {
    return channels.sort((a, b) => {
      // Prioritize by: unread, recent activity, favorites, alphabetical
      if (a.hasUnread !== b.hasUnread) {
        return a.hasUnread ? -1 : 1;
      }
      
      if (a.isFavorite !== b.isFavorite) {
        return a.isFavorite ? -1 : 1;
      }
      
      if (a.lastActivity !== b.lastActivity) {
        return b.lastActivity.getTime() - a.lastActivity.getTime();
      }
      
      return a.name.localeCompare(b.name);
    });
  }
}
```

### Key Patterns Used

1. **Event Sourcing**: All changes tracked as events
2. **Optimistic Updates**: Immediate UI feedback with server confirmation
3. **Smart Reconnection**: Automatic reconnection with exponential backoff
4. **Offline Support**: Message queuing and sync when reconnected
5. **Priority Queuing**: Smart channel ordering based on activity and unread status

## Case Study 3: Trello - Card Management System

### Overview
Trello's drag-and-drop interface with real-time collaboration demonstrates sophisticated state management and conflict resolution.

### Core Architecture Patterns

#### 1. Board State Management

```typescript
// trello-board.service.ts
@Injectable({
  providedIn: 'root'
})
export class TrelloBoardService {
  private boardState$ = new BehaviorSubject<BoardState | null>(null);
  private pendingOperations$ = new BehaviorSubject<Operation[]>([]);
  
  // Optimistic updates with operation log
  board$: Observable<BoardState | null> = combineLatest([
    this.boardState$,
    this.pendingOperations$
  ]).pipe(
    map(([baseState, operations]) => 
      operations.reduce((state, op) => this.applyOperation(state, op), baseState)
    ),
    shareReplay(1)
  );
  
  // Real-time collaboration
  collaborativeUpdates$: Observable<CollaborativeUpdate> = this.websocketService.connect().pipe(
    filter(event => event.type === 'board_update'),
    map(event => this.parseCollaborativeUpdate(event.data)),
    shareReplay(1)
  );
  
  constructor(
    private websocketService: WebSocketService,
    private http: HttpClient,
    private conflictResolver: ConflictResolverService
  ) {
    this.setupCollaborativeSync();
    this.setupOperationSync();
  }
  
  private setupCollaborativeSync() {
    // Handle remote updates
    this.collaborativeUpdates$.subscribe(update => {
      this.handleRemoteUpdate(update);
    });
  }
  
  private setupOperationSync() {
    // Sync pending operations to server
    this.pendingOperations$.pipe(
      filter(operations => operations.length > 0),
      debounceTime(500), // Batch operations
      switchMap(operations => this.syncOperations(operations))
    ).subscribe({
      next: (result) => this.handleSyncSuccess(result),
      error: (error) => this.handleSyncError(error)
    });
  }
  
  // Card movement with conflict resolution
  moveCard(cardId: string, fromListId: string, toListId: string, position: number): void {
    const operation: MoveCardOperation = {
      id: this.generateOperationId(),
      type: 'MOVE_CARD',
      cardId,
      fromListId,
      toListId,
      position,
      timestamp: new Date(),
      userId: this.getCurrentUserId()
    };
    
    this.addPendingOperation(operation);
  }
  
  // List management
  createList(title: string, position: number): Observable<List> {
    const tempId = this.generateTempId();
    const optimisticList: List = {
      id: tempId,
      title,
      position,
      cards: [],
      isOptimistic: true
    };
    
    const operation: CreateListOperation = {
      id: this.generateOperationId(),
      type: 'CREATE_LIST',
      list: optimisticList,
      timestamp: new Date(),
      userId: this.getCurrentUserId()
    };
    
    this.addPendingOperation(operation);
    
    return this.http.post<List>('/api/lists', { title, position }).pipe(
      tap(actualList => {
        this.replaceOptimisticList(tempId, actualList);
      }),
      catchError(error => {
        this.removeOptimisticList(tempId);
        return throwError(error);
      })
    );
  }
  
  private handleRemoteUpdate(update: CollaborativeUpdate): void {
    const currentState = this.boardState$.value;
    if (!currentState) return;
    
    // Check for conflicts with pending operations
    const conflicts = this.detectConflicts(update, this.pendingOperations$.value);
    
    if (conflicts.length > 0) {
      // Resolve conflicts using operational transformation
      const resolvedUpdate = this.conflictResolver.resolve(update, conflicts);
      this.applyResolvedUpdate(resolvedUpdate);
    } else {
      // Apply update directly
      this.applyRemoteUpdate(update);
    }
  }
  
  private detectConflicts(
    remoteUpdate: CollaborativeUpdate, 
    pendingOps: Operation[]
  ): Conflict[] {
    return pendingOps
      .filter(op => this.operationsConflict(remoteUpdate, op))
      .map(op => ({ remoteUpdate, localOperation: op }));
  }
  
  private applyOperation(state: BoardState | null, operation: Operation): BoardState | null {
    if (!state) return null;
    
    switch (operation.type) {
      case 'MOVE_CARD':
        return this.applyMoveCard(state, operation as MoveCardOperation);
      case 'CREATE_LIST':
        return this.applyCreateList(state, operation as CreateListOperation);
      case 'UPDATE_CARD':
        return this.applyUpdateCard(state, operation as UpdateCardOperation);
      default:
        return state;
    }
  }
  
  private applyMoveCard(state: BoardState, operation: MoveCardOperation): BoardState {
    const { cardId, fromListId, toListId, position } = operation;
    
    // Create new state with immutable updates
    const newLists = state.lists.map(list => {
      if (list.id === fromListId) {
        return {
          ...list,
          cards: list.cards.filter(card => card.id !== cardId)
        };
      }
      
      if (list.id === toListId) {
        const card = this.findCard(state, cardId);
        if (!card) return list;
        
        const newCards = [...list.cards];
        newCards.splice(position, 0, card);
        
        return {
          ...list,
          cards: newCards
        };
      }
      
      return list;
    });
    
    return {
      ...state,
      lists: newLists,
      lastModified: operation.timestamp
    };
  }
}
```

#### 2. Conflict Resolution Service

```typescript
// conflict-resolver.service.ts
@Injectable({
  providedIn: 'root'
})
export class ConflictResolverService {
  
  resolve(remoteUpdate: CollaborativeUpdate, conflicts: Conflict[]): ResolvedUpdate {
    return conflicts.reduce((update, conflict) => {
      return this.resolveConflict(update, conflict);
    }, remoteUpdate);
  }
  
  private resolveConflict(update: CollaborativeUpdate, conflict: Conflict): CollaborativeUpdate {
    const { remoteUpdate, localOperation } = conflict;
    
    // Operational Transformation based on operation types
    if (remoteUpdate.type === 'MOVE_CARD' && localOperation.type === 'MOVE_CARD') {
      return this.resolveMoveCardConflict(remoteUpdate, localOperation);
    }
    
    if (remoteUpdate.type === 'UPDATE_CARD' && localOperation.type === 'UPDATE_CARD') {
      return this.resolveUpdateCardConflict(remoteUpdate, localOperation);
    }
    
    // Default: remote wins with timestamp comparison
    return remoteUpdate.timestamp > localOperation.timestamp ? remoteUpdate : update;
  }
  
  private resolveMoveCardConflict(
    remote: MoveCardUpdate, 
    local: MoveCardOperation
  ): CollaborativeUpdate {
    // If same card moved to different positions
    if (remote.cardId === local.cardId) {
      // Use timestamp to determine winner, adjust position
      if (remote.timestamp > local.timestamp) {
        return remote;
      } else {
        // Transform remote position based on local move
        return {
          ...remote,
          position: this.transformPosition(remote.position, local)
        };
      }
    }
    
    // Different cards: transform positions
    return {
      ...remote,
      position: this.adjustPositionForLocalMove(remote, local)
    };
  }
  
  private transformPosition(remotePosition: number, localMove: MoveCardOperation): number {
    // Complex position transformation logic based on move directions
    // This is simplified - real implementation would be more sophisticated
    return remotePosition;
  }
}
```

### Key Patterns Used

1. **Operational Transformation**: Conflict resolution for concurrent edits
2. **Command Pattern**: All changes as reversible operations
3. **Optimistic UI**: Immediate visual feedback with server reconciliation
4. **Event Sourcing**: Operation log for undo/redo functionality
5. **CRDT-like Resolution**: Conflict-free replicated data types concepts

## Case Study 4: Netflix - Video Streaming Platform

### Overview
Netflix's platform manages complex video streaming, recommendation systems, and user behavior tracking using reactive patterns.

### Core Architecture Patterns

#### 1. Video Player State Management

```typescript
// netflix-player.service.ts
@Injectable({
  providedIn: 'root'
})
export class NetflixPlayerService {
  private playerState$ = new BehaviorSubject<PlayerState>({
    isPlaying: false,
    currentTime: 0,
    duration: 0,
    volume: 1,
    quality: 'auto',
    isBuffering: false,
    hasError: false
  });
  
  private playbackEvents$ = new Subject<PlaybackEvent>();
  private qualityChanges$ = new Subject<QualityChangeEvent>();
  private bufferEvents$ = new Subject<BufferEvent>();
  
  // Adaptive quality streaming
  adaptiveQuality$: Observable<VideoQuality> = merge(
    this.getNetworkConditions(),
    this.getDeviceCapabilities(),
    this.getUserPreferences()
  ).pipe(
    debounceTime(1000),
    map(([network, device, preferences]) => 
      this.calculateOptimalQuality(network, device, preferences)
    ),
    distinctUntilChanged(),
    shareReplay(1)
  );
  
  // Playback analytics
  playbackAnalytics$: Observable<PlaybackAnalytics> = this.playbackEvents$.pipe(
    buffer(interval(30000)), // Batch every 30 seconds
    filter(events => events.length > 0),
    map(events => this.aggregatePlaybackEvents(events)),
    shareReplay(1)
  );
  
  // Continuous watching experience
  nextEpisode$: Observable<VideoContent | null> = this.playerState$.pipe(
    filter(state => state.currentTime / state.duration > 0.9), // Near end
    switchMap(state => this.getNextEpisode(state.contentId)),
    shareReplay(1)
  );
  
  constructor(
    private http: HttpClient,
    private analyticsService: AnalyticsService,
    private recommendationService: RecommendationService
  ) {
    this.setupPlaybackTracking();
    this.setupAdaptiveStreaming();
    this.setupContinuousPlay();
  }
  
  private setupPlaybackTracking() {
    // Track detailed playback metrics
    this.playerState$.pipe(
      distinctUntilChanged((a, b) => Math.abs(a.currentTime - b.currentTime) < 1),
      throttleTime(5000) // Track every 5 seconds
    ).subscribe(state => {
      this.trackPlaybackProgress(state);
    });
    
    // Track quality changes
    this.qualityChanges$.subscribe(event => {
      this.analyticsService.trackQualityChange(event);
    });
    
    // Track buffering events
    this.bufferEvents$.pipe(
      debounceTime(1000) // Avoid spam during unstable connections
    ).subscribe(event => {
      this.analyticsService.trackBuffering(event);
    });
  }
  
  private setupAdaptiveStreaming() {
    // Automatically adjust quality based on conditions
    this.adaptiveQuality$.subscribe(quality => {
      this.changeQuality(quality);
    });
    
    // Monitor buffer health
    interval(1000).pipe(
      map(() => this.getBufferHealth()),
      distinctUntilChanged()
    ).subscribe(bufferHealth => {
      if (bufferHealth.isLow) {
        this.handleLowBuffer();
      }
    });
  }
  
  private setupContinuousPlay() {
    // Auto-play next episode
    this.nextEpisode$.pipe(
      filter(nextEpisode => !!nextEpisode && this.shouldAutoPlay())
    ).subscribe(nextEpisode => {
      this.preloadNextEpisode(nextEpisode!);
    });
  }
  
  // Smart buffering strategy
  private getBufferStrategy(): Observable<BufferStrategy> {
    return combineLatest([
      this.getNetworkSpeed(),
      this.getBatteryLevel(),
      this.getDeviceMemory()
    ]).pipe(
      map(([speed, battery, memory]) => ({
        bufferSize: this.calculateBufferSize(speed, memory),
        prefetchEnabled: battery > 20 && speed > 1000, // 1Mbps
        qualityLevels: this.getAvailableQualities(speed)
      }))
    );
  }
  
  // User experience optimization
  play(contentId: string): Observable<void> {
    return this.http.post<void>(`/api/play/${contentId}`, {}).pipe(
      tap(() => {
        this.playerState$.next({
          ...this.playerState$.value,
          isPlaying: true
        });
      }),
      switchMap(() => this.startPlaybackTracking(contentId)),
      catchError(error => {
        this.handlePlaybackError(error);
        return throwError(error);
      })
    );
  }
  
  private startPlaybackTracking(contentId: string): Observable<void> {
    return interval(1000).pipe(
      takeUntil(this.playerState$.pipe(filter(state => !state.isPlaying))),
      tap(tick => {
        const currentState = this.playerState$.value;
        this.playbackEvents$.next({
          type: 'progress',
          contentId,
          timestamp: new Date(),
          currentTime: currentState.currentTime,
          quality: currentState.quality
        });
      }),
      map(() => void 0)
    );
  }
}
```

#### 2. Recommendation Engine

```typescript
// netflix-recommendations.service.ts
@Injectable({
  providedIn: 'root'
})
export class NetflixRecommendationsService {
  private userBehavior$ = new BehaviorSubject<UserBehavior[]>([]);
  private watchHistory$ = new BehaviorSubject<WatchedContent[]>([]);
  private currentContext$ = new BehaviorSubject<ViewingContext>({
    timeOfDay: 'evening',
    device: 'tv',
    location: 'home'
  });
  
  // Personalized recommendations
  recommendations$: Observable<RecommendationSection[]> = combineLatest([
    this.userBehavior$,
    this.watchHistory$,
    this.currentContext$
  ]).pipe(
    debounceTime(2000), // Avoid excessive recalculation
    switchMap(([behavior, history, context]) => 
      this.generateRecommendations(behavior, history, context)
    ),
    shareReplay(1)
  );
  
  // Real-time behavior tracking
  behaviorTracking$: Observable<BehaviorEvent> = merge(
    this.trackScrolling(),
    this.trackHovering(),
    this.trackSelections(),
    this.trackWatchTime()
  ).pipe(
    shareReplay(1)
  );
  
  constructor(
    private http: HttpClient,
    private mlService: MachineLearningService,
    private abTestService: ABTestService
  ) {
    this.setupBehaviorTracking();
    this.setupRecommendationOptimization();
  }
  
  private setupBehaviorTracking() {
    // Aggregate behavior events
    this.behaviorTracking$.pipe(
      buffer(interval(60000)), // Batch every minute
      filter(events => events.length > 0),
      map(events => this.aggregateBehaviorEvents(events))
    ).subscribe(aggregatedBehavior => {
      this.updateUserBehavior(aggregatedBehavior);
    });
  }
  
  private setupRecommendationOptimization() {
    // A/B test different recommendation algorithms
    this.abTestService.getVariant('recommendation_algorithm').pipe(
      switchMap(variant => this.loadRecommendationAlgorithm(variant))
    ).subscribe(algorithm => {
      this.setRecommendationAlgorithm(algorithm);
    });
  }
  
  private trackScrolling(): Observable<BehaviorEvent> {
    return fromEvent(document, 'scroll').pipe(
      throttleTime(1000),
      map(event => ({
        type: 'scroll',
        timestamp: new Date(),
        data: { scrollPosition: window.scrollY }
      }))
    );
  }
  
  private trackHovering(): Observable<BehaviorEvent> {
    return fromEvent(document, 'mouseover').pipe(
      filter(event => (event.target as Element).closest('.content-item')),
      debounceTime(500),
      map(event => {
        const contentElement = (event.target as Element).closest('.content-item');
        return {
          type: 'hover',
          timestamp: new Date(),
          data: { 
            contentId: contentElement?.getAttribute('data-content-id'),
            duration: 500
          }
        };
      })
    );
  }
  
  private generateRecommendations(
    behavior: UserBehavior[],
    history: WatchedContent[],
    context: ViewingContext
  ): Observable<RecommendationSection[]> {
    return this.mlService.predict({
      userBehavior: behavior,
      watchHistory: history,
      context: context,
      timestamp: new Date()
    }).pipe(
      map(predictions => this.formatRecommendations(predictions)),
      catchError(error => {
        console.error('Recommendation generation failed:', error);
        return this.getFallbackRecommendations();
      })
    );
  }
  
  // Continuous learning from user interactions
  recordInteraction(interaction: UserInteraction): void {
    const behaviorEvent: BehaviorEvent = {
      type: interaction.type,
      timestamp: new Date(),
      data: interaction.data
    };
    
    this.behaviorTracking$.next(behaviorEvent);
    
    // Immediate feedback for critical interactions
    if (interaction.type === 'play' || interaction.type === 'rate') {
      this.updateRecommendationsImmediately(interaction);
    }
  }
}
```

### Key Patterns Used

1. **Adaptive Streaming**: Dynamic quality adjustment based on conditions
2. **Behavioral Analytics**: Real-time user behavior tracking and analysis
3. **Machine Learning Integration**: Reactive ML model updates
4. **A/B Testing**: Reactive experimentation framework
5. **Predictive Preloading**: Smart content prefetching

## Case Study 5: VS Code - Code Editor Architecture

### Overview
VS Code's reactive architecture handles complex text editing, extensions, and real-time collaboration.

### Core Architecture Patterns

#### 1. Document Management

```typescript
// vscode-document.service.ts
@Injectable({
  providedIn: 'root'
})
export class VSCodeDocumentService {
  private openDocuments$ = new BehaviorSubject<Map<string, Document>>(new Map());
  private activeDocument$ = new BehaviorSubject<string | null>(null);
  private documentChanges$ = new Subject<DocumentChange>();
  
  // Language server integration
  languageFeatures$: Observable<LanguageFeatures> = this.activeDocument$.pipe(
    filter(docId => !!docId),
    switchMap(docId => this.getLanguageFeatures(docId!)),
    shareReplay(1)
  );
  
  // Real-time collaboration
  collaborativeEdits$: Observable<CollaborativeEdit[]> = this.websocketService.connect().pipe(
    filter(event => event.type === 'document_edit'),
    map(event => this.parseCollaborativeEdit(event.data)),
    shareReplay(1)
  );
  
  // Undo/Redo stack
  undoStack$: Observable<UndoStack> = this.documentChanges$.pipe(
    scan((stack, change) => this.updateUndoStack(stack, change), new UndoStack()),
    shareReplay(1)
  );
  
  constructor(
    private languageServerService: LanguageServerService,
    private websocketService: WebSocketService,
    private extensionService: ExtensionService
  ) {
    this.setupCollaborativeEditing();
    this.setupLanguageServices();
  }
  
  private setupCollaborativeEditing() {
    // Apply remote edits with operational transformation
    this.collaborativeEdits$.subscribe(edits => {
      edits.forEach(edit => this.applyRemoteEdit(edit));
    });
  }
  
  private setupLanguageServices() {
    // Trigger language analysis on document changes
    this.documentChanges$.pipe(
      debounceTime(300), // Avoid excessive analysis
      switchMap(change => this.analyzeDocument(change.documentId))
    ).subscribe(analysis => {
      this.updateLanguageAnalysis(analysis);
    });
  }
  
  // Text editing with operational transformation
  editDocument(documentId: string, edit: TextEdit): void {
    const document = this.getDocument(documentId);
    if (!document) return;
    
    // Apply edit locally with operational transformation
    const transformedEdit = this.transformEdit(edit, document.pendingEdits);
    const newDocument = this.applyEdit(document, transformedEdit);
    
    this.updateDocument(documentId, newDocument);
    
    // Broadcast to collaborators
    this.broadcastEdit(documentId, transformedEdit);
    
    // Track for undo/redo
    this.documentChanges$.next({
      type: 'edit',
      documentId,
      edit: transformedEdit,
      timestamp: new Date()
    });
  }
  
  private transformEdit(edit: TextEdit, pendingEdits: TextEdit[]): TextEdit {
    return pendingEdits.reduce((transformedEdit, pendingEdit) => {
      return this.operationalTransform(transformedEdit, pendingEdit);
    }, edit);
  }
  
  private operationalTransform(edit1: TextEdit, edit2: TextEdit): TextEdit {
    // Simplified operational transformation
    if (edit1.position.line < edit2.position.line) {
      return edit1; // No transformation needed
    }
    
    if (edit1.position.line > edit2.position.line) {
      return {
        ...edit1,
        position: {
          line: edit1.position.line + this.getLineDelta(edit2),
          character: edit1.position.character
        }
      };
    }
    
    // Same line - more complex transformation
    return this.transformSameLine(edit1, edit2);
  }
}
```

### Key Patterns Used

1. **Operational Transformation**: Real-time collaborative editing
2. **Language Server Protocol**: Reactive language features
3. **Extension Architecture**: Plugin-based reactive system
4. **Command Pattern**: All actions as reversible commands
5. **Reactive Debugging**: Stream-based debugging tools

## Performance Optimization Patterns

### 1. Smart Caching Strategy

```typescript
// smart-cache.service.ts
@Injectable({
  providedIn: 'root'
})
export class SmartCacheService {
  private cache = new Map<string, CacheEntry>();
  private accessFrequency = new Map<string, number>();
  private lastAccess = new Map<string, Date>();
  
  get<T>(key: string): Observable<T | null> {
    const entry = this.cache.get(key);
    
    if (entry && this.isValid(entry)) {
      this.recordAccess(key);
      return of(entry.data);
    }
    
    return of(null);
  }
  
  set<T>(key: string, data: T, ttl: number = 300000): void {
    // Implement LRU with frequency consideration
    if (this.cache.size >= this.maxSize) {
      this.evictLeastUseful();
    }
    
    this.cache.set(key, {
      data,
      timestamp: new Date(),
      ttl,
      accessCount: 1
    });
    
    this.recordAccess(key);
  }
  
  private evictLeastUseful(): void {
    let leastUsefulKey = '';
    let leastUsefulScore = Infinity;
    
    for (const [key] of this.cache) {
      const score = this.calculateUsefulnessScore(key);
      if (score < leastUsefulScore) {
        leastUsefulScore = score;
        leastUsefulKey = key;
      }
    }
    
    if (leastUsefulKey) {
      this.cache.delete(leastUsefulKey);
      this.accessFrequency.delete(leastUsefulKey);
      this.lastAccess.delete(leastUsefulKey);
    }
  }
  
  private calculateUsefulnessScore(key: string): number {
    const frequency = this.accessFrequency.get(key) || 0;
    const lastAccessTime = this.lastAccess.get(key);
    const recency = lastAccessTime ? 
      (Date.now() - lastAccessTime.getTime()) / 1000 : Infinity;
    
    // Combine frequency and recency
    return frequency / Math.log(recency + 1);
  }
}
```

### 2. Memory-Efficient Streams

```typescript
// memory-efficient-streams.service.ts
@Injectable({
  providedIn: 'root'
})
export class MemoryEfficientStreamsService {
  
  createLargeDataStream<T>(
    dataSource: () => Observable<T[]>,
    pageSize: number = 100
  ): Observable<T[]> {
    return new Observable(subscriber => {
      let currentPage = 0;
      let allData: T[] = [];
      let isComplete = false;
      
      const loadNextPage = () => {
        if (isComplete) return;
        
        dataSource().pipe(
          map(data => data.slice(currentPage * pageSize, (currentPage + 1) * pageSize)),
          take(1)
        ).subscribe({
          next: (pageData) => {
            if (pageData.length === 0) {
              isComplete = true;
              subscriber.complete();
              return;
            }
            
            allData = [...allData, ...pageData];
            subscriber.next(allData);
            currentPage++;
            
            // Clean up old data to prevent memory leaks
            if (allData.length > pageSize * 5) {
              allData = allData.slice(-pageSize * 3);
            }
          },
          error: (error) => subscriber.error(error)
        });
      };
      
      loadNextPage();
      
      return () => {
        allData = [];
        isComplete = true;
      };
    });
  }
  
  createVirtualizedStream<T>(
    items: Observable<T[]>,
    viewportSize: number,
    itemHeight: number
  ): Observable<VirtualizedData<T>> {
    return combineLatest([
      items,
      this.scrollPosition$
    ]).pipe(
      map(([allItems, scrollTop]) => {
        const startIndex = Math.floor(scrollTop / itemHeight);
        const endIndex = Math.min(
          startIndex + Math.ceil(viewportSize / itemHeight) + 2,
          allItems.length
        );
        
        return {
          visibleItems: allItems.slice(startIndex, endIndex),
          startIndex,
          endIndex,
          totalItems: allItems.length,
          offsetY: startIndex * itemHeight
        };
      }),
      distinctUntilChanged((a, b) => 
        a.startIndex === b.startIndex && 
        a.endIndex === b.endIndex
      )
    );
  }
}
```

## Common Anti-Patterns to Avoid

### 1. **Subscription Leaks**
```typescript
// ❌ Bad: Manual subscription without cleanup
ngOnInit() {
  this.dataService.getData().subscribe(data => {
    this.data = data;
  });
}

// ✅ Good: Proper cleanup
private destroy$ = new Subject<void>();

ngOnInit() {
  this.dataService.getData().pipe(
    takeUntil(this.destroy$)
  ).subscribe(data => {
    this.data = data;
  });
}

ngOnDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

### 2. **Excessive API Calls**
```typescript
// ❌ Bad: No debouncing or caching
searchInput.valueChanges.subscribe(query => {
  this.searchService.search(query).subscribe(results => {
    this.searchResults = results;
  });
});

// ✅ Good: Debounced with caching
searchInput.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(query => this.searchService.search(query).pipe(
    startWith(this.getCachedResults(query))
  ))
).subscribe(results => {
  this.searchResults = results;
});
```

### 3. **Nested Subscriptions**
```typescript
// ❌ Bad: Nested subscriptions
this.userService.getCurrentUser().subscribe(user => {
  this.permissionService.getPermissions(user.id).subscribe(permissions => {
    this.permissions = permissions;
  });
});

// ✅ Good: Flattened with operators
this.userService.getCurrentUser().pipe(
  switchMap(user => this.permissionService.getPermissions(user.id))
).subscribe(permissions => {
  this.permissions = permissions;
});
```

## Key Takeaways

1. **Smart State Management**: Use reactive state with proper caching and synchronization
2. **Optimistic Updates**: Provide immediate feedback with server reconciliation
3. **Conflict Resolution**: Implement operational transformation for real-time collaboration
4. **Performance Optimization**: Use throttling, debouncing, and smart caching
5. **Memory Management**: Proper subscription cleanup and efficient data handling
6. **Error Handling**: Graceful degradation and recovery strategies
7. **Real-time Sync**: WebSocket integration with offline support
8. **User Experience**: Progressive loading and smooth interactions

## Exercise: Implement a Real-Time Dashboard

Create a real-time dashboard that demonstrates multiple patterns:

1. **Data Streams**: Multiple real-time data sources
2. **Smart Caching**: Efficient data caching with TTL
3. **Optimistic Updates**: Immediate UI feedback
4. **Conflict Resolution**: Handle concurrent updates
5. **Performance**: Virtual scrolling and lazy loading
6. **Offline Support**: Queue operations when offline

### Requirements:
- Handle 1000+ concurrent data points
- Sub-100ms UI response times
- Graceful offline/online transitions
- Memory usage under 50MB
- Comprehensive error handling

## Summary

In this lesson, we've analyzed:

- ✅ Real-world architectures from major applications
- ✅ Production-ready reactive patterns
- ✅ Performance optimization strategies
- ✅ Conflict resolution techniques
- ✅ Memory management best practices
- ✅ Error handling at scale
- ✅ Common anti-patterns to avoid

These case studies provide proven patterns and architectural decisions for building scalable, reactive applications.

## Next Steps

In the next lesson, we'll explore **RxJS in Microservices Architecture**, covering how reactive patterns scale across distributed systems and service boundaries.
