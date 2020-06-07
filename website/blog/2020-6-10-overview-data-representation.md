---
title: An overview of new data representation in BuckleScript
---

In the next version of BuckleScript, we made several major changes to tweak the data representation for various data types, it's more idiomatic and debugger friendly. 

Note since V8 or other JavaScript engines are tweaked to make idiomatic JS code running fast, so it also results in faster running code. 

Another property, for a compiled language like BuckleScript, is that we can reason about [IC](https://en.wikipedia.org/wiki/Inline_caching) friendliness by just looking at the type definitions locally; this is very helpful for advanced users to write performance reliable JS code, if you are unfamiliar with how IC works in general, feel free to skip the section about IC.

Note this article is quite dense so that we will skip the old encoding.

## Record (stable)

We compile records to idiomatic JS objects since [version 7](https://bucklescript.github.io/blog/2019/11/18/whats-new-in-7), this is great for performance and debugging. We also support label renaming to shorten field names to save sizes.

Take the code below for example:

```reasonml
type int64 = {
    loBits : int [@bs.as "lo"],
    hiBits : int [@bs.as "hi]
}
let value = {hiBits : 33 , loBits : 32 };
let rand = ({loBits; hiBits}) => loBits + hiBits;
```

It will generate JS output as below:
```js
var value = {lo : 32, hi : 33}
function rand (param){
    return param.lo + param.hi
}
```

If users want to make it even  shorter, they can choose to compile record as an array as below:

```reasonml
type int64 = {
    loBits : int [@bs.as "0"],
    hiBits : int [@bs.as "1"]
}
let value = {hiBits : 33 , loBits : 32 };
let rand = ({loBits; hiBits}) => loBits + hiBits;
```

Now output JS as below:

```js
var value = [32,33]
function rand(param){
    return param[0] + param[1]
}
```

The label renaming techniques can be applied systematically using a syntactic macro, in the future we may provide an advanced mode to apply it automatically. Another nice property is that only the type definition needs to be adapted, other parts of code is untouched.

### IC friendliness

Record is always in a perfect IC position, the compiler can ensure all generated records are of the same shape

## Variant (internal)
This encoding for variants may be  subject to change in the future, but it is so simple that it makes sense for users to have a basic understanding.

Take such type definition for example:

```reasonml
type t =
  | Black(t, int, t)
  | Red(t, int, t)
  | Empty;
let empty = Empty  ;
let v0 = Black (empty, 3, empty);
let v1 = Red (empty, 3, empty);
```
The generated JS code would be:

```js
var empty =/*Empty*/ 0;
var v0  = {TAG : 0/*Black*/, _0 : /*Empty*/ 0 , _1 : 3 , _2 : /*Empty */ 0};
var v1 = {TAG : 1/*Red*/, _0 : /*Empty*/ 0 , _1 : 3 , _2 : /*Empty */ 0};
```

As you can see, variants are divided into two categories, the variant which does not have payload is compiled into a number starting from 0, while variants which has the payload is compiled into an object which has the first slot named `TAG` and the following slots named as `_0`, `_1` ..

### variant with inline records

Users can give names to the payload, and the compiler respect it, however, we don't support user level renaming, i.e, using `bs.as`, at this time.

```reasonml
type t =
  | Black ({l:t, value: int, r: t})
  | Red({l:t, value: int, r: t})
  | Empty;
let empty = Empty  ;
let v0 = Black ({l:empty, value: 3, r:empty});
let v1 = Red ({l:empty, value:3, r: empty});
```
The generated JS code would be:
```js
var empty =/*Empty*/ 0;
var v0  = {TAG : 0/*Black*/, l: /*Empty*/ 0 , value : 3 , r: : /*Empty */ 0};
var v1 = {TAG : 1/*Red*/, l : /*Empty*/ 0 , value : 3 , r : /*Empty */ 0};
``` 


### Special case when the number of variants which has payload is only 1.

Take the types below for example:

```reasonml
type list = 
    | Nil
    | Cons (int * list);
```

Since the number of variants which has payload is only 1, the compiler does not need add `TAG` when we destruct the data for pattern matching, so the code below:

```reasonml
let u = Cons(1,Nil)
```
Will generate such JS output:

```js
var u = {_0: 1, _1 : /*Nil*/ 0 }; // No TAG data.
```

### Specialized for immutable list

The `list` type is a built-in type, its type definition is similar to this :

```reasonml
type t ('a) = 
    | []
    | (::) ('a * t ('a))
```

Without any customization,  it will generate js objects with indexes like `_0`, `_1`, since list is so pervasive,
we provide some special treatment so that 
```reasonml
let u = [0,1,2,3]
```
Will generate js code as below:
```js
var u = {hd : 0, {hd : 1, {hd: 2, {hd :3 , tl : /*[]*/0 }}}}
```

This is a minor change, we changed the name of `_0` to `hd` and `_1` to `tl`.

### IC friendliness

For types whose number of variants which has payload is 1, it will be in a perfect IC position. 
The number of variants which does not carry payload will not affect IC, since the pattern match will do a split first. 

For types whose number of variants which has the same number of payloads, it will also be in a perfect IC position, like the red-black-tree example above. 

For other cases, it will hit a polymorphic IC in the V8 jit compiler, this is not the fastest running case.

Note that user can always tweak the variant layout to make it IC friendly, for example, it can always introduce one level of indirection to make all variants share the same number of payloads:

```reasonml
type t = 
    | A0 of a0 // 1 paylaod
    | A1 of a1 // 1 payload
    | A2 of a2 // 1 payload
    | C0
    | C1
    | C2 // This will not affect IC 
```

### Variant in debug mode

Note we only generate constructor names in comments for debugging,  when constructor names are attached to the data, it will be more useful for debugging. When debug mode is activated using `-bs-g`, 

The generated code will be changed from below 
```js
var v0  = {TAG : 0/*Black*/, _0 : /*Empty*/ 0 , _1 : 3 , _2 : /*Empty */ 0};
```
to

```js
var v0  = {TAG : 0, _0 : /*Empty*/ 0 , _1 : 3 , _2 : /*Empty */ 0, [Symbol.for("name")]: "Black"};
```

## Polymorphic-variant (internal)

Polymorphic variant allows users to use the types without declaring it first:

```reasonml
let u = 3 -> `hello
```

It will generate
```js
var u = {HASH : MAGIC_NUMBER, VAL: 3 }
```
The field of `HASH` is the hash of name `"hello"`, while the `VAL` is the payload

### IC friendliness

Polymorphic variant is always in a perfect IC position, the compiler can ensure all generated records are of the same shape. This is due to that the payload is not unpacked, it is always just one payload


### Polymorphic variant in debug mode

In debug mode, similar to variant, we carry the name in generated code for debugging,

So instead of 
```js
var u = {HASH : MAGIC_NUMBER, VAL: 3 }
```

It will generate

```js
var u = {HASH : MAGIC_NUMBER, VAL: 3 , [Symbol.for("name")] : "hello"}
```