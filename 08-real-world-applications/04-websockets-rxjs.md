# WebSockets with RxJS

## Introduction

WebSockets provide full-duplex communication between client and server, enabling real-time applications. This lesson explores how to integrate WebSockets with RxJS to create robust, reactive real-time communication systems with proper connection management, error handling, and reconnection strategies.

## Learning Objectives

- Master WebSocket integration with RxJS Observables
- Implement connection management and lifecycle handling
- Create robust reconnection and error recovery strategies
- Build real-time messaging and notification systems
- Handle WebSocket events reactively with RxJS operators
- Implement connection pooling and resource management
- Test WebSocket-based reactive applications

## WebSocket Fundamentals with RxJS

### 1. Basic WebSocket Observable

```typescript
// ✅ Creating a basic WebSocket Observable
import { Observable, Subject, BehaviorSubject } from 'rxjs';
import { filter, map, takeUntil, retry, catchError } from 'rxjs/operators';

export class WebSocketService {
  private socket$: Observable<MessageEvent>;
  private messagesSubject$ = new Subject<any>();
  private connectionStatus$ = new BehaviorSubject<boolean>(false);

  constructor(private url: string) {
    this.socket$ = this.createWebSocketObservable();
  }

  private createWebSocketObservable(): Observable<MessageEvent> {
    return new Observable<MessageEvent>(observer => {
      const websocket = new WebSocket(this.url);

      websocket.onopen = () => {
        console.log('WebSocket connected');
        this.connectionStatus$.next(true);
      };

      websocket.onmessage = (event) => {
        observer.next(event);
      };

      websocket.onerror = (error) => {
        console.error('WebSocket error:', error);
        observer.error(error);
      };

      websocket.onclose = (event) => {
        console.log('WebSocket closed:', event);
        this.connectionStatus$.next(false);
        
        if (event.wasClean) {
          observer.complete();
        } else {
          observer.error(new Error('WebSocket connection lost'));
        }
      };

      // Cleanup function
      return () => {
        if (websocket.readyState === WebSocket.OPEN) {
          websocket.close();
        }
      };
    });
  }

  // Send messages through WebSocket
  send(message: any): void {
    if (this.connectionStatus$.value) {
      // Implementation depends on how you maintain the WebSocket reference
      this.sendMessage(message);
    } else {
      console.warn('WebSocket not connected. Message not sent:', message);
    }
  }

  // Get connection status
  get isConnected$(): Observable<boolean> {
    return this.connectionStatus$.asObservable();
  }

  // Get messages stream
  get messages$(): Observable<any> {
    return this.socket$.pipe(
      map(event => JSON.parse(event.data)),
      catchError(error => {
        console.error('Error parsing message:', error);
        return [];
      })
    );
  }
}
```

### 2. Enhanced WebSocket Service with Reconnection

```typescript
// ✅ Production-ready WebSocket service with reconnection
import { webSocket, WebSocketSubject } from 'rxjs/webSocket';
import { timer, EMPTY, Subject, BehaviorSubject } from 'rxjs';
import { retryWhen, tap, delayWhen, switchMap, catchError } from 'rxjs/operators';

export interface WebSocketConfig {
  url: string;
  reconnectInterval?: number;
  maxReconnectAttempts?: number;
  reconnectBackoff?: boolean;
  protocols?: string[];
}

export interface WebSocketMessage {
  type: string;
  payload: any;
  timestamp?: Date;
  id?: string;
}

@Injectable({ providedIn: 'root' })
export class RxWebSocketService {
  private config: Required<WebSocketConfig>;
  private socket$: WebSocketSubject<any>;
  private messagesSubject$ = new Subject<WebSocketMessage>();
  private connectionStatus$ = new BehaviorSubject<ConnectionStatus>('DISCONNECTED');
  private reconnectAttempts = 0;

  constructor(config: WebSocketConfig) {
    this.config = {
      reconnectInterval: 5000,
      maxReconnectAttempts: 10,
      reconnectBackoff: true,
      protocols: [],
      ...config
    };
  }

  // Connect to WebSocket
  connect(): void {
    if (this.socket$) {
      return;
    }

    this.socket$ = webSocket({
      url: this.config.url,
      protocols: this.config.protocols,
      openObserver: {
        next: () => {
          console.log('WebSocket connected successfully');
          this.connectionStatus$.next('CONNECTED');
          this.reconnectAttempts = 0;
        }
      },
      closeObserver: {
        next: () => {
          console.log('WebSocket connection closed');
          this.connectionStatus$.next('DISCONNECTED');
          this.socket$ = null;
          this.attemptReconnect();
        }
      }
    });

    // Subscribe to messages with error handling and reconnection
    this.socket$.pipe(
      retryWhen(errors => this.handleReconnection(errors)),
      catchError(error => {
        console.error('WebSocket error:', error);
        this.connectionStatus$.next('ERROR');
        return EMPTY;
      })
    ).subscribe(
      message => this.messagesSubject$.next(message),
      error => console.error('WebSocket subscription error:', error)
    );
  }

  // Disconnect from WebSocket
  disconnect(): void {
    if (this.socket$) {
      this.socket$.complete();
      this.socket$ = null;
      this.connectionStatus$.next('DISCONNECTED');
    }
  }

  // Send message through WebSocket
  sendMessage(message: WebSocketMessage): void {
    if (this.socket$ && this.connectionStatus$.value === 'CONNECTED') {
      const messageWithMetadata = {
        ...message,
        timestamp: new Date(),
        id: this.generateMessageId()
      };
      
      this.socket$.next(messageWithMetadata);
    } else {
      console.warn('Cannot send message: WebSocket not connected');
      // Optionally queue messages for later sending
      this.queueMessage(message);
    }
  }

  // Get messages observable
  get messages$(): Observable<WebSocketMessage> {
    return this.messagesSubject$.asObservable();
  }

  // Get connection status
  get connectionStatus(): Observable<ConnectionStatus> {
    return this.connectionStatus$.asObservable();
  }

  // Filter messages by type
  getMessagesByType(type: string): Observable<WebSocketMessage> {
    return this.messages$.pipe(
      filter(message => message.type === type)
    );
  }

  // Private methods
  private handleReconnection(errors: Observable<any>): Observable<any> {
    return errors.pipe(
      tap(() => {
        this.reconnectAttempts++;
        this.connectionStatus$.next('RECONNECTING');
      }),
      delayWhen(() => {
        if (this.reconnectAttempts >= this.config.maxReconnectAttempts) {
          throw new Error('Max reconnection attempts reached');
        }

        const delay = this.config.reconnectBackoff
          ? this.config.reconnectInterval * Math.pow(2, this.reconnectAttempts - 1)
          : this.config.reconnectInterval;

        console.log(`Reconnecting in ${delay}ms (attempt ${this.reconnectAttempts})`);
        return timer(delay);
      })
    );
  }

  private attemptReconnect(): void {
    if (this.reconnectAttempts < this.config.maxReconnectAttempts) {
      setTimeout(() => this.connect(), this.config.reconnectInterval);
    }
  }

  private generateMessageId(): string {
    return `msg_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }

  private queueMessage(message: WebSocketMessage): void {
    // Implementation for message queuing
    // Store messages in a queue and send when reconnected
  }
}

type ConnectionStatus = 'CONNECTING' | 'CONNECTED' | 'DISCONNECTING' | 'DISCONNECTED' | 'RECONNECTING' | 'ERROR';
```

## Real-Time Messaging Patterns

### 1. Chat Application Pattern

```typescript
// ✅ Real-time chat application with RxJS
export interface ChatMessage {
  id: string;
  userId: string;
  username: string;
  message: string;
  timestamp: Date;
  roomId: string;
}

export interface ChatRoom {
  id: string;
  name: string;
  participants: string[];
  lastMessage?: ChatMessage;
}

@Injectable({ providedIn: 'root' })
export class ChatService {
  private wsService: RxWebSocketService;
  private messagesSubject$ = new Subject<ChatMessage>();
  private roomsSubject$ = new BehaviorSubject<ChatRoom[]>([]);
  private currentUser$ = new BehaviorSubject<User | null>(null);

  constructor(private authService: AuthService) {
    this.wsService = new RxWebSocketService({
      url: 'wss://api.example.com/chat'
    });

    this.setupMessageHandling();
  }

  // Initialize chat service
  initializeChat(user: User): void {
    this.currentUser$.next(user);
    this.wsService.connect();
    
    // Join user's rooms
    this.joinUserRooms(user.id);
  }

  // Send chat message
  sendMessage(roomId: string, message: string): void {
    const chatMessage: ChatMessage = {
      id: this.generateId(),
      userId: this.currentUser$.value?.id!,
      username: this.currentUser$.value?.username!,
      message,
      timestamp: new Date(),
      roomId
    };

    this.wsService.sendMessage({
      type: 'CHAT_MESSAGE',
      payload: chatMessage
    });
  }

  // Join chat room
  joinRoom(roomId: string): void {
    this.wsService.sendMessage({
      type: 'JOIN_ROOM',
      payload: { roomId, userId: this.currentUser$.value?.id }
    });
  }

  // Leave chat room
  leaveRoom(roomId: string): void {
    this.wsService.sendMessage({
      type: 'LEAVE_ROOM',
      payload: { roomId, userId: this.currentUser$.value?.id }
    });
  }

