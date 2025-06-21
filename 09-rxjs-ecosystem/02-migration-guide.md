# RxJS Migration Guide

## Introduction

This comprehensive migration guide covers transitioning between different versions of RxJS, handling breaking changes, and modernizing your reactive codebase. We'll explore version-specific changes, automated migration tools, and best practices for smooth upgrades.

## Learning Objectives

- Understand major RxJS version differences and breaking changes
- Learn migration strategies for different RxJS versions
- Master automated migration tools and techniques
- Handle legacy code modernization effectively
- Implement backward compatibility patterns
- Plan and execute large-scale RxJS upgrades

## RxJS Version History & Breaking Changes

### 1. RxJS 5 to RxJS 6 Migration

#### Major Changes Overview

```typescript
// RxJS 5 - Old Import Style
import { Observable } from 'rxjs/Observable';
import { Subject } from 'rxjs/Subject';
import 'rxjs/add/operator/map';
import 'rxjs/add/operator/filter';
import 'rxjs/add/operator/debounceTime';

// RxJS 6 - New Import Style
import { Observable, Subject } from 'rxjs';
import { map, filter, debounceTime } from 'rxjs/operators';
```

#### Operator Import Changes

```typescript
// migration-v5-to-v6.ts

// ❌ RxJS 5 - Patch Operators
import { Observable } from 'rxjs/Observable';
import 'rxjs/add/operator/map';
import 'rxjs/add/operator/filter';
import 'rxjs/add/operator/switchMap';

const example$ = new Observable(subscriber => {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);
}).map(x => x * 2)
  .filter(x => x > 2)
  .switchMap(x => Observable.of(x));

// ✅ RxJS 6 - Pipeable Operators
import { Observable } from 'rxjs';
import { map, filter, switchMap } from 'rxjs/operators';
import { of } from 'rxjs';

const example$ = new Observable(subscriber => {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);
}).pipe(
  map(x => x * 2),
  filter(x => x > 2),
  switchMap(x => of(x))
);
```

#### Creation Function Changes

```typescript
// creation-functions-migration.ts

// ❌ RxJS 5 - Static Methods
import { Observable } from 'rxjs/Observable';

const from$ = Observable.from([1, 2, 3]);
const of$ = Observable.of(1, 2, 3);
const timer$ = Observable.timer(1000);
const interval$ = Observable.interval(1000);
const empty$ = Observable.empty();
const never$ = Observable.never();
const throw$ = Observable.throw(new Error('error'));

// ✅ RxJS 6 - Standalone Functions
import { from, of, timer, interval, EMPTY, NEVER, throwError } from 'rxjs';

const from$ = from([1, 2, 3]);
const of$ = of(1, 2, 3);
const timer$ = timer(1000);
const interval$ = interval(1000);
const empty$ = EMPTY;
const never$ = NEVER;
const throw$ = throwError(new Error('error'));
```

#### Automated Migration Tool

