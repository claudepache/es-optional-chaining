# Status of this repository

This repository represents the sole opinion of its author. The author’s main goal is that the material presented here could be used as a sound basis for the official ECMAScript proposal.

***This repository is now obsolete. The official ECMAScript proposal is here: https://github.com/tc39/proposal-optional-chaining***

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
new C?.(...args)  // optional constructor invocation. (But is that case useful?)
```

### Notes

* In order to allow `foo?.3:0` to be parsed as `foo  ?  .3  :  0` (as required for backward compatibility), a simple lookahead is added at the level of the lexical grammar, so that the sequence of characters `?.` is not interpreted as a single token in that situation (the `?.` token must not be immediately followed by a decimal digit).

* We don’t use the `obj?[expr]` and `func?(...arg)` syntax, because of the difficulty for the parser to distinguish those forms from the conditional operator, e.g. `obj?[expr].filter(fun):0` and `func?(x - 2) + 3 :1`.
  
  Alternative syntaxes for those two cases have each their own flaws, and deciding which is one the least bad is mostly a question of personal preference. Here is how we made our choice: 
  - pick the best syntax for the `a?.b` case, which is expected to occurs most often;  
  - extend the use of the `?.` sequence of characters to other cases, in order to have a uniform look: `a?.[b]`, `a?.(b)`.

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

**Locality.** Apart from short-circuiting, the semantics is strictly local. For instance, the meaning of a `.` token is not modified by a previous `?.` token found earlier in the chain.
```js
a?.b.c().d      // If  a  is not null/undefined, and  a.b  is nevertheless undefined,
                // short-circuiting does *not* apply: the meaning of  .c  is not modified.
```
Short-circuiting semantics may be compared to an early return instruction in a function.

**Free grouping?** As currently specced, use of parentheses for mere grouping does not stop short-circuiting. However that semantics is debatable and may be changed.

```js
(a?.b).c().d     // equivalent to: a?.b.c().d
```

**Use in write context.** In absence of clear use cases and semantics, the `?.` operator is statically forbidden at the left of an assignment operator. On the other hand, optional deletion is allowed, because it has clear semantics, has [known use case](https://github.com/babel/babel/blob/28ae47a174f67a8ae6f4527e0a66e88896814170/packages/babel-helper-builder-react-jsx/src/index.js#L66-L69), and is consistent with the general ”just ignore nonsensical argument” semantics of the `delete` operator.

```js
a?.b = 42     // trigger an early ReferenceError (same error as `a.b() = c`, etc.)
delete a?.b   // no-op if a is null/undefined
```

## Specification

Technically the semantics are enforced by introducing a special Reference, called Nil, which is propagated without further evaluation through left-hand side expressions (property accesses, method calls, etc.), and which dereferences to **undefined**.

See [the spec text](https://claudepache.github.io/es-optional-chaining/) for more details.

## Compiling
The Nil Reference is a spec artefact that is used because it is more pleasant to write the spec that way. But if you intend to transform code using optional chaining into es6-compatible code, it is more useful to have an “abrupt completion” mental model. Concretely, the expression:

```js
a?.b.c?.d
```

could be rewritten using an IIAFE and early return statements:

```js
(() => {
    let _ = a
    if (_ == null)
        return
    _ = _.b.c
    if (_ == null)
        return
    return _.d
})()
```

For an existing Babel implementation, see: https://github.com/babel/babel/pull/5813