  // Get messages for specific room
  getRoomMessages(roomId: string): Observable<ChatMessage[]> {
    return this.messagesSubject$.pipe(
      filter(message => message.roomId === roomId),
      scan((messages, newMessage) => [...messages, newMessage], [] as ChatMessage[]),
      map(messages => messages.slice(-50)) // Keep last 50 messages
    );
  }

  // Get all rooms
  get rooms$(): Observable<ChatRoom[]> {
    return this.roomsSubject$.asObservable();
  }

  // Get connection status
  get connectionStatus$(): Observable<ConnectionStatus> {
    return this.wsService.connectionStatus;
  }

  // Get typing indicators
  getTypingIndicators(roomId: string): Observable<string[]> {
    return this.wsService.getMessagesByType('TYPING_INDICATOR').pipe(
      filter(msg => msg.payload.roomId === roomId),
      map(msg => msg.payload.typingUsers as string[])
    );
  }

  // Send typing indicator
  sendTypingIndicator(roomId: string, isTyping: boolean): void {
    this.wsService.sendMessage({
      type: 'TYPING_INDICATOR',
      payload: {
        roomId,
        userId: this.currentUser$.value?.id,
        isTyping
      }
    });
  }

  // Private methods
  private setupMessageHandling(): void {
    // Handle incoming chat messages
    this.wsService.getMessagesByType('CHAT_MESSAGE').subscribe(
      message => this.messagesSubject$.next(message.payload)
    );

    // Handle room updates
    this.wsService.getMessagesByType('ROOM_UPDATE').subscribe(
      message => this.handleRoomUpdate(message.payload)
    );

    // Handle user joined/left events
    this.wsService.getMessagesByType('USER_JOINED').subscribe(
      message => this.handleUserJoined(message.payload)
    );

    this.wsService.getMessagesByType('USER_LEFT').subscribe(
      message => this.handleUserLeft(message.payload)
    );
  }

  private joinUserRooms(userId: string): void {
    this.wsService.sendMessage({
      type: 'GET_USER_ROOMS',
      payload: { userId }
    });
  }

  private handleRoomUpdate(payload: any): void {
    const currentRooms = this.roomsSubject$.value;
    const updatedRooms = currentRooms.map(room => 
      room.id === payload.roomId ? { ...room, ...payload.updates } : room
    );
    this.roomsSubject$.next(updatedRooms);
  }

  private handleUserJoined(payload: any): void {
    console.log(`User ${payload.username} joined room ${payload.roomId}`);
  }

  private handleUserLeft(payload: any): void {
    console.log(`User ${payload.username} left room ${payload.roomId}`);
  }

