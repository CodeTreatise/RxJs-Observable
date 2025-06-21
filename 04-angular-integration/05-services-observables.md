# Building Reactive Services

## Overview

Reactive services are the backbone of Angular applications built with RxJS. They provide data streams, manage state, handle business logic, and coordinate communication between components. This lesson covers comprehensive patterns for building robust, scalable, and maintainable reactive services using RxJS operators and Angular's dependency injection system.

## Reactive Service Fundamentals

### Basic Reactive Service Structure

**Core Service Pattern:**
```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { BehaviorSubject, Observable, Subject, combineLatest, throwError } from 'rxjs';
import { map, switchMap, catchError, tap, shareReplay, filter, distinctUntilChanged } from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class UserService {
  private readonly baseUrl = '/api/users';
  
  // Private state subjects
  private usersSubject = new BehaviorSubject<User[]>([]);
  private selectedUserSubject = new BehaviorSubject<User | null>(null);
  private loadingSubject = new BehaviorSubject<boolean>(false);
  private errorSubject = new BehaviorSubject<string | null>(null);
  
  // Public observables
  public readonly users$ = this.usersSubject.asObservable();
  public readonly selectedUser$ = this.selectedUserSubject.asObservable();
  public readonly loading$ = this.loadingSubject.asObservable();
  public readonly error$ = this.errorSubject.asObservable();
  
  // Derived observables
  public readonly userCount$ = this.users$.pipe(
    map(users => users.length)
  );
  
  public readonly activeUsers$ = this.users$.pipe(
    map(users => users.filter(user => user.isActive))
  );
  
  public readonly selectedUserPosts$ = this.selectedUser$.pipe(
    filter(user => !!user),
    switchMap(user => this.http.get<Post[]>(`${this.baseUrl}/${user!.id}/posts`)),
    shareReplay(1)
  );

  constructor(private http: HttpClient) {
    this.initializeService();
  }

  private initializeService(): void {
    // Load initial data
    this.loadUsers().subscribe();
  }

  // Public methods
  loadUsers(): Observable<User[]> {
    this.setLoading(true);
    this.clearError();
    
    return this.http.get<User[]>(this.baseUrl).pipe(
      tap(users => {
        this.usersSubject.next(users);
        this.setLoading(false);
      }),
      catchError(error => {
        this.handleError('Failed to load users', error);
        return throwError(() => error);
      })
    );
  }

  getUserById(id: number): Observable<User> {
    return this.http.get<User>(`${this.baseUrl}/${id}`).pipe(
      tap(user => {
        // Update user in the list if it exists
        const currentUsers = this.usersSubject.value;
        const index = currentUsers.findIndex(u => u.id === id);
        if (index !== -1) {
          const updatedUsers = [...currentUsers];
          updatedUsers[index] = user;
          this.usersSubject.next(updatedUsers);
        }
      }),
      catchError(error => {
        this.handleError(`Failed to load user ${id}`, error);
        return throwError(() => error);
      })
    );
  }

  createUser(userData: Partial<User>): Observable<User> {
    this.setLoading(true);
    this.clearError();
    
    return this.http.post<User>(this.baseUrl, userData).pipe(
      tap(newUser => {
        const currentUsers = this.usersSubject.value;
        this.usersSubject.next([...currentUsers, newUser]);
        this.setLoading(false);
      }),
      catchError(error => {
        this.handleError('Failed to create user', error);
        return throwError(() => error);
      })
    );
  }

  updateUser(id: number, updates: Partial<User>): Observable<User> {
    this.setLoading(true);
    this.clearError();
    
    return this.http.put<User>(`${this.baseUrl}/${id}`, updates).pipe(
      tap(updatedUser => {
        const currentUsers = this.usersSubject.value;
        const index = currentUsers.findIndex(u => u.id === id);
        if (index !== -1) {
          const newUsers = [...currentUsers];
          newUsers[index] = updatedUser;
          this.usersSubject.next(newUsers);
        }
        
        // Update selected user if it's the same
        const selectedUser = this.selectedUserSubject.value;
        if (selectedUser && selectedUser.id === id) {
          this.selectedUserSubject.next(updatedUser);
        }
        
        this.setLoading(false);
      }),
      catchError(error => {
        this.handleError('Failed to update user', error);
        return throwError(() => error);
      })
    );
  }

  deleteUser(id: number): Observable<void> {
    this.setLoading(true);
    this.clearError();
    
    return this.http.delete<void>(`${this.baseUrl}/${id}`).pipe(
      tap(() => {
        const currentUsers = this.usersSubject.value;
        const filteredUsers = currentUsers.filter(u => u.id !== id);
        this.usersSubject.next(filteredUsers);
        
        // Clear selected user if it was deleted
        const selectedUser = this.selectedUserSubject.value;
        if (selectedUser && selectedUser.id === id) {
          this.selectedUserSubject.next(null);
        }
        
        this.setLoading(false);
      }),
      catchError(error => {
        this.handleError('Failed to delete user', error);
        return throwError(() => error);
      })
    );
  }

  selectUser(user: User | null): void {
    this.selectedUserSubject.next(user);
  }

  searchUsers(query: string): Observable<User[]> {
    if (!query.trim()) {
      return this.users$;
    }
    
    const searchTerm = query.toLowerCase();
    return this.users$.pipe(
      map(users => users.filter(user => 
        user.name.toLowerCase().includes(searchTerm) ||
        user.email.toLowerCase().includes(searchTerm)
      ))
    );
  }

  // Private helper methods
  private setLoading(loading: boolean): void {
    this.loadingSubject.next(loading);
  }

  private clearError(): void {
    this.errorSubject.next(null);
  }

  private handleError(message: string, error: any): void {
    console.error(message, error);
    this.setLoading(false);
    this.errorSubject.next(message);
  }

  // Cleanup method
  destroy(): void {
    this.usersSubject.complete();
    this.selectedUserSubject.complete();
    this.loadingSubject.complete();
    this.errorSubject.complete();
  }
}

interface User {
  id: number;
  name: string;
  email: string;
  isActive: boolean;
  role: string;
  lastLogin: Date;
}

interface Post {
  id: number;
  title: string;
  content: string;
  userId: number;
  createdAt: Date;
}
```

