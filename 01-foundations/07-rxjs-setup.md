# Setting up RxJS in Angular 🟢

## 🎯 Learning Objectives
By the end of this lesson, you will understand:
- How to set up RxJS in different Angular versions
- Angular's built-in RxJS integration
- How to import and use RxJS operators
- Best practices for RxJS in Angular projects
- Common setup issues and their solutions

## 🚀 Angular & RxJS Integration

### Why Angular Uses RxJS

Angular is built with **reactive programming** at its core:

- **HttpClient** returns Observables
- **Reactive Forms** use Observables for value changes
- **Router** provides Observable-based navigation
- **Component lifecycle** integrates with RxJS patterns
- **Event handling** is streamlined with RxJS

```typescript
// Angular is reactive by design
export class AppComponent {
  // HTTP requests return Observables
  users$ = this.http.get<User[]>('/api/users');
  
  // Forms emit value changes as Observables
  searchValue$ = this.searchForm.get('query')!.valueChanges;
  
  // Router provides Observable navigation
  route$ = this.router.events;
}
```

## 📦 RxJS Versions & Angular Compatibility

### Angular-RxJS Version Matrix

| Angular Version | RxJS Version | Release Date | Status |
|----------------|--------------|--------------|---------|
| Angular 2-5    | RxJS 5.x     | 2016-2018    | ⚠️ Legacy |
| Angular 6-7    | RxJS 6.x     | 2018-2019    | ⚠️ Legacy |
| Angular 8-11   | RxJS 6.x     | 2019-2021    | ⚠️ Legacy |
| Angular 12-15  | RxJS 7.x     | 2021-2023    | ✅ Supported |
| Angular 16+    | RxJS 7.4+    | 2023+        | ✅ Current |

### Key Changes Between Versions

#### RxJS 5 → RxJS 6 (Breaking Changes)
```typescript
// RxJS 5 (Old way)
import { Observable } from 'rxjs/Observable';
import 'rxjs/add/operator/map';
import 'rxjs/add/operator/filter';

observable
  .map(x => x * 2)
  .filter(x => x > 10)
  .subscribe();

// RxJS 6+ (Current way)
import { Observable } from 'rxjs';
import { map, filter } from 'rxjs/operators';

observable.pipe(
  map(x => x * 2),
  filter(x => x > 10)
).subscribe();
```

#### RxJS 6 → RxJS 7 (Minor Changes)
```typescript
// RxJS 6
import { combineLatest } from 'rxjs';

combineLatest(obs1, obs2); // Deprecated

// RxJS 7
import { combineLatest } from 'rxjs';

combineLatest([obs1, obs2]); // Array syntax required
```

## 🛠️ Setting Up New Angular Project

### Using Angular CLI

```bash
# Create new Angular project (includes RxJS by default)
ng new my-rxjs-app

# Navigate to project
cd my-rxjs-app

# Check RxJS version
npm list rxjs
```

### Manual RxJS Installation

```bash
# If RxJS is not included (rare)
npm install rxjs

# Install specific version
npm install rxjs@^7.8.0

# For development
npm install --save-dev @types/rxjs
```

### Package.json Dependencies

```json
{
  "dependencies": {
    "@angular/animations": "^16.0.0",
    "@angular/common": "^16.0.0",
    "@angular/compiler": "^16.0.0",
    "@angular/core": "^16.0.0",
    "@angular/forms": "^16.0.0",
    "@angular/platform-browser": "^16.0.0",
    "@angular/platform-browser-dynamic": "^16.0.0",
    "@angular/router": "^16.0.0",
    "rxjs": "~7.8.0",
    "tslib": "^2.3.0",
    "zone.js": "~0.13.0"
  }
}
```

## 📝 TypeScript Configuration

### tsconfig.json Setup

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "dom"],
    "module": "ES2022",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "allowSyntheticDefaultImports": true
  }
}
```

### Angular-specific Configuration

```typescript
// angular.json - Build optimization
{
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "options": {
            "optimization": true,
            "buildOptimizer": true,
            "aot": true,
            "extractLicenses": true,
            "sourceMap": false
          }
        }
      }
    }
  }
}
```

## 📚 RxJS Import Patterns

### Core Imports

```typescript
// Essential RxJS imports
import { Observable, Subject, BehaviorSubject, ReplaySubject } from 'rxjs';
import { Observer, Subscription } from 'rxjs';

