# Real-time Notification Patterns

## Learning Objectives
By the end of this lesson, you will be able to:
- Build comprehensive notification systems using RxJS
- Implement push notifications with service workers
- Create in-app notification management with queuing and prioritization
- Handle notification permissions and user preferences
- Build notification templates and customizable alerts
- Implement notification analytics and tracking
- Create cross-platform notification strategies

## Table of Contents
1. [Notification System Architecture](#notification-system-architecture)
2. [In-App Notification Service](#in-app-notification-service)
3. [Push Notification Integration](#push-notification-integration)
4. [Notification Templates & Rendering](#notification-templates--rendering)
5. [User Preferences & Permissions](#user-preferences--permissions)
6. [Notification Analytics & Tracking](#notification-analytics--tracking)
7. [Cross-Platform Strategies](#cross-platform-strategies)
8. [Advanced Notification Patterns](#advanced-notification-patterns)

## Notification System Architecture

### Core Notification Infrastructure

```typescript
// notification-system.service.ts
import { Injectable, inject } from '@angular/core';
import { 
  BehaviorSubject, 
  Observable, 
  Subject, 
  merge, 
  timer,
  fromEvent,
  of,
  throwError
} from 'rxjs';
import { 
  filter, 
  map, 
  tap, 
  take, 
  takeUntil, 
  switchMap,
  debounceTime,
  distinctUntilChanged,
  scan,
  shareReplay
} from 'rxjs/operators';

export interface Notification {
  id: string;
  type: NotificationType;
  title: string;
  message: string;
  category: NotificationCategory;
  priority: NotificationPriority;
  data?: any;
  actions?: NotificationAction[];
  timestamp: Date;
  expiresAt?: Date;
  isRead: boolean;
  isPersistent: boolean;
  source: NotificationSource;
  userId?: string;
  groupId?: string;
}

export interface NotificationAction {
  id: string;
  label: string;
  action: string;
  style?: 'primary' | 'secondary' | 'danger';
  icon?: string;
}

export type NotificationType = 
  | 'info' 
  | 'success' 
  | 'warning' 
  | 'error' 
  | 'system' 
  | 'user' 
  | 'marketing';

export type NotificationCategory = 
  | 'chat' 
  | 'system' 
  | 'order' 
  | 'security' 
  | 'reminder' 
  | 'news' 
  | 'social';

export type NotificationPriority = 'low' | 'normal' | 'high' | 'urgent';

export type NotificationSource = 
  | 'websocket' 
  | 'push' 
  | 'local' 
  | 'api' 
  | 'system';

export interface NotificationState {
  notifications: Notification[];
  unreadCount: number;
  lastUpdated: Date;
  isLoading: boolean;
}

@Injectable({
  providedIn: 'root'
})
export class NotificationSystemService {
  private stateSubject = new BehaviorSubject<NotificationState>({
    notifications: [],
    unreadCount: 0,
    lastUpdated: new Date(),
    isLoading: false
  });

  private newNotificationSubject = new Subject<Notification>();
  private actionSubject = new Subject<{ notificationId: string; actionId: string }>();
  private destroySubject = new Subject<void>();

  // Public observables
  public state$ = this.stateSubject.asObservable();
  public notifications$ = this.state$.pipe(map(state => state.notifications));
  public unreadCount$ = this.state$.pipe(map(state => state.unreadCount));
  public newNotification$ = this.newNotificationSubject.asObservable();
  public notificationActions$ = this.actionSubject.asObservable();

  // Filtered notification streams
  public urgentNotifications$ = this.notifications$.pipe(
    map(notifications => notifications.filter(n => n.priority === 'urgent'))
  );

  public unreadNotifications$ = this.notifications$.pipe(
    map(notifications => notifications.filter(n => !n.isRead))
  );

  constructor(
    private webSocketService: WebSocketService,
    private pushNotificationService: PushNotificationService,
    private storageService: StorageService
  ) {
    this.initializeNotificationStreams();
    this.loadPersistedNotifications();
    this.setupAutoCleanup();
  }

  // Add notification
  addNotification(notification: Omit<Notification, 'id' | 'timestamp' | 'isRead'>): Observable<Notification> {
    const fullNotification: Notification = {
      ...notification,
      id: this.generateNotificationId(),
      timestamp: new Date(),
      isRead: false
    };

    this.updateState(state => ({
      ...state,
      notifications: [...state.notifications, fullNotification],
      unreadCount: state.unreadCount + 1,
      lastUpdated: new Date()
    }));

    // Emit new notification event
    this.newNotificationSubject.next(fullNotification);

    // Persist if required
    if (fullNotification.isPersistent) {
      this.persistNotification(fullNotification);
    }

    // Auto-expire if configured
    if (fullNotification.expiresAt) {
      this.scheduleExpiration(fullNotification);
    }

    return of(fullNotification);
  }

  // Mark as read
  markAsRead(notificationId: string): Observable<void> {
    this.updateState(state => {
      const notifications = state.notifications.map(n =>
        n.id === notificationId ? { ...n, isRead: true } : n
      );
      
      const unreadCount = notifications.filter(n => !n.isRead).length;

      return {
        ...state,
        notifications,
        unreadCount,
        lastUpdated: new Date()
      };
    });

    return of(void 0);
  }

  // Mark all as read
  markAllAsRead(): Observable<void> {
    this.updateState(state => ({
      ...state,
      notifications: state.notifications.map(n => ({ ...n, isRead: true })),
      unreadCount: 0,
      lastUpdated: new Date()
    }));

    return of(void 0);
  }

  // Remove notification
  removeNotification(notificationId: string): Observable<void> {
    this.updateState(state => {
      const notifications = state.notifications.filter(n => n.id !== notificationId);
      const unreadCount = notifications.filter(n => !n.isRead).length;

      return {
        ...state,
        notifications,
        unreadCount,
        lastUpdated: new Date()
      };
    });

    this.removePersistedNotification(notificationId);
    return of(void 0);
  }

  // Clear all notifications
  clearAll(): Observable<void> {
    this.updateState(state => ({
      ...state,
      notifications: [],
      unreadCount: 0,
      lastUpdated: new Date()
    }));

    this.clearPersistedNotifications();
    return of(void 0);
  }

  // Handle notification action
  executeAction(notificationId: string, actionId: string): Observable<void> {
    const notification = this.findNotification(notificationId);
    if (!notification) {
      return throwError(() => new Error('Notification not found'));
    }

    const action = notification.actions?.find(a => a.id === actionId);
    if (!action) {
      return throwError(() => new Error('Action not found'));
    }

    // Emit action event
    this.actionSubject.next({ notificationId, actionId });

    // Auto-mark as read when action is taken
    this.markAsRead(notificationId).subscribe();

    return of(void 0);
  }

  // Get notifications by category
  getNotificationsByCategory(category: NotificationCategory): Observable<Notification[]> {
    return this.notifications$.pipe(
      map(notifications => notifications.filter(n => n.category === category))
    );
  }

  // Get notifications by priority
  getNotificationsByPriority(priority: NotificationPriority): Observable<Notification[]> {
    return this.notifications$.pipe(
      map(notifications => notifications.filter(n => n.priority === priority))
    );
  }

  private initializeNotificationStreams(): void {
    // WebSocket notifications
    this.webSocketService.messages$.pipe(
      filter(message => message.type === 'notification'),
      map(message => message.payload as Notification),
      takeUntil(this.destroySubject)
    ).subscribe(notification => {
      this.addNotification({
        ...notification,
        source: 'websocket'
      }).subscribe();
    });

    // Push notifications
    this.pushNotificationService.notifications$.pipe(
      takeUntil(this.destroySubject)
    ).subscribe(notification => {
      this.addNotification({
        ...notification,
        source: 'push'
      }).subscribe();
    });
  }

  private loadPersistedNotifications(): void {
    const stored = this.storageService.getItem('notifications');
    if (stored) {
      const notifications: Notification[] = JSON.parse(stored);
      this.updateState(state => ({
        ...state,
        notifications,
        unreadCount: notifications.filter(n => !n.isRead).length
      }));
    }
  }

  private persistNotification(notification: Notification): void {
    const currentState = this.stateSubject.value;
    const persistentNotifications = currentState.notifications.filter(n => n.isPersistent);
    this.storageService.setItem('notifications', JSON.stringify(persistentNotifications));
  }

  private removePersistedNotification(notificationId: string): void {
    const currentState = this.stateSubject.value;
    const remainingNotifications = currentState.notifications.filter(
      n => n.id !== notificationId && n.isPersistent
    );
    this.storageService.setItem('notifications', JSON.stringify(remainingNotifications));
  }

  private clearPersistedNotifications(): void {
    this.storageService.removeItem('notifications');
  }

  private scheduleExpiration(notification: Notification): void {
    if (!notification.expiresAt) return;

    const expirationTime = notification.expiresAt.getTime() - Date.now();
    if (expirationTime > 0) {
      timer(expirationTime).subscribe(() => {
        this.removeNotification(notification.id).subscribe();
      });
    }
  }

  private setupAutoCleanup(): void {
    // Clean up old notifications every hour
    timer(0, 3600000).pipe(
      takeUntil(this.destroySubject)
    ).subscribe(() => {
      this.cleanupExpiredNotifications();
    });
  }

  private cleanupExpiredNotifications(): void {
    const now = new Date();
    this.updateState(state => {
      const validNotifications = state.notifications.filter(n => 
        !n.expiresAt || n.expiresAt > now
      );
      
      return {
        ...state,
        notifications: validNotifications,
        unreadCount: validNotifications.filter(n => !n.isRead).length
      };
    });
  }

  private updateState(updateFn: (state: NotificationState) => NotificationState): void {
    const currentState = this.stateSubject.value;
    const newState = updateFn(currentState);
    this.stateSubject.next(newState);
  }

  private findNotification(id: string): Notification | undefined {
    return this.stateSubject.value.notifications.find(n => n.id === id);
  }

  private generateNotificationId(): string {
    return `notification-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }

  ngOnDestroy(): void {
    this.destroySubject.next();
    this.destroySubject.complete();
  }
}
```

## In-App Notification Service

### Advanced In-App Notifications

```typescript
// in-app-notification.service.ts
@Injectable({
  providedIn: 'root'
})
export class InAppNotificationService {
  private toastQueue: ToastNotification[] = [];
  private activeToasts$ = new BehaviorSubject<ToastNotification[]>([]);
  private bannerNotification$ = new BehaviorSubject<BannerNotification | null>(null);
  
  // Configuration
  private readonly config = {
    maxToasts: 5,
    defaultDuration: 5000,
    positionClasses: {
      'top-right': 'toast-top-right',
      'top-left': 'toast-top-left',
      'bottom-right': 'toast-bottom-right',
      'bottom-left': 'toast-bottom-left',
      'center': 'toast-center'
    }
  };

  public toasts$ = this.activeToasts$.asObservable();
  public banner$ = this.bannerNotification$.asObservable();

  constructor(
    private notificationSystem: NotificationSystemService,
    private animationService: AnimationService
  ) {
    this.setupNotificationHandlers();
  }

  // Show toast notification
  showToast(toast: Partial<ToastNotification>): Observable<string> {
    const fullToast: ToastNotification = {
      id: this.generateId(),
      type: toast.type || 'info',
      title: toast.title || '',
      message: toast.message || '',
      duration: toast.duration ?? this.config.defaultDuration,
      position: toast.position || 'top-right',
      actions: toast.actions || [],
      isCloseable: toast.isCloseable ?? true,
      isPersistent: toast.isPersistent || false,
      timestamp: new Date()
    };

    this.addToastToQueue(fullToast);
    this.processToastQueue();

    return of(fullToast.id);
  }

  // Show banner notification
  showBanner(banner: Partial<BannerNotification>): Observable<string> {
    const fullBanner: BannerNotification = {
      id: this.generateId(),
      type: banner.type || 'info',
      title: banner.title || '',
      message: banner.message || '',
      actions: banner.actions || [],
      isCloseable: banner.isCloseable ?? true,
      isDismissible: banner.isDismissible ?? true,
      timestamp: new Date()
    };

    this.bannerNotification$.next(fullBanner);

    return of(fullBanner.id);
  }

  // Close toast
  closeToast(toastId: string): Observable<void> {
    const currentToasts = this.activeToasts$.value;
    const toast = currentToasts.find(t => t.id === toastId);
    
    if (toast) {
      return this.animationService.fadeOut(toastId).pipe(
        tap(() => {
          const updatedToasts = currentToasts.filter(t => t.id !== toastId);
          this.activeToasts$.next(updatedToasts);
          this.processToastQueue();
        }),
        map(() => void 0)
      );
    }

    return of(void 0);
  }

  // Close banner
  closeBanner(): Observable<void> {
    const banner = this.bannerNotification$.value;
    if (banner) {
      return this.animationService.slideUp(banner.id).pipe(
        tap(() => this.bannerNotification$.next(null)),
        map(() => void 0)
      );
    }
    return of(void 0);
  }

  // Close all toasts
  closeAllToasts(): Observable<void> {
    const currentToasts = this.activeToasts$.value;
    const closeAnimations = currentToasts.map(toast => 
      this.animationService.fadeOut(toast.id)
    );

    if (closeAnimations.length === 0) {
      return of(void 0);
    }

    return merge(...closeAnimations).pipe(
      tap(() => {
        this.activeToasts$.next([]);
        this.toastQueue = [];
      }),
      map(() => void 0)
    );
  }

  // Handle toast action
  handleToastAction(toastId: string, actionId: string): Observable<void> {
    const toast = this.activeToasts$.value.find(t => t.id === toastId);
    if (!toast) {
      return throwError(() => new Error('Toast not found'));
    }

    const action = toast.actions.find(a => a.id === actionId);
    if (!action) {
      return throwError(() => new Error('Action not found'));
    }

    // Execute action
    this.executeToastAction(action, toast);

    // Auto-close unless persistent
    if (!toast.isPersistent) {
      return this.closeToast(toastId);
    }

    return of(void 0);
  }

  // Smart notification display
  showSmartNotification(notification: Notification): Observable<void> {
    // Determine best display method based on priority and type
    if (notification.priority === 'urgent') {
      return this.showBanner({
        type: this.mapNotificationTypeToDisplay(notification.type),
        title: notification.title,
        message: notification.message,
        actions: notification.actions,
        isCloseable: true
      }).pipe(map(() => void 0));
    } else {
      return this.showToast({
        type: this.mapNotificationTypeToDisplay(notification.type),
        title: notification.title,
        message: notification.message,
        actions: notification.actions,
        duration: this.calculateDuration(notification),
        isPersistent: notification.priority === 'high'
      }).pipe(map(() => void 0));
    }
  }

  private setupNotificationHandlers(): void {
    // Auto-display new notifications
    this.notificationSystem.newNotification$.subscribe(notification => {
      this.showSmartNotification(notification).subscribe();
    });
  }

  private addToastToQueue(toast: ToastNotification): void {
    this.toastQueue.push(toast);
  }

  private processToastQueue(): void {
    const currentToasts = this.activeToasts$.value;
    const availableSlots = this.config.maxToasts - currentToasts.length;
    
    if (availableSlots > 0 && this.toastQueue.length > 0) {
      const toastsToShow = this.toastQueue.splice(0, availableSlots);
      const newToasts = [...currentToasts, ...toastsToShow];
      
      this.activeToasts$.next(newToasts);

      // Schedule auto-removal for non-persistent toasts
      toastsToShow.forEach(toast => {
        if (!toast.isPersistent && toast.duration > 0) {
          timer(toast.duration).subscribe(() => {
            this.closeToast(toast.id).subscribe();
          });
        }
      });

      // Show animations
      toastsToShow.forEach(toast => {
        this.animationService.fadeIn(toast.id).subscribe();
      });
    }
  }

  private executeToastAction(action: NotificationAction, toast: ToastNotification): void {
    // Handle common actions
    switch (action.action) {
      case 'dismiss':
        this.closeToast(toast.id).subscribe();
        break;
      case 'view':
        // Navigate to relevant page
        break;
      case 'approve':
      case 'decline':
        // Handle approval actions
        break;
      default:
        // Custom action handling
        console.log(`Executing action: ${action.action}`, { action, toast });
    }
  }

  private mapNotificationTypeToDisplay(type: NotificationType): ToastType {
    const typeMap: { [key in NotificationType]: ToastType } = {
      'info': 'info',
      'success': 'success',
      'warning': 'warning',
      'error': 'error',
      'system': 'info',
      'user': 'info',
      'marketing': 'info'
    };
    return typeMap[type];
  }

  private calculateDuration(notification: Notification): number {
    const baseDuration = this.config.defaultDuration;
    const priorityMultiplier = {
      'low': 0.8,
      'normal': 1,
      'high': 1.5,
      'urgent': 2
    };
    
    return baseDuration * priorityMultiplier[notification.priority];
  }

  private generateId(): string {
    return `toast-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
}

// Toast interfaces
interface ToastNotification {
  id: string;
  type: ToastType;
  title: string;
  message: string;
  duration: number;
  position: ToastPosition;
  actions: NotificationAction[];
  isCloseable: boolean;
  isPersistent: boolean;
  timestamp: Date;
}

interface BannerNotification {
  id: string;
  type: ToastType;
  title: string;
  message: string;
  actions: NotificationAction[];
  isCloseable: boolean;
  isDismissible: boolean;
  timestamp: Date;
}

type ToastType = 'info' | 'success' | 'warning' | 'error';
type ToastPosition = 'top-right' | 'top-left' | 'bottom-right' | 'bottom-left' | 'center';
```

## Push Notification Integration

### Service Worker Push Notifications

```typescript
// push-notification.service.ts
@Injectable({
  providedIn: 'root'
})
export class PushNotificationService {
  private swRegistration: ServiceWorkerRegistration | null = null;
  private pushSubscription: PushSubscription | null = null;
  private permissionState$ = new BehaviorSubject<NotificationPermission>('default');
  private incomingNotifications$ = new Subject<Notification>();

  public permission$ = this.permissionState$.asObservable();
  public notifications$ = this.incomingNotifications$.asObservable();
  public isSupported = 'serviceWorker' in navigator && 'PushManager' in window;

  constructor(
    private http: HttpClient,
    private swUpdate: SwUpdate
  ) {
    this.initializeServiceWorker();
    this.setupMessageListener();
  }

  // Request permission for push notifications
  requestPermission(): Observable<NotificationPermission> {
    if (!this.isSupported) {
      return throwError(() => new Error('Push notifications not supported'));
    }

    return new Observable(observer => {
      Notification.requestPermission().then(permission => {
        this.permissionState$.next(permission);
        observer.next(permission);
        observer.complete();
      }).catch(error => {
        observer.error(error);
      });
    });
  }

  // Subscribe to push notifications
  subscribeToNotifications(): Observable<PushSubscription> {
    if (!this.swRegistration) {
      return throwError(() => new Error('Service Worker not registered'));
    }

    if (this.permissionState$.value !== 'granted') {
      return throwError(() => new Error('Permission not granted'));
    }

    return new Observable(observer => {
      this.swRegistration!.pushManager.subscribe({
        userVisibleOnly: true,
        applicationServerKey: this.urlBase64ToUint8Array(environment.vapidPublicKey)
      }).then(subscription => {
        this.pushSubscription = subscription;
        
        // Send subscription to server
        this.sendSubscriptionToServer(subscription).subscribe({
          next: () => {
            observer.next(subscription);
            observer.complete();
          },
          error: (error) => observer.error(error)
        });
      }).catch(error => {
        observer.error(error);
      });
    });
  }

  // Unsubscribe from push notifications
  unsubscribeFromNotifications(): Observable<void> {
    if (!this.pushSubscription) {
      return of(void 0);
    }

    return new Observable(observer => {
      this.pushSubscription!.unsubscribe().then(successful => {
        if (successful) {
          // Remove subscription from server
          this.removeSubscriptionFromServer().subscribe({
            next: () => {
              this.pushSubscription = null;
              observer.next();
              observer.complete();
            },
            error: (error) => observer.error(error)
          });
        } else {
          observer.error(new Error('Failed to unsubscribe'));
        }
      }).catch(error => {
        observer.error(error);
      });
    });
  }

  // Send push notification (server-side trigger)
  sendPushNotification(notification: PushNotificationPayload): Observable<void> {
    return this.http.post<void>('/api/notifications/push', notification);
  }

  // Check subscription status
  checkSubscriptionStatus(): Observable<boolean> {
    if (!this.swRegistration) {
      return of(false);
    }

    return new Observable(observer => {
      this.swRegistration!.pushManager.getSubscription().then(subscription => {
        this.pushSubscription = subscription;
        observer.next(!!subscription);
        observer.complete();
      }).catch(() => {
        observer.next(false);
        observer.complete();
      });
    });
  }

  // Handle incoming push messages
  private setupMessageListener(): void {
    if ('serviceWorker' in navigator) {
      navigator.serviceWorker.addEventListener('message', event => {
        if (event.data && event.data.type === 'PUSH_NOTIFICATION') {
          this.incomingNotifications$.next(event.data.notification);
        }
      });
    }
  }

  private async initializeServiceWorker(): Promise<void> {
    if (!this.isSupported) {
      return;
    }

    try {
      // Wait for service worker to be ready
      this.swRegistration = await navigator.serviceWorker.ready;
      
      // Check current permission
      this.permissionState$.next(Notification.permission);
      
      // Check existing subscription
      this.checkSubscriptionStatus().subscribe();
      
    } catch (error) {
      console.error('Service Worker initialization failed:', error);
    }
  }

  private sendSubscriptionToServer(subscription: PushSubscription): Observable<void> {
    const subscriptionData = {
      endpoint: subscription.endpoint,
      keys: {
        auth: subscription.getKey('auth') ? btoa(String.fromCharCode(...new Uint8Array(subscription.getKey('auth')!))) : '',
        p256dh: subscription.getKey('p256dh') ? btoa(String.fromCharCode(...new Uint8Array(subscription.getKey('p256dh')!))) : ''
      }
    };

    return this.http.post<void>('/api/notifications/subscribe', subscriptionData);
  }

  private removeSubscriptionFromServer(): Observable<void> {
    if (!this.pushSubscription) {
      return of(void 0);
    }

    return this.http.post<void>('/api/notifications/unsubscribe', {
      endpoint: this.pushSubscription.endpoint
    });
  }

  private urlBase64ToUint8Array(base64String: string): Uint8Array {
    const padding = '='.repeat((4 - base64String.length % 4) % 4);
    const base64 = (base64String + padding)
      .replace(/-/g, '+')
      .replace(/_/g, '/');

    const rawData = window.atob(base64);
    const outputArray = new Uint8Array(rawData.length);

    for (let i = 0; i < rawData.length; ++i) {
      outputArray[i] = rawData.charCodeAt(i);
    }
    return outputArray;
  }
}

// Service Worker notification handler (sw.js)
self.addEventListener('push', event => {
  if (!event.data) {
    return;
  }

  const notification = event.data.json();
  
  const options = {
    body: notification.message,
    icon: notification.icon || '/assets/icons/notification-icon.png',
    badge: notification.badge || '/assets/icons/notification-badge.png',
    image: notification.image,
    data: notification.data,
    actions: notification.actions || [],
    requireInteraction: notification.priority === 'urgent',
    silent: notification.priority === 'low',
    tag: notification.tag || 'default',
    renotify: true,
    timestamp: Date.now()
  };

  event.waitUntil(
    self.registration.showNotification(notification.title, options)
  );
});

self.addEventListener('notificationclick', event => {
  event.notification.close();

  if (event.action) {
    // Handle action click
    handleNotificationAction(event.action, event.notification.data);
  } else {
    // Handle notification click
    event.waitUntil(
      clients.openWindow('/')
    );
  }
});

function handleNotificationAction(action, data) {
  switch (action) {
    case 'view':
      clients.openWindow(data.url || '/');
      break;
    case 'dismiss':
      // Just close the notification
      break;
    default:
      // Send message to main app
      clients.matchAll().then(clients => {
        clients.forEach(client => {
          client.postMessage({
            type: 'NOTIFICATION_ACTION',
            action,
            data
          });
        });
      });
  }
}

interface PushNotificationPayload {
  title: string;
  message: string;
  icon?: string;
  badge?: string;
  image?: string;
  data?: any;
  actions?: NotificationAction[];
  priority?: NotificationPriority;
  tag?: string;
  userIds?: string[];
  scheduleAt?: Date;
}
```

## Notification Templates & Rendering

### Dynamic Notification Templates

```typescript
// notification-template.service.ts
@Injectable({
  providedIn: 'root'
})
export class NotificationTemplateService {
  private templates = new Map<string, NotificationTemplate>();
  private customRenderers = new Map<string, ComponentRef<any>>();

  constructor(
    private componentFactoryResolver: ComponentFactoryResolver,
    private viewContainerRef: ViewContainerRef
  ) {
    this.registerDefaultTemplates();
  }

  // Register notification template
  registerTemplate(template: NotificationTemplate): void {
    this.templates.set(template.id, template);
  }

  // Get template by ID
  getTemplate(templateId: string): NotificationTemplate | undefined {
    return this.templates.get(templateId);
  }

  // Render notification using template
  renderNotification(notification: Notification, templateId?: string): Observable<RenderedNotification> {
    const template = templateId 
      ? this.getTemplate(templateId)
      : this.selectTemplate(notification);

    if (!template) {
      return throwError(() => new Error(`Template not found: ${templateId}`));
    }

    return this.processTemplate(notification, template);
  }

  // Process template with notification data
  private processTemplate(
    notification: Notification, 
    template: NotificationTemplate
  ): Observable<RenderedNotification> {
    const context = this.createTemplateContext(notification);
    
    return of({
      id: notification.id,
      html: this.interpolateTemplate(template.html, context),
      css: template.css,
      component: template.component,
      data: notification,
      template: template
    });
  }

  // Create template context
  private createTemplateContext(notification: Notification): TemplateContext {
    return {
      notification,
      user: this.getCurrentUser(),
      helpers: {
        formatDate: (date: Date) => date.toLocaleDateString(),
        formatTime: (date: Date) => date.toLocaleTimeString(),
        formatRelativeTime: (date: Date) => this.getRelativeTime(date),
        truncate: (text: string, length: number) => 
          text.length > length ? text.substring(0, length) + '...' : text,
        capitalize: (text: string) => 
          text.charAt(0).toUpperCase() + text.slice(1),
        getIcon: (type: NotificationType) => this.getTypeIcon(type),
        getColor: (type: NotificationType) => this.getTypeColor(type)
      }
    };
  }

  // Template interpolation
  private interpolateTemplate(template: string, context: TemplateContext): string {
    return template.replace(/\{\{([^}]+)\}\}/g, (match, path) => {
      const value = this.getValueByPath(context, path.trim());
      return value !== undefined ? String(value) : match;
    });
  }

  // Select appropriate template based on notification
  private selectTemplate(notification: Notification): NotificationTemplate | undefined {
    // Priority-based template selection
    const templateSelectors = [
      // Custom template by category and type
      `${notification.category}-${notification.type}`,
      // Category-specific template
      notification.category,
      // Type-specific template
      notification.type,
      // Priority-specific template
      `priority-${notification.priority}`,
      // Default template
      'default'
    ];

    for (const selector of templateSelectors) {
      const template = this.templates.get(selector);
      if (template) {
        return template;
      }
    }

    return this.templates.get('default');
  }

  // Register default templates
  private registerDefaultTemplates(): void {
    // Default template
    this.registerTemplate({
      id: 'default',
      name: 'Default Notification',
      html: `
        <div class="notification notification-{{notification.type}}">
          <div class="notification-icon">
            <i class="{{helpers.getIcon(notification.type)}}"></i>
          </div>
          <div class="notification-content">
            <h4 class="notification-title">{{notification.title}}</h4>
            <p class="notification-message">{{notification.message}}</p>
            <div class="notification-meta">
              <span class="notification-time">{{helpers.formatRelativeTime(notification.timestamp)}}</span>
              <span class="notification-category">{{notification.category}}</span>
            </div>
          </div>
          <div class="notification-actions">
            <!-- Actions will be rendered separately -->
          </div>
        </div>
      `,
      css: `
        .notification {
          display: flex;
          padding: 16px;
          margin: 8px 0;
          border-radius: 8px;
          background: white;
          box-shadow: 0 2px 8px rgba(0,0,0,0.1);
          border-left: 4px solid var(--notification-color);
        }
        .notification-icon {
          margin-right: 12px;
          font-size: 24px;
          color: var(--notification-color);
        }
        .notification-content {
          flex: 1;
        }
        .notification-title {
          margin: 0 0 4px 0;
          font-weight: 600;
          color: #333;
        }
        .notification-message {
          margin: 0 0 8px 0;
          color: #666;
          line-height: 1.4;
        }
        .notification-meta {
          font-size: 12px;
          color: #999;
        }
      `
    });

    // Chat message template
    this.registerTemplate({
      id: 'chat',
      name: 'Chat Notification',
      html: `
        <div class="notification notification-chat">
          <div class="notification-avatar">
            <img src="{{notification.data.avatar}}" alt="{{notification.data.senderName}}">
          </div>
          <div class="notification-content">
            <h4 class="notification-sender">{{notification.data.senderName}}</h4>
            <p class="notification-preview">{{helpers.truncate(notification.message, 60)}}</p>
            <span class="notification-room">in {{notification.data.roomName}}</span>
          </div>
        </div>
      `,
      css: `
        .notification-chat {
          background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
          color: white;
        }
        .notification-avatar img {
          width: 40px;
          height: 40px;
          border-radius: 50%;
        }
        .notification-sender {
          color: white;
          font-weight: bold;
        }
        .notification-preview {
          color: rgba(255,255,255,0.9);
        }
        .notification-room {
          color: rgba(255,255,255,0.7);
          font-size: 12px;
        }
      `
    });

    // System alert template
    this.registerTemplate({
      id: 'system',
      name: 'System Alert',
      html: `
        <div class="notification notification-system">
          <div class="notification-header">
            <i class="fas fa-exclamation-triangle"></i>
            <h4>System Alert</h4>
          </div>
          <div class="notification-body">
            <p>{{notification.message}}</p>
            <div class="notification-details" *ngIf="notification.data.details">
              <pre>{{notification.data.details}}</pre>
            </div>
          </div>
        </div>
      `,
      css: `
        .notification-system {
          background: #fff3cd;
          border-color: #ffc107;
          color: #856404;
        }
        .notification-header {
          display: flex;
          align-items: center;
          margin-bottom: 8px;
        }
        .notification-header i {
          margin-right: 8px;
          color: #ffc107;
        }
        .notification-details pre {
          background: rgba(0,0,0,0.1);
          padding: 8px;
          border-radius: 4px;
          font-size: 12px;
          margin-top: 8px;
        }
      `
    });
  }

  private getValueByPath(obj: any, path: string): any {
    return path.split('.').reduce((current, key) => current?.[key], obj);
  }

  private getRelativeTime(date: Date): string {
    const now = new Date();
    const diffInSeconds = Math.floor((now.getTime() - date.getTime()) / 1000);

    if (diffInSeconds < 60) return 'just now';
    if (diffInSeconds < 3600) return `${Math.floor(diffInSeconds / 60)}m ago`;
    if (diffInSeconds < 86400) return `${Math.floor(diffInSeconds / 3600)}h ago`;
    return `${Math.floor(diffInSeconds / 86400)}d ago`;
  }

  private getTypeIcon(type: NotificationType): string {
    const iconMap: { [key in NotificationType]: string } = {
      'info': 'fas fa-info-circle',
      'success': 'fas fa-check-circle',
      'warning': 'fas fa-exclamation-triangle',
      'error': 'fas fa-times-circle',
      'system': 'fas fa-cog',
      'user': 'fas fa-user',
      'marketing': 'fas fa-bullhorn'
    };
    return iconMap[type];
  }

  private getTypeColor(type: NotificationType): string {
    const colorMap: { [key in NotificationType]: string } = {
      'info': '#17a2b8',
      'success': '#28a745',
      'warning': '#ffc107',
      'error': '#dc3545',
      'system': '#6c757d',
      'user': '#007bff',
      'marketing': '#e83e8c'
    };
    return colorMap[type];
  }

  private getCurrentUser(): any {
    // Get current user from auth service
    return { name: 'Current User', id: '1' };
  }
}

// Interfaces
interface NotificationTemplate {
  id: string;
  name: string;
  html: string;
  css?: string;
  component?: Type<any>;
  category?: NotificationCategory;
  type?: NotificationType;
  priority?: NotificationPriority;
}

interface TemplateContext {
  notification: Notification;
  user: any;
  helpers: TemplateHelpers;
}

interface TemplateHelpers {
  formatDate: (date: Date) => string;
  formatTime: (date: Date) => string;
  formatRelativeTime: (date: Date) => string;
  truncate: (text: string, length: number) => string;
  capitalize: (text: string) => string;
  getIcon: (type: NotificationType) => string;
  getColor: (type: NotificationType) => string;
}

interface RenderedNotification {
  id: string;
  html: string;
  css?: string;
  component?: Type<any>;
  data: Notification;
  template: NotificationTemplate;
}
```

## User Preferences & Permissions

### Notification Preferences Management

```typescript
// notification-preferences.service.ts
@Injectable({
  providedIn: 'root'
})
export class NotificationPreferencesService {
  private preferencesSubject = new BehaviorSubject<NotificationPreferences>(
    this.getDefaultPreferences()
  );

  public preferences$ = this.preferencesSubject.asObservable();

  constructor(
    private storageService: StorageService,
    private http: HttpClient
  ) {
    this.loadPreferences();
  }

  // Update preferences
  updatePreferences(updates: Partial<NotificationPreferences>): Observable<NotificationPreferences> {
    const current = this.preferencesSubject.value;
    const updated = { ...current, ...updates, lastUpdated: new Date() };
    
    this.preferencesSubject.next(updated);
    this.savePreferences(updated);
    
    return this.syncWithServer(updated);
  }

  // Update category preferences
  updateCategoryPreference(
    category: NotificationCategory, 
    preference: CategoryPreference
  ): Observable<NotificationPreferences> {
    const current = this.preferencesSubject.value;
    const updated = {
      ...current,
      categories: {
        ...current.categories,
        [category]: preference
      },
      lastUpdated: new Date()
    };

    this.preferencesSubject.next(updated);
    this.savePreferences(updated);
    
    return this.syncWithServer(updated);
  }

  // Check if notification should be shown
  shouldShowNotification(notification: Notification): Observable<boolean> {
    return this.preferences$.pipe(
      map(prefs => this.evaluateNotificationPermission(notification, prefs)),
      take(1)
    );
  }

  // Get quiet hours status
  isInQuietHours(): Observable<boolean> {
    return this.preferences$.pipe(
      map(prefs => {
        if (!prefs.quietHours.enabled) return false;
        
        const now = new Date();
        const currentTime = now.getHours() * 60 + now.getMinutes();
        const startTime = this.timeStringToMinutes(prefs.quietHours.start);
        const endTime = this.timeStringToMinutes(prefs.quietHours.end);
        
        if (startTime <= endTime) {
          return currentTime >= startTime && currentTime <= endTime;
        } else {
          // Overnight quiet hours
          return currentTime >= startTime || currentTime <= endTime;
        }
      })
    );
  }

  // Get notification permission for category
  getCategoryPermission(category: NotificationCategory): Observable<CategoryPreference> {
    return this.preferences$.pipe(
      map(prefs => prefs.categories[category] || this.getDefaultCategoryPreference()),
      take(1)
    );
  }

  // Request browser notification permission
  requestBrowserPermission(): Observable<NotificationPermission> {
    return new Observable(observer => {
      if (!('Notification' in window)) {
        observer.error(new Error('Notifications not supported'));
        return;
      }

      if (Notification.permission === 'granted') {
        observer.next('granted');
        observer.complete();
        return;
      }

      Notification.requestPermission().then(permission => {
        this.updatePreferences({
          browserPermission: permission
        }).subscribe();
        
        observer.next(permission);
        observer.complete();
      }).catch(error => {
        observer.error(error);
      });
    });
  }

  private evaluateNotificationPermission(
    notification: Notification, 
    prefs: NotificationPreferences
  ): boolean {
    // Check global enabled state
    if (!prefs.enabled) return false;

    // Check browser permission
    if (prefs.browserPermission !== 'granted') return false;

    // Check quiet hours
    if (prefs.quietHours.enabled) {
      const now = new Date();
      const isQuietTime = this.isTimeInQuietHours(now, prefs.quietHours);
      if (isQuietTime && notification.priority !== 'urgent') {
        return false;
      }
    }

    // Check category preferences
    const categoryPref = prefs.categories[notification.category];
    if (!categoryPref?.enabled) return false;

    // Check priority filter
    const priorityLevels = ['low', 'normal', 'high', 'urgent'];
    const notificationLevel = priorityLevels.indexOf(notification.priority);
    const minimumLevel = priorityLevels.indexOf(categoryPref.minimumPriority);
    
    return notificationLevel >= minimumLevel;
  }

  private isTimeInQuietHours(time: Date, quietHours: QuietHours): boolean {
    const currentTime = time.getHours() * 60 + time.getMinutes();
    const startTime = this.timeStringToMinutes(quietHours.start);
    const endTime = this.timeStringToMinutes(quietHours.end);
    
    if (startTime <= endTime) {
      return currentTime >= startTime && currentTime <= endTime;
    } else {
      return currentTime >= startTime || currentTime <= endTime;
    }
  }

  private timeStringToMinutes(timeString: string): number {
    const [hours, minutes] = timeString.split(':').map(Number);
    return hours * 60 + minutes;
  }

  private loadPreferences(): void {
    const stored = this.storageService.getItem('notification-preferences');
    if (stored) {
      try {
        const preferences = JSON.parse(stored);
        this.preferencesSubject.next(preferences);
      } catch (error) {
        console.error('Failed to parse stored preferences:', error);
      }
    }
  }

  private savePreferences(preferences: NotificationPreferences): void {
    this.storageService.setItem('notification-preferences', JSON.stringify(preferences));
  }

  private syncWithServer(preferences: NotificationPreferences): Observable<NotificationPreferences> {
    return this.http.put<NotificationPreferences>('/api/user/notification-preferences', preferences)
      .pipe(
        catchError(error => {
          console.error('Failed to sync preferences with server:', error);
          return of(preferences);
        })
      );
  }

  private getDefaultPreferences(): NotificationPreferences {
    return {
      enabled: true,
      browserPermission: Notification.permission,
      quietHours: {
        enabled: false,
        start: '22:00',
        end: '08:00'
      },
      categories: {
        chat: this.getDefaultCategoryPreference(),
        system: { ...this.getDefaultCategoryPreference(), minimumPriority: 'normal' },
        order: this.getDefaultCategoryPreference(),
        security: { ...this.getDefaultCategoryPreference(), minimumPriority: 'high' },
        reminder: this.getDefaultCategoryPreference(),
        news: { ...this.getDefaultCategoryPreference(), enabled: false },
        social: this.getDefaultCategoryPreference()
      },
      lastUpdated: new Date()
    };
  }

  private getDefaultCategoryPreference(): CategoryPreference {
    return {
      enabled: true,
      inApp: true,
      push: true,
      email: false,
      minimumPriority: 'low'
    };
  }
}

// Interfaces
interface NotificationPreferences {
  enabled: boolean;
  browserPermission: NotificationPermission;
  quietHours: QuietHours;
  categories: { [key in NotificationCategory]: CategoryPreference };
  lastUpdated: Date;
}

interface QuietHours {
  enabled: boolean;
  start: string; // HH:MM format
  end: string;   // HH:MM format
}

interface CategoryPreference {
  enabled: boolean;
  inApp: boolean;
  push: boolean;
  email: boolean;
  minimumPriority: NotificationPriority;
}
```

## Summary

This lesson covered comprehensive real-time notification patterns for Angular applications:

### Key Takeaways:
1. **Notification System Architecture**: Complete notification infrastructure with queuing, prioritization, and state management
2. **In-App Notifications**: Toast and banner notifications with smart display logic and animations
3. **Push Notifications**: Service Worker integration with proper permission handling and subscription management
4. **Dynamic Templates**: Flexible templating system for customizable notification rendering
5. **User Preferences**: Comprehensive preference management with quiet hours and category-specific settings
6. **Permission Management**: Proper handling of browser permissions and user consent
7. **Performance**: Efficient queuing, batching, and cleanup strategies
8. **Cross-Platform**: Strategies for web, mobile, and desktop notification delivery

### Next Steps:
- Implement notification systems in your applications
- Add proper permission flows and user preferences
- Consider notification analytics and engagement tracking
- Test across different browsers and devices

In the next lesson, we'll explore **Search & Autocomplete Patterns** for building sophisticated search experiences with RxJS.