### Advanced State Management Service

**Complex State Service with Reducers:**
```typescript
@Injectable({
  providedIn: 'root'
})
export class ShoppingCartService {
  // State interface
  private readonly initialState: CartState = {
    items: [],
    total: 0,
    itemCount: 0,
    discounts: [],
    shippingInfo: null,
    loading: false,
    error: null
  };

  // State subject
  private stateSubject = new BehaviorSubject<CartState>(this.initialState);
  
  // Action subjects for different operations
  private actionsSubject = new Subject<CartAction>();
  
  // Public state observables
  public readonly state$ = this.stateSubject.asObservable();
  public readonly items$ = this.state$.pipe(
    map(state => state.items),
    distinctUntilChanged()
  );
  public readonly total$ = this.state$.pipe(
    map(state => state.total),
    distinctUntilChanged()
  );
  public readonly itemCount$ = this.state$.pipe(
    map(state => state.itemCount),
    distinctUntilChanged()
  );
  public readonly isEmpty$ = this.itemCount$.pipe(
    map(count => count === 0)
  );
  public readonly loading$ = this.state$.pipe(
    map(state => state.loading),
    distinctUntilChanged()
  );
  public readonly error$ = this.state$.pipe(
    map(state => state.error),
    distinctUntilChanged()
  );

  // Derived observables for complex calculations
  public readonly subtotal$ = this.items$.pipe(
    map(items => items.reduce((sum, item) => sum + (item.price * item.quantity), 0))
  );

  public readonly tax$ = this.subtotal$.pipe(
    map(subtotal => subtotal * 0.08) // 8% tax
  );

  public readonly shipping$ = combineLatest([this.subtotal$, this.state$]).pipe(
    map(([subtotal, state]) => {
      if (subtotal >= 50) return 0; // Free shipping over $50
      if (state.shippingInfo?.expedited) return 15;
      return 5;
    })
  );

  public readonly finalTotal$ = combineLatest([
    this.subtotal$,
    this.tax$,
    this.shipping$,
    this.state$
  ]).pipe(
    map(([subtotal, tax, shipping, state]) => {
      const discountAmount = state.discounts.reduce((sum, discount) => sum + discount.amount, 0);
      return subtotal + tax + shipping - discountAmount;
    })
  );

  constructor(
    private http: HttpClient,
    private storageService: StorageService
  ) {
    this.initializeService();
    this.setupActionHandlers();
  }

  private initializeService(): void {
    // Load cart from storage
    const savedCart = this.storageService.getItem('shopping-cart');
    if (savedCart) {
      try {
        const cartData = JSON.parse(savedCart);
        this.updateState({ items: cartData.items || [] });
        this.recalculateCart();
      } catch (error) {
        console.warn('Failed to load saved cart:', error);
      }
    }

    // Auto-save cart changes
    this.state$.pipe(
      distinctUntilChanged((prev, curr) => 
        JSON.stringify(prev.items) === JSON.stringify(curr.items)
      )
    ).subscribe(state => {
      this.storageService.setItem('shopping-cart', JSON.stringify({
        items: state.items,
        timestamp: Date.now()
      }));
    });
  }

  private setupActionHandlers(): void {
    this.actionsSubject.pipe(
      switchMap(action => this.processAction(action))
    ).subscribe();
  }

  private processAction(action: CartAction): Observable<any> {
    switch (action.type) {
      case 'ADD_ITEM':
        return this.handleAddItem(action.payload);
      case 'REMOVE_ITEM':
        return this.handleRemoveItem(action.payload);
      case 'UPDATE_QUANTITY':
        return this.handleUpdateQuantity(action.payload);
      case 'APPLY_DISCOUNT':
        return this.handleApplyDiscount(action.payload);
      case 'SET_SHIPPING':
        return this.handleSetShipping(action.payload);
      case 'CLEAR_CART':
        return this.handleClearCart();
      default:
        return of(null);
    }
  }

  // Public action methods
  addItem(product: Product, quantity: number = 1): Observable<void> {
    return new Observable(subscriber => {
      this.actionsSubject.next({
        type: 'ADD_ITEM',
        payload: { product, quantity }
      });
      subscriber.next();
      subscriber.complete();
    });
  }

  removeItem(productId: string): Observable<void> {
    return new Observable(subscriber => {
      this.actionsSubject.next({
        type: 'REMOVE_ITEM',
        payload: { productId }
      });
      subscriber.next();
      subscriber.complete();
    });
  }

  updateQuantity(productId: string, quantity: number): Observable<void> {
    return new Observable(subscriber => {
      this.actionsSubject.next({
        type: 'UPDATE_QUANTITY',
        payload: { productId, quantity }
      });
      subscriber.next();
      subscriber.complete();
    });
  }

  applyDiscount(discountCode: string): Observable<boolean> {
    this.updateState({ loading: true, error: null });
    
    return this.http.post<DiscountResponse>('/api/cart/discount', { code: discountCode }).pipe(
      tap(response => {
        if (response.valid) {
          this.actionsSubject.next({
            type: 'APPLY_DISCOUNT',
            payload: response.discount
          });
        } else {
          this.updateState({ 
            loading: false, 
            error: 'Invalid discount code' 
          });
        }
      }),
      map(response => response.valid),
      catchError(error => {
        this.updateState({ 
          loading: false, 
          error: 'Failed to apply discount code' 
        });
        return of(false);
      })
    );
  }

  setShippingInfo(shippingInfo: ShippingInfo): void {
    this.actionsSubject.next({
      type: 'SET_SHIPPING',
      payload: shippingInfo
    });
  }

  clearCart(): void {
    this.actionsSubject.next({ type: 'CLEAR_CART' });
  }

  checkout(): Observable<CheckoutResult> {
    this.updateState({ loading: true, error: null });
    
    const checkoutData$ = combineLatest([
      this.items$,
      this.finalTotal$,
      this.state$
    ]).pipe(take(1));

    return checkoutData$.pipe(
      switchMap(([items, total, state]) => {
        const checkoutPayload = {
          items: items.map(item => ({
            productId: item.product.id,
            quantity: item.quantity,
            price: item.price
          })),
          total,
          shippingInfo: state.shippingInfo,
          discounts: state.discounts
        };

        return this.http.post<CheckoutResult>('/api/checkout', checkoutPayload);
      }),
      tap(result => {
        if (result.success) {
          this.clearCart();
        }
        this.updateState({ loading: false });
      }),
      catchError(error => {
        this.updateState({ 
          loading: false, 
          error: 'Checkout failed. Please try again.' 
        });
        return throwError(() => error);
      })
    );
  }

  // Action handlers
  private handleAddItem(payload: { product: Product; quantity: number }): Observable<any> {
    const currentState = this.stateSubject.value;
    const existingItemIndex = currentState.items.findIndex(
      item => item.product.id === payload.product.id
    );

    let newItems: CartItem[];
    if (existingItemIndex !== -1) {
      // Update existing item
      newItems = [...currentState.items];
      newItems[existingItemIndex] = {
        ...newItems[existingItemIndex],
        quantity: newItems[existingItemIndex].quantity + payload.quantity
      };
    } else {
      // Add new item
      const newItem: CartItem = {
        id: this.generateId(),
        product: payload.product,
        quantity: payload.quantity,
        price: payload.product.price,
        addedAt: new Date()
      };
      newItems = [...currentState.items, newItem];
    }

    this.updateState({ items: newItems });
    this.recalculateCart();
    return of(null);
  }

  private handleRemoveItem(payload: { productId: string }): Observable<any> {
    const currentState = this.stateSubject.value;
    const newItems = currentState.items.filter(
      item => item.product.id !== payload.productId
    );
    
    this.updateState({ items: newItems });
    this.recalculateCart();
    return of(null);
  }

  private handleUpdateQuantity(payload: { productId: string; quantity: number }): Observable<any> {
    const currentState = this.stateSubject.value;
    
    if (payload.quantity <= 0) {
      return this.handleRemoveItem({ productId: payload.productId });
    }

    const newItems = currentState.items.map(item =>
      item.product.id === payload.productId
        ? { ...item, quantity: payload.quantity }
        : item
    );

    this.updateState({ items: newItems });
    this.recalculateCart();
    return of(null);
  }

  private handleApplyDiscount(discount: Discount): Observable<any> {
    const currentState = this.stateSubject.value;
    const existingDiscountIndex = currentState.discounts.findIndex(
      d => d.code === discount.code
    );

    let newDiscounts: Discount[];
    if (existingDiscountIndex !== -1) {
      // Replace existing discount
      newDiscounts = [...currentState.discounts];
      newDiscounts[existingDiscountIndex] = discount;
    } else {
      // Add new discount
      newDiscounts = [...currentState.discounts, discount];
    }

    this.updateState({ 
      discounts: newDiscounts, 
      loading: false, 
      error: null 
    });
    this.recalculateCart();
    return of(null);
  }

  private handleSetShipping(shippingInfo: ShippingInfo): Observable<any> {
    this.updateState({ shippingInfo });
    this.recalculateCart();
    return of(null);
  }

  private handleClearCart(): Observable<any> {
    this.updateState({ 
      items: [], 
      discounts: [], 
      shippingInfo: null,
      total: 0,
      itemCount: 0
    });
    return of(null);
  }

  private recalculateCart(): void {
    const currentState = this.stateSubject.value;
    const itemCount = currentState.items.reduce((sum, item) => sum + item.quantity, 0);
    const total = currentState.items.reduce((sum, item) => sum + (item.price * item.quantity), 0);
    
    this.updateState({ itemCount, total });
  }

  private updateState(partialState: Partial<CartState>): void {
    const currentState = this.stateSubject.value;
    const newState = { ...currentState, ...partialState };
    this.stateSubject.next(newState);
  }

  private generateId(): string {
    return Math.random().toString(36).substr(2, 9);
  }

  // Cleanup
  destroy(): void {
    this.stateSubject.complete();
    this.actionsSubject.complete();
  }
}

// Interfaces
interface CartState {
  items: CartItem[];
  total: number;
  itemCount: number;
  discounts: Discount[];
  shippingInfo: ShippingInfo | null;
  loading: boolean;
  error: string | null;
}

interface CartItem {
  id: string;
  product: Product;
  quantity: number;
  price: number;
  addedAt: Date;
}

interface Product {
  id: string;
  name: string;
  price: number;
  imageUrl: string;
  description: string;
}

interface Discount {
  code: string;
  amount: number;
  type: 'fixed' | 'percentage';
  description: string;
}

interface ShippingInfo {
  expedited: boolean;
  address: string;
  city: string;
  zipCode: string;
}

interface CartAction {
  type: string;
  payload?: any;
}

interface DiscountResponse {
  valid: boolean;
  discount?: Discount;
  message?: string;
}

interface CheckoutResult {
  success: boolean;
  orderId?: string;
  error?: string;
}
```

