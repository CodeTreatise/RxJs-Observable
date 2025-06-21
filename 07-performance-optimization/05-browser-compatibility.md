# Browser Compatibility & Polyfills

## Table of Contents
- [Introduction](#introduction)
- [RxJS Browser Support](#rxjs-browser-support)
- [ES6 Features Used by RxJS](#es6-features-used-by-rxjs)
- [Polyfill Requirements](#polyfill-requirements)
- [Angular CLI Configuration](#angular-cli-configuration)
- [Manual Polyfill Setup](#manual-polyfill-setup)
- [Testing Browser Compatibility](#testing-browser-compatibility)
- [Performance Considerations](#performance-considerations)
- [Common Browser Issues](#common-browser-issues)
- [Progressive Enhancement](#progressive-enhancement)
- [Legacy Browser Support](#legacy-browser-support)
- [Best Practices](#best-practices)

## Introduction

RxJS relies on modern JavaScript features that may not be available in all browsers. This lesson covers how to ensure your RxJS applications work across different browsers and devices through proper polyfill configuration and compatibility strategies.

### Browser Support Overview

```typescript
// RxJS 7.x Browser Requirements
const browserSupport = {
  chrome: '>=60',      // ES2015+ support
  firefox: '>=60',     // ES2015+ support
  safari: '>=12',      // ES2015+ support
  edge: '>=79',        // Chromium-based Edge
  ie: 'Not supported', // IE11 and below not supported
  
  // Mobile browsers
  ios: '>=12',         // Safari mobile
  android: '>=7.0'     // Chrome mobile
};

// Key features required:
// - ES2015 (ES6) support
// - Symbol
// - Map/Set
// - Promise
// - Object.assign
// - Array methods (find, includes, etc.)
```

## RxJS Browser Support

### RxJS Version Compatibility

```typescript
// RxJS version browser support matrix
export const RxJSBrowserMatrix = {
  'rxjs@7': {
    chrome: '>=60',
    firefox: '>=60',
    safari: '>=12',
    edge: '>=79',
    ie: false,
    requiresPolyfills: ['Symbol', 'Map', 'Set', 'Promise']
  },
  
  'rxjs@6': {
    chrome: '>=45',
    firefox: '>=45',
    safari: '>=10',
    edge: '>=14',
    ie: '>=11', // With polyfills
    requiresPolyfills: ['Symbol', 'Map', 'Set', 'Promise', 'Object.assign']
  }
};

// Feature detection for RxJS compatibility
export class BrowserCompatibility {
  static checkRxJSSupport(): CompatibilityReport {
    const features = this.detectFeatures();
    const missing = this.findMissingFeatures(features);
    
    return {
      isSupported: missing.length === 0,
      missingFeatures: missing,
      recommendedPolyfills: this.getRecommendedPolyfills(missing),
      browserInfo: this.getBrowserInfo()
    };
  }
  
  private static detectFeatures(): FeatureSupport {
    return {
      symbol: typeof Symbol !== 'undefined',
      map: typeof Map !== 'undefined',
      set: typeof Set !== 'undefined',
      promise: typeof Promise !== 'undefined',
      objectAssign: typeof Object.assign === 'function',
      arrayFind: Array.prototype.find !== undefined,
      arrayIncludes: Array.prototype.includes !== undefined
    };
  }
}

interface FeatureSupport {
  symbol: boolean;
  map: boolean;
  set: boolean;
  promise: boolean;
  objectAssign: boolean;
  arrayFind: boolean;
  arrayIncludes: boolean;
}
```

### Browser Feature Detection

```typescript
// Comprehensive browser feature detection
export class FeatureDetector {
  static getRxJSRequirements(): RequirementCheck[] {
    return [
      {
        name: 'Symbol',
        required: true,
        available: this.hasSymbol(),
        polyfill: 'es6-symbol'
      },
      {
        name: 'Map',
        required: true,
        available: this.hasMap(),
        polyfill: 'core-js/features/map'
      },
      {
        name: 'Set',
        required: true,
        available: this.hasSet(),
        polyfill: 'core-js/features/set'
      },
      {
        name: 'Promise',
        required: true,
        available: this.hasPromise(),
        polyfill: 'core-js/features/promise'
      },
      {
        name: 'Object.assign',
        required: true,
        available: this.hasObjectAssign(),
        polyfill: 'core-js/features/object/assign'
      }
    ];
  }
  
  private static hasSymbol(): boolean {
    return typeof Symbol !== 'undefined' && 
           typeof Symbol.iterator !== 'undefined';
  }
  
  private static hasMap(): boolean {
    return typeof Map !== 'undefined' && 
           typeof Map.prototype.forEach === 'function';
  }
  
  private static hasSet(): boolean {
    return typeof Set !== 'undefined' && 
           typeof Set.prototype.forEach === 'function';
  }
  
  private static hasPromise(): boolean {
    return typeof Promise !== 'undefined' && 
           typeof Promise.resolve === 'function';
  }
  
  private static hasObjectAssign(): boolean {
    return typeof Object.assign === 'function';
  }
}

interface RequirementCheck {
  name: string;
  required: boolean;
  available: boolean;
  polyfill: string;
}
```

## ES6 Features Used by RxJS

### Core ES6 Dependencies

```typescript
// ES6 features that RxJS depends on
export const RxJSES6Dependencies = {
  // Symbol - Used for Symbol.iterator and Symbol.observable
  symbol: {
    usage: 'Iterator protocol, Observable symbol',
    critical: true,
    example: `
      // RxJS uses Symbol.iterator for array conversion
      const obs$ = from([1, 2, 3]); // Uses Symbol.iterator
      
      // Symbol.observable for interoperability
      const customObservable = {
        [Symbol.observable]() {
          return this;
        }
      };
    `
  },
  
  // Map - Used internally for operator caching and state management
  map: {
    usage: 'Internal caching, operator state',
    critical: true,
    example: `
      // RxJS operators use Map for caching
      const cache = new Map();
      const shareReplay$ = source$.pipe(shareReplay(1));
    `
  },
  
  // Set - Used for tracking subscriptions and preventing duplicates
  set: {
    usage: 'Subscription tracking, duplicate prevention',
    critical: true,
    example: `
      // Used internally for managing subscriber sets
      const subscribers = new Set();
    `
  },
  
  // Promise - Used for async operations and conversions
  promise: {
    usage: 'Promise conversion, async operations',
    critical: false,
    example: `
      // Converting Promises to Observables
      const obs$ = from(fetch('/api/data'));
    `
  }
};
```

### Operator-Specific Requirements

```typescript
// Specific operators and their browser requirements
export const OperatorRequirements = {
  // Most operators work with basic ES6
  basic: {
    operators: ['map', 'filter', 'tap', 'take', 'skip'],
    requirements: ['Symbol', 'Map'],
    browserSupport: 'IE11+ with polyfills'
  },
  
  // Async operators need Promise support
  async: {
    operators: ['switchMap', 'mergeMap', 'concatMap', 'exhaustMap'],
    requirements: ['Symbol', 'Map', 'Promise'],
    browserSupport: 'IE11+ with polyfills'
  },
  
  // Time-based operators need additional features
  time: {
    operators: ['debounceTime', 'throttleTime', 'delay', 'timeout'],
    requirements: ['Symbol', 'Map', 'setTimeout', 'clearTimeout'],
    browserSupport: 'All modern browsers'
  },
  
  // Advanced operators may need more features
  advanced: {
    operators: ['shareReplay', 'publishReplay', 'multicast'],
    requirements: ['Symbol', 'Map', 'Set', 'WeakMap'],
    browserSupport: 'Modern browsers only'
  }
};
```

## Polyfill Requirements

### Core Polyfills

```typescript
// Essential polyfills for RxJS
export const CorePolyfills = {
  // Core-js configuration for RxJS
  essential: [
    'core-js/features/symbol',
    'core-js/features/symbol/iterator',
    'core-js/features/map',
    'core-js/features/set',
    'core-js/features/promise',
    'core-js/features/object/assign'
  ],
  
  // Optional but recommended
  recommended: [
    'core-js/features/array/find',
    'core-js/features/array/includes',
    'core-js/features/array/from',
    'core-js/features/weak-map',
    'core-js/features/weak-set'
  ],
  
  // For specific use cases
  conditional: {
    ajax: ['whatwg-fetch'], // For older browsers
    websocket: [], // Native WebSocket support is widespread
    testing: ['core-js/features/reflect']
  }
};

// Polyfill loader service
export class PolyfillLoader {
  static async loadRequiredPolyfills(): Promise<void> {
    const requirements = FeatureDetector.getRxJSRequirements();
    const missing = requirements.filter(req => !req.available);
    
    if (missing.length === 0) {
      return; // No polyfills needed
    }
    
    console.log('Loading polyfills for:', missing.map(m => m.name));
    
    // Load polyfills dynamically
    const polyfillPromises = missing.map(req => 
      this.loadPolyfill(req.polyfill)
    );
    
    await Promise.all(polyfillPromises);
    console.log('All polyfills loaded successfully');
  }
  
  private static async loadPolyfill(polyfillName: string): Promise<void> {
    try {
      await import(polyfillName);
    } catch (error) {
      console.warn(`Failed to load polyfill: ${polyfillName}`, error);
    }
  }
}
```

### Selective Polyfill Loading

```typescript
// Smart polyfill loading based on browser detection
export class SmartPolyfillLoader {
  private static loadedPolyfills = new Set<string>();
  
  static async loadPolyfillsForBrowser(): Promise<void> {
    const browserInfo = this.detectBrowser();
    const requiredPolyfills = this.getPolyfillsForBrowser(browserInfo);
    
    await this.loadPolyfills(requiredPolyfills);
  }
  
  private static getPolyfillsForBrowser(browser: BrowserInfo): string[] {
    const polyfills: string[] = [];
    
    // Internet Explorer 11
    if (browser.name === 'IE' && browser.version <= 11) {
      polyfills.push(
        'core-js/features/symbol',
        'core-js/features/map',
        'core-js/features/set',
        'core-js/features/promise',
        'core-js/features/object/assign',
        'core-js/features/array/find',
        'core-js/features/array/includes'
      );
    }
    
    // Older Chrome (< 60)
    if (browser.name === 'Chrome' && browser.version < 60) {
      polyfills.push('core-js/features/symbol');
    }
    
    // Older Safari (< 12)
    if (browser.name === 'Safari' && browser.version < 12) {
      polyfills.push(
        'core-js/features/symbol',
        'core-js/features/promise'
      );
    }
    
    return polyfills;
  }
  
  private static async loadPolyfills(polyfills: string[]): Promise<void> {
    const newPolyfills = polyfills.filter(p => !this.loadedPolyfills.has(p));
    
    if (newPolyfills.length === 0) return;
    
    const loadPromises = newPolyfills.map(async (polyfill) => {
      try {
        await import(polyfill);
        this.loadedPolyfills.add(polyfill);
      } catch (error) {
        console.error(`Failed to load polyfill: ${polyfill}`, error);
      }
    });
    
    await Promise.all(loadPromises);
  }
}

interface BrowserInfo {
  name: string;
  version: number;
  engine: string;
}
```

## Angular CLI Configuration

### Browserslist Configuration

```javascript
// .browserslistrc - Configure target browsers
# Production browsers
> 0.5%
last 2 versions
Firefox ESR
not dead
not IE 9-11

# Development browsers (more permissive)
[development]
last 1 chrome version
last 1 firefox version
last 1 safari version
```

### Angular.json Polyfill Setup

```json
// angular.json - Build configuration
{
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "options": {
            "polyfills": "src/polyfills.ts",
            "es5BrowserSupport": false
          },
          "configurations": {
            "legacy": {
              "es5BrowserSupport": true,
              "polyfills": "src/polyfills-legacy.ts"
            }
          }
        }
      }
    }
  }
}
```

### Polyfills.ts Configuration

```typescript
// src/polyfills.ts - Main polyfills file
/**
 * Required for RxJS to work properly
 */

// Core polyfills for RxJS
import 'core-js/features/symbol';
import 'core-js/features/symbol/iterator';
import 'core-js/features/map';
import 'core-js/features/set';
import 'core-js/features/promise';

// Object methods
import 'core-js/features/object/assign';

// Array methods (recommended)
import 'core-js/features/array/find';
import 'core-js/features/array/includes';
import 'core-js/features/array/from';

// Zone.js for Angular
import 'zone.js/dist/zone';

// Browser compatibility check
import { BrowserCompatibility } from './app/utils/browser-compatibility';

// Check and report compatibility issues
const compatibilityReport = BrowserCompatibility.checkRxJSSupport();
if (!compatibilityReport.isSupported) {
  console.warn('Browser compatibility issues detected:', compatibilityReport);
}
```

### Legacy Browser Support

```typescript
// src/polyfills-legacy.ts - Extended polyfills for legacy browsers
/**
 * Polyfills for legacy browsers (IE11, older Chrome/Safari)
 */

// All standard polyfills
import './polyfills';

// Additional polyfills for legacy browsers
import 'core-js/features/weak-map';
import 'core-js/features/weak-set';
import 'core-js/features/reflect';

// String methods
import 'core-js/features/string/includes';
import 'core-js/features/string/starts-with';
import 'core-js/features/string/ends-with';

// Number methods
import 'core-js/features/number/is-finite';
import 'core-js/features/number/is-integer';
import 'core-js/features/number/is-nan';

// Fetch API for older browsers
import 'whatwg-fetch';

// Custom Elements (if using Angular Elements)
import '@webcomponents/custom-elements/custom-elements.min.js';

// Intersection Observer (for lazy loading)
import 'intersection-observer';
```

## Manual Polyfill Setup

### Dynamic Polyfill Loading

```typescript
// Dynamic polyfill loading based on feature detection
export class DynamicPolyfillLoader {
  static async loadPolyfillsIfNeeded(): Promise<void> {
    const polyfillsToLoad: Promise<any>[] = [];
    
    // Check Symbol support
    if (typeof Symbol === 'undefined') {
      polyfillsToLoad.push(import('core-js/features/symbol'));
    }
    
    // Check Map support
    if (typeof Map === 'undefined') {
      polyfillsToLoad.push(import('core-js/features/map'));
    }
    
    // Check Set support
    if (typeof Set === 'undefined') {
      polyfillsToLoad.push(import('core-js/features/set'));
    }
    
    // Check Promise support
    if (typeof Promise === 'undefined') {
      polyfillsToLoad.push(import('core-js/features/promise'));
    }
    
    // Check Object.assign support
    if (typeof Object.assign !== 'function') {
      polyfillsToLoad.push(import('core-js/features/object/assign'));
    }
    
    if (polyfillsToLoad.length > 0) {
      console.log(`Loading ${polyfillsToLoad.length} polyfills...`);
      await Promise.all(polyfillsToLoad);
      console.log('Polyfills loaded successfully');
    }
  }
}

// Usage in main.ts
import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';
import { AppModule } from './app/app.module';
import { DynamicPolyfillLoader } from './polyfills/dynamic-loader';

async function bootstrap() {
  // Load polyfills first
  await DynamicPolyfillLoader.loadPolyfillsIfNeeded();
  
  // Then bootstrap Angular
  platformBrowserDynamic()
    .bootstrapModule(AppModule)
    .catch(err => console.error(err));
}

bootstrap();
```

### Conditional RxJS Loading

```typescript
// Conditional RxJS feature loading
export class ConditionalRxJSLoader {
  static async loadRxJSFeatures(): Promise<void> {
    // Basic RxJS always loads
    const { Observable } = await import('rxjs');
    const { map, filter } = await import('rxjs/operators');
    
    // Advanced features only on modern browsers
    if (this.isModernBrowser()) {
      const { ajax } = await import('rxjs/ajax');
      const { webSocket } = await import('rxjs/webSocket');
    }
    
    // Polyfill-dependent features
    if (this.hasRequiredFeatures()) {
      const { shareReplay, publishReplay } = await import('rxjs/operators');
    }
  }
  
  private static isModernBrowser(): boolean {
    // Check for modern browser features
    return !!(
      window.fetch &&
      window.Promise &&
      window.Symbol &&
      window.Map &&
      window.Set
    );
  }
  
  private static hasRequiredFeatures(): boolean {
    return !!(
      typeof WeakMap !== 'undefined' &&
      typeof WeakSet !== 'undefined'
    );
  }
}
```

## Testing Browser Compatibility

### Automated Testing Setup

```typescript
// Browser compatibility testing service
export class CompatibilityTester {
  static async runCompatibilityTests(): Promise<TestResults> {
    const results: TestResults = {
      browser: this.getBrowserInfo(),
      features: await this.testFeatures(),
      rxjs: await this.testRxJSFunctionality(),
      performance: await this.testPerformance()
    };
    
    return results;
  }
  
  private static async testFeatures(): Promise<FeatureTestResults> {
    return {
      symbol: this.testSymbol(),
      map: this.testMap(),
      set: this.testSet(),
      promise: this.testPromise(),
      objectAssign: this.testObjectAssign()
    };
  }
  
  private static async testRxJSFunctionality(): Promise<RxJSTestResults> {
    try {
      const { Observable } = await import('rxjs');
      const { map, filter } = await import('rxjs/operators');
      
      // Test basic Observable functionality
      const basicTest = await this.testBasicObservable(Observable, map);
      
      // Test operator chaining
      const operatorTest = await this.testOperatorChaining(Observable, map, filter);
      
      return {
        basic: basicTest,
        operators: operatorTest,
        success: basicTest && operatorTest
      };
    } catch (error) {
      return {
        basic: false,
        operators: false,
        success: false,
        error: error.message
      };
    }
  }
  
  private static testBasicObservable(Observable: any, map: any): Promise<boolean> {
    return new Promise((resolve) => {
      try {
        const obs$ = new Observable(subscriber => {
          subscriber.next(1);
          subscriber.complete();
        });
        
        obs$.pipe(map(x => x * 2)).subscribe({
          next: (value) => resolve(value === 2),
          error: () => resolve(false)
        });
      } catch (error) {
        resolve(false);
      }
    });
  }
}

interface TestResults {
  browser: BrowserInfo;
  features: FeatureTestResults;
  rxjs: RxJSTestResults;
  performance: PerformanceTestResults;
}
```

### Cross-Browser Testing

```typescript
// Cross-browser testing configuration
export const BrowserTestMatrix = {
  desktop: [
    { name: 'Chrome', versions: ['latest', 'latest-1'] },
    { name: 'Firefox', versions: ['latest', 'latest-1'] },
    { name: 'Safari', versions: ['latest', 'latest-1'] },
    { name: 'Edge', versions: ['latest', 'latest-1'] }
  ],
  
  mobile: [
    { name: 'Chrome Mobile', versions: ['latest'] },
    { name: 'Safari Mobile', versions: ['latest'] },
    { name: 'Samsung Internet', versions: ['latest'] }
  ],
  
  legacy: [
    { name: 'IE', versions: ['11'] },
    { name: 'Chrome', versions: ['60', '70'] },
    { name: 'Safari', versions: ['12', '13'] }
  ]
};

// Automated testing runner
export class CrossBrowserTester {
  static async runTestSuite(): Promise<BrowserTestReport[]> {
    const results: BrowserTestReport[] = [];
    
    for (const category of Object.keys(BrowserTestMatrix)) {
      const browsers = BrowserTestMatrix[category];
      
      for (const browser of browsers) {
        for (const version of browser.versions) {
          const result = await this.testBrowserVersion(browser.name, version);
          results.push(result);
        }
      }
    }
    
    return results;
  }
  
  private static async testBrowserVersion(
    browserName: string, 
    version: string
  ): Promise<BrowserTestReport> {
    // This would integrate with tools like Selenium, Puppeteer, or BrowserStack
    console.log(`Testing ${browserName} ${version}...`);
    
    // Run compatibility tests
    const compatibilityResult = await CompatibilityTester.runCompatibilityTests();
    
    return {
      browser: browserName,
      version,
      passed: compatibilityResult.rxjs.success,
      features: compatibilityResult.features,
      errors: compatibilityResult.rxjs.error ? [compatibilityResult.rxjs.error] : []
    };
  }
}
```

## Performance Considerations

### Polyfill Performance Impact

```typescript
// Performance monitoring for polyfills
export class PolyfillPerformanceMonitor {
  private static metrics: PerformanceMetric[] = [];
  
  static measurePolyfillImpact(): PolyfillImpactReport {
    const baselinePerformance = this.measureBaseline();
    const polyfillPerformance = this.measureWithPolyfills();
    
    return {
      baseline: baselinePerformance,
      withPolyfills: polyfillPerformance,
      impact: {
        loadTime: polyfillPerformance.loadTime - baselinePerformance.loadTime,
        memoryUsage: polyfillPerformance.memoryUsage - baselinePerformance.memoryUsage,
        bundleSize: polyfillPerformance.bundleSize - baselinePerformance.bundleSize
      },
      recommendations: this.generateRecommendations(polyfillPerformance)
    };
  }
  
  private static measureBaseline(): PerformanceMetrics {
    return {
      loadTime: performance.now(),
      memoryUsage: this.getMemoryUsage(),
      bundleSize: this.getBundleSize()
    };
  }
  
  private static generateRecommendations(metrics: PerformanceMetrics): string[] {
    const recommendations = [];
    
    if (metrics.loadTime > 1000) {
      recommendations.push('Consider lazy loading polyfills');
    }
    
    if (metrics.bundleSize > 100000) {
      recommendations.push('Use selective polyfill loading');
    }
    
    if (metrics.memoryUsage > 50000000) {
      recommendations.push('Monitor memory usage with polyfills');
    }
    
    return recommendations;
  }
}

interface PerformanceMetrics {
  loadTime: number;
  memoryUsage: number;
  bundleSize: number;
}
```

### Optimized Polyfill Strategy

```typescript
// Optimized polyfill loading strategy
export class OptimizedPolyfillStrategy {
  private static cache = new Map<string, Promise<any>>();
  
  static async loadOptimizedPolyfills(): Promise<void> {
    // Check what's actually needed
    const needed = this.analyzeNeeds();
    
    // Load only required polyfills
    const loadPromises = needed.map(polyfill => 
      this.loadPolyfillCached(polyfill)
    );
    
    await Promise.all(loadPromises);
  }
  
  private static analyzeNeeds(): string[] {
    const needed: string[] = [];
    
    // Only check features actually used by your app
    if (this.usesSymbol() && typeof Symbol === 'undefined') {
      needed.push('core-js/features/symbol');
    }
    
    if (this.usesMap() && typeof Map === 'undefined') {
      needed.push('core-js/features/map');
    }
    
    if (this.usesPromise() && typeof Promise === 'undefined') {
      needed.push('core-js/features/promise');
    }
    
    return needed;
  }
  
  private static async loadPolyfillCached(polyfill: string): Promise<void> {
    if (!this.cache.has(polyfill)) {
      this.cache.set(polyfill, import(polyfill));
    }
    
    await this.cache.get(polyfill);
  }
  
  // Feature usage detection
  private static usesSymbol(): boolean {
    // Detect if your app actually uses Symbol
    return document.querySelector('[data-uses-symbol]') !== null;
  }
  
  private static usesMap(): boolean {
    // Detect if your app actually uses Map
    return document.querySelector('[data-uses-map]') !== null;
  }
  
  private static usesPromise(): boolean {
    // Detect if your app actually uses Promise
    return document.querySelector('[data-uses-promise]') !== null;
  }
}
```

## Common Browser Issues

### Internet Explorer 11 Issues

```typescript
// IE11 specific fixes and workarounds
export class IE11Compatibility {
  static applyFixes(): void {
    // Fix Symbol.observable polyfill
    this.fixSymbolObservable();
    
    // Fix Array.from issues
    this.fixArrayFrom();
    
    // Fix Object.assign edge cases
    this.fixObjectAssign();
  }
  
  private static fixSymbolObservable(): void {
    if (typeof Symbol !== 'undefined' && !Symbol.observable) {
      Symbol.observable = Symbol('observable');
    }
  }
  
  private static fixArrayFrom(): void {
    // IE11 Array.from doesn't handle iterables correctly
    if (!Array.from || !this.testArrayFrom()) {
      Array.from = function(arrayLike: any, mapFn?: any, thisArg?: any) {
        const items = Object(arrayLike);
        if (arrayLike == null) {
          throw new TypeError('Array.from requires an array-like object');
        }
        
        const len = parseInt(items.length) || 0;
        const result = typeof this === 'function' ? Object(new this(len)) : new Array(len);
        
        for (let i = 0; i < len; i++) {
          const value = items[i];
          result[i] = mapFn ? mapFn.call(thisArg, value, i) : value;
        }
        
        result.length = len;
        return result;
      };
    }
  }
  
  private static testArrayFrom(): boolean {
    try {
      Array.from([1, 2, 3]);
      return true;
    } catch {
      return false;
    }
  }
}
```

### Safari Issues

```typescript
// Safari specific compatibility fixes
export class SafariCompatibility {
  static applyFixes(): void {
    // Fix Symbol.iterator in older Safari versions
    this.fixSymbolIterator();
    
    // Fix Promise constructor issues
    this.fixPromiseConstructor();
  }
  
  private static fixSymbolIterator(): void {
    if (typeof Symbol !== 'undefined' && !Symbol.iterator) {
      Symbol.iterator = Symbol('Symbol.iterator');
    }
  }
  
  private static fixPromiseConstructor(): void {
    // Some Safari versions have Promise but with issues
    if (typeof Promise !== 'undefined') {
      try {
        new Promise(resolve => resolve(1));
      } catch {
        // Promise constructor is broken, load polyfill
        import('core-js/features/promise');
      }
    }
  }
}
```

## Progressive Enhancement

### Feature-Based Loading

```typescript
// Progressive enhancement strategy
export class ProgressiveEnhancement {
  static async enhanceApplication(): Promise<void> {
    // Start with basic functionality
    await this.loadCore();
    
    // Add features based on browser capabilities
    if (this.supportsAdvancedFeatures()) {
      await this.loadAdvancedFeatures();
    }
    
    // Add experimental features for cutting-edge browsers
    if (this.supportsCuttingEdge()) {
      await this.loadExperimentalFeatures();
    }
  }
  
  private static async loadCore(): Promise<void> {
    // Basic RxJS functionality that works everywhere
    const { Observable } = await import('rxjs');
    const { map, filter } = await import('rxjs/operators');
    
    // Register as globally available
    (window as any).RxJS = { Observable, operators: { map, filter } };
  }
  
  private static async loadAdvancedFeatures(): Promise<void> {
    // Advanced operators and features
    const operators = await import('rxjs/operators');
    const { ajax } = await import('rxjs/ajax');
    
    // Extend global RxJS object
    Object.assign((window as any).RxJS.operators, operators);
    (window as any).RxJS.ajax = ajax;
  }
  
  private static async loadExperimentalFeatures(): Promise<void> {
    // Experimental or newer features
    if (this.supportsWebSocket()) {
      const { webSocket } = await import('rxjs/webSocket');
      (window as any).RxJS.webSocket = webSocket;
    }
  }
  
  private static supportsAdvancedFeatures(): boolean {
    return !!(
      window.Map &&
      window.Set &&
      window.WeakMap &&
      window.Promise
    );
  }
  
  private static supportsCuttingEdge(): boolean {
    return !!(
      window.fetch &&
      window.AbortController &&
      window.IntersectionObserver
    );
  }
  
  private static supportsWebSocket(): boolean {
    return typeof WebSocket !== 'undefined';
  }
}
```

## Legacy Browser Support

### IE11 Support Strategy

```typescript
// Complete IE11 support implementation
export class IE11Support {
  static async setupComplete(): Promise<void> {
    console.log('Setting up IE11 support...');
    
    // Load all required polyfills
    await this.loadPolyfills();
    
    // Apply compatibility fixes
    this.applyCompatibilityFixes();
    
    // Set up performance monitoring
    this.setupPerformanceMonitoring();
    
    console.log('IE11 support configured successfully');
  }
  
  private static async loadPolyfills(): Promise<void> {
    const polyfills = [
      'core-js/features/symbol',
      'core-js/features/symbol/iterator',
      'core-js/features/map',
      'core-js/features/set',
      'core-js/features/weak-map',
      'core-js/features/promise',
      'core-js/features/object/assign',
      'core-js/features/array/from',
      'core-js/features/array/find',
      'core-js/features/array/includes',
      'whatwg-fetch'
    ];
    
    await Promise.all(polyfills.map(p => import(p)));
  }
  
  private static applyCompatibilityFixes(): void {
    // Fix console methods
    this.fixConsole();
    
    // Fix event handling
    this.fixEventHandling();
    
    // Apply IE11 specific RxJS fixes
    IE11Compatibility.applyFixes();
  }
  
  private static fixConsole(): void {
    // IE11 console methods aren't always available
    const consoleMethods = ['log', 'warn', 'error', 'info', 'debug'];
    
    consoleMethods.forEach(method => {
      if (typeof console[method] !== 'function') {
        console[method] = function() {
          // Fallback implementation
        };
      }
    });
  }
  
  private static setupPerformanceMonitoring(): void {
    // Monitor performance in IE11
    if (typeof performance === 'undefined') {
      (window as any).performance = {
        now: () => Date.now(),
        mark: () => {},
        measure: () => {}
      };
    }
  }
}
```

## Best Practices

### Polyfill Best Practices

```typescript
// Best practices for polyfill management
export const PolyfillBestPractices = {
  // 1. Load polyfills conditionally
  loadConditionally: async () => {
    // Only load what's actually needed
    if (typeof Map === 'undefined') {
      await import('core-js/features/map');
    }
  },
  
  // 2. Use feature detection, not browser detection
  useFeatureDetection: () => {
    // ✅ GOOD
    if (typeof Symbol === 'undefined') {
      // Load Symbol polyfill
    }
    
    // ❌ BAD
    // if (navigator.userAgent.includes('IE')) {
    //   // Load all polyfills
    // }
  },
  
  // 3. Minimize polyfill bundle size
  minimizeSize: () => {
    // Use specific imports, not entire libraries
    // ✅ GOOD
    import('core-js/features/map');
    
    // ❌ BAD
    // import('core-js');
  },
  
  // 4. Test polyfill functionality
  testFunctionality: () => {
    // Always test that polyfills work as expected
    const map = new Map();
    map.set('test', 'value');
    console.assert(map.get('test') === 'value', 'Map polyfill failed');
  }
};

// Comprehensive setup function
export class BrowserCompatibilitySetup {
  static async setup(): Promise<void> {
    try {
      // 1. Detect browser and features
      const browserInfo = this.detectBrowser();
      const features = this.detectFeatures();
      
      // 2. Load required polyfills
      await this.loadPolyfills(browserInfo, features);
      
      // 3. Apply browser-specific fixes
      this.applyBrowserFixes(browserInfo);
      
      // 4. Test functionality
      this.testFunctionality();
      
      // 5. Report status
      this.reportStatus(browserInfo, features);
      
    } catch (error) {
      console.error('Browser compatibility setup failed:', error);
      throw error;
    }
  }
  
  private static async loadPolyfills(browser: any, features: any): Promise<void> {
    if (browser.isIE11) {
      await IE11Support.setupComplete();
    } else if (browser.isOldSafari) {
      await SafariCompatibility.applyFixes();
    } else {
      await OptimizedPolyfillStrategy.loadOptimizedPolyfills();
    }
  }
  
  private static testFunctionality(): void {
    // Test core RxJS functionality
    try {
      const { Observable } = require('rxjs');
      const { map } = require('rxjs/operators');
      
      // Quick test
      new Observable(sub => sub.next(1))
        .pipe(map(x => x * 2))
        .subscribe();
        
      console.log('✅ RxJS compatibility test passed');
    } catch (error) {
      console.error('❌ RxJS compatibility test failed:', error);
      throw new Error('RxJS is not compatible with this browser');
    }
  }
}
```

## Summary

Browser compatibility for RxJS applications requires careful consideration of:

1. **Feature Detection**: Use feature detection rather than browser detection
2. **Selective Polyfilling**: Load only the polyfills you actually need
3. **Performance Impact**: Monitor the cost of polyfills on load time and bundle size
4. **Testing**: Test across target browsers and versions
5. **Progressive Enhancement**: Start with basic functionality and enhance based on capabilities

Key polyfills for RxJS:
- **Symbol** (required for iterators)
- **Map/Set** (used internally)
- **Promise** (for async operations)
- **Object.assign** (for object merging)

By following these practices, you can ensure your RxJS applications work reliably across all target browsers while maintaining optimal performance.

---

**Next:** [Real-World Applications & Patterns](../08-real-world-applications/01-project-setup.md)
**Previous:** [Bundle Optimization](./04-bundle-optimization.md)