```typescript
// rxjs-migration-tool.ts
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root'
})
export class RxJSMigrationToolService {
  
  // Migration patterns for automated replacement
  private migrationPatterns = {
    imports: [
      {
        pattern: /import\s*{\s*Observable\s*}\s*from\s*['"]rxjs\/Observable['"]/g,
        replacement: "import { Observable } from 'rxjs'"
      },
      {
        pattern: /import\s*{\s*Subject\s*}\s*from\s*['"]rxjs\/Subject['"]/g,
        replacement: "import { Subject } from 'rxjs'"
      },
      {
        pattern: /import\s*['"]rxjs\/add\/operator\/(\w+)['"]/g,
        replacement: "// Migrated: import { $1 } from 'rxjs/operators'"
      }
    ],
    operators: [
      {
        pattern: /\.map\(/g,
        replacement: '.pipe(map('
      },
      {
        pattern: /\.filter\(/g,
        replacement: '.pipe(filter('
      },
      {
        pattern: /\.switchMap\(/g,
        replacement: '.pipe(switchMap('
      },
      {
        pattern: /\.mergeMap\(/g,
        replacement: '.pipe(mergeMap('
      },
      {
        pattern: /\.flatMap\(/g,
        replacement: '.pipe(mergeMap('
      },
      {
        pattern: /\.do\(/g,
        replacement: '.pipe(tap('
      }
    ],
    creationFunctions: [
      {
        pattern: /Observable\.from\(/g,
        replacement: 'from('
      },
      {
        pattern: /Observable\.of\(/g,
        replacement: 'of('
      },
      {
        pattern: /Observable\.timer\(/g,
        replacement: 'timer('
      },
      {
        pattern: /Observable\.interval\(/g,
        replacement: 'interval('
      },
      {
        pattern: /Observable\.empty\(\)/g,
        replacement: 'EMPTY'
      },
      {
        pattern: /Observable\.never\(\)/g,
        replacement: 'NEVER'
      },
      {
        pattern: /Observable\.throw\(/g,
        replacement: 'throwError('
      }
    ]
  };
  
  // Migrate a single file
  migrateFile(fileContent: string): MigrationResult {
    let migratedContent = fileContent;
    const appliedMigrations: string[] = [];
    
    // Apply import migrations
    this.migrationPatterns.imports.forEach(pattern => {
      if (pattern.pattern.test(migratedContent)) {
        migratedContent = migratedContent.replace(pattern.pattern, pattern.replacement);
        appliedMigrations.push(`Import: ${pattern.pattern.source}`);
      }
    });
    
    // Apply operator migrations
    migratedContent = this.migrateOperatorChains(migratedContent, appliedMigrations);
    
    // Apply creation function migrations
    this.migrationPatterns.creationFunctions.forEach(pattern => {
      if (pattern.pattern.test(migratedContent)) {
        migratedContent = migratedContent.replace(pattern.pattern, pattern.replacement);
        appliedMigrations.push(`Creation: ${pattern.pattern.source}`);
      }
    });
    
    // Add required imports
    migratedContent = this.addRequiredImports(migratedContent);
    
    return {
      originalContent: fileContent,
      migratedContent,
      appliedMigrations,
      requiresManualReview: this.requiresManualReview(migratedContent)
    };
  }
  
  private migrateOperatorChains(content: string, appliedMigrations: string[]): string {
    // Complex regex to handle operator chains
    const operatorChainPattern = /(\w+\$[^;]*?)(\.[a-zA-Z]+\([^)]*\))([^;]*?)(;|\))/g;
    
    return content.replace(operatorChainPattern, (match, start, operators, end, terminator) => {
      const operatorMatches = operators.match(/\.[a-zA-Z]+\([^)]*\)/g);
      
      if (operatorMatches && operatorMatches.length > 1) {
        // Convert to pipe syntax
        const pipeOperators = operatorMatches
          .map(op => op.substring(1)) // Remove leading dot
          .join(', ');
        
        appliedMigrations.push(`Operator chain: ${operatorMatches.join('')}`);
        return `${start}.pipe(${pipeOperators})${end}${terminator}`;
      }
      
      return match;
    });
  }
  
  private addRequiredImports(content: string): string {
    const requiredOperators = this.extractUsedOperators(content);
    const requiredCreationFunctions = this.extractUsedCreationFunctions(content);
    
    let importsToAdd = '';
    
    if (requiredOperators.length > 0) {
      importsToAdd += `import { ${requiredOperators.join(', ')} } from 'rxjs/operators';\n`;
    }
    
    if (requiredCreationFunctions.length > 0) {
      importsToAdd += `import { ${requiredCreationFunctions.join(', ')} } from 'rxjs';\n`;
    }
    
    // Insert imports after existing imports
    const importSection = content.match(/^(import[^;]+;[\s\S]*?)(?=\n\n|\nexport|\nclass|\ninterface|\nconst|\nlet|\nvar)/m);
    
    if (importSection && importsToAdd) {
      return content.replace(importSection[0], importSection[0] + '\n' + importsToAdd);
    }
    
    return importsToAdd + content;
  }
  
  private extractUsedOperators(content: string): string[] {
    const operatorPattern = /\.pipe\([^)]*(\w+)\(/g;
    const operators = new Set<string>();
    let match;
    
    while ((match = operatorPattern.exec(content)) !== null) {
      operators.add(match[1]);
    }
    
    return Array.from(operators);
  }
  
  private extractUsedCreationFunctions(content: string): string[] {
    const creationFunctions = ['from', 'of', 'timer', 'interval', 'EMPTY', 'NEVER', 'throwError'];
    const used = new Set<string>();
    
    creationFunctions.forEach(func => {
      if (content.includes(func + '(') || content.includes(func)) {
        used.add(func);
      }
    });
    
    return Array.from(used);
  }
  
  private requiresManualReview(content: string): boolean {
    // Check for patterns that require manual review
    const manualReviewPatterns = [
      /Observable\.create/,
      /Observable\.bindCallback/,
      /Observable\.bindNodeCallback/,
      /\.do\(/,  // Should be .tap()
      /\.finally\(/,  // Should be .finalize()
      /\.catch\(/,  // Should be .catchError()
      /\.switch\(/,  // Should be .switchAll()
      /\.merge\(/,  // Context-dependent
      /\.concat\(/,  // Context-dependent
    ];
    
    return manualReviewPatterns.some(pattern => pattern.test(content));
  }
}

// Usage example
export interface MigrationResult {
  originalContent: string;
  migratedContent: string;
  appliedMigrations: string[];
  requiresManualReview: boolean;
}
```