### Data Synchronization Service

**Real-time Data Sync Service:**
```typescript
@Injectable({
  providedIn: 'root'
})
export class DataSyncService {
  private readonly reconnectInterval = 5000;
  private readonly maxReconnectAttempts = 5;
  
  private connectionSubject = new BehaviorSubject<ConnectionState>('disconnected');
  private messagesSubject = new Subject<SyncMessage>();
  private reconnectAttempts = 0;
  private reconnectTimer?: any;

  // Public observables
  public readonly connectionState$ = this.connectionSubject.asObservable();
  public readonly isConnected$ = this.connectionState$.pipe(
    map(state => state === 'connected')
  );
  public readonly messages$ = this.messagesSubject.asObservable();

  // Filtered message streams
  public readonly userUpdates$ = this.messages$.pipe(
    filter(msg => msg.type === 'user_update'),
    map(msg => msg.payload as User)
  );

  public readonly orderUpdates$ = this.messages$.pipe(
    filter(msg => msg.type === 'order_update'),
    map(msg => msg.payload as Order)
  );

  public readonly notifications$ = this.messages$.pipe(
    filter(msg => msg.type === 'notification'),
    map(msg => msg.payload as Notification)
  );

  private websocket?: WebSocket;

  constructor(
    private configService: ConfigService,
    private authService: AuthService,
    private notificationService: NotificationService
  ) {
    this.initializeConnection();
    this.setupConnectionMonitoring();
  }

  private initializeConnection(): void {
    // Wait for authentication before connecting
    this.authService.isAuthenticated$.pipe(
      filter(authenticated => authenticated),
      switchMap(() => this.configService.getWebSocketUrl()),
      take(1)
    ).subscribe(url => {
      this.connect(url);
    });
  }

  private setupConnectionMonitoring(): void {
    // Monitor connection state
    this.connectionState$.pipe(
      distinctUntilChanged()
    ).subscribe(state => {
      console.log('WebSocket connection state:', state);
      
      if (state === 'connected') {
        this.reconnectAttempts = 0;
        this.clearReconnectTimer();
        this.notificationService.showSuccess('Connected to real-time updates');
      } else if (state === 'disconnected') {
        this.scheduleReconnect();
      }
    });

    // Handle page visibility changes
    document.addEventListener('visibilitychange', () => {
      if (document.visibilityState === 'visible') {
        this.ensureConnection();
      }
    });

    // Handle online/offline events
    window.addEventListener('online', () => {
      this.ensureConnection();
    });

    window.addEventListener('offline', () => {
      this.disconnect();
    });
  }

  private connect(url: string): void {
    if (this.websocket?.readyState === WebSocket.OPEN) {
      return;
    }

    this.connectionSubject.next('connecting');

    try {
      this.websocket = new WebSocket(url);
      this.setupWebSocketHandlers();
    } catch (error) {
      console.error('Failed to create WebSocket connection:', error);
      this.connectionSubject.next('disconnected');
    }
  }

  private setupWebSocketHandlers(): void {
    if (!this.websocket) return;

    this.websocket.onopen = () => {
      console.log('WebSocket connected');
      this.connectionSubject.next('connected');
      this.authenticate();
    };

    this.websocket.onmessage = (event) => {
      try {
        const message: SyncMessage = JSON.parse(event.data);
        this.handleMessage(message);
      } catch (error) {
        console.error('Failed to parse WebSocket message:', error);
      }
    };

    this.websocket.onclose = (event) => {
      console.log('WebSocket disconnected:', event.code, event.reason);
      this.connectionSubject.next('disconnected');
    };

    this.websocket.onerror = (error) => {
      console.error('WebSocket error:', error);
      this.connectionSubject.next('error');
    };
  }

  private authenticate(): void {
    const token = this.authService.getToken();
    if (token && this.websocket?.readyState === WebSocket.OPEN) {
      this.send({
        type: 'authenticate',
        payload: { token }
      });
    }
  }

  private handleMessage(message: SyncMessage): void {
    switch (message.type) {
      case 'authenticated':
        console.log('WebSocket authentication successful');
        this.subscribeToUserChannels();
        break;
      
      case 'authentication_failed':
        console.error('WebSocket authentication failed');
        this.disconnect();
        this.authService.logout();
        break;
      
      case 'ping':
        this.send({ type: 'pong' });
        break;
      
      default:
        this.messagesSubject.next(message);
    }
  }

  private subscribeToUserChannels(): void {
    const userId = this.authService.getCurrentUserId();
    if (userId) {
      this.send({
        type: 'subscribe',
        payload: {
          channels: [
            `user.${userId}`,
            'global_notifications',
            'system_updates'
          ]
        }
      });
    }
  }

  private send(message: any): void {
    if (this.websocket?.readyState === WebSocket.OPEN) {
      this.websocket.send(JSON.stringify(message));
    } else {
      console.warn('Cannot send message: WebSocket not connected');
    }
  }

  private disconnect(): void {
    if (this.websocket) {
      this.websocket.close();
      this.websocket = undefined;
    }
    this.connectionSubject.next('disconnected');
  }

  private ensureConnection(): void {
    if (!this.websocket || this.websocket.readyState !== WebSocket.OPEN) {
      this.configService.getWebSocketUrl().pipe(
        take(1)
      ).subscribe(url => {
        this.connect(url);
      });
    }
  }

  private scheduleReconnect(): void {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      console.log('Max reconnection attempts reached');
      this.notificationService.showError('Unable to connect to real-time updates');
      return;
    }

    this.clearReconnectTimer();
    
    const delay = Math.min(
      this.reconnectInterval * Math.pow(2, this.reconnectAttempts),
      30000 // Max 30 seconds
    );

    console.log(`Scheduling reconnection attempt ${this.reconnectAttempts + 1} in ${delay}ms`);
    
    this.reconnectTimer = setTimeout(() => {
      this.reconnectAttempts++;
      this.ensureConnection();
    }, delay);
  }

  private clearReconnectTimer(): void {
    if (this.reconnectTimer) {
      clearTimeout(this.reconnectTimer);
      this.reconnectTimer = undefined;
    }
  }

  // Public methods
  subscribeToChannel(channel: string): Observable<any> {
    return this.isConnected$.pipe(
      filter(connected => connected),
      take(1),
      tap(() => {
        this.send({
          type: 'subscribe',
          payload: { channels: [channel] }
        });
      }),
      switchMap(() => 
        this.messages$.pipe(
          filter(msg => msg.channel === channel),
          map(msg => msg.payload)
        )
      )
    );
  }

  unsubscribeFromChannel(channel: string): void {
    this.send({
      type: 'unsubscribe',
      payload: { channels: [channel] }
    });
  }

  sendMessage(type: string, payload: any, channel?: string): void {
    this.send({
      type,
      payload,
      channel
    });
  }

  // Cleanup
  destroy(): void {
    this.clearReconnectTimer();
    this.disconnect();
    this.connectionSubject.complete();
    this.messagesSubject.complete();
  }
}

type ConnectionState = 'disconnected' | 'connecting' | 'connected' | 'error';

interface SyncMessage {
  type: string;
  payload?: any;
  channel?: string;
  timestamp?: string;
}

interface Order {
  id: string;
  status: string;
  total: number;
  items: any[];
}

interface Notification {
  id: string;
  title: string;
  message: string;
  type: 'info' | 'success' | 'warning' | 'error';
  timestamp: Date;
}
```