// Creation operators
import { of, from, interval, timer, fromEvent } from 'rxjs';

// Pipeable operators
import { 
  map, 
  filter, 
  tap, 
  switchMap, 
  mergeMap, 
  concatMap, 
  catchError,
  retry,
  debounceTime,
  distinctUntilChanged,
  takeUntil,
  share,
  shareReplay
} from 'rxjs/operators';
```

### Angular-specific Imports

```typescript
// Angular HTTP
import { HttpClient, HttpErrorResponse } from '@angular/common/http';

// Angular Forms
import { FormControl, FormGroup, FormBuilder } from '@angular/forms';

// Angular Router
import { Router, ActivatedRoute, NavigationEnd } from '@angular/router';

// Angular Animations
import { AnimationEvent } from '@angular/animations';
```

### Organized Import Structure

```typescript
// Group imports logically
// 1. Angular core
import { Component, OnInit, OnDestroy, Injectable } from '@angular/core';

// 2. Angular modules
import { HttpClient } from '@angular/common/http';
import { FormBuilder, FormGroup } from '@angular/forms';

// 3. RxJS core
import { Observable, Subject, BehaviorSubject } from 'rxjs';

// 4. RxJS operators
import { 
  map, 
  filter, 
  switchMap, 
  takeUntil, 
  catchError 
} from 'rxjs/operators';

// 5. Local imports
import { UserService } from './user.service';
import { User } from './models/user.interface';
```

## 🏗️ Angular Service Setup

### Basic Observable Service

```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, BehaviorSubject } from 'rxjs';
import { map, tap, catchError } from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class UserService {
  private apiUrl = 'https://jsonplaceholder.typicode.com';
  private usersSubject = new BehaviorSubject<User[]>([]);
  
  // Public Observable (read-only)
  users$ = this.usersSubject.asObservable();

  constructor(private http: HttpClient) {}

  // Load users and update state
  loadUsers(): Observable<User[]> {
    return this.http.get<User[]>(`${this.apiUrl}/users`).pipe(
      tap(users => this.usersSubject.next(users)),
      catchError(error => {
        console.error('Failed to load users:', error);
        throw error;
      })
    );
  }

  // Get single user
  getUser(id: number): Observable<User> {
    return this.http.get<User>(`${this.apiUrl}/users/${id}`);
  }

  // Add user
  addUser(user: Omit<User, 'id'>): Observable<User> {
    return this.http.post<User>(`${this.apiUrl}/users`, user).pipe(
      tap(newUser => {
        const currentUsers = this.usersSubject.value;
        this.usersSubject.next([...currentUsers, newUser]);
      })
    );
  }
}
```

### Interface Definitions

```typescript
// models/user.interface.ts
export interface User {
  id: number;
  name: string;
  username: string;
  email: string;
  address: Address;
  phone: string;
  website: string;
  company: Company;
}

export interface Address {
  street: string;
  suite: string;
  city: string;
  zipcode: string;
  geo: Geo;
}

export interface Geo {
  lat: string;
  lng: string;
}

export interface Company {
  name: string;
  catchPhrase: string;
  bs: string;
}
```

## 🧩 Component Integration

### Basic Component with RxJS

```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';
import { UserService } from './services/user.service';
import { User } from './models/user.interface';

@Component({
  selector: 'app-users',
  template: `
    <h2>Users</h2>
    <div *ngIf="loading">Loading...</div>
    <div *ngIf="error" class="error">{{ error }}</div>
    
    <div *ngFor="let user of users" class="user-card">
      <h3>{{ user.name }}</h3>
      <p>{{ user.email }}</p>
    </div>
  `,
  styleUrls: ['./users.component.scss']
})
export class UsersComponent implements OnInit, OnDestroy {
  users: User[] = [];
  loading = false;
  error: string | null = null;
  
  // Destroy subject for cleanup
  private destroy$ = new Subject<void>();

  constructor(private userService: UserService) {}

  ngOnInit(): void {
    this.loadUsers();
  }