### 2. RxJS 6 to RxJS 7 Migration

#### Breaking Changes in RxJS 7

```typescript
// rxjs-7-migration.ts

// ❌ RxJS 6 - Deprecated operators
import { combineLatest } from 'rxjs';
import { toPromise } from 'rxjs/operators';

// Deprecated toPromise()
observable$.pipe(
  toPromise()
).then(result => console.log(result));

// Deprecated combineLatest with array
combineLatest([obs1$, obs2$, obs3$]);

// ✅ RxJS 7 - Modern alternatives
import { combineLatest, firstValueFrom, lastValueFrom } from 'rxjs';

// Use firstValueFrom or lastValueFrom
firstValueFrom(observable$)
  .then(result => console.log(result));

lastValueFrom(observable$)
  .then(result => console.log(result));

// Use object syntax for combineLatest
combineLatest({
  first: obs1$,
  second: obs2$,
  third: obs3$
});
```

#### Null and Undefined Handling

```typescript
// null-undefined-handling.ts

// ❌ RxJS 6 - Less strict
import { of } from 'rxjs';
import { map } from 'rxjs/operators';

of(null, undefined, 0, '').pipe(
  map(x => x?.toString()) // May cause issues
);

// ✅ RxJS 7 - Stricter null/undefined handling
import { of } from 'rxjs';
import { map, filter } from 'rxjs/operators';

of(null, undefined, 0, '').pipe(
  filter((x): x is NonNullable<typeof x> => x != null),
  map(x => x.toString()) // Type-safe
);
```

### 3. RxJS 7 to RxJS 8+ Migration

#### Modern RxJS Features

```typescript
// rxjs-8-features.ts

// ✅ Enhanced type safety
import { Observable } from 'rxjs';
import { map, filter } from 'rxjs/operators';

// Better type inference
const numbers$ = new Observable<number>(subscriber => {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);
});

const strings$ = numbers$.pipe(
  map(n => n.toString()) // Automatically typed as Observable<string>
);

// ✅ Improved error handling
import { catchError, retry } from 'rxjs/operators';
import { throwError } from 'rxjs';

const resilientOperation$ = numbers$.pipe(
  map(n => {
    if (n === 2) throw new Error('Invalid number');
    return n * 2;
  }),
  retry({
    count: 3,
    delay: (error, retryIndex) => timer(retryIndex * 1000)
  }),
  catchError(error => {
    console.error('Operation failed:', error);
    return throwError(() => error);
  })
);
```

## Comprehensive Migration Strategy

### 1. Pre-Migration Assessment