## Service Communication Patterns

### Inter-Service Communication

**Service Coordination Hub:**
```typescript
@Injectable({
  providedIn: 'root'
})
export class ServiceCoordinator {
  private eventBus = new Subject<ServiceEvent>();
  
  // Event streams by type
  public readonly events$ = this.eventBus.asObservable();
  public readonly userEvents$ = this.events$.pipe(
    filter(event => event.source === 'user')
  );
  public readonly orderEvents$ = this.events$.pipe(
    filter(event => event.source === 'order')
  );
  public readonly cartEvents$ = this.events$.pipe(
    filter(event => event.source === 'cart')
  );

  constructor(
    private userService: UserService,
    private orderService: OrderService,
    private cartService: ShoppingCartService,
    private notificationService: NotificationService,
    private analyticsService: AnalyticsService
  ) {
    this.setupEventHandlers();
  }

  private setupEventHandlers(): void {
    // User-related event handling
    this.userEvents$.pipe(
      filter(event => event.type === 'user_logged_in')
    ).subscribe(event => {
      this.handleUserLogin(event.payload);
    });

    this.userEvents$.pipe(
      filter(event => event.type === 'user_profile_updated')
    ).subscribe(event => {
      this.handleUserProfileUpdate(event.payload);
    });

    // Order-related event handling
    this.orderEvents$.pipe(
      filter(event => event.type === 'order_created')
    ).subscribe(event => {
      this.handleOrderCreated(event.payload);
    });

    this.orderEvents$.pipe(
      filter(event => event.type === 'order_status_changed')
    ).subscribe(event => {
      this.handleOrderStatusChange(event.payload);
    });

    // Cart-related event handling
    this.cartEvents$.pipe(
      filter(event => event.type === 'item_added_to_cart')
    ).subscribe(event => {
      this.handleCartItemAdded(event.payload);
    });
  }

  // Event publishing methods
  publishEvent(source: string, type: string, payload: any): void {
    const event: ServiceEvent = {
      id: this.generateEventId(),
      source,
      type,
      payload,
      timestamp: new Date()
    };
    
    this.eventBus.next(event);
  }

  // Event handlers
  private handleUserLogin(payload: { user: User }): void {
    // Load user's cart from server
    this.cartService.loadUserCart(payload.user.id);
    
    // Update analytics
    this.analyticsService.trackUserLogin(payload.user.id);
    
    // Show welcome notification
    this.notificationService.showSuccess(`Welcome back, ${payload.user.name}!`);
  }

  private handleUserProfileUpdate(payload: { user: User }): void {
    // Sync profile changes across services
    this.orderService.updateUserInfo(payload.user);
    this.cartService.updateUserInfo(payload.user);
    
    // Track profile update
    this.analyticsService.trackProfileUpdate(payload.user.id);
  }

  private handleOrderCreated(payload: { order: Order }): void {
    // Clear cart after successful order
    this.cartService.clearCart();
    
    // Send confirmation notification
    this.notificationService.showSuccess('Order placed successfully!');
    
    // Track order creation
    this.analyticsService.trackOrderCreated(payload.order);
    
    // Update user's order history
    this.userService.refreshUserOrders(payload.order.userId);
  }

  private handleOrderStatusChange(payload: { order: Order; previousStatus: string }): void {
    // Send status update notification
    const message = this.getOrderStatusMessage(payload.order.status);
    this.notificationService.showInfo(message);
    
    // Track status change
    this.analyticsService.trackOrderStatusChange(payload.order);
    
    // Update real-time order tracking
    this.publishEvent('tracking', 'order_status_updated', payload);
  }

  private handleCartItemAdded(payload: { item: CartItem; user: User }): void {
    // Track add to cart event
    this.analyticsService.trackAddToCart(payload.item, payload.user);
    
    // Check for related products
    this.publishEvent('recommendations', 'suggest_related_products', {
      productId: payload.item.product.id,
      userId: payload.user.id
    });
  }

  private getOrderStatusMessage(status: string): string {
    const messages = {
      'confirmed': 'Your order has been confirmed',
      'processing': 'Your order is being processed',
      'shipped': 'Your order has been shipped',
      'delivered': 'Your order has been delivered',
      'cancelled': 'Your order has been cancelled'
    };
    
    return messages[status as keyof typeof messages] || 'Order status updated';
  }

  private generateEventId(): string {
    return `event_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }

  // Cleanup
  destroy(): void {
    this.eventBus.complete();
  }
}

