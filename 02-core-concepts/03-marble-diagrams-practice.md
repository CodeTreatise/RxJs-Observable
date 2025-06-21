# Reading & Drawing Marble Diagrams Practice 🟢

## 🎯 Learning Objectives
By the end of this lesson, you will be able to:
- Read complex marble diagrams with confidence
- Draw accurate marble diagrams for any Observable sequence
- Use marble diagrams for debugging and communication
- Create interactive marble diagram exercises
- Apply marble thinking to real-world scenarios

## 📚 Marble Diagram Review

### Basic Notation Refresher

| Symbol | Meaning | Example |
|--------|---------|---------|
| `─` | Timeline | `──────────` |
| `a`, `b`, `c` | Values | `──a───b───c──` |
| `\|` | Completion | `──a───b──\|` |
| `X` or `#` | Error | `──a───b──X` |
| `()` | Synchronous grouping | `(abc)\|` |
| `^` | Subscription point | `^──a───b──\|` |
| `!` | Unsubscription point | `^──a───b──!` |

## 📖 Reading Complex Marble Diagrams

### Exercise 1: Basic Reading

Read these marble diagrams and predict the output:

```typescript
// Diagram 1: Simple sequence
source: ──a───b───c───|
```

**What happens?**
- Emits `a` at time 2
- Emits `b` at time 6  
- Emits `c` at time 10
- Completes at time 14

```typescript
// Diagram 2: Error case
source: ──a───b───X
```

**What happens?**
- Emits `a` at time 2
- Emits `b` at time 6
- Errors at time 10 (never completes)

```typescript
// Diagram 3: Synchronous burst
source: (abc)───d───|
```

**What happens?**
- Emits `a`, `b`, `c` synchronously at time 0
- Emits `d` at time 8
- Completes at time 12

### Exercise 2: Operator Reading

Practice reading operator transformations:

```typescript
// map() transformation
source: ──1───2───3───|
        map(x => x * 10)
result: ──10──20──30──|
```

```typescript
// filter() transformation  
source: ──1───2───3───4───5───|
        filter(x => x % 2 === 0)
result: ──────2───────4───────|
```

```typescript
// debounceTime() transformation
source: ──a─b─c─────d───e─f───|
        debounceTime(2)
result: ─────────c─────d─────f|
```

### Exercise 3: Advanced Reading

```typescript
// switchMap() with inner Observables
source:     ──a─────b─────c───|
             \       \     \
              \       \     \
               1─2─|   3─4─|  5─6─|
              
switchMap:  ────1─2───3─4───5─6─|
```

**Reading Steps:**
1. `a` emitted → creates inner Observable `1─2─|`
2. `1` and `2` are emitted from first inner Observable
3. `b` emitted → switches to new inner Observable `3─4─|` (cancels previous)
4. `3` and `4` are emitted from second inner Observable
5. `c` emitted → switches to new inner Observable `5─6─|`
6. `5` and `6` are emitted from third inner Observable

## ✏️ Drawing Marble Diagrams

### Drawing Process

1. **Start with timeline** `──────────────────`
2. **Mark time intervals** (optional) `0─1─2─3─4─5─6─7─8─9`
3. **Add value emissions** `──a───b───c───`
4. **Add termination** `──a───b───c───|` or `──a───b───c───X`

### Exercise 4: Draw from Code

Draw marble diagrams for these RxJS operations:

```typescript
// 1. Simple of() Observable
of(1, 2, 3).subscribe(console.log);
```

**Your drawing:**
```
result: (123)|
```

```typescript
// 2. interval() with take()
interval(1000).pipe(take(3)).subscribe(console.log);
```

**Your drawing:**
```
result: ────0────1────2|
```

```typescript
// 3. timer() with single emission
timer(2000).subscribe(console.log);
```

**Your drawing:**
```
result: ────────0|
```

### Exercise 5: Complex Transformations

Draw the result for these operator chains:

```typescript
// Example: range + map + filter
range(1, 5).pipe(
  map(x => x * 2),
  filter(x => x > 4)
).subscribe(console.log);
```

**Step by step:**
```
range(1,5): (12345)|
map(x*2):   (246810)|  
filter>4:   (6810)|
```

**Final result:**
```
result: (6810)|
```

## 🎮 Interactive Exercises

### Exercise 6: Debugging with Marble Diagrams

You have this buggy code. Draw marble diagrams to identify the issue:

```typescript
const search$ = fromEvent(searchInput, 'input').pipe(
  map(event => event.target.value),
  switchMap(query => this.http.get(`/search?q=${query}`))
);
```

**Problem:** No debouncing - API called on every keystroke!

**Current behavior:**
```
keystrokes: ──h─e─l─l─o─w─o─r─l─d──|
searches:   ──r─r─r─r─r─r─r─r─r─r──|  (r = HTTP request)
```

**Solution with debouncing:**
```typescript
const search$ = fromEvent(searchInput, 'input').pipe(
  map(event => event.target.value),
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(query => this.http.get(`/search?q=${query}`))
);
```