```typescript
// migration-assessment.service.ts
import { Injectable } from '@angular/core';

export interface ProjectAssessment {
  currentRxJSVersion: string;
  targetVersion: string;
  filesToMigrate: string[];
  complexityScore: number;
  estimatedEffort: string;
  riskFactors: string[];
  dependencies: PackageDependency[];
}

export interface PackageDependency {
  name: string;
  version: string;
  rxjsCompatibility: 'compatible' | 'needs-update' | 'incompatible';
}

@Injectable({
  providedIn: 'root'
})
export class MigrationAssessmentService {
  
  // Analyze project for migration readiness
  assessProject(projectPath: string): Observable<ProjectAssessment> {
    return this.scanProjectFiles(projectPath).pipe(
      switchMap(files => this.analyzeFiles(files)),
      map(analysis => this.generateAssessment(analysis))
    );
  }
  
  private scanProjectFiles(projectPath: string): Observable<string[]> {
    return from(this.getTypeScriptFiles(projectPath));
  }
  
  private analyzeFiles(files: string[]): Observable<FileAnalysis[]> {
    return from(files).pipe(
      mergeMap(file => this.analyzeFile(file), 5), // Parallel analysis
      toArray()
    );
  }
  
  private analyzeFile(filePath: string): Observable<FileAnalysis> {
    return this.readFile(filePath).pipe(
      map(content => ({
        filePath,
        rxjsUsage: this.detectRxJSUsage(content),
        migrationComplexity: this.calculateComplexity(content),
        issues: this.identifyIssues(content)
      }))
    );
  }
  
  private detectRxJSUsage(content: string): RxJSUsage {
    return {
      hasObservables: /Observable|Subject|BehaviorSubject|ReplaySubject/.test(content),
      hasOperators: /\.pipe\(|\.map\(|\.filter\(|\.switchMap\(/.test(content),
      hasCreationFunctions: /of\(|from\(|timer\(|interval\(/.test(content),
      hasDeprecatedPatterns: this.hasDeprecatedPatterns(content),
      importStyle: this.detectImportStyle(content)
    };
  }
  
  private calculateComplexity(content: string): number {
    let complexity = 0;
    
    // Operator chain complexity
    const operatorChains = content.match(/\.pipe\([^}]+\)/g) || [];
    complexity += operatorChains.reduce((sum, chain) => {
      const operators = (chain.match(/,/g) || []).length + 1;
      return sum + Math.min(operators, 10); // Cap at 10
    }, 0);
    
    // Custom operator complexity
    const customOperators = content.match(/return\s+\(source:\s*Observable/g) || [];
    complexity += customOperators.length * 5;
    
    // Subject usage complexity
    const subjects = content.match(/(Subject|BehaviorSubject|ReplaySubject)/g) || [];
    complexity += subjects.length * 2;
    
    return complexity;
  }
  
  private identifyIssues(content: string): MigrationIssue[] {
    const issues: MigrationIssue[] = [];
    
    // Deprecated imports
    if (/import.*['"]rxjs\/Observable['"]/.test(content)) {
      issues.push({
        type: 'deprecated-import',
        severity: 'error',
        message: 'Uses deprecated Observable import',
        line: this.findLineNumber(content, /import.*['"]rxjs\/Observable['"]/)
      });
    }
    
    // Patch operators
    if (/import\s*['"]rxjs\/add\/operator/.test(content)) {
      issues.push({
        type: 'patch-operator',
        severity: 'error',
        message: 'Uses patch operators',
        line: this.findLineNumber(content, /import\s*['"]rxjs\/add\/operator/)
      });
    }
    
    // toPromise() usage
    if (/\.toPromise\(\)/.test(content)) {
      issues.push({
        type: 'deprecated-method',
        severity: 'warning',
        message: 'Uses deprecated toPromise() method',
        line: this.findLineNumber(content, /\.toPromise\(\)/)
      });
    }
    
    return issues;
  }
  
  private generateAssessment(analyses: FileAnalysis[]): ProjectAssessment {
    const filesToMigrate = analyses
      .filter(analysis => analysis.rxjsUsage.hasObservables)
      .map(analysis => analysis.filePath);
    
    const totalComplexity = analyses.reduce((sum, analysis) => sum + analysis.migrationComplexity, 0);
    const totalIssues = analyses.reduce((sum, analysis) => sum + analysis.issues.length, 0);
    
    return {
      currentRxJSVersion: this.getCurrentRxJSVersion(),
      targetVersion: '7.0.0',
      filesToMigrate,
      complexityScore: totalComplexity,
      estimatedEffort: this.estimateEffort(totalComplexity, totalIssues),
      riskFactors: this.identifyRiskFactors(analyses),
      dependencies: this.analyzeDependencies()
    };
  }
  
  private estimateEffort(complexity: number, issues: number): string {
    const totalWork = complexity + (issues * 2);
    
    if (totalWork < 20) return 'Small (1-2 days)';
    if (totalWork < 100) return 'Medium (1-2 weeks)';
    if (totalWork < 300) return 'Large (2-4 weeks)';
    return 'Very Large (1+ months)';
  }
}
```

### 2. Automated Migration Pipeline