  private generateId(): string {
    return `chat_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}
```

### 2. Real-Time Notifications System

```typescript
// ✅ Real-time notifications with RxJS
export interface Notification {
  id: string;
  type: NotificationType;
  title: string;
  message: string;
  timestamp: Date;
  read: boolean;
  data?: any;
  priority: 'low' | 'medium' | 'high' | 'critical';
}

export type NotificationType = 'info' | 'success' | 'warning' | 'error' | 'system';

@Injectable({ providedIn: 'root' })
export class NotificationService {
  private wsService: RxWebSocketService;
  private notifications$ = new BehaviorSubject<Notification[]>([]);
  private unreadCount$ = new BehaviorSubject<number>(0);
  private soundEnabled$ = new BehaviorSubject<boolean>(true);

  constructor() {
    this.wsService = new RxWebSocketService({
      url: 'wss://api.example.com/notifications'
    });

    this.setupNotificationHandling();
  }

  // Initialize notifications
  initialize(userId: string): void {
    this.wsService.connect();
    
    // Subscribe to user notifications
    this.wsService.sendMessage({
      type: 'SUBSCRIBE_NOTIFICATIONS',
      payload: { userId }
    });

    // Request existing notifications
    this.loadExistingNotifications(userId);
  }

  // Get all notifications
  get allNotifications$(): Observable<Notification[]> {
    return this.notifications$.asObservable();
  }

  // Get unread notifications
  get unreadNotifications$(): Observable<Notification[]> {
    return this.notifications$.pipe(
      map(notifications => notifications.filter(n => !n.read))
    );
  }

  // Get unread count
  get unreadCount(): Observable<number> {
    return this.unreadCount$.asObservable();
  }

  // Get notifications by type
  getNotificationsByType(type: NotificationType): Observable<Notification[]> {
    return this.notifications$.pipe(
      map(notifications => notifications.filter(n => n.type === type))
    );
  }

  // Get high priority notifications
  get priorityNotifications$(): Observable<Notification[]> {
    return this.notifications$.pipe(
      map(notifications => 
        notifications.filter(n => n.priority === 'high' || n.priority === 'critical')
      )
    );
  }

  // Mark notification as read
  markAsRead(notificationId: string): void {
    const notifications = this.notifications$.value;
    const updated = notifications.map(n => 
      n.id === notificationId ? { ...n, read: true } : n
    );
    
    this.notifications$.next(updated);
    this.updateUnreadCount();

    // Notify server
    this.wsService.sendMessage({
      type: 'MARK_READ',
      payload: { notificationId }
    });
  }

  // Mark all as read
  markAllAsRead(): void {
    const notifications = this.notifications$.value;
    const updated = notifications.map(n => ({ ...n, read: true }));
    
    this.notifications$.next(updated);
    this.updateUnreadCount();

    this.wsService.sendMessage({
      type: 'MARK_ALL_READ',
      payload: {}
    });
  }

  // Remove notification
  removeNotification(notificationId: string): void {
    const notifications = this.notifications$.value;
    const filtered = notifications.filter(n => n.id !== notificationId);
    
    this.notifications$.next(filtered);
    this.updateUnreadCount();

    this.wsService.sendMessage({
      type: 'REMOVE_NOTIFICATION',
      payload: { notificationId }
    });
  }

  // Clear all notifications
  clearAll(): void {
    this.notifications$.next([]);
    this.unreadCount$.next(0);

    this.wsService.sendMessage({
      type: 'CLEAR_ALL',
      payload: {}
    });
  }

  // Toggle sound notifications
  toggleSound(): void {
    const enabled = !this.soundEnabled$.value;
    this.soundEnabled$.next(enabled);
    localStorage.setItem('notifications-sound', enabled.toString());
  }

  // Get sound setting
  get soundEnabled(): Observable<boolean> {
    return this.soundEnabled$.asObservable();
  }

  // Private methods
  private setupNotificationHandling(): void {
    // Handle incoming notifications
    this.wsService.getMessagesByType('NEW_NOTIFICATION').subscribe(
      message => this.handleNewNotification(message.payload)
    );

    // Handle notification updates
    this.wsService.getMessagesByType('NOTIFICATION_UPDATE').subscribe(
      message => this.handleNotificationUpdate(message.payload)
    );

    // Handle bulk notifications
    this.wsService.getMessagesByType('BULK_NOTIFICATIONS').subscribe(
      message => this.handleBulkNotifications(message.payload)
    );

    // Load sound setting
    const soundSetting = localStorage.getItem('notifications-sound');
    if (soundSetting !== null) {
      this.soundEnabled$.next(soundSetting === 'true');
    }
  }

  private handleNewNotification(notification: Notification): void {
    const currentNotifications = this.notifications$.value;
    const updated = [notification, ...currentNotifications];
    
    this.notifications$.next(updated);
    this.updateUnreadCount();

    // Play notification sound
    if (this.soundEnabled$.value && notification.priority !== 'low') {
      this.playNotificationSound(notification.priority);
    }

    // Show browser notification for high priority
    if (notification.priority === 'high' || notification.priority === 'critical') {
      this.showBrowserNotification(notification);
    }
  }

  private handleNotificationUpdate(update: any): void {
    const notifications = this.notifications$.value;
    const updated = notifications.map(n => 
      n.id === update.id ? { ...n, ...update } : n
    );
    
    this.notifications$.next(updated);
    this.updateUnreadCount();
  }

  private handleBulkNotifications(notifications: Notification[]): void {
    this.notifications$.next(notifications);
    this.updateUnreadCount();
  }

  private updateUnreadCount(): void {
    const unreadCount = this.notifications$.value.filter(n => !n.read).length;
    this.unreadCount$.next(unreadCount);
  }

  private loadExistingNotifications(userId: string): void {
    this.wsService.sendMessage({
      type: 'GET_NOTIFICATIONS',
      payload: { userId, limit: 100 }
    });
  }

  private playNotificationSound(priority: string): void {
    const audio = new Audio();
    audio.src = `/assets/sounds/notification-${priority}.mp3`;
    audio.play().catch(error => {
      console.warn('Could not play notification sound:', error);
    });
  }

  private showBrowserNotification(notification: Notification): void {
    if ('Notification' in window && Notification.permission === 'granted') {
      new Notification(notification.title, {
        body: notification.message,
        icon: '/assets/icons/notification-icon.png',
        tag: notification.id
      });
    }
  }
}
```

## Connection Management and Error Handling

### 1. Connection Pool and Resource Management

```typescript
// ✅ WebSocket connection pool for multiple endpoints
@Injectable({ providedIn: 'root' })
export class WebSocketPool {
  private connections = new Map<string, RxWebSocketService>();
  private connectionLimits = new Map<string, number>();
  private defaultLimit = 5;

  constructor() {}

  // Get or create connection
  getConnection(url: string, config?: Partial<WebSocketConfig>): RxWebSocketService {
    const existing = this.connections.get(url);
    
    if (existing) {
      return existing;
    }

    // Check connection limits
    const domain = this.extractDomain(url);
    const currentConnections = this.getConnectionsForDomain(domain);
    const limit = this.connectionLimits.get(domain) || this.defaultLimit;

    if (currentConnections >= limit) {
      throw new Error(`Connection limit reached for domain: ${domain}`);
    }

    // Create new connection
    const connection = new RxWebSocketService({
      url,
      ...config
    });

    this.connections.set(url, connection);
    
    // Auto-cleanup on disconnect
    connection.connectionStatus.pipe(
      filter(status => status === 'DISCONNECTED'),
      take(1)
    ).subscribe(() => {
      this.removeConnection(url);
    });

    return connection;
  }

  // Remove connection
  removeConnection(url: string): void {
    const connection = this.connections.get(url);
    
    if (connection) {
      connection.disconnect();
      this.connections.delete(url);
    }
  }

  // Set connection limit for domain
  setConnectionLimit(domain: string, limit: number): void {
    this.connectionLimits.set(domain, limit);
  }

  // Get all active connections
  getActiveConnections(): RxWebSocketService[] {
    return Array.from(this.connections.values());
  }

  // Cleanup all connections
  cleanup(): void {
    this.connections.forEach(connection => connection.disconnect());
    this.connections.clear();
  }

  // Private methods
  private extractDomain(url: string): string {
    try {
      return new URL(url).hostname;
    } catch {
      return 'localhost';
    }
  }

  private getConnectionsForDomain(domain: string): number {
    return Array.from(this.connections.keys())
      .filter(url => this.extractDomain(url) === domain)
      .length;
  }
}
```

### 2. Advanced Error Handling and Recovery

```typescript
// ✅ Advanced error handling for WebSocket connections
export class WebSocketErrorHandler {
  private errorHistory: WebSocketError[] = [];
  private maxErrorHistory = 100;

  handleError(error: any, context: string): Observable<never> {
    const wsError: WebSocketError = {
      error,
      context,
      timestamp: new Date(),
      code: this.extractErrorCode(error),
      recoverable: this.isRecoverable(error)
    };

    this.logError(wsError);
    this.addToHistory(wsError);

    // Determine recovery strategy
    if (wsError.recoverable) {
      return this.handleRecoverableError(wsError);
    } else {
      return this.handleNonRecoverableError(wsError);
    }
  }

  private handleRecoverableError(error: WebSocketError): Observable<never> {
    // Implement backoff strategy
    const retryDelay = this.calculateRetryDelay(error);
    
    console.warn(`Recoverable WebSocket error, retrying in ${retryDelay}ms`, error);
    
    return timer(retryDelay).pipe(
      switchMap(() => throwError(() => error.error))
    );
  }

  private handleNonRecoverableError(error: WebSocketError): Observable<never> {
    console.error('Non-recoverable WebSocket error:', error);
    
    // Notify user or trigger alternative flow
    this.notifyError(error);
    
    return throwError(() => error.error);
  }

  private isRecoverable(error: any): boolean {
    if (error.code) {
      // Network errors are usually recoverable
      return [1006, 1011, 1012, 1013, 1014].includes(error.code);
    }

    // Check error message for recoverable patterns
    const message = error.message?.toLowerCase() || '';
    return message.includes('network') || 
           message.includes('connection') ||
           message.includes('timeout');
  }

  private calculateRetryDelay(error: WebSocketError): number {
    const baseDelay = 1000;
    const maxDelay = 30000;
    const backoffFactor = 2;

    // Count recent errors for exponential backoff
    const recentErrors = this.getRecentErrors(60000); // Last minute
    const delay = Math.min(
      baseDelay * Math.pow(backoffFactor, recentErrors.length),
      maxDelay
    );

    return delay;
  }

  private extractErrorCode(error: any): number | null {
    return error.code || error.status || null;
  }

  private logError(error: WebSocketError): void {
    const logLevel = error.recoverable ? 'warn' : 'error';
    console[logLevel]('WebSocket error:', {
      context: error.context,
      code: error.code,
      message: error.error.message,
      recoverable: error.recoverable,
      timestamp: error.timestamp
    });
  }

  private addToHistory(error: WebSocketError): void {
    this.errorHistory.push(error);
    
    if (this.errorHistory.length > this.maxErrorHistory) {
      this.errorHistory.shift();
    }
  }

  private getRecentErrors(timeWindow: number): WebSocketError[] {
    const cutoff = new Date(Date.now() - timeWindow);
    return this.errorHistory.filter(error => error.timestamp > cutoff);
  }

  private notifyError(error: WebSocketError): void {
    // Implement user notification system
    // Could integrate with your app's notification service
  }
}

interface WebSocketError {
  error: any;
  context: string;
  timestamp: Date;
  code: number | null;
  recoverable: boolean;
}
```

## Testing WebSocket Applications

### 1. WebSocket Testing with RxJS

```typescript
// ✅ Testing WebSocket services with marble diagrams
describe('RxWebSocketService', () => {
  let service: RxWebSocketService;
  let mockWebSocket: jasmine.SpyObj<WebSocket>;
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });

    // Mock WebSocket
    mockWebSocket = jasmine.createSpyObj('WebSocket', ['send', 'close']);
    spyOn(window, 'WebSocket').and.returnValue(mockWebSocket);

    service = new RxWebSocketService({
      url: 'ws://test.example.com'
    });
  });

  it('should handle connection lifecycle', () => {
    testScheduler.run(({ cold, hot, expectObservable }) => {
      // Simulate connection events
      const connectionEvents$ = hot('a-b-c', {
        a: { type: 'open' },
        b: { type: 'message', data: '{"type":"test","payload":"data"}' },
        c: { type: 'close' }
      });

      // Test connection status changes
      const expectedStatus$ = cold('a-b-c', {
        a: 'CONNECTED',
        b: 'CONNECTED',
        c: 'DISCONNECTED'
      });

      expectObservable(service.connectionStatus).toBe(expectedStatus$);
    });
  });

  it('should handle message sending and receiving', () => {
    testScheduler.run(({ cold, expectObservable }) => {
      const testMessage = { type: 'test', payload: 'data' };
      
      // Simulate sending message
      service.sendMessage(testMessage);
      
      // Verify WebSocket.send was called
      expect(mockWebSocket.send).toHaveBeenCalledWith(
        JSON.stringify(jasmine.objectContaining(testMessage))
      );
    });
  });

  it('should retry on connection failures', () => {
    testScheduler.run(({ cold, hot, expectObservable }) => {
      // Simulate connection failures and recovery
      const errors$ = hot('a--b--c', {
        a: new Error('Connection failed'),
        b: new Error('Network error'),
        c: null // Success
      });

      const retryAttempts$ = cold('---a--b--c', {
        a: 1,
        b: 2,
        c: 0 // Reset after success
      });

      // Test retry behavior
      expectObservable(service.connectionStatus).toBe(retryAttempts$);
    });
  });
});

// Mock WebSocket for testing
class MockWebSocket {
  static CONNECTING = 0;
  static OPEN = 1;
  static CLOSING = 2;
  static CLOSED = 3;

  readyState = MockWebSocket.CONNECTING;
  onopen: ((event: Event) => void) | null = null;
  onclose: ((event: CloseEvent) => void) | null = null;
  onmessage: ((event: MessageEvent) => void) | null = null;
  onerror: ((event: Event) => void) | null = null;

  constructor(public url: string) {
    // Simulate async connection
    setTimeout(() => {
      this.readyState = MockWebSocket.OPEN;
      this.onopen?.(new Event('open'));
    }, 0);
  }

  send(data: string): void {
    if (this.readyState !== MockWebSocket.OPEN) {
      throw new Error('WebSocket is not open');
    }
    // Message sending logic
  }

  close(code?: number, reason?: string): void {
    this.readyState = MockWebSocket.CLOSED;
    this.onclose?.(new CloseEvent('close', { code, reason }));
  }

  // Helper methods for testing
  simulateMessage(data: any): void {
    const event = new MessageEvent('message', {
      data: JSON.stringify(data)
    });
    this.onmessage?.(event);
  }

  simulateError(error: any): void {
    this.onerror?.(new Event('error'));
  }

  simulateClose(code = 1000, reason = 'Normal closure'): void {
    this.readyState = MockWebSocket.CLOSED;
    this.onclose?.(new CloseEvent('close', { code, reason }));
  }
}
```

### 2. Integration Testing

```typescript
// ✅ Integration testing for WebSocket applications
@Component({
  template: `
    <div class="chat-container">
      <div class="messages" *ngFor="let message of messages$ | async">
        {{ message.username }}: {{ message.message }}
      </div>
      <input 
        #messageInput 
        (keyup.enter)="sendMessage(messageInput.value); messageInput.value=''"
        placeholder="Type a message..."
      >
      <div class="connection-status" [class]="connectionStatus$ | async">
        {{ connectionStatus$ | async }}
      </div>
    </div>
  `
})
class ChatTestComponent {
  messages$ = this.chatService.getRoomMessages('test-room');
  connectionStatus$ = this.chatService.connectionStatus$;

  constructor(private chatService: ChatService) {}

  sendMessage(message: string): void {
    if (message.trim()) {
      this.chatService.sendMessage('test-room', message);
    }
  }
}

describe('Chat Integration Test', () => {
  let component: ChatTestComponent;
  let fixture: ComponentFixture<ChatTestComponent>;
  let chatService: jasmine.SpyObj<ChatService>;

  beforeEach(async () => {
    const chatServiceSpy = jasmine.createSpyObj('ChatService', [
      'sendMessage',
      'getRoomMessages',
      'connectionStatus$'
    ]);

    await TestBed.configureTestingModule({
      declarations: [ChatTestComponent],
      providers: [
        { provide: ChatService, useValue: chatServiceSpy }
      ]
    }).compileComponents();

    fixture = TestBed.createComponent(ChatTestComponent);
    component = fixture.componentInstance;
    chatService = TestBed.inject(ChatService) as jasmine.SpyObj<ChatService>;

    // Setup service returns
    chatService.getRoomMessages.and.returnValue(of([]));
    chatService.connectionStatus$ = of('CONNECTED');
  });

  it('should send message when enter is pressed', () => {
    const input = fixture.debugElement.query(By.css('input'));
    const testMessage = 'Hello WebSocket!';

    // Set input value and trigger enter
    input.nativeElement.value = testMessage;
    input.nativeElement.dispatchEvent(new KeyboardEvent('keyup', { key: 'Enter' }));

    expect(chatService.sendMessage).toHaveBeenCalledWith('test-room', testMessage);
  });

  it('should display connection status', () => {
    fixture.detectChanges();
    
    const statusElement = fixture.debugElement.query(By.css('.connection-status'));
    expect(statusElement.nativeElement.textContent.trim()).toBe('CONNECTED');
  });

  it('should display messages from service', fakeAsync(() => {
    const testMessages = [
      { id: '1', username: 'User1', message: 'Hello', timestamp: new Date(), roomId: 'test-room' }
    ];

    chatService.getRoomMessages.and.returnValue(of(testMessages));
    component.messages$ = chatService.getRoomMessages('test-room');
    
    fixture.detectChanges();
    tick();

    const messageElements = fixture.debugElement.queryAll(By.css('.messages'));
    expect(messageElements.length).toBe(1);
    expect(messageElements[0].nativeElement.textContent).toContain('User1: Hello');
  }));
});
```

## Performance Optimization

### 1. Message Throttling and Buffering

```typescript
// ✅ Optimized message handling for high-frequency updates
@Injectable({ providedIn: 'root' })
export class OptimizedWebSocketService extends RxWebSocketService {
  private messageBuffer$ = new Subject<WebSocketMessage>();
  private batchSize = 50;
  private batchInterval = 100; // ms

  constructor(config: WebSocketConfig) {
    super(config);
    this.setupOptimizedMessageHandling();
  }

  // Batched message processing
  get batchedMessages$(): Observable<WebSocketMessage[]> {
    return this.messageBuffer$.pipe(
      bufferTime(this.batchInterval, null, this.batchSize),
      filter(batch => batch.length > 0),
      shareReplay(1)
    );
  }

  // Throttled specific message types
  getThrottledMessages(type: string, throttleMs = 1000): Observable<WebSocketMessage> {
    return this.getMessagesByType(type).pipe(
      throttleTime(throttleMs, undefined, { leading: true, trailing: true })
    );
  }

  // Debounced user input messages
  getDebouncedMessages(type: string, debounceMs = 300): Observable<WebSocketMessage> {
    return this.getMessagesByType(type).pipe(
      debounceTime(debounceMs)
    );
  }

  // Priority-based message processing
  getPriorityMessages(): Observable<WebSocketMessage> {
    return this.messages$.pipe(
      filter(message => message.payload?.priority === 'high' || message.payload?.priority === 'critical'),
      tap(message => console.log('Priority message received:', message))
    );
  }

  private setupOptimizedMessageHandling(): void {
    // Route messages to buffer for batch processing
    this.messages$.subscribe(message => {
      this.messageBuffer$.next(message);
    });

    // Handle high-priority messages immediately
    this.getPriorityMessages().subscribe(message => {
      this.handlePriorityMessage(message);
    });
  }

  private handlePriorityMessage(message: WebSocketMessage): void {
    // Immediate processing for critical messages
    console.log('Processing priority message immediately:', message);
  }
}
```

### 2. Connection Optimization

```typescript
// ✅ Connection optimization strategies
export class ConnectionOptimizer {
  private connectionMetrics = new Map<string, ConnectionMetrics>();

  optimizeConnection(url: string): WebSocketConfig {
    const metrics = this.connectionMetrics.get(url);
    
    if (!metrics) {
      return this.getDefaultConfig();
    }

    return {
      url,
      reconnectInterval: this.calculateOptimalReconnectInterval(metrics),
      maxReconnectAttempts: this.calculateOptimalMaxAttempts(metrics),
      reconnectBackoff: metrics.averageLatency > 1000, // Use backoff for slow connections
      protocols: this.selectOptimalProtocols(metrics)
    };
  }

  recordConnectionMetrics(url: string, metrics: Partial<ConnectionMetrics>): void {
    const existing = this.connectionMetrics.get(url) || {
      successfulConnections: 0,
      failedConnections: 0,
      averageLatency: 0,
      averageReconnectTime: 0,
      lastConnectionTime: new Date()
    };

    this.connectionMetrics.set(url, { ...existing, ...metrics });
  }

  private calculateOptimalReconnectInterval(metrics: ConnectionMetrics): number {
    const baseInterval = 1000;
    const failureRate = metrics.failedConnections / (metrics.successfulConnections + metrics.failedConnections);
    
    // Increase interval based on failure rate
    return Math.min(baseInterval * (1 + failureRate * 5), 30000);
  }

  private calculateOptimalMaxAttempts(metrics: ConnectionMetrics): number {
    const baseAttempts = 5;
    const successRate = metrics.successfulConnections / (metrics.successfulConnections + metrics.failedConnections);
    
    // Increase attempts for reliable connections
    return Math.floor(baseAttempts * (1 + successRate));
  }

  private selectOptimalProtocols(metrics: ConnectionMetrics): string[] {
    // Select protocols based on performance metrics
    if (metrics.averageLatency < 100) {
      return ['v2.websocket.org', 'v1.websocket.org'];
    } else {
      return ['v1.websocket.org'];
    }
  }

  private getDefaultConfig(): WebSocketConfig {
    return {
      url: '',
      reconnectInterval: 5000,
      maxReconnectAttempts: 5,
      reconnectBackoff: true,
      protocols: []
    };
  }
}

interface ConnectionMetrics {
  successfulConnections: number;
  failedConnections: number;
  averageLatency: number;
  averageReconnectTime: number;
  lastConnectionTime: Date;
}
```

## Summary

This comprehensive lesson covered WebSocket integration with RxJS:

### 🎯 Key Concepts
1. **Basic WebSocket Observable**: Creating reactive WebSocket connections
2. **Connection Management**: Robust reconnection and lifecycle handling
3. **Real-Time Patterns**: Chat, notifications, and live updates
4. **Error Handling**: Recovery strategies and error classification
5. **Resource Management**: Connection pooling and optimization
6. **Testing**: Unit and integration testing strategies
7. **Performance**: Throttling, batching, and optimization techniques

### 🚀 Best Practices
- Always implement reconnection logic
- Handle connection states reactively
- Use appropriate operators for message processing
- Implement proper error recovery
- Optimize for high-frequency message scenarios
- Test with realistic network conditions

### 🛠️ Production Considerations
- Monitor connection health and metrics
- Implement graceful degradation
- Handle browser compatibility
- Secure WebSocket connections (WSS)
- Implement proper authentication
- Consider message queuing for offline scenarios

WebSockets with RxJS provide powerful reactive real-time communication capabilities for modern web applications.
  catchError,
  share,
  filter,
  map
} from 'rxjs/operators';

export interface WebSocketMessage {
  type: string;
  payload: any;
  id?: string;
  timestamp?: number;
  correlationId?: string;
}

export interface ConnectionState {
  status: 'connecting' | 'connected' | 'disconnected' | 'error';
  lastConnected?: Date;
  reconnectAttempts: number;
  error?: any;
}

@Injectable({
  providedIn: 'root'
})
export class WebSocketClientService {
  private socket$!: WebSocketSubject<WebSocketMessage>;
  private messagesSubject = new Subject<WebSocketMessage>();
  private connectionStateSubject = new BehaviorSubject<ConnectionState>({
    status: 'disconnected',
    reconnectAttempts: 0
  });