**Fixed behavior:**
```
keystrokes: ──h─e─l─l─o─w─o─r─l─d──|
debounced:  ─────────────────────d─|
searches:   ─────────────────────r─|
```

### Exercise 7: Real-World Scenario

**Scenario:** Button click with loading state

```typescript
const button = document.getElementById('saveButton');
const clicks$ = fromEvent(button, 'click');
const saveData$ = clicks$.pipe(
  switchMap(() => this.api.saveData())
);
```

**Draw the interaction:**
```
clicks:    ──c─────────c─c───c─────|  (c = click)
API calls: ──r─────────r─r───r─────|  (r = request)
responses: ────R─────────R─R─────R─|  (R = response)
```

**Problem:** No loading state management!

**Improved version:**
```typescript
const saveData$ = clicks$.pipe(
  tap(() => this.setLoading(true)),
  switchMap(() => this.api.saveData().pipe(
    finalize(() => this.setLoading(false))
  ))
);
```

**With loading state:**
```
clicks:     ──c─────────c─c───c─────|
loading:    ──L─────────L─L───L─────|  (L = loading start)
responses:  ────R─────────R─R─────R─|  (R = response + loading end)
```

## 🧩 Combination Operators Practice

### Exercise 8: combineLatest()

Draw the marble diagram for `combineLatest`:

```typescript
const source1$ = cold('──a─────c───e──|');
const source2$ = cold('────b─────d────f|');
const result$ = combineLatest([source1$, source2$]);
```

**Step-by-step analysis:**
1. `a` emitted but no `b` yet → no emission
2. `b` emitted, we have `a` and `b` → emit `[a,b]`
3. `c` emitted, latest from source2 is `b` → emit `[c,b]`
4. `d` emitted, latest from source1 is `c` → emit `[c,d]`
5. `e` emitted, latest from source2 is `d` → emit `[e,d]`
6. `f` emitted, latest from source1 is `e` → emit `[e,f]`

**Result diagram:**
```
source1: ──a─────c───e──|
source2: ────b─────d────f|
result:  ────[a,b]─[c,b]─[c,d]─[e,d]─[e,f]|
```

### Exercise 9: merge()

```typescript
const source1$ = cold('──a───c───e|');
const source2$ = cold('────b───d───f|');
const result$ = merge(source1$, source2$);
```

**Result diagram:**
```
source1: ──a───c───e|
source2: ────b───d───f|
result:  ──a─b─c─d─e─f|
```

### Exercise 10: zip()

```typescript
const source1$ = cold('──a───c───e──|');
const source2$ = cold('────b─────d──f|');
const result$ = zip(source1$, source2$);
```

**Zip waits for pairs:**
```
source1: ──a───c───e──|
source2: ────b─────d──f|
result:  ────[a,b]──[c,d]──[e,f]|
```

## 🔧 Error Handling Diagrams

### Exercise 11: catchError()

```typescript
const source$ = cold('──a───b───X');
const result$ = source$.pipe(
  catchError(err => of('error handled'))
);
```

**Result diagram:**
```
source: ──a───b───X
result: ──a───b───(error handled)|
```

### Exercise 12: retry()

```typescript
const source$ = cold('──a───b───X');
const result$ = source$.pipe(retry(2));
```

**Shows multiple attempts:**
```
attempt1: ──a───b───X
attempt2: ──────────a───b───X
attempt3: ──────────────────a───b───X
```

**Timeline view:**
```
result: ──a───b───a───b───a───b───X
```

## 🎭 Higher-Order Observable Diagrams

### Exercise 13: switchMap() vs mergeMap() vs concatMap()

Given this setup:
```typescript
const source$ = cold('─a───b───c|');
const project = (x) => cold(`──${x}1──${x}2|`);
```

**switchMap() - Cancels previous:**
```
source:     ─a───b───c|
             \   \   \
              \   \   \
               ──a1──a2|
                   ──b1──b2|
                       ──c1──c2|
switchMap:  ───a1─b1─c1──c2|
```

**mergeMap() - Keeps all:**
```
source:     ─a───b───c|
             \   \   \
              \   \   \
               ──a1──a2|
                   ──b1──b2|
                       ──c1──c2|
mergeMap:   ───a1─b1─(a2,c1)─(b2,c2)|
```

**concatMap() - Queues:**
```
source:     ─a───b───c|
             \   
              \   
               ──a1──a2|
                       ──b1──b2|
                               ──c1──c2|
concatMap:  ───a1──a2──b1──b2──c1──c2|
```

## 🧪 Testing with Marble Diagrams

### Exercise 14: Writing Marble Tests

```typescript
import { TestScheduler } from 'rxjs/testing';

describe('Marble Diagram Tests', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should debounce correctly', () => {
    testScheduler.run(({ cold, expectObservable }) => {
      const source$ = cold('──a─b─c─────d───e─f───|');
      const expected =     '─────────c─────d─────f|';
      
      const result$ = source$.pipe(
        debounceTime(3, testScheduler)
      );
      
      expectObservable(result$).toBe(expected);
    });
  });

  it('should handle errors correctly', () => {
    testScheduler.run(({ cold, expectObservable }) => {
      const source$ = cold('──a───b───X');
      const expected =     '──a───b───(c|)';
      
      const result$ = source$.pipe(
        catchError(() => of('c'))
      );
      
      expectObservable(result$).toBe(expected);
    });
  });
});
```