```typescript
// migration-pipeline.service.ts
import { Injectable } from '@angular/core';

export interface MigrationStep {
  name: string;
  description: string;
  execute: (context: MigrationContext) => Observable<MigrationStepResult>;
  rollback?: (context: MigrationContext) => Observable<void>;
}

export interface MigrationContext {
  projectPath: string;
  sourceVersion: string;
  targetVersion: string;
  options: MigrationOptions;
  backup?: string;
}

@Injectable({
  providedIn: 'root'
})
export class MigrationPipelineService {
  
  private migrationSteps: MigrationStep[] = [
    {
      name: 'backup',
      description: 'Create project backup',
      execute: (context) => this.createBackup(context)
    },
    {
      name: 'dependency-check',
      description: 'Check dependency compatibility',
      execute: (context) => this.checkDependencies(context)
    },
    {
      name: 'import-migration',
      description: 'Migrate import statements',
      execute: (context) => this.migrateImports(context),
      rollback: (context) => this.rollbackImports(context)
    },
    {
      name: 'operator-migration',
      description: 'Migrate operators to pipeable format',
      execute: (context) => this.migrateOperators(context),
      rollback: (context) => this.rollbackOperators(context)
    },
    {
      name: 'creation-function-migration',
      description: 'Migrate creation functions',
      execute: (context) => this.migrateCreationFunctions(context)
    },
    {
      name: 'type-check',
      description: 'Verify TypeScript compilation',
      execute: (context) => this.typeCheck(context)
    },
    {
      name: 'test-execution',
      description: 'Run test suite',
      execute: (context) => this.runTests(context)
    },
    {
      name: 'cleanup',
      description: 'Clean up temporary files',
      execute: (context) => this.cleanup(context)
    }
  ];
  
  // Execute complete migration pipeline
  executeMigration(context: MigrationContext): Observable<MigrationResult> {
    return new Observable(subscriber => {
      const results: MigrationStepResult[] = [];
      let currentStepIndex = 0;
      
      const executeNextStep = () => {
        if (currentStepIndex >= this.migrationSteps.length) {
          subscriber.next({
            success: true,
            steps: results,
            context
          });
          subscriber.complete();
          return;
        }
        
        const step = this.migrationSteps[currentStepIndex];
        console.log(`Executing step: ${step.name}`);
        
        step.execute(context).subscribe({
          next: (result) => {
            results.push(result);
            
            if (result.success) {
              currentStepIndex++;
              executeNextStep();
            } else {
              // Step failed, initiate rollback
              this.rollbackMigration(context, currentStepIndex - 1).subscribe({
                next: () => {
                  subscriber.next({
                    success: false,
                    steps: results,
                    context,
                    failedStep: step.name,
                    error: result.error
                  });
                  subscriber.complete();
                },
                error: (rollbackError) => {
                  subscriber.error(new Error(`Migration failed and rollback failed: ${rollbackError.message}`));
                }
              });
            }
          },
          error: (error) => {
            subscriber.error(error);
          }
        });
      };
      
      executeNextStep();
    });
  }
  
  private rollbackMigration(context: MigrationContext, lastSuccessfulStep: number): Observable<void> {
    const stepsToRollback = this.migrationSteps.slice(0, lastSuccessfulStep + 1).reverse();
    
    return from(stepsToRollback).pipe(
      concatMap(step => {
        if (step.rollback) {
          console.log(`Rolling back step: ${step.name}`);
          return step.rollback(context);
        }
        return of(void 0);
      }),
      toArray(),
      map(() => void 0)
    );
  }
  
  private createBackup(context: MigrationContext): Observable<MigrationStepResult> {
    const backupPath = `${context.projectPath}_backup_${Date.now()}`;
    
    return this.copyDirectory(context.projectPath, backupPath).pipe(
      map(() => {
        context.backup = backupPath;
        return {
          stepName: 'backup',
          success: true,
          message: `Backup created at ${backupPath}`
        };
      }),
      catchError(error => of({
        stepName: 'backup',
        success: false,
        error: error.message
      }))
    );
  }
  
  private migrateImports(context: MigrationContext): Observable<MigrationStepResult> {
    return this.getTypeScriptFiles(context.projectPath).pipe(
      mergeMap(files => from(files).pipe(
        mergeMap(file => this.migrateFileImports(file), 3),
        toArray()
      )),
      map(results => ({
        stepName: 'import-migration',
        success: results.every(r => r.success),
        message: `Migrated imports in ${results.length} files`,
        details: results
      }))
    );
  }
  
  private migrateFileImports(filePath: string): Observable<FileMigrationResult> {
    return this.readFile(filePath).pipe(
      map(content => {
        let migratedContent = content;
        
        // Migrate Observable imports
        migratedContent = migratedContent.replace(
          /import\s*{\s*Observable\s*}\s*from\s*['"]rxjs\/Observable['"]/g,
          "import { Observable } from 'rxjs'"
        );
        
        // Migrate Subject imports
        migratedContent = migratedContent.replace(
          /import\s*{\s*Subject\s*}\s*from\s*['"]rxjs\/Subject['"]/g,
          "import { Subject } from 'rxjs'"
        );
        
        // Remove patch operator imports
        migratedContent = migratedContent.replace(
          /import\s*['"]rxjs\/add\/operator\/\w+['"];\s*\n?/g,
          ''
        );
        
        return { filePath, content: migratedContent };
      }),
      switchMap(({ filePath, content }) => 
        this.writeFile(filePath, content).pipe(
          map(() => ({ filePath, success: true })),
          catchError(error => of({ filePath, success: false, error: error.message }))
        )
      )
    );
  }
  
  private typeCheck(context: MigrationContext): Observable<MigrationStepResult> {
    return this.runTypeScriptCompiler(context.projectPath).pipe(
      map(result => ({
        stepName: 'type-check',
        success: result.exitCode === 0,
        message: result.exitCode === 0 ? 'TypeScript compilation successful' : 'TypeScript compilation failed',
        details: result.output
      }))
    );
  }
  
  private runTests(context: MigrationContext): Observable<MigrationStepResult> {
    return this.executeTestSuite(context.projectPath).pipe(
      map(result => ({
        stepName: 'test-execution',
        success: result.passed,
        message: `Tests: ${result.passed}/${result.total} passed`,
        details: result
      }))
    );
  }
}
```