  // Configuration
  private readonly config = {
    url: 'wss://api.example.com/ws',
    reconnectInterval: 5000,
    maxReconnectAttempts: 10,
    heartbeatInterval: 30000,
    messageTimeout: 10000
  };

  // Public observables
  public messages$ = this.messagesSubject.asObservable();
  public connectionState$ = this.connectionStateSubject.asObservable();
  public isConnected$ = this.connectionState$.pipe(
    map(state => state.status === 'connected')
  );

  constructor() {
    this.connect();
  }

  connect(): void {
    if (this.socket$ && !this.socket$.closed) {
      return;
    }

    this.updateConnectionState({ status: 'connecting' });

    const config: WebSocketSubjectConfig<WebSocketMessage> = {
      url: this.config.url,
      openObserver: {
        next: () => {
          console.log('WebSocket connected');
          this.updateConnectionState({
            status: 'connected',
            lastConnected: new Date(),
            reconnectAttempts: 0
          });
          this.startHeartbeat();
        }
      },
      closeObserver: {
        next: (event) => {
          console.log('WebSocket disconnected', event);
          this.updateConnectionState({ status: 'disconnected' });
          this.handleDisconnection();
        }
      },
      serializer: (msg) => JSON.stringify(msg),
      deserializer: (e) => {
        try {
          return JSON.parse(e.data);
        } catch (error) {
          console.error('Failed to parse message:', error);
          return { type: 'error', payload: 'Invalid message format' };
        }
      }
    };

    this.socket$ = webSocket<WebSocketMessage>(config);

    // Subscribe to incoming messages
    this.socket$.subscribe({
      next: (message) => this.handleMessage(message),
      error: (error) => this.handleError(error),
      complete: () => console.log('WebSocket connection completed')
    });
  }