interface ServiceEvent {
  id: string;
  source: string;
  type: string;
  payload: any;
  timestamp: Date;
}
```

### Service Factory Pattern

**Dynamic Service Creation:**
```typescript
@Injectable({
  providedIn: 'root'
})
export class ServiceFactory {
  private serviceCache = new Map<string, any>();
  private configCache = new Map<string, ServiceConfig>();

  constructor(
    private injector: Injector,
    private http: HttpClient
  ) {}

  // Create service instance based on configuration
  createService<T>(serviceType: string, config: ServiceConfig): Observable<T> {
    const cacheKey = `${serviceType}_${JSON.stringify(config)}`;
    
    if (this.serviceCache.has(cacheKey)) {
      return of(this.serviceCache.get(cacheKey));
    }

    return this.loadServiceConfig(serviceType).pipe(
      map(serviceConfig => {
        const mergedConfig = { ...serviceConfig, ...config };
        const service = this.instantiateService<T>(serviceType, mergedConfig);
        this.serviceCache.set(cacheKey, service);
        return service;
      })
    );
  }

  // Create API service with specific configuration
  createApiService(apiName: string, baseUrl: string, options?: ApiServiceOptions): Observable<ApiService> {
    const config: ServiceConfig = {
      type: 'api',
      baseUrl,
      ...options
    };

    return this.createService<ApiService>('api', config).pipe(
      map(service => this.configureApiService(service as ApiService, apiName, config))
    );
  }