### 3. Manual Migration Patterns

```typescript
// manual-migration-patterns.ts

// Pattern 1: Complex Operator Chains
// ❌ Before (RxJS 5)
source$
  .map(x => x.value)
  .filter(x => x > 0)
  .switchMap(x => this.service.getData(x))
  .do(x => console.log(x))
  .catch(error => Observable.of(null));

// ✅ After (RxJS 6+)
source$.pipe(
  map(x => x.value),
  filter(x => x > 0),
  switchMap(x => this.service.getData(x)),
  tap(x => console.log(x)),
  catchError(error => of(null))
);

// Pattern 2: Custom Operators
// ❌ Before (RxJS 5)
Observable.prototype.debug = function(tag: string) {
  return this.do(
    next => console.log(`${tag}: next`, next),
    error => console.log(`${tag}: error`, error),
    () => console.log(`${tag}: complete`)
  );
};

// ✅ After (RxJS 6+)
export function debug<T>(tag: string): MonoTypeOperatorFunction<T> {
  return (source: Observable<T>) => source.pipe(
    tap({
      next: value => console.log(`${tag}: next`, value),
      error: error => console.log(`${tag}: error`, error),
      complete: () => console.log(`${tag}: complete`)
    })
  );
}

// Pattern 3: Subject Patterns
// ❌ Before (Complex inheritance)
class CustomSubject<T> extends Subject<T> {
  private _isActive = false;
  
  next(value: T) {
    if (this._isActive) {
      super.next(value);
    }
  }
}

// ✅ After (Composition over inheritance)
class CustomSubject<T> {
  private _subject = new Subject<T>();
  private _isActive = false;
  
  get asObservable(): Observable<T> {
    return this._subject.asObservable();
  }
  
  next(value: T) {
    if (this._isActive) {
      this._subject.next(value);
    }
  }
  
  activate() { this._isActive = true; }
  deactivate() { this._isActive = false; }
}
```

## Version-Specific Migration Tools

### 1. Angular CLI Integration

```typescript
// ng-update-migration.ts

// Angular CLI migration schematic
export function updateToRxJS7(): Rule {
  return (tree: Tree, context: SchematicContext) => {
    const packageJsonPath = '/package.json';
    const packageJson = JSON.parse(tree.read(packageJsonPath)!.toString());
    
    // Update RxJS version
    if (packageJson.dependencies?.rxjs) {
      packageJson.dependencies.rxjs = '^7.0.0';
    }
    
    tree.overwrite(packageJsonPath, JSON.stringify(packageJson, null, 2));
    
    // Apply code transformations
    return chain([
      updateSourceFiles(),
      updateTestFiles(),
      updateConfigFiles()
    ])(tree, context);
  };
}

function updateSourceFiles(): Rule {
  return (tree: Tree) => {
    tree.visit(path => {
      if (path.endsWith('.ts') && !path.endsWith('.d.ts')) {
        const content = tree.read(path)?.toString();
        if (content) {
          const updatedContent = applyRxJS7Migrations(content);
          tree.overwrite(path, updatedContent);
        }
      }
    });
  };
}
```

### 2. ESLint Rules for Migration

```typescript
// eslint-rxjs-migration.ts

// Custom ESLint rules for RxJS migration
export const rxjsMigrationRules = {
  'rxjs/no-deprecated-imports': {
    create(context: any) {
      return {
        ImportDeclaration(node: any) {
          const source = node.source.value;
          
          if (source === 'rxjs/Observable') {
            context.report({
              node,
              message: 'Use "import { Observable } from \'rxjs\'" instead',
              fix(fixer: any) {
                return fixer.replaceText(node.source, "'rxjs'");
              }
            });
          }
          
          if (source.startsWith('rxjs/add/operator/')) {
            const operator = source.split('/').pop();
            context.report({
              node,
              message: `Import ${operator} from 'rxjs/operators' instead`,
              fix(fixer: any) {
                return fixer.replaceText(
                  node.source, 
                  "'rxjs/operators'"
                );
              }
            });
          }
        }
      };
    }
  },
  
  'rxjs/no-patch-operators': {
    create(context: any) {
      return {
        MemberExpression(node: any) {
          if (node.property && node.property.name) {
            const operatorName = node.property.name;
            const patchOperators = ['map', 'filter', 'switchMap', 'mergeMap'];
            
            if (patchOperators.includes(operatorName)) {
              const parent = node.parent;
              if (parent.type === 'CallExpression' && parent.callee === node) {
                context.report({
                  node,
                  message: `Use pipe(${operatorName}(...)) instead of .${operatorName}(...)`,
                  fix(fixer: any) {
                    // Complex fix logic for converting to pipe syntax
                    return applyPipeFix(fixer, node, parent);
                  }
                });
              }
            }
          }
        }
      };
    }
  }
};
```