  disconnect(): void {
    if (this.socket$) {
      this.socket$.complete();
      this.updateConnectionState({ status: 'disconnected' });
    }
  }

  send(message: WebSocketMessage): Observable<void> {
    if (!this.socket$ || this.socket$.closed) {
      return throwError(() => new Error('WebSocket is not connected'));
    }

    const messageWithTimestamp = {
      ...message,
      id: this.generateMessageId(),
      timestamp: Date.now()
    };

    try {
      this.socket$.next(messageWithTimestamp);
      return of(void 0);
    } catch (error) {
      return throwError(() => error);
    }
  }

  // Send message and wait for response
  sendWithResponse<T>(
    message: WebSocketMessage, 
    responseType: string,
    timeout: number = this.config.messageTimeout
  ): Observable<T> {
    const correlationId = this.generateMessageId();
    const messageWithCorrelation = {
      ...message,
      correlationId
    };

    // Listen for response
    const response$ = this.messages$.pipe(
      filter(msg => 
        msg.type === responseType && 
        msg.correlationId === correlationId
      ),
      map(msg => msg.payload as T),
      take(1)
    );

    // Send message and return response observable
    return this.send(messageWithCorrelation).pipe(
      switchMap(() => response$),
      timeout(timeout),
      catchError(error => {
        if (error.name === 'TimeoutError') {
          return throwError(() => new Error(`Response timeout for ${responseType}`));
        }
        return throwError(() => error);
      })
    );
  }

  private handleMessage(message: WebSocketMessage): void {
    switch (message.type) {
      case 'heartbeat-response':
        this.handleHeartbeatResponse();
        break;
      case 'error':
        console.error('Server error:', message.payload);
        break;
      default:
        this.messagesSubject.next(message);
    }
  }

  private handleError(error: any): void {
    console.error('WebSocket error:', error);
    this.updateConnectionState({ 
      status: 'error', 
      error 
    });
    this.handleDisconnection();
  }

  private handleDisconnection(): void {
    const currentState = this.connectionStateSubject.value;
    
    if (currentState.reconnectAttempts < this.config.maxReconnectAttempts) {
      this.scheduleReconnect();
    } else {
      console.error('Max reconnection attempts reached');
      this.updateConnectionState({ status: 'error' });
    }
  }

  private scheduleReconnect(): void {
    const currentState = this.connectionStateSubject.value;
    const attempts = currentState.reconnectAttempts + 1;
    const delay = this.calculateReconnectDelay(attempts);

    this.updateConnectionState({ reconnectAttempts: attempts });

    timer(delay).subscribe(() => {
      console.log(`Reconnection attempt ${attempts}`);
      this.connect();
    });
  }

  private calculateReconnectDelay(attempt: number): number {
    // Exponential backoff with jitter
    const baseDelay = this.config.reconnectInterval;
    const maxDelay = 30000;
    const exponentialDelay = Math.min(baseDelay * Math.pow(2, attempt - 1), maxDelay);
    const jitter = Math.random() * 0.1 * exponentialDelay;
    return exponentialDelay + jitter;
  }

  private startHeartbeat(): void {
    timer(0, this.config.heartbeatInterval).pipe(
      takeUntil(this.connectionState$.pipe(
        filter(state => state.status !== 'connected')
      ))
    ).subscribe(() => {
      this.send({
        type: 'heartbeat',
        payload: { timestamp: Date.now() }
      }).subscribe();
    });
  }

  private handleHeartbeatResponse(): void {
    // Heartbeat acknowledged - connection is healthy
  }

  private updateConnectionState(partial: Partial<ConnectionState>): void {
    const current = this.connectionStateSubject.value;
    this.connectionStateSubject.next({ ...current, ...partial });
  }

  private generateMessageId(): string {
    return Date.now().toString(36) + Math.random().toString(36).substr(2);
  }
}
```

## Connection Management

### Advanced Connection Manager

```typescript
// websocket-connection-manager.service.ts
@Injectable({
  providedIn: 'root'
})
export class WebSocketConnectionManager {
  private connections = new Map<string, WebSocketClientService>();
  private connectionConfigs = new Map<string, WebSocketConfig>();
  
  // Global connection state
  private globalStateSubject = new BehaviorSubject<GlobalConnectionState>({
    totalConnections: 0,
    activeConnections: 0,
    failedConnections: 0
  });

  public globalState$ = this.globalStateSubject.asObservable();

  constructor(private authService: AuthService) {}

  // Create or get existing connection
  getConnection(connectionId: string, config?: WebSocketConfig): WebSocketClientService {
    if (this.connections.has(connectionId)) {
      return this.connections.get(connectionId)!;
    }

    return this.createConnection(connectionId, config);
  }

  createConnection(connectionId: string, config?: WebSocketConfig): WebSocketClientService {
    const finalConfig = {
      ...this.getDefaultConfig(),
      ...config
    };

    this.connectionConfigs.set(connectionId, finalConfig);

    const connection = new WebSocketClientService();
    this.connections.set(connectionId, connection);

    // Monitor connection state
    connection.connectionState$.subscribe(state => {
      this.updateGlobalState();
    });

    this.updateGlobalState();
    return connection;
  }

  // Connection with authentication
  createAuthenticatedConnection(
    connectionId: string, 
    config?: WebSocketConfig
  ): Observable<WebSocketClientService> {
    return this.authService.getToken().pipe(
      switchMap(token => {
        const authConfig = {
          ...config,
          url: this.addAuthToUrl(config?.url || this.getDefaultConfig().url, token),
          headers: {
            ...config?.headers,
            'Authorization': `Bearer ${token}`
          }
        };

        const connection = this.createConnection(connectionId, authConfig);
        return of(connection);
      })
    );
  }

  // Multi-endpoint connection
  createMultiEndpointConnection(
    connectionId: string,
    endpoints: string[]
  ): Observable<WebSocketClientService> {
    return this.findBestEndpoint(endpoints).pipe(
      switchMap(bestEndpoint => {
        const config: WebSocketConfig = {
          url: bestEndpoint,
          fallbackUrls: endpoints.filter(url => url !== bestEndpoint)
        };
        
        return of(this.createConnection(connectionId, config));
      })
    );
  }

  closeConnection(connectionId: string): void {
    const connection = this.connections.get(connectionId);
    if (connection) {
      connection.disconnect();
      this.connections.delete(connectionId);
      this.connectionConfigs.delete(connectionId);
      this.updateGlobalState();
    }
  }

  closeAllConnections(): void {
    this.connections.forEach((connection, id) => {
      connection.disconnect();
    });
    this.connections.clear();
    this.connectionConfigs.clear();
    this.updateGlobalState();
  }

  // Health check for all connections
  healthCheck(): Observable<ConnectionHealth[]> {
    const healthChecks = Array.from(this.connections.entries()).map(
      ([id, connection]) => this.checkConnectionHealth(id, connection)
    );

    return Promise.all(healthChecks).then(results => of(results));
  }

  private async checkConnectionHealth(
    id: string, 
    connection: WebSocketClientService
  ): Promise<ConnectionHealth> {
    try {
      const pingStart = Date.now();
      
      await connection.sendWithResponse(
        { type: 'ping', payload: {} },
        'pong',
        5000
      ).toPromise();

      const latency = Date.now() - pingStart;

      return {
        connectionId: id,
        status: 'healthy',
        latency,
        lastCheck: new Date()
      };
    } catch (error) {
      return {
        connectionId: id,
        status: 'unhealthy',
        error: error.message,
        lastCheck: new Date()
      };
    }
  }

  private findBestEndpoint(endpoints: string[]): Observable<string> {
    // Test each endpoint and return the fastest responding one
    const tests = endpoints.map(url => this.testEndpoint(url));
    
    return Promise.race(tests).then(winner => of(winner));
  }

  private async testEndpoint(url: string): Promise<string> {
    return new Promise((resolve, reject) => {
      const testSocket = new WebSocket(url);
      const timeout = setTimeout(() => {
        testSocket.close();
        reject(new Error('Timeout'));
      }, 3000);

      testSocket.onopen = () => {
        clearTimeout(timeout);
        testSocket.close();
        resolve(url);
      };

      testSocket.onerror = () => {
        clearTimeout(timeout);
        reject(new Error('Connection failed'));
      };
    });
  }

  private addAuthToUrl(url: string, token: string): string {
    const separator = url.includes('?') ? '&' : '?';
    return `${url}${separator}token=${encodeURIComponent(token)}`;
  }

  private getDefaultConfig(): WebSocketConfig {
    return {
      url: 'wss://api.example.com/ws',
      reconnectInterval: 5000,
      maxReconnectAttempts: 10,
      heartbeatInterval: 30000
    };
  }