  // Create data service with caching
  createDataService<T>(entityType: string, options?: DataServiceOptions): Observable<DataService<T>> {
    const config: ServiceConfig = {
      type: 'data',
      entityType,
      ...options
    };

    return this.createService<DataService<T>>('data', config);
  }

  private loadServiceConfig(serviceType: string): Observable<ServiceConfig> {
    if (this.configCache.has(serviceType)) {
      return of(this.configCache.get(serviceType)!);
    }

    return this.http.get<ServiceConfig>(`/api/config/services/${serviceType}`).pipe(
      tap(config => this.configCache.set(serviceType, config)),
      catchError(() => of(this.getDefaultConfig(serviceType)))
    );
  }

  private instantiateService<T>(serviceType: string, config: ServiceConfig): T {
    switch (serviceType) {
      case 'api':
        return new ApiService(this.injector.get(HttpClient), config) as unknown as T;
      case 'data':
        return new DataService(this.injector.get(HttpClient), config) as unknown as T;
      case 'websocket':
        return new WebSocketService(config) as unknown as T;
      default:
        throw new Error(`Unknown service type: ${serviceType}`);
    }
  }

  private configureApiService(service: ApiService, apiName: string, config: ServiceConfig): ApiService {
    // Configure authentication
    if (config.requiresAuth) {
      const authService = this.injector.get(AuthService);
      service.setAuthProvider(() => authService.getToken());
    }

    // Configure error handling
    service.setErrorHandler((error: any) => {
      console.error(`API Error in ${apiName}:`, error);
      return throwError(() => error);
    });

    // Configure request interceptors
    if (config.interceptors) {
      config.interceptors.forEach(interceptor => {
        service.addInterceptor(interceptor);
      });
    }

    return service;
  }

  private getDefaultConfig(serviceType: string): ServiceConfig {
    const defaults: Record<string, ServiceConfig> = {
      api: {
        type: 'api',
        timeout: 30000,
        retries: 3,
        requiresAuth: true
      },
      data: {
        type: 'data',
        cacheTimeout: 300000, // 5 minutes
        enableOffline: true
      },
      websocket: {
        type: 'websocket',
        reconnectInterval: 5000,
        maxReconnectAttempts: 5
      }
    };

    return defaults[serviceType] || {};
  }

  // Clear service cache
  clearCache(): void {
    this.serviceCache.clear();
    this.configCache.clear();
  }

  // Get cached service
  getCachedService<T>(serviceType: string, config: ServiceConfig): T | null {
    const cacheKey = `${serviceType}_${JSON.stringify(config)}`;
    return this.serviceCache.get(cacheKey) || null;
  }
}