## Testing Migration Results

### 1. Migration Verification Suite

```typescript
// migration-verification.service.ts
import { Injectable } from '@angular/core';

export interface VerificationResult {
  passed: boolean;
  testName: string;
  details: string;
  performance?: PerformanceMetrics;
}

@Injectable({
  providedIn: 'root'
})
export class MigrationVerificationService {
  
  // Comprehensive verification suite
  verifyMigration(beforeSnapshot: ProjectSnapshot, afterSnapshot: ProjectSnapshot): Observable<VerificationReport> {
    return forkJoin({
      syntax: this.verifySyntax(afterSnapshot),
      functionality: this.verifyFunctionality(beforeSnapshot, afterSnapshot),
      performance: this.verifyPerformance(beforeSnapshot, afterSnapshot),
      typeChecking: this.verifyTypeChecking(afterSnapshot),
      bundleSize: this.verifyBundleSize(beforeSnapshot, afterSnapshot)
    }).pipe(
      map(results => this.generateReport(results))
    );
  }
  
  private verifySyntax(snapshot: ProjectSnapshot): Observable<VerificationResult> {
    return this.runTypeScriptCompiler(snapshot.projectPath).pipe(
      map(result => ({
        passed: result.exitCode === 0,
        testName: 'Syntax Verification',
        details: result.exitCode === 0 
          ? 'All files compile successfully' 
          : `Compilation errors: ${result.errors.length}`,
        errors: result.errors
      }))
    );
  }
  
  private verifyFunctionality(before: ProjectSnapshot, after: ProjectSnapshot): Observable<VerificationResult> {
    return forkJoin({
      beforeTests: this.runTestSuite(before.projectPath),
      afterTests: this.runTestSuite(after.projectPath)
    }).pipe(
      map(({ beforeTests, afterTests }) => ({
        passed: afterTests.passed >= beforeTests.passed,
        testName: 'Functionality Verification',
        details: `Before: ${beforeTests.passed}/${beforeTests.total}, After: ${afterTests.passed}/${afterTests.total}`,
        beforeResults: beforeTests,
        afterResults: afterTests
      }))
    );
  }
  
  private verifyPerformance(before: ProjectSnapshot, after: ProjectSnapshot): Observable<VerificationResult> {
    return forkJoin({
      beforeMetrics: this.measurePerformance(before.projectPath),
      afterMetrics: this.measurePerformance(after.projectPath)
    }).pipe(
      map(({ beforeMetrics, afterMetrics }) => {
        const improvement = this.calculateImprovement(beforeMetrics, afterMetrics);
        
        return {
          passed: improvement.bundleSize <= 1.1, // Allow 10% bundle size increase
          testName: 'Performance Verification',
          details: `Bundle size change: ${improvement.bundleSize}x, Runtime performance: ${improvement.runtime}x`,
          performance: {
            before: beforeMetrics,
            after: afterMetrics,
            improvement
          }
        };
      })
    );
  }
  
  // Test backward compatibility
  verifyBackwardCompatibility(snapshot: ProjectSnapshot): Observable<VerificationResult> {
    const compatibilityTests = [
      this.testObservableCreation,
      this.testOperatorChaining,
      this.testSubjectBehavior,
      this.testErrorHandling,
      this.testSchedulerBehavior
    ];
    
    return from(compatibilityTests).pipe(
      mergeMap(test => test(snapshot), 3),
      toArray(),
      map(results => ({
        passed: results.every(r => r.passed),
        testName: 'Backward Compatibility',
        details: `${results.filter(r => r.passed).length}/${results.length} compatibility tests passed`,
        testResults: results
      }))
    );
  }
  
  private testObservableCreation(snapshot: ProjectSnapshot): Observable<TestResult> {
    return this.executeTestCode(snapshot, `
      // Test Observable creation patterns
      const obs1 = new Observable(subscriber => subscriber.next(1));
      const obs2 = of(1, 2, 3);
      const obs3 = from([1, 2, 3]);
      const obs4 = timer(1000);
      
      // Verify they emit expected values
      return Promise.all([
        firstValueFrom(obs2),
        firstValueFrom(obs3),
        lastValueFrom(obs1)
      ]);
    `);
  }
  
  private testOperatorChaining(snapshot: ProjectSnapshot): Observable<TestResult> {
    return this.executeTestCode(snapshot, `
      // Test operator chaining
      const result = of(1, 2, 3).pipe(
        map(x => x * 2),
        filter(x => x > 2),
        reduce((acc, val) => acc + val, 0)
      );
      
      return firstValueFrom(result).then(value => {
        if (value !== 10) throw new Error('Unexpected result: ' + value);
        return value;
      });
    `);
  }
}
```