## 🎨 Visual Marble Diagram Tools

### ASCII Art Diagrams

```typescript
// Complex operator chain visualization
/*
source:    ──1───2───3───4───5───|
           
map(x*2):  ──2───4───6───8───10──|
           
filter>5:  ──────────6───8───10──|
           
take(2):   ──────────6───8───|
*/

const result$ = source$.pipe(
  map(x => x * 2),
  filter(x => x > 5),
  take(2)
);
```

### HTML/CSS Marble Diagrams

```html
<!-- Visual marble diagram in HTML -->
<div class="marble-diagram">
  <div class="timeline">
    <div class="value" style="left: 20%">a</div>
    <div class="value" style="left: 40%">b</div>
    <div class="value" style="left: 60%">c</div>
    <div class="complete" style="left: 80%">|</div>
  </div>
</div>

<style>
.marble-diagram {
  position: relative;
  height: 40px;
  border-bottom: 2px solid #333;
  margin: 20px 0;
}

.value {
  position: absolute;
  bottom: -5px;
  background: #4CAF50;
  color: white;
  padding: 2px 6px;
  border-radius: 50%;
}

.complete {
  position: absolute;
  bottom: -10px;
  font-size: 20px;
  font-weight: bold;
}
</style>
```

## 🎯 Practice Challenges

### Challenge 1: Debug This Behavior

Given this code, draw what actually happens vs what's expected:

```typescript
const button1$ = fromEvent(btn1, 'click').pipe(map(() => 'A'));
const button2$ = fromEvent(btn2, 'click').pipe(map(() => 'B'));
const merged$ = merge(button1$, button2$).pipe(take(1));
```

**Expected:** Take first click from either button
**Draw the actual behavior and identify any issues**

### Challenge 2: Design a Solution

**Requirement:** Auto-save user input, but:
- Only save if user stops typing for 2 seconds
- Don't save duplicate values
- Show saving status
- Handle save errors gracefully

**Draw the marble diagram for your solution:**

```typescript
// Your solution here
const autoSave$ = input$.pipe(
  debounceTime(2000),
  distinctUntilChanged(),
  tap(() => showSaving()),
  switchMap(value => saveToServer(value).pipe(
    tap(() => hideSaving()),
    catchError(err => {
      showError(err);
      return EMPTY;
    })
  ))
);
```

**Draw your marble diagram:**
```
input:     ──h─e─l─l─o─────w─o─r─l─d─────|
debounced: ─────────────o─────────────d─|
saving:    ─────────────S─────────────S─|  (S = start saving)
saved:     ─────────────────R─────────────R|  (R = save response)
```

## 📚 Reference Guide

### Common Patterns Quick Reference

```typescript
// Debounced search
input$.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(query => search(query))
)
// Marble: ──h─e─l─l─o─ → ─────────────o

// Retry with backoff  
request$.pipe(
  retryWhen(errors => errors.pipe(
    scan((count, err) => count + 1, 0),
    delay(1000)
  ))
)
// Marble: ──r─X─────r─X─────r─X (r=request, X=error)

// Race condition handling
button$.pipe(
  switchMap(() => api.call())
)
// Marble: Latest request cancels previous

// Batching
source$.pipe(
  bufferTime(1000),
  filter(batch => batch.length > 0)
)
// Marble: ──a─b─c─────d─e─ → ─────[a,b,c]─────[d,e]
```

## 🎯 Quick Assessment

**Drawing Challenges:**

1. Draw the marble diagram for `timer(1000, 500).pipe(take(3))`
2. Draw `combineLatest` of two sources where one completes early
3. Draw `switchMap` with overlapping inner Observables
4. Draw error recovery with `catchError` and `retry`

**Sample Solutions:**

1. `timer`: `────0──1──2|`
2. `combineLatest`: Stops when first source completes
3. `switchMap`: Inner Observables get cancelled on new emissions
4. `retry + catchError`: Multiple attempts then fallback value

## 🌟 Key Takeaways

- **Marble diagrams** are essential for understanding Observable behavior
- **Time visualization** helps debug complex async operations
- **Drawing practice** improves reactive thinking skills
- **Testing integration** ensures correct behavior
- **Visual tools** aid communication with team members
- **Pattern recognition** speeds up development
- **Debugging skills** improve with marble diagram expertise

## 🚀 Next Steps

Now that you're proficient with marble diagrams, you're ready to learn about **Subscription Management & Teardown Logic** to understand how to properly manage Observable lifecycles in applications.

**Next Lesson**: [Subscription Management & Teardown Logic](./04-subscribing-unsubscribing.md) 🟢

---

🎉 **Outstanding!** You can now read and draw marble diagrams like a pro. This visual thinking skill will serve you well throughout your reactive programming journey!