// Generic API Service
export class ApiService {
  private authProvider?: () => string | null;
  private errorHandler?: (error: any) => Observable<never>;
  private interceptors: RequestInterceptor[] = [];

  constructor(
    private http: HttpClient,
    private config: ServiceConfig
  ) {}

  setAuthProvider(provider: () => string | null): void {
    this.authProvider = provider;
  }

  setErrorHandler(handler: (error: any) => Observable<never>): void {
    this.errorHandler = handler;
  }

  addInterceptor(interceptor: RequestInterceptor): void {
    this.interceptors.push(interceptor);
  }

  get<T>(endpoint: string, options?: any): Observable<T> {
    return this.request<T>('GET', endpoint, null, options);
  }

  post<T>(endpoint: string, data: any, options?: any): Observable<T> {
    return this.request<T>('POST', endpoint, data, options);
  }

  put<T>(endpoint: string, data: any, options?: any): Observable<T> {
    return this.request<T>('PUT', endpoint, data, options);
  }

  delete<T>(endpoint: string, options?: any): Observable<T> {
    return this.request<T>('DELETE', endpoint, null, options);
  }

  private request<T>(method: string, endpoint: string, data?: any, options?: any): Observable<T> {
    const url = `${this.config.baseUrl}${endpoint}`;
    let headers = new HttpHeaders(options?.headers || {});

    // Apply authentication
    if (this.config.requiresAuth && this.authProvider) {
      const token = this.authProvider();
      if (token) {
        headers = headers.set('Authorization', `Bearer ${token}`);
      }
    }

    // Apply interceptors
    let requestOptions = { ...options, headers };
    this.interceptors.forEach(interceptor => {
      requestOptions = interceptor(requestOptions);
    });

    // Make request
    let request$: Observable<T>;
    switch (method.toUpperCase()) {
      case 'GET':
        request$ = this.http.get<T>(url, requestOptions);
        break;
      case 'POST':
        request$ = this.http.post<T>(url, data, requestOptions);
        break;
      case 'PUT':
        request$ = this.http.put<T>(url, data, requestOptions);
        break;
      case 'DELETE':
        request$ = this.http.delete<T>(url, requestOptions);
        break;
      default:
        throw new Error(`Unsupported HTTP method: ${method}`);
    }

    // Apply timeout and error handling
    return request$.pipe(
      timeout(this.config.timeout || 30000),
      retry(this.config.retries || 0),
      catchError(error => this.errorHandler ? this.errorHandler(error) : throwError(() => error))
    );
  }
}

// Generic Data Service
export class DataService<T> {
  private cache = new Map<string, { data: T; timestamp: number }>();

  constructor(
    private http: HttpClient,
    private config: ServiceConfig
  ) {}

  getAll(): Observable<T[]> {
    return this.getCachedOrFetch<T[]>('all', `/api/${this.config.entityType}`);
  }

  getById(id: string): Observable<T> {
    return this.getCachedOrFetch<T>(id, `/api/${this.config.entityType}/${id}`);
  }

  create(item: Partial<T>): Observable<T> {
    return this.http.post<T>(`/api/${this.config.entityType}`, item).pipe(
      tap(() => this.invalidateCache())
    );
  }

  update(id: string, updates: Partial<T>): Observable<T> {
    return this.http.put<T>(`/api/${this.config.entityType}/${id}`, updates).pipe(
      tap(() => this.invalidateCache())
    );
  }

  delete(id: string): Observable<void> {
    return this.http.delete<void>(`/api/${this.config.entityType}/${id}`).pipe(
      tap(() => this.invalidateCache())
    );
  }

  private getCachedOrFetch<K>(key: string, url: string): Observable<K> {
    const cached = this.cache.get(key);
    const now = Date.now();
    
    if (cached && (now - cached.timestamp) < (this.config.cacheTimeout || 300000)) {
      return of(cached.data as K);
    }

    return this.http.get<K>(url).pipe(
      tap(data => {
        this.cache.set(key, { data: data as any, timestamp: now });
      })
    );
  }

  private invalidateCache(): void {
    this.cache.clear();
  }
}

interface ServiceConfig {
  type: string;
  baseUrl?: string;
  timeout?: number;
  retries?: number;
  requiresAuth?: boolean;
  entityType?: string;
  cacheTimeout?: number;
  enableOffline?: boolean;
  reconnectInterval?: number;
  maxReconnectAttempts?: number;
  interceptors?: RequestInterceptor[];
}

interface ApiServiceOptions {
  timeout?: number;
  retries?: number;
  requiresAuth?: boolean;
  interceptors?: RequestInterceptor[];
}

interface DataServiceOptions {
  cacheTimeout?: number;
  enableOffline?: boolean;
}