  private updateGlobalState(): void {
    const totalConnections = this.connections.size;
    let activeConnections = 0;
    let failedConnections = 0;

    this.connections.forEach(connection => {
      connection.connectionState$.pipe(take(1)).subscribe(state => {
        if (state.status === 'connected') {
          activeConnections++;
        } else if (state.status === 'error') {
          failedConnections++;
        }
      });
    });

    this.globalStateSubject.next({
      totalConnections,
      activeConnections,
      failedConnections
    });
  }
}

// Supporting interfaces
interface WebSocketConfig {
  url: string;
  reconnectInterval?: number;
  maxReconnectAttempts?: number;
  heartbeatInterval?: number;
  headers?: { [key: string]: string };
  fallbackUrls?: string[];
}

interface GlobalConnectionState {
  totalConnections: number;
  activeConnections: number;
  failedConnections: number;
}

interface ConnectionHealth {
  connectionId: string;
  status: 'healthy' | 'unhealthy';
  latency?: number;
  error?: string;
  lastCheck: Date;
}
```

## Message Handling Patterns

### Message Router and Queue

```typescript
// websocket-message-router.service.ts
@Injectable({
  providedIn: 'root'
})
export class WebSocketMessageRouter {
  private routeHandlers = new Map<string, MessageHandler[]>();
  private messageQueue: QueuedMessage[] = [];
  private isProcessingQueue = false;
  
  // Message middleware chain
  private middleware: MessageMiddleware[] = [];

  constructor(private connectionManager: WebSocketConnectionManager) {}

  // Register message handler
  registerHandler(messageType: string, handler: MessageHandler): void {
    if (!this.routeHandlers.has(messageType)) {
      this.routeHandlers.set(messageType, []);
    }
    this.routeHandlers.get(messageType)!.push(handler);
  }

  // Remove message handler
  unregisterHandler(messageType: string, handler: MessageHandler): void {
    const handlers = this.routeHandlers.get(messageType);
    if (handlers) {
      const index = handlers.indexOf(handler);
      if (index > -1) {
        handlers.splice(index, 1);
      }
    }
  }

  // Add middleware
  addMiddleware(middleware: MessageMiddleware): void {
    this.middleware.push(middleware);
  }

  // Route incoming message
  routeMessage(connectionId: string, message: WebSocketMessage): void {
    // Apply middleware chain
    const processedMessage = this.applyMiddleware(message);
    
    if (!processedMessage) {
      return; // Message was filtered out by middleware
    }

    // Get handlers for message type
    const handlers = this.routeHandlers.get(processedMessage.type) || [];
    
    // Execute handlers
    handlers.forEach(handler => {
      try {
        handler.handle(connectionId, processedMessage);
      } catch (error) {
        console.error(`Handler error for ${processedMessage.type}:`, error);
      }
    });
  }

  // Queue message for later processing
  queueMessage(
    connectionId: string, 
    message: WebSocketMessage, 
    priority: MessagePriority = 'normal'
  ): void {
    const queuedMessage: QueuedMessage = {
      connectionId,
      message,
      priority,
      timestamp: Date.now(),
      retries: 0
    };

    // Insert based on priority
    if (priority === 'high') {
      this.messageQueue.unshift(queuedMessage);
    } else {
      this.messageQueue.push(queuedMessage);
    }

    this.processQueue();
  }

  // Process queued messages
  private async processQueue(): Promise<void> {
    if (this.isProcessingQueue || this.messageQueue.length === 0) {
      return;
    }

    this.isProcessingQueue = true;

    while (this.messageQueue.length > 0) {
      const queuedMessage = this.messageQueue.shift()!;
      
      try {
        await this.processQueuedMessage(queuedMessage);
      } catch (error) {
        console.error('Queue processing error:', error);
        
        // Retry logic
        if (queuedMessage.retries < 3) {
          queuedMessage.retries++;
          this.messageQueue.push(queuedMessage);
        }
      }
    }

    this.isProcessingQueue = false;
  }

  private async processQueuedMessage(queuedMessage: QueuedMessage): Promise<void> {
    const connection = this.connectionManager.getConnection(queuedMessage.connectionId);
    
    if (!connection) {
      throw new Error(`Connection ${queuedMessage.connectionId} not found`);
    }

    return connection.send(queuedMessage.message).toPromise();
  }

  private applyMiddleware(message: WebSocketMessage): WebSocketMessage | null {
    let currentMessage: WebSocketMessage | null = message;

    for (const middleware of this.middleware) {
      currentMessage = middleware.process(currentMessage);
      if (!currentMessage) {
        break; // Message was filtered out
      }
    }

    return currentMessage;
  }

  // Built-in middleware examples
  static createLoggingMiddleware(): MessageMiddleware {
    return {
      process: (message: WebSocketMessage | null): WebSocketMessage | null => {
        if (message) {
          console.log(`[WS] ${message.type}:`, message.payload);
        }
        return message;
      }
    };
  }

  static createValidationMiddleware(schema: any): MessageMiddleware {
    return {
      process: (message: WebSocketMessage | null): WebSocketMessage | null => {
        if (!message) return null;
        
        // Validate message against schema
        const isValid = this.validateMessage(message, schema);
        return isValid ? message : null;
      }
    };
  }

  private static validateMessage(message: WebSocketMessage, schema: any): boolean {
    // Implement validation logic based on your schema system
    return true; // Placeholder
  }
}

// Message handler implementation
export abstract class MessageHandler {
  abstract handle(connectionId: string, message: WebSocketMessage): void;
}

// Specific handler example
@Injectable()
export class ChatMessageHandler extends MessageHandler {
  constructor(private chatService: ChatService) {
    super();
  }

  handle(connectionId: string, message: WebSocketMessage): void {
    if (message.type === 'chat-message') {
      this.chatService.addMessage(message.payload);
    }
  }
}

// Interfaces
interface QueuedMessage {
  connectionId: string;
  message: WebSocketMessage;
  priority: MessagePriority;
  timestamp: number;
  retries: number;
}

interface MessageMiddleware {
  process(message: WebSocketMessage | null): WebSocketMessage | null;
}

type MessagePriority = 'low' | 'normal' | 'high';
```

## Real-time Communication Services

### Real-time Chat Service

```typescript
// real-time-chat.service.ts
@Injectable({
  providedIn: 'root'
})
export class RealTimeChatService {
  private readonly connectionId = 'chat-connection';
  private connection!: WebSocketClientService;
  
  // Chat state
  private messagesSubject = new BehaviorSubject<ChatMessage[]>([]);
  private roomsSubject = new BehaviorSubject<ChatRoom[]>([]);
  private usersSubject = new BehaviorSubject<ChatUser[]>([]);
  private typingUsersSubject = new BehaviorSubject<TypingUser[]>([]);
  
  // Public streams
  public messages$ = this.messagesSubject.asObservable();
  public rooms$ = this.roomsSubject.asObservable();
  public users$ = this.usersSubject.asObservable();
  public typingUsers$ = this.typingUsersSubject.asObservable();
  
  // Connection status
  public isConnected$ = this.connectionManager.globalState$.pipe(
    map(state => state.activeConnections > 0)
  );

  constructor(
    private connectionManager: WebSocketConnectionManager,
    private messageRouter: WebSocketMessageRouter,
    private userService: UserService
  ) {
    this.initializeConnection();
    this.setupMessageHandlers();
  }

  private async initializeConnection(): Promise<void> {
    try {
      this.connection = await this.connectionManager.createAuthenticatedConnection(
        this.connectionId,
        { url: 'wss://chat.example.com/ws' }
      ).toPromise();

      // Route all messages through the router
      this.connection.messages$.subscribe(message => {
        this.messageRouter.routeMessage(this.connectionId, message);
      });

    } catch (error) {
      console.error('Failed to initialize chat connection:', error);
    }
  }

  private setupMessageHandlers(): void {
    // Message handlers
    this.messageRouter.registerHandler('chat-message', {
      handle: (connectionId, message) => this.handleChatMessage(message)
    });

    this.messageRouter.registerHandler('user-joined', {
      handle: (connectionId, message) => this.handleUserJoined(message)
    });

    this.messageRouter.registerHandler('user-left', {
      handle: (connectionId, message) => this.handleUserLeft(message)
    });

    this.messageRouter.registerHandler('typing-start', {
      handle: (connectionId, message) => this.handleTypingStart(message)
    });

    this.messageRouter.registerHandler('typing-stop', {
      handle: (connectionId, message) => this.handleTypingStop(message)
    });
  }

  // Public methods
  joinRoom(roomId: string): Observable<void> {
    return this.connection.sendWithResponse(
      {
        type: 'join-room',
        payload: { roomId }
      },
      'room-joined'
    ).pipe(
      tap(() => this.loadRoomData(roomId)),
      map(() => void 0)
    );
  }

  leaveRoom(roomId: string): Observable<void> {
    return this.connection.send({
      type: 'leave-room',
      payload: { roomId }
    });
  }

  sendMessage(roomId: string, content: string): Observable<ChatMessage> {
    const tempMessage: ChatMessage = {
      id: this.generateTempId(),
      roomId,
      content,
      userId: this.userService.getCurrentUserId(),
      userName: this.userService.getCurrentUserName(),
      timestamp: new Date(),
      status: 'sending'
    };

    // Optimistically add to messages
    this.addMessage(tempMessage);

    return this.connection.sendWithResponse(
      {
        type: 'send-message',
        payload: { roomId, content }
      },
      'message-sent'
    ).pipe(
      map(response => response as ChatMessage),
      tap(confirmedMessage => {
        // Replace temp message with confirmed one
        this.replaceMessage(tempMessage.id, confirmedMessage);
      }),
      catchError(error => {
        // Mark message as failed
        this.updateMessageStatus(tempMessage.id, 'failed');
        return throwError(() => error);
      })
    );
  }

