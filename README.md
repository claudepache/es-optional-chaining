# Optional Chaining

This is a proposal for introducing Optional Chaining feature (aka Existential Operator, aka Null Propagation) in [ECMAScript](https://github.com/tc39/ecma262/)).

## Motivation

When looking for a property value deeply in a tree structure, one has often to check whether intermediate nodes exist:
```js
var street = user.address && user.address.street
```

Also, many API return either an object or null/undefined, and one may want to extract a property from the result only when it is not null:
```js
var fooInput = myForm.querySelector('input[name=foo]')
var fooValue = fooInput ? fooInput.value : undefined
```

The Optional Chaining Operator allows to handle many of those cases without repeating yourself and/or assigning intermediate results in temporary variables:

```js
var street = user.address?.street

var fooValue = myForm.querySelector('input[name=foo]')?.value
```

<!--
## Further examples

The following code:
```js
var SetProto
if (Object.prototype.hasOwnProperty('__proto__')) {
    SetProto = Object.getOwnPropertyDescriptor(Object.prototype, '__proto__').set
}
```
could become shorter:
```js
var SetProto = Object.getOwnPropertyDescriptor(Object.prototype, '__proto__')?.set
```


The following code:
```js
if (foo != null && foo[Symbol.iterator] != null) { // foo is iterable
    var iterator = foo[Symbol.iterator]()
    // ...
}
```
could be rewritten as (disregarding the buggy case where `foo[Symbol.iterator]()` would not produce a function):
```js
var iterator = foo?.[Symbol.iterator]?.()
if (iterator) { // foo is iterable
    // ...
}
```


The following code:
```js
var _
var list = (_ = node._tree) && (_ = _.editionParams) && _.fooList || []
```
could become more readable:
```js
var list = node._tree?.editionParams?.fooList || []
```
-->

## Syntax

The operator is spelt `?.` and may be used at the following positions:

```js
obj?.prop         // optional property access
obj?.[expr]       // ditto
func?.(...args)   // optional function or method call
new C?.(...args)  // optional constructor invocation
```

### Notes

* In order to allow `foo?.3:0` to be parsed as `foo ? .3 : 0` rather than `foo ?. 3 : 0`, a simple lookahead is added at the level of the lexical grammar (the `?.` token should not be followed by a decimal digit).

* We don’t use the `obj?[expr]` and `func?(...arg)` syntax, because of the difficulty for the parser to distinguish those forms from the conditional operator, e.g. `obj?[expr].filter(fun):0` and `func?(x - 2) + 3 :1`.

<!--

Here are other alternatives that don’t need lookahead are proposed:

```js
obj.?prop         // optional property access
obj.?[expr]       // ditto
func.?(...args)   // optional function or method call
new C.?(...args)  // optional constructor invocation
```

And:

```js
obj..prop         // optional property access
obj..[expr]       // ditto
func..(...args)   // optional function or method call
new C..(...args)  // optional constructor invocation
```

And, minimising the number of characters (but the question mark inside the brackets don’t look good):

```js
obj.?prop        // optional property access
obj[?expr]       // ditto
func(?...args)   // optional function or method call
new C(?...args)  // optional constructor invocation
```

-->

## Semantics

(The explanations here are optimised for the human mind. For a more machine-friendly version, look at the spec text.)

**Base case.** If the expression at the left-hand side of the `?.` operator evaluates to undefined or null, its right-hand side is _not_ evaluated and the whole expression returns undefined.

```js
a?.b      // undefined if a is null/undefined, a.b otherwise
a?.[++x]   // If a evaluates to null/undefined, the variable x is *not* incremented.
```

**Short-circuiting.** A value of undefined produced by the `?.` operator is propagated without further evaluation to an entire chain of property accesses, method calls, constructor invocations, etc. (or, in spec parlance, a [Left-Hand-Side Expression](https://tc39.github.io/ecma262/#sec-left-hand-side-expressions)).

```js
a?.b.c().d      // undefined if a is null/undefined, a.b.c().d otherwise.
                // NB: If a is not null/undefined, and a.b is nevertheless undefined,
                //     short-circuiting does *not* apply
```

**Free grouping.** Use of parentheses for mere grouping does *not* stop short-circuiting.
```js
(a?.b).c().d     // equivalent to: a?.b.c().d
```

**Use in write context.** The `?.` operator may also be used for optional property writing and deletion:

```js
a?.b = 42     // does nothing if a is null/undefined, equivalent to a.b = 42 otherwise
delete a?.b   // no-op if a is null/undefined
```

## Specification

Technically the semantics are enforced by introducing a special Reference, called Nil, which is propagated without further evaluation through left-hand side expressions (property accesses, method calls, etc.), and which dereferences to **undefined** (or to /dev/null in write context).

See [the spec text](https://claudepache.github.io/es-optional-chaining/) for more details.