type RequestInterceptor = (options: any) => any;
```

## Testing Reactive Services

### Service Testing Patterns

**Comprehensive Service Testing:**
```typescript
describe('UserService', () => {
  let service: UserService;
  let httpMock: HttpTestingController;
  let mockUsers: User[];

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [UserService]
    });

    service = TestBed.inject(UserService);
    httpMock = TestBed.inject(HttpTestingController);
    
    mockUsers = [
      { id: 1, name: 'John Doe', email: 'john@example.com', isActive: true, role: 'user', lastLogin: new Date() },
      { id: 2, name: 'Jane Smith', email: 'jane@example.com', isActive: true, role: 'admin', lastLogin: new Date() }
    ];
  });

  afterEach(() => {
    httpMock.verify();
  });

  describe('loadUsers', () => {
    it('should load users and update state', () => {
      let users: User[] = [];
      let loading = false;
      let error: string | null = null;

      // Subscribe to observables
      service.users$.subscribe(u => users = u);
      service.loading$.subscribe(l => loading = l);
      service.error$.subscribe(e => error = e);

      // Trigger load
      service.loadUsers().subscribe();

      // Initially loading should be true
      expect(loading).toBe(true);
      expect(error).toBe(null);

      // Mock HTTP response
      const req = httpMock.expectOne('/api/users');
      expect(req.request.method).toBe('GET');
      req.flush(mockUsers);

      // After response, state should be updated
      expect(users).toEqual(mockUsers);
      expect(loading).toBe(false);
      expect(error).toBe(null);
    });

    it('should handle load error', () => {
      let users: User[] = [];
      let loading = false;
      let error: string | null = null;

      service.users$.subscribe(u => users = u);
      service.loading$.subscribe(l => loading = l);
      service.error$.subscribe(e => error = e);

      service.loadUsers().subscribe({
        error: () => {} // Handle error
      });

      const req = httpMock.expectOne('/api/users');
      req.error(new ErrorEvent('Network error'));

      expect(users).toEqual([]);
      expect(loading).toBe(false);
      expect(error).toBe('Failed to load users');
    });
  });

  describe('createUser', () => {
    it('should create user and add to list', () => {
      const newUser = { name: 'New User', email: 'new@example.com' };
      const createdUser: User = { 
        id: 3, 
        name: 'New User', 
        email: 'new@example.com', 
        isActive: true, 
        role: 'user', 
        lastLogin: new Date() 
      };

      let users: User[] = [];
      service.users$.subscribe(u => users = u);

      // Set initial users
      service.loadUsers().subscribe();
      httpMock.expectOne('/api/users').flush(mockUsers);

      // Create new user
      service.createUser(newUser).subscribe();
      
      const req = httpMock.expectOne('/api/users');
      expect(req.request.method).toBe('POST');
      expect(req.request.body).toEqual(newUser);
      req.flush(createdUser);

      expect(users).toContain(createdUser);
      expect(users.length).toBe(3);
    });
  });

  describe('searchUsers', () => {
    it('should filter users by search term', () => {
      // Set up initial data
      service.loadUsers().subscribe();
      httpMock.expectOne('/api/users').flush(mockUsers);

      // Test search
      service.searchUsers('john').subscribe(results => {
        expect(results.length).toBe(1);
        expect(results[0].name).toBe('John Doe');
      });

      service.searchUsers('example.com').subscribe(results => {
        expect(results.length).toBe(2);
      });

      service.searchUsers('').subscribe(results => {
        expect(results.length).toBe(2);
      });
    });
  });

  describe('derived observables', () => {
    beforeEach(() => {
      service.loadUsers().subscribe();
      httpMock.expectOne('/api/users').flush(mockUsers);
    });

    it('should calculate user count', () => {
      service.userCount$.subscribe(count => {
        expect(count).toBe(2);
      });
    });

    it('should filter active users', () => {
      service.activeUsers$.subscribe(activeUsers => {
        expect(activeUsers.length).toBe(2);
        expect(activeUsers.every(user => user.isActive)).toBe(true);
      });
    });

    it('should load selected user posts', () => {
      const mockPosts = [
        { id: 1, title: 'Post 1', content: 'Content 1', userId: 1, createdAt: new Date() }
      ];

      // Select a user
      service.selectUser(mockUsers[0]);

      // Subscribe to posts
      service.selectedUserPosts$.subscribe(posts => {
        expect(posts).toEqual(mockPosts);
      });

      // Mock the posts request
      const req = httpMock.expectOne('/api/users/1/posts');
      req.flush(mockPosts);
    });
  });
});
```

## Best Practices

### 1. Service Architecture
```typescript
// ✅ Good: Clear separation of concerns
@Injectable({
  providedIn: 'root'
})
export class WellStructuredService {
  // Private state
  private state = new BehaviorSubject(initialState);
  
  // Public observables
  public readonly data$ = this.state.pipe(map(s => s.data));
  public readonly loading$ = this.state.pipe(map(s => s.loading));
  
  // Clear API
  public loadData(): Observable<Data> { /* ... */ }
  public updateData(updates: Partial<Data>): Observable<Data> { /* ... */ }
}
```

### 2. Error Handling
```typescript
// ✅ Good: Comprehensive error handling
private handleError(operation: string, error: any): void {
  console.error(`${operation} failed:`, error);
  this.updateState({ 
    loading: false, 
    error: this.getErrorMessage(error) 
  });
}
```

### 3. Memory Management
```typescript
// ✅ Good: Proper cleanup
destroy(): void {
  this.stateSubject.complete();
  this.actionsSubject.complete();
}
```

### 4. Type Safety
```typescript
// ✅ Good: Strong typing
interface ServiceState {
  data: Data[];
  loading: boolean;
  error: string | null;
}
```

## Summary

Building reactive services with RxJS requires:

- **Clear State Management**: Use BehaviorSubject for state, expose as observables
- **Comprehensive Error Handling**: Handle all error scenarios gracefully  
- **Derived Observables**: Create computed values from base state
- **Service Coordination**: Use event buses for inter-service communication
- **Factory Patterns**: Create configurable, reusable services
- **Testing Strategy**: Test all observable streams and state changes
- **Memory Management**: Always clean up subscriptions and subjects

Key principles:
- Expose state as observables, never as direct access
- Use proper TypeScript interfaces for all data structures
- Implement comprehensive error handling and loading states
- Follow single responsibility principle
- Use dependency injection effectively
- Write comprehensive tests for all observable streams