  // Real-time typing indicators
  startTyping(roomId: string): void {
    this.connection.send({
      type: 'typing-start',
      payload: { roomId }
    }).subscribe();
  }

  stopTyping(roomId: string): void {
    this.connection.send({
      type: 'typing-stop',
      payload: { roomId }
    }).subscribe();
  }

  // Message reactions
  addReaction(messageId: string, emoji: string): Observable<void> {
    return this.connection.send({
      type: 'add-reaction',
      payload: { messageId, emoji }
    });
  }

  // File sharing
  sendFile(roomId: string, file: File): Observable<ChatMessage> {
    // First upload file
    return this.uploadFile(file).pipe(
      switchMap(fileUrl => {
        return this.connection.sendWithResponse(
          {
            type: 'send-file',
            payload: { 
              roomId, 
              fileUrl, 
              fileName: file.name,
              fileSize: file.size,
              fileType: file.type
            }
          },
          'file-sent'
        );
      }),
      map(response => response as ChatMessage)
    );
  }

  // Message handlers
  private handleChatMessage(message: WebSocketMessage): void {
    const chatMessage = message.payload as ChatMessage;
    this.addMessage(chatMessage);
  }

  private handleUserJoined(message: WebSocketMessage): void {
    const user = message.payload as ChatUser;
    this.addUser(user);
  }

  private handleUserLeft(message: WebSocketMessage): void {
    const userId = message.payload.userId;
    this.removeUser(userId);
  }

  private handleTypingStart(message: WebSocketMessage): void {
    const typingUser = message.payload as TypingUser;
    this.addTypingUser(typingUser);
  }

  private handleTypingStop(message: WebSocketMessage): void {
    const userId = message.payload.userId;
    this.removeTypingUser(userId);
  }

  // State management methods
  private addMessage(message: ChatMessage): void {
    const currentMessages = this.messagesSubject.value;
    const updatedMessages = [...currentMessages, message]
      .sort((a, b) => a.timestamp.getTime() - b.timestamp.getTime());
    this.messagesSubject.next(updatedMessages);
  }

  private replaceMessage(tempId: string, newMessage: ChatMessage): void {
    const currentMessages = this.messagesSubject.value;
    const index = currentMessages.findIndex(m => m.id === tempId);
    
    if (index > -1) {
      const updatedMessages = [...currentMessages];
      updatedMessages[index] = newMessage;
      this.messagesSubject.next(updatedMessages);
    }
  }

  private updateMessageStatus(messageId: string, status: MessageStatus): void {
    const currentMessages = this.messagesSubject.value;
    const updatedMessages = currentMessages.map(message =>
      message.id === messageId ? { ...message, status } : message
    );
    this.messagesSubject.next(updatedMessages);
  }

  private addUser(user: ChatUser): void {
    const currentUsers = this.usersSubject.value;
    if (!currentUsers.find(u => u.id === user.id)) {
      this.usersSubject.next([...currentUsers, user]);
    }
  }

  private removeUser(userId: string): void {
    const currentUsers = this.usersSubject.value;
    const filteredUsers = currentUsers.filter(u => u.id !== userId);
    this.usersSubject.next(filteredUsers);
  }

  private addTypingUser(typingUser: TypingUser): void {
    const currentTyping = this.typingUsersSubject.value;
    const filtered = currentTyping.filter(u => u.userId !== typingUser.userId);
    this.typingUsersSubject.next([...filtered, typingUser]);

    // Auto-remove after timeout
    timer(3000).subscribe(() => {
      this.removeTypingUser(typingUser.userId);
    });
  }

  private removeTypingUser(userId: string): void {
    const currentTyping = this.typingUsersSubject.value;
    const filtered = currentTyping.filter(u => u.userId !== userId);
    this.typingUsersSubject.next(filtered);
  }

  private loadRoomData(roomId: string): void {
    // Load initial room data (messages, users, etc.)
    this.connection.send({
      type: 'load-room-data',
      payload: { roomId }
    }).subscribe();
  }

  private uploadFile(file: File): Observable<string> {
    // Implement file upload logic
    return of('https://example.com/uploaded-file.jpg');
  }

