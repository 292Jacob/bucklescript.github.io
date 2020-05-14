---
title: Enhanced lazy encoding in BuckleScript
---



Recently we made some significant improvements with our new encoding for lazy values, and we find it so exciting that we want to highlight the changes. The new encoding generates very idiomatic JS output like hand-written code.

For people who are not familiar with lazy evaluation, it is documented here: https://ocaml.org/releases/4.10/htmlman/expr.html#sss:expr-lazy.

# What's the difference?

Take this code snippet, for example:

```reasonml
let lazy1 = lazy {
    "Hello, lazy" -> Js.log;
     1
}; // create a lazy value

let lazy2 = lazy 3 ; // artifical lazy values for demo purpose

Js.log2 (lazy1, lazy2); // logging the lazy values

let (lazy la, lazy lb) = (lazy1, lazy2); // pattern match to force evaluation

Js.log2 (la, lb); // logging forced values
```

Running this code in node, the output is as below:
```bash
lazy_demo$node src/lazy_demo.bs.js 
[ [Function], tag: 246 ] 3 # logging the output of two lazy blocks
Hello, lazy # lazy1, laz2 evaluated forced by pattern match, hence logging
1 3 #logging the evaluated lazy block
```

With the new encoding, the output is as below:
```bash
{ RE_LAZY_DONE: false, value: [Function: value] } { RE_LAZY_DONE: true, value: 3 } # logging block one with new encoding
Hello, lazy
1 3
```

As you can see, with the new encoding, no magic tags like 246  appear, and the lazy status is clearly marked via `RE_LAZY_DONE: (true | false) `.

More than that, the generated code quality is also improved. In the old mode, the generated JS code was like this:

```js
var lazy1 = Caml_obj.caml_lazy_make((function (param) {
        console.log("Hello, lazy");
        return 1;
      }));

console.log(lazy1, 3);

var la = CamlinternalLazy.force(lazy1);

var lb = CamlinternalLazy.force(3);

console.log(la, lb);

var lazy2 = 3;
```

In the new mode, it is simplified:
```js
var lazy1 = {
  RE_LAZY_DONE: false,
  value: (function () { // closure now is uncurried arity-0 function
      console.log("Hello, lazy");
      return 1;
    })
};

var lazy2 = {
  RE_LAZY_DONE: true,
  value: 3
};

console.log(lazy1, lazy2);

var la = CamlinternalLazy.force(lazy1);

var lb = CamlinternalLazy.force(lazy2);

console.log(la, lb);
```

## What changes did we make?

In native, the encoding of lazy values is rather complicated: 

- It is an array, which is not friendly for debugging in JS context.
- It has some special tags which are not meaningful, for example, magic number 246, in JS context.
- It tries to unbox lazy values with the help of native GC. However, such complexity does not pay off in JS since the JSVM does not expose its GC semantics.

So in the master, our encoding scheme is much simplified to take advantage of JS as much as possible:

- The encoding is uniform; it is always an object of two key value pairs. One is `RE_LAZY_DONE` to mark its status, 
the other is either a closure or an evaluated value.

- The compiler optimization still kicks in at compile time: if it knows a lazy value is already evaluated or does not need to be evaluated, it will promote its status to be 'done'. However, unlike native, unboxing is not happening. This makes sense since the most interesting unboxing scenario happens in runtime instead of compile time where it is impossible in JSVM.


With the new encoding, `lazy` has a much nicer sugar, and we encourage you to use it whenever it is convenient!

# Caveats:

Don't rely on the special name `RE_LAZY_DONE` for JS interop; we may change it to a symbol in the future.