  ngOnDestroy(): void {
    // Clean up all subscriptions
    this.destroy$.next();
    this.destroy$.complete();
  }

  private loadUsers(): void {
    this.loading = true;
    this.error = null;

    this.userService.loadUsers().pipe(
      takeUntil(this.destroy$) // Automatic cleanup
    ).subscribe({
      next: (users) => {
        this.users = users;
        this.loading = false;
      },
      error: (error) => {
        this.error = 'Failed to load users';
        this.loading = false;
        console.error('Error loading users:', error);
      }
    });
  }
}
```

### Using Async Pipe (Recommended)

```typescript
@Component({
  selector: 'app-users-async',
  template: `
    <h2>Users (Async Pipe)</h2>
    
    <!-- Async pipe handles subscription/unsubscription automatically -->
    <div *ngIf="users$ | async as users; else loading">
      <div *ngFor="let user of users" class="user-card">
        <h3>{{ user.name }}</h3>
        <p>{{ user.email }}</p>
      </div>
    </div>
    
    <ng-template #loading>
      <div>Loading users...</div>
    </ng-template>
  `,
  styleUrls: ['./users.component.scss']
})
export class UsersAsyncComponent implements OnInit {
  users$ = this.userService.users$;

  constructor(private userService: UserService) {}

  ngOnInit(): void {
    // Just trigger the load - async pipe handles the rest
    this.userService.loadUsers().subscribe();
  }
}
```

## 🎨 Reactive Forms Integration

### Form with Observable Value Changes

```typescript
import { Component, OnInit } from '@angular/core';
import { FormBuilder, FormGroup } from '@angular/forms';
import { debounceTime, distinctUntilChanged, switchMap } from 'rxjs/operators';

@Component({
  selector: 'app-search',
  template: `
    <form [formGroup]="searchForm">
      <input 
        type="text" 
        formControlName="query" 
        placeholder="Search users..."
      >
    </form>
    
    <div *ngFor="let result of searchResults$ | async">
      {{ result.name }}
    </div>
  `
})
export class SearchComponent implements OnInit {
  searchForm: FormGroup;
  searchResults$ = this.searchForm?.get('query')?.valueChanges.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap(query => 
      query ? this.userService.searchUsers(query) : []
    )
  );

  constructor(
    private fb: FormBuilder,
    private userService: UserService
  ) {
    this.searchForm = this.fb.group({
      query: ['']
    });
  }

  ngOnInit(): void {
    // Form is already reactive through valueChanges
  }
}
```

## 🛣️ Router Integration

### Route Data and Parameters

```typescript
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute, Router } from '@angular/router';
import { switchMap, map } from 'rxjs/operators';

@Component({
  selector: 'app-user-detail',
  template: `
    <div *ngIf="user$ | async as user">
      <h2>{{ user.name }}</h2>
      <p>{{ user.email }}</p>
      <button (click)="goBack()">Back</button>
    </div>
  `
})
export class UserDetailComponent implements OnInit {
  user$ = this.route.paramMap.pipe(
    map(params => +params.get('id')!),
    switchMap(id => this.userService.getUser(id))
  );

  constructor(
    private route: ActivatedRoute,
    private router: Router,
    private userService: UserService
  ) {}

  ngOnInit(): void {}

  goBack(): void {
    this.router.navigate(['/users']);
  }
}
```

## ⚡ Performance Optimization

### Tree Shaking Configuration

```typescript
// Import only what you need for better tree shaking
import { map } from 'rxjs/operators';
import { Observable } from 'rxjs';

// Avoid importing entire operator modules
// ❌ Bad
import * from 'rxjs/operators';

// ✅ Good
import { map, filter, switchMap } from 'rxjs/operators';
```

### Bundle Size Analysis

```bash
# Analyze bundle size
ng build --stats-json
npx webpack-bundle-analyzer dist/my-app/stats.json