  private generateTempId(): string {
    return `temp-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
}

// Chat interfaces
interface ChatMessage {
  id: string;
  roomId: string;
  content: string;
  userId: string;
  userName: string;
  timestamp: Date;
  status?: MessageStatus;
  reactions?: MessageReaction[];
  fileUrl?: string;
  fileName?: string;
}

interface ChatRoom {
  id: string;
  name: string;
  description?: string;
  userCount: number;
  isPrivate: boolean;
}

interface ChatUser {
  id: string;
  name: string;
  avatar?: string;
  status: 'online' | 'away' | 'offline';
  lastSeen?: Date;
}

interface TypingUser {
  userId: string;
  userName: string;
  roomId: string;
}

interface MessageReaction {
  emoji: string;
  users: string[];
  count: number;
}

type MessageStatus = 'sending' | 'sent' | 'delivered' | 'read' | 'failed';
```

## Authentication & Authorization

### Secure WebSocket Authentication

```typescript
// websocket-auth.service.ts
@Injectable({
  providedIn: 'root'
})
export class WebSocketAuthService {
  private authTokenSubject = new BehaviorSubject<string | null>(null);
  private refreshTokenSubject = new BehaviorSubject<string | null>(null);
  
  public authToken$ = this.authTokenSubject.asObservable();
  
  constructor(
    private authService: AuthService,
    private connectionManager: WebSocketConnectionManager
  ) {
    this.setupTokenRefresh();
  }

  // Authenticate WebSocket connection
  authenticateConnection(connectionId: string): Observable<WebSocketClientService> {
    return this.getValidToken().pipe(
      switchMap(token => {
        return this.connectionManager.createAuthenticatedConnection(
          connectionId,
          {
            url: this.buildAuthenticatedUrl(token),
            headers: { 'Authorization': `Bearer ${token}` }
          }
        );
      }),
      tap(connection => this.setupTokenValidation(connection))
    );
  }

  // Handle token refresh for WebSocket
  private setupTokenRefresh(): void {
    // Monitor token expiry
    this.authService.tokenExpiry$.pipe(
      switchMap(expiryTime => {
        const timeUntilExpiry = expiryTime - Date.now() - 60000; // Refresh 1 min early
        return timer(Math.max(0, timeUntilExpiry));
      })
    ).subscribe(() => {
      this.refreshAllConnections();
    });
  }

  private setupTokenValidation(connection: WebSocketClientService): void {
    // Listen for auth-related messages
    connection.messages$.pipe(
      filter(message => message.type === 'auth-error' || message.type === 'token-expired')
    ).subscribe(message => {
      this.handleAuthError(connection, message);
    });
  }

  private handleAuthError(connection: WebSocketClientService, message: WebSocketMessage): void {
    if (message.type === 'token-expired') {
      this.refreshConnectionAuth(connection);
    } else {
      console.error('Authentication error:', message.payload);
      // Handle auth failure (redirect to login, etc.)
    }
  }

  private refreshConnectionAuth(connection: WebSocketClientService): void {
    this.getValidToken().subscribe(newToken => {
      // Send new token to server
      connection.send({
        type: 'refresh-auth',
        payload: { token: newToken }
      }).subscribe();
    });
  }

  private refreshAllConnections(): void {
    // Get all active connections and refresh their auth
    this.connectionManager.healthCheck().subscribe(healthStates => {
      healthStates.forEach(health => {
        if (health.status === 'healthy') {
          const connection = this.connectionManager.getConnection(health.connectionId);
          this.refreshConnectionAuth(connection);
        }
      });
    });
  }

  private getValidToken(): Observable<string> {
    return this.authService.getToken().pipe(
      switchMap(token => {
        if (this.isTokenExpiringSoon(token)) {
          return this.authService.refreshToken();
        }
        return of(token);
      })
    );
  }

  private buildAuthenticatedUrl(token: string): string {
    const baseUrl = 'wss://api.example.com/ws';
    return `${baseUrl}?token=${encodeURIComponent(token)}`;
  }

  private isTokenExpiringSoon(token: string): boolean {
    // Check if token expires within 5 minutes
    const decoded = this.decodeToken(token);
    const expiryTime = decoded.exp * 1000;
    return (expiryTime - Date.now()) < 300000;
  }

  private decodeToken(token: string): any {
    // Simple JWT decode (use a proper library in production)
    const payload = token.split('.')[1];
    return JSON.parse(atob(payload));
  }
}

// Role-based message filtering
@Injectable()
export class WebSocketAuthMiddleware implements MessageMiddleware {
  constructor(private authService: AuthService) {}

  process(message: WebSocketMessage | null): WebSocketMessage | null {
    if (!message) return null;

    const userRoles = this.authService.getCurrentUserRoles();
    const requiredRole = this.getRequiredRole(message.type);

    if (requiredRole && !userRoles.includes(requiredRole)) {
      console.warn(`Access denied for message type: ${message.type}`);
      return null;
    }

    return message;
  }

  private getRequiredRole(messageType: string): string | null {
    const roleMap: { [key: string]: string } = {
      'admin-broadcast': 'admin',
      'moderator-action': 'moderator',
      'private-message': 'user'
    };

    return roleMap[messageType] || null;
  }
}
```

## Offline & Recovery Strategies

### Offline Message Queue

```typescript
// offline-websocket.service.ts
@Injectable({
  providedIn: 'root'
})
export class OfflineWebSocketService {
  private offlineQueue: OfflineMessage[] = [];
  private isOnline$ = new BehaviorSubject<boolean>(navigator.onLine);
  
  constructor(
    private connectionManager: WebSocketConnectionManager,
    private storage: LocalStorageService
  ) {
    this.setupOnlineMonitoring();
    this.loadOfflineQueue();
  }

  // Send message with offline support
  sendMessageWithOfflineSupport(
    connectionId: string,
    message: WebSocketMessage
  ): Observable<void> {
    return this.isOnline$.pipe(
      take(1),
      switchMap(isOnline => {
        if (isOnline) {
          const connection = this.connectionManager.getConnection(connectionId);
          return connection.send(message);
        } else {
          this.queueOfflineMessage(connectionId, message);
          return of(void 0);
        }
      })
    );
  }

  private setupOnlineMonitoring(): void {
    // Monitor online/offline status
    fromEvent(window, 'online').subscribe(() => {
      this.isOnline$.next(true);
      this.processOfflineQueue();
    });

    fromEvent(window, 'offline').subscribe(() => {
      this.isOnline$.next(false);
    });

    // Monitor connection status
    this.connectionManager.globalState$.pipe(
      map(state => state.activeConnections > 0),
      distinctUntilChanged()
    ).subscribe(hasActiveConnections => {
      if (hasActiveConnections && this.isOnline$.value) {
        this.processOfflineQueue();
      }
    });
  }

  private queueOfflineMessage(connectionId: string, message: WebSocketMessage): void {
    const offlineMessage: OfflineMessage = {
      id: this.generateId(),
      connectionId,
      message,
      timestamp: Date.now(),
      retries: 0
    };

    this.offlineQueue.push(offlineMessage);
    this.saveOfflineQueue();
  }

  private async processOfflineQueue(): Promise<void> {
    if (this.offlineQueue.length === 0) {
      return;
    }

    const messagesToProcess = [...this.offlineQueue];
    this.offlineQueue = [];

    for (const offlineMessage of messagesToProcess) {
      try {
        await this.sendOfflineMessage(offlineMessage);
      } catch (error) {
        console.error('Failed to send offline message:', error);
        
        if (offlineMessage.retries < 3) {
          offlineMessage.retries++;
          this.offlineQueue.push(offlineMessage);
        }
      }
    }

    this.saveOfflineQueue();
  }

  private async sendOfflineMessage(offlineMessage: OfflineMessage): Promise<void> {
    const connection = this.connectionManager.getConnection(offlineMessage.connectionId);
    return connection.send(offlineMessage.message).toPromise();
  }

  private loadOfflineQueue(): void {
    const stored = this.storage.getItem('websocket-offline-queue');
    if (stored) {
      this.offlineQueue = JSON.parse(stored);
    }
  }

  private saveOfflineQueue(): void {
    this.storage.setItem('websocket-offline-queue', JSON.stringify(this.offlineQueue));
  }

  private generateId(): string {
    return Date.now().toString(36) + Math.random().toString(36).substr(2);
  }
}

interface OfflineMessage {
  id: string;
  connectionId: string;
  message: WebSocketMessage;
  timestamp: number;
  retries: number;
}
```

## Performance Optimization

### Message Batching and Throttling

```typescript
// websocket-performance.service.ts
@Injectable({
  providedIn: 'root'
})
export class WebSocketPerformanceService {
  private messageBuffer = new Map<string, WebSocketMessage[]>();
  private readonly batchConfig = {
    maxBatchSize: 10,
    batchTimeout: 100,
    throttleTime: 50
  };

  constructor(private connectionManager: WebSocketConnectionManager) {}

  // Send messages in batches
  sendBatch(connectionId: string, messages: WebSocketMessage[]): Observable<void> {
    if (!this.messageBuffer.has(connectionId)) {
      this.messageBuffer.set(connectionId, []);
    }

    const buffer = this.messageBuffer.get(connectionId)!;
    buffer.push(...messages);

    // Process batch if size threshold reached
    if (buffer.length >= this.batchConfig.maxBatchSize) {
      return this.processBatch(connectionId);
    }

    // Schedule batch processing
    timer(this.batchConfig.batchTimeout).subscribe(() => {
      if (buffer.length > 0) {
        this.processBatch(connectionId).subscribe();
      }
    });

    return of(void 0);
  }

  // Throttled message sending
  sendThrottled(connectionId: string, message: WebSocketMessage): Observable<void> {
    return of(message).pipe(
      throttleTime(this.batchConfig.throttleTime),
      switchMap(() => {
        const connection = this.connectionManager.getConnection(connectionId);
        return connection.send(message);
      })
    );
  }

  // Debounced message sending (useful for typing indicators)
  sendDebounced(
    connectionId: string, 
    message: WebSocketMessage,
    debounceMs: number = 300
  ): Observable<void> {
    return of(message).pipe(
      debounceTime(debounceMs),
      switchMap(() => {
        const connection = this.connectionManager.getConnection(connectionId);
        return connection.send(message);
      })
    );
  }

  private processBatch(connectionId: string): Observable<void> {
    const buffer = this.messageBuffer.get(connectionId);
    if (!buffer || buffer.length === 0) {
      return of(void 0);
    }

    const messagesToSend = buffer.splice(0, this.batchConfig.maxBatchSize);
    const connection = this.connectionManager.getConnection(connectionId);

    // Send as single batch message
    const batchMessage: WebSocketMessage = {
      type: 'batch',
      payload: { messages: messagesToSend }
    };

    return connection.send(batchMessage);
  }
}
```

## Testing WebSocket Services

### Mock WebSocket for Testing

```typescript
// websocket-testing.service.ts
export class MockWebSocketService {
  private messageSubject = new Subject<WebSocketMessage>();
  private connectionStateSubject = new BehaviorSubject<ConnectionState>({
    status: 'disconnected',
    reconnectAttempts: 0
  });

  public messages$ = this.messageSubject.asObservable();
  public connectionState$ = this.connectionStateSubject.asObservable();
  public isConnected$ = this.connectionState$.pipe(
    map(state => state.status === 'connected')
  );

  connect(): void {
    this.connectionStateSubject.next({
      status: 'connected',
      lastConnected: new Date(),
      reconnectAttempts: 0
    });
  }

  disconnect(): void {
    this.connectionStateSubject.next({
      status: 'disconnected',
      reconnectAttempts: 0
    });
  }

  send(message: WebSocketMessage): Observable<void> {
    // Simulate sending
    return of(void 0).pipe(delay(10));
  }

  sendWithResponse<T>(
    message: WebSocketMessage,
    responseType: string,
    timeout?: number
  ): Observable<T> {
    // Simulate request-response
    return of({} as T).pipe(delay(50));
  }

  // Test helpers
  simulateMessage(message: WebSocketMessage): void {
    this.messageSubject.next(message);
  }

  simulateDisconnection(): void {
    this.connectionStateSubject.next({
      status: 'disconnected',
      reconnectAttempts: 0
    });
  }

  simulateError(error: any): void {
    this.connectionStateSubject.next({
      status: 'error',
      error,
      reconnectAttempts: 0
    });
  }
}

// Test example
describe('RealTimeChatService', () => {
  let service: RealTimeChatService;
  let mockWebSocket: MockWebSocketService;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        RealTimeChatService,
        { provide: WebSocketClientService, useClass: MockWebSocketService }
      ]
    });

    service = TestBed.inject(RealTimeChatService);
    mockWebSocket = TestBed.inject(WebSocketClientService) as any;
  });

  it('should handle incoming chat messages', () => {
    const testMessage: ChatMessage = {
      id: '1',
      roomId: 'room1',
      content: 'Hello',
      userId: 'user1',
      userName: 'John',
      timestamp: new Date()
    };

    let receivedMessages: ChatMessage[] = [];
    service.messages$.subscribe(messages => {
      receivedMessages = messages;
    });

    mockWebSocket.simulateMessage({
      type: 'chat-message',
      payload: testMessage
    });

    expect(receivedMessages).toContain(testMessage);
  });

  it('should handle connection failures gracefully', () => {
    let connectionStatus: boolean = true;
    service.isConnected$.subscribe(status => {
      connectionStatus = status;
    });

    mockWebSocket.simulateDisconnection();

    expect(connectionStatus).toBeFalse();
  });
});
```

## Summary

This lesson covered comprehensive WebSocket integration with RxJS for real-time Angular applications:

### Key Takeaways:
1. **WebSocket Fundamentals**: Proper Observable-based WebSocket implementation with error handling
2. **Connection Management**: Advanced connection lifecycle management with authentication and health monitoring
3. **Message Handling**: Sophisticated message routing, queuing, and middleware patterns
4. **Real-time Services**: Production-ready chat service with optimistic updates and conflict resolution
5. **Authentication**: Secure WebSocket authentication with token refresh and role-based access
6. **Offline Support**: Robust offline message queuing and automatic recovery
7. **Performance**: Message batching, throttling, and optimization strategies
8. **Testing**: Comprehensive testing strategies with mock implementations

### Next Steps:
- Implement WebSocket connections in your applications
- Add proper error handling and reconnection logic
- Consider message queuing for offline scenarios
- Implement authentication and authorization
- Monitor connection health and performance

In the next lesson, we'll explore **Real-time Notification Patterns** for building comprehensive notification systems.