## Migration Best Practices

### 1. Planning and Preparation

```typescript
// migration-planning.ts

export class MigrationPlanner {
  
  // Create migration roadmap
  createMigrationRoadmap(assessment: ProjectAssessment): MigrationRoadmap {
    return {
      phases: [
        {
          name: 'Preparation',
          duration: '1-2 weeks',
          tasks: [
            'Set up development environment',
            'Create comprehensive backup',
            'Establish testing baseline',
            'Train team on new patterns'
          ]
        },
        {
          name: 'Dependencies',
          duration: '1 week',
          tasks: [
            'Update package.json',
            'Resolve dependency conflicts',
            'Update dev dependencies',
            'Test build process'
          ]
        },
        {
          name: 'Core Migration',
          duration: '2-4 weeks',
          tasks: [
            'Migrate imports',
            'Convert operators',
            'Update creation functions',
            'Fix TypeScript errors'
          ]
        },
        {
          name: 'Testing & Verification',
          duration: '1-2 weeks',
          tasks: [
            'Run full test suite',
            'Performance testing',
            'Integration testing',
            'User acceptance testing'
          ]
        },
        {
          name: 'Deployment',
          duration: '1 week',
          tasks: [
            'Staging deployment',
            'Production deployment',
            'Monitoring setup',
            'Rollback preparation'
          ]
        }
      ],
      risks: this.identifyRisks(assessment),
      mitigation: this.createMitigationPlan(assessment)
    };
  }
  
  // Risk assessment
  private identifyRisks(assessment: ProjectAssessment): MigrationRisk[] {
    const risks: MigrationRisk[] = [];
    
    if (assessment.complexityScore > 200) {
      risks.push({
        type: 'complexity',
        severity: 'high',
        description: 'High complexity migration may introduce bugs',
        mitigation: 'Extensive testing and gradual rollout'
      });
    }
    
    if (assessment.dependencies.some(dep => dep.rxjsCompatibility === 'incompatible')) {
      risks.push({
        type: 'dependencies',
        severity: 'high',
        description: 'Incompatible dependencies found',
        mitigation: 'Update or replace incompatible dependencies'
      });
    }
    
    return risks;
  }
}
```

### 2. Team Training and Communication

```typescript
// training-materials.ts

export const migrationTrainingMaterials = {
  quickReference: {
    title: 'RxJS Migration Quick Reference',
    sections: [
      {
        title: 'Import Changes',
        before: `import { Observable } from 'rxjs/Observable';`,
        after: `import { Observable } from 'rxjs';`
      },
      {
        title: 'Operator Usage',
        before: `source.map(x => x * 2).filter(x => x > 5)`,
        after: `source.pipe(map(x => x * 2), filter(x => x > 5))`
      },
      {
        title: 'toPromise() Replacement',
        before: `observable.toPromise()`,
        after: `firstValueFrom(observable) or lastValueFrom(observable)`
      }
    ]
  },
  
  commonPatterns: {
    title: 'Common Migration Patterns',
    patterns: [
      {
        name: 'Error Handling',
        description: 'Use catchError instead of catch',
        example: `
          // Before
          source.catch(error => Observable.of(fallback))
          
          // After
          source.pipe(catchError(error => of(fallback)))
        `
      },
      {
        name: 'Side Effects',
        description: 'Use tap instead of do',
        example: `
          // Before
          source.do(value => console.log(value))
          
          // After
          source.pipe(tap(value => console.log(value)))
        `
      }
    ]
  }
};
```

## Summary

This comprehensive migration guide covers:

- ✅ Version-specific breaking changes and migration paths
- ✅ Automated migration tools and pipelines
- ✅ Manual migration patterns for complex scenarios
- ✅ Testing and verification strategies
- ✅ Project assessment and planning tools
- ✅ Risk mitigation and best practices
- ✅ Team training and communication resources

These strategies ensure smooth, reliable migrations between RxJS versions while minimizing disruption to your development workflow.

## Next Steps

Next, we'll explore **RxJS Best Practices & Guidelines** to establish coding standards and patterns for optimal RxJS usage in your projects.