# Check for RxJS bloat
npm install -g bundlephobia
bundlephobia rxjs
```

### Custom Build Configuration

```typescript
// angular.json - Custom build configurations
{
  "configurations": {
    "production": {
      "optimization": {
        "scripts": true,
        "styles": true,
        "fonts": true
      },
      "outputHashing": "all",
      "sourceMap": false,
      "namedChunks": false,
      "extractLicenses": true,
      "vendorChunk": false,
      "buildOptimizer": true
    }
  }
}
```

## 🔧 Development Tools

### Angular DevTools

```bash
# Install Angular DevTools (Chrome extension)
# Provides RxJS stream inspection
```

### RxJS DevTools

```typescript
// Enable RxJS debugging in development
import { tap } from 'rxjs/operators';

const debugStream$ = source$.pipe(
  tap(value => console.log('Debug:', value))
);
```

### VSCode Extensions

```json
// .vscode/extensions.json
{
  "recommendations": [
    "angular.ng-template",
    "ms-vscode.vscode-typescript-next",
    "bradlc.vscode-tailwindcss",
    "esbenp.prettier-vscode"
  ]
}
```

## 🚨 Common Setup Issues

### Issue 1: Import Errors

```typescript
// ❌ Common mistake
import { Observable } from 'rxjs/Observable'; // RxJS 5 syntax

// ✅ Correct for RxJS 6+
import { Observable } from 'rxjs';
```

### Issue 2: Operator Import Errors

```typescript
// ❌ Wrong
import { map } from 'rxjs/add/operator/map';

// ✅ Correct
import { map } from 'rxjs/operators';
```

### Issue 3: Version Conflicts

```bash
# Check for version conflicts
npm ls rxjs

# Fix conflicts
npm install rxjs@^7.8.0 --save
```

### Issue 4: Memory Leaks

```typescript
// ❌ Memory leak
export class Component {
  ngOnInit() {
    this.service.data$.subscribe(data => {
      // No unsubscription - memory leak!
    });
  }
}

// ✅ Proper cleanup
export class Component implements OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit() {
    this.service.data$.pipe(
      takeUntil(this.destroy$)
    ).subscribe(data => {
      // Automatic cleanup
    });
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## 📋 Best Practices Checklist

### ✅ Setup Best Practices

- [ ] Use latest stable Angular and RxJS versions
- [ ] Import only needed operators for tree shaking
- [ ] Set up proper TypeScript configuration
- [ ] Use async pipe when possible
- [ ] Implement proper subscription cleanup
- [ ] Use meaningful naming conventions ($ suffix)
- [ ] Handle errors gracefully
- [ ] Use shareReplay for expensive operations
- [ ] Avoid nested subscriptions
- [ ] Use reactive patterns throughout

### ✅ Development Best Practices

- [ ] Use Angular DevTools for debugging
- [ ] Write marble tests for complex streams
- [ ] Document complex reactive flows
- [ ] Use TypeScript strict mode
- [ ] Implement error boundaries
- [ ] Monitor bundle size regularly
- [ ] Use linting rules for RxJS
- [ ] Follow reactive programming principles

## 🎯 Quick Assessment

**Setup Questions:**

1. What RxJS version should you use with Angular 16?
2. How do you properly clean up subscriptions?
3. What's the difference between manual subscription and async pipe?
4. How do you import operators in RxJS 6+?

**Answers:**

1. **RxJS 7.4+** (comes bundled with Angular 16)
2. **takeUntil pattern** with destroy Subject or **async pipe**
3. **Manual**: Requires cleanup. **Async pipe**: Automatic subscription/unsubscription
4. **import { map, filter } from 'rxjs/operators'** (pipeable operators)

## 🌟 Key Takeaways

- **Angular includes RxJS** by default with proper version compatibility
- **Async pipe** is preferred for automatic subscription management
- **Proper imports** are crucial for tree shaking and bundle size
- **Subscription cleanup** prevents memory leaks
- **Reactive patterns** should be used consistently throughout the app
- **Development tools** help debug and optimize RxJS usage

## 🚀 Next Steps

Now that you have RxJS properly set up in Angular, you're ready to explore the **RxJS Library Architecture & Design** to understand how the library is structured and organized.

**Next Lesson**: [RxJS Library Architecture & Design](./08-rxjs-architecture.md) 🟡

---

🎉 **Excellent!** You now have a solid foundation for using RxJS in Angular applications. Your development environment is ready for building reactive applications with confidence!
