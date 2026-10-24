# JavaScript

## Theory questions

### 1. Difference between reassignment and redeclaration (PDF p. 7)
**Answer:** Reassignment changes the value held by an existing binding; redeclaration declares a binding with the same name again in a scope. `let` and `const` generally cannot be redeclared in the same lexical scope, while `var` can be redeclared there; `const` cannot be reassigned.

### 2. `var` vs. `let` vs. `const` (PDF p. 7)
**Answer:** `var` is function-scoped and hoisted with an initial `undefined`; `let` and `const` are block-scoped and inaccessible before declaration due to the temporal dead zone. `let` can be reassigned; `const` cannot be reassigned, though an object it references can still be mutated.

### 3. Types of object creation in JavaScript (PDF p. 7)
**Answer:** Common approaches include object literals (`{}`), constructors/classes with `new`, factory functions, `Object.create(proto)`, and built-in constructors. Prefer literals or clear factories for simple objects; use classes when instance behavior/inheritance is useful.

### 4. What is hoisting in JavaScript? (PDF p. 7)
**Answer:** Hoisting describes how declarations are processed before execution within their scope. Function declarations can be called before their source location; `var` bindings exist as `undefined` before assignment, while `let`/`const` bindings exist but cannot be accessed before initialization.

### 5. What is the temporal dead zone? (PDF p. 7)
**Answer:** It is the period from entering a block scope until a `let` or `const` declaration is initialized. Accessing that binding during the period throws a `ReferenceError`.

### 6. What is a closure in JavaScript? (PDF p. 7)
**Answer:** A closure is a function together with access to the lexical environment in which it was created. It lets a function retain private state or refer to outer variables after the outer function has returned.

### 7. What is a prototype? (PDF p. 7)
**Answer:** A prototype is an object used in JavaScript's inheritance lookup chain. When a property is not found on an object itself, lookup proceeds through its prototype chain; class syntax builds on this model.

### 8. What is the spread operator? (PDF p. 7)
**Answer:** `...` expands an iterable's elements or an object's enumerable own properties into another expression, such as `[...items]` or `{...settings}`. These are shallow copies, not deep clones.

### 9. What is the rest operator? (PDF p. 7)
**Answer:** Rest syntax gathers remaining values into an array or object, as in `function f(first, ...others)` or `const { id, ...rest } = record`.

### 10. Difference between spread and rest operators (PDF p. 7)
**Answer:** They use the same `...` syntax but have opposite roles: spread expands values into a new call/array/object; rest collects remaining values into one array or object.

### 11. `splice` vs. `slice` (PDF p. 7)
**Answer:** `slice(start, end)` returns a shallow copy of a range without changing the original array. `splice(start, count, ...items)` mutates the original by removing, replacing, or inserting elements and returns the removed elements.

### 12. What is an arrow function? (PDF p. 7)
**Answer:** It is concise function syntax, such as `(x) => x * 2`. Arrow functions capture lexical `this`, `arguments`, and `super`, and cannot be used as constructors; use a regular function when dynamic `this` or construction is needed.

### 13. What is an anonymous function? (PDF p. 7)
**Answer:** It is a function expression without an explicit function name, often passed as a callback. It can still be assigned to a variable or be given an inferred name by the runtime.

### 14. What is an array and its types? (PDF p. 7)
**Answer:** A JavaScript array is an ordered, zero-indexed object used to store a sequence and can hold mixed values. Arrays may be one-dimensional or nested/multidimensional by convention; typed arrays such as `Uint8Array` store fixed-width numeric data.

### 15. How do you flatten a multidimensional array? (PDF p. 7)
**Answer:** Use `array.flat(depth)` for a known nesting depth, or `array.flat(Infinity)` for arbitrary nesting. For custom streaming or special traversal behavior, use recursion or an explicit stack.

### 16. `=`, `==`, and `===`: state the operator names and explain their use (PDF p. 7)
**Answer:** `=` is assignment. `==` is loose equality and may coerce types; `===` is strict equality and compares without type coercion. Prefer `===` (and `!==`) for predictable comparisons.

## JavaScript output questions

### 17. What is printed by `console.log(true + 1)`, `console.log(1 + '2')`, and `console.log('2' + 1)`? (Questpond Virtusa interview, PDF p. 53)
**Answer:** `2`, `"12"`, and `"21"`, respectively. Boolean is converted to a number for numeric addition; when string concatenation is selected, the other operand is converted to a string.

### 18. What happens when `var` and `let` declarations are logged before their source lines? (Questpond Virtusa interview, PDF p. 53)
```js
let c;

console.log(a);
console.log(b);
console.log(c);

var a;
let b;
```
**Answer:** The first log prints `undefined` because the `var` binding is initialized to `undefined` before execution. The second log throws a `ReferenceError` because `b` is in its temporal dead zone; the third log is not reached.
