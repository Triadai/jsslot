# jsslot
Exploring E-like slot sugar for JavaScript

A number of separate usability concerns --- including "friends" and "protected" for private state, template literal patterns, functional reactive programming, and more --- would be made more pleasant by the addition of a slot syntax to standard JavaScript. The semantics is defined by a semi-local rewrite that can initially be implemented by transpilers such as a Babel plugin.

Disclaimers

> These disclaimer sections can be skipped on a first skim of this document.

> Starting with [EcmaScript 2015 Annex A: Grammar Summary](http://www.ecma-international.org/ecma-262/6.0/#sec-grammar-summary), we add the following productions. When the name being defined by a production is a name already defined in this [official grammar](http://www.ecma-international.org/ecma-262/6.0/#sec-grammar-summary), the right side of the production contains proposed additions --- additional disjuncts --- to the right side already defined.

> For concreteness, we propose what would have been the most pleasant syntactic choice in the absence of *ASI hazards* (Automatic Semicolon Insertion hazards). Before this could become real, we would need a syntax robust against these hazards. We ignore this issue for now in the remainder of this document.

> This [official grammar](http://www.ecma-international.org/ecma-262/6.0/#sec-grammar-summary) uses a BNF notation enhanced with parameters such a `[~Yield]`. Of course these would need to be propagated into any correct extension of these productions. However, all the issues addressed by these parameters are orthogonal to this proposal, so we ignore this issue as well in the remainder of this document.

> Of course, we are only concerned with strict mode. We prefer that none of this be available in sloppy mode. However, we don't care what happens to sloppy mode. If adding these same extensions to both modes makes adoption easier, whatever.

> In the rewrites shown below, any variable name starting with "`t_`" is assumed to be replaced by a unique variable name -- one which does not appear anywhere elsewhere in the universe. If we assume a hygienic expansion, then we need not be further concerned with this issue.

## Grammar and Rewrites

### As an expression

```
UnaryExpression:
    & LeftHandSideExpression

```
expands approximately to
```
({
  get: () => (LeftHardSideExpression),
  set: (t_1) => { LeftHandSideExpression = t_1; }
})
```
In the existing grammar, a `LeftHardSideExpression` can be used as an *r-value*, evaluated to produce a value, or as an *l-value*, used at the target of an assignment. The mnemonic is that, in the context of an `AssignmentExpression` an r-value can appear on the right side of an assignment operator ("`=`") and an l-value can appear on the left side.

In this proposal, a unary prefix ampersand ("`&`") followed by an assignable expression reifies the ability both to evaluate this expression to a value and to assign to it. The difference from the above rewrite is that any subexpression that would be evaluated as an r-value in both cases is instead evaluated only once, and in the containing scope.

As implied by the rewrite, the resulting expression can only be used as an r-value.

Disclaimers

> The residual LeftHandSideExpression appearing in the two functions above is assumed not to violate *Tennet's Correspondence Principal* (TCP). That is, the r-value use within the `get` method should mean the same thing as it would if evaluated as an r-value in the surrounding context; and the l-value use in the `set` method should mean the same thing as it would if used as the target of an assignment expression in the surrounding context.

> This is why we show the rewrite using arrow functions rather than method shorthand syntax --- to minimize remaining TCP issues. Those syntactic issues which could still cause a TCP violation, such as `yield` or `await`, are assumed disallowed in this residual LeftHandSideExpression. This probably follows anyway from the extraction of r-value subexpressions.

Examples

```JavaScript
&x
```
expands to
```JavaScript
Object.freeze({
  get: () => (x),
  set: (t_1) => { x = t_1; }
})
```
but
```JavaScript
&foo()[i+1]
```
expands to
```JavaScript
let t_1, t_2;
//...
(
  t_1 = foo(),
  t_2 = i+1,
  Object.freeze({
    get: () => (t_1[t_2]),
    set: (t_3) => {t_1[t_2] = t_3;}
  })
)
```
Notice that the expansion above never introduces a reference to `this`. If any `this` did appear in the original expression, we assume that `this` would have already been extracted into the surrounding context, like `foo()` above was, so no `this` remains in the functions introduced by the rewrite. This will be important later.

If the assignment that would be generated by the above expansion would be a static error, then `set` method is instead defined to have value `undefined`. Revisiting our first example, if `x` were defined `const`

```JavaScript
const x = 3;
```
then `&x` would expand to
```JavaScript
Object.freeze({
  get: () => (x),
  set: void 0
})
```

(Unlike `undefined`, the expression `void 0` always reliably evaluates to the value `undefined`).


### As a definition

```
BindingIdentifier:
    & Identifier
```

Currently, a `BindingIdentifier` can only be an Identifier. This is a defining occurrence of the Identifier, defining a variable. In strict mode, for all other occurrences of this same identifier an an r-value or l-value, we can statically determine whether they are scoped to this definition, that is, whether they refer to the variable defined by this defining occurrence.

Here we propose that anywhere a `BindingIdentifier` may currently appear, an ampersand ("`&`") followed by an identifier may also appear.

The rewrite in this case is only semi-local. It rewrite an occurrence of the BindingIdentifier to a unique variable name, say `t_1`. It rewrites all use occurrences of this same Identifier scoped as an r-value for this variable to `t_1.get()`. It rewrites all use occurrences of this same Identifier scoped as an r-value to `t_1.set(t_2)`, where remaining rewrites have assigned to `t_2` the value that the original would have assigned to the Identifier.

Examples

```JavaScript
let &y = foo();
...
(y = y + 1)
```
expands to
```JavaScript
const t_1 = foo();
let t_2;
...
(
  t_2 = t_1.get() + 1,
  t_1.set(t_2),
  t_2
)
```
The last line of the expansion above ensures that the value of the AssignmentExpression as a whole remain the value assigned.

Expression that treat an identifier as both l-value and r-value at the same time, such as those involving ("`++`") or ("`+=`"), are first rewritten to separate these uses before the above rewrite could apply.

To prepare for the next section, we must first repair a bug in the expansion above. By using method call syntax, the above `get` and `set` calls give the `get` and `set` methods access to their slot object itself, via their `this` binding. Instead, as at this stage, we must ensure that they are called with their `this` binding set to the value `undefined`. Since we assume strict-only code, we can accomplish this simply:

```JavaScript
const t_1 = foo();
let t_2;
...
(
  t_2 = (1,t_1.get()) + 1,
  (1,t_1.set)(t_2),
  t_2
)
```
If `foo()` returns a slot created by our first expansion, this makes no difference, since that first expansion


## Class friends

The first motivation was to support class friends, by reifying access to the [private state](https://github.com/tc39/proposal-private-fields) as a first class slot object. However, the above rewrite rules are not adequate, as shown by the following example.

```JavaScript
{
  let xSlot;

  class Foo {
    @x = 1;
    ...
    getXSlot() { return &@x; }

    static {
      xSlot = &@x;
    }
  }
  ...
}
```
would expand to
```JavaScript
{
  let xSlot;

  class Foo {
    @x = 1;
    ...
    getXSlot() {
      return Object.freeze({
        get: () => (@x),
        set: (t_1) => {@x = t_1;}
      });
    }

    static {
      xSlot = Object.freeze({
        get: () => (@x),
        set: (t_2) => {@x = t_2;}
      });
    }
  }
  ...
}
```
The meaning of `&@x` in the instance method means what we expect -- it reifies access only to the `@x` slot of `this` instance. However, in the static context, it is useless, since there is no instance to access. To express "friend" relationships among classes, we need instead is an expansion such as that shown at [Why we don't need help reifying the private field name](https://github.com/tc39/proposal-private-fields/issues/26), where the object containing the private state is also passed as an argument to the `get` and `set` methods. Recall that
  * `@x` is always only sugar for `this.@x`
  * The `get` and `set` expansions above have no `this` in their method bodies.
  * The calls to `get` and `set` generated by the expansions above always pass the value `undefined` as the `this` binding. The following expansion shows what we were reserving it for.

For clarity, below we expand `@x` to `this.@x` and then follow the previously stated rules for extracting `this` as a subexpression.

  ```JavaScript
  {
    let xSlot;

    class Foo {
      @x = 1;
      ...
      getXSlot() { return &this.@x; }

      static {
        xSlot = &&@x;
      }
    }
    ...
  }
  ```
expands to
```JavaScript
{
  let xSlot;

  class Foo {
    @x = 1;
    ...
    getXSlot() {
      let t_1;
      return (
        t_1 = this,
        Object.freeze({
          get: () => (t_1.@x),
          set: (t_2) => {t_1.@x = t_2;}
        })
      );
    }

    static {
      xSlot = Object.freeze({
        get() { return(this.@x); },
        set(t_3) {this.@x = t_3;}
      });
    }
  }
  ...
}
```
The expression `aFoo.getXSlot()` still reifies access only to the private `@x` slot of the `aFoo` instance it was called on, since that instance is bound to `this` within the `getXSlot` method, and therefore to the `t_1` used within the generated arrow functions.

The expansion of `&&@x` is much like the expansion of `&@x`, except that the implicit `this` is not extracted, and the `get` and `set` methods use the normal method shorthand. They are normal functions whose `this` is unbound, and determined by their caller.

Disclaimers
> The `&&` syntax is ugly, especially since class friends is the motivating usecase. OTOH,
  * It is unambiguous, since `&anything` is not itself a LeftHandSideExpression, and so cannot other appear after a unary `&`.
  * If we interpret a unary prefix `&` as reifying a meta view of the meaning of its operand expression, along some meta direction, then `&&@x` makes some sense as reifying a meta view (though in a different meta direction) of the meaning of `&@x`.

During the static block, which executes during the initialization of the `Foo` class, the `xSlot` variable is initialized to a slot that gives access to the `@x` of any instance of `Foo` it is applied to. This expresses the exporting half of a friend relationship.

Say that the following class `Bar` also appears with the scope of `xSlot`:

```JavaScript
{
  let xSlot;
  ...
  class Foo {...}
  ...
  class Bar {
    &&@y = xSlot;
    ...
    incrFooX(aFoo) {
      aFoo.@y = aFoo.@y + 1;
    }
  }
}
```
expands to
```JavaScript
{
  let xSlot;
  ...
  class Foo {...}
  ...
  class Bar {
    static @t_1 = xSlot;
    ...
    incrFooX(aFoo) {
      const t_2 = @t_1.get.call(aFoo);
      @t_1.set.call(aFoo, t_2 + 1);
    }
  }
}
```
Disclaimers
> This use of `call` is not actually safe. Assuming the `applyFn` from [Safe meta programming](http://wiki.ecmascript.org/doku.php?id=conventions:safe_meta_programming), the following expansion would be safer
```JavaScript
incrFooX(aFoo) {
  const t_2 = applyFn(@t_1.get, aFoo, []);
  applyFn(@t_1.set, aFoo, [t_2 + 1]);
}
```
For the remainder of this document we ignore this safety issue

## Protected

We can take a first step towards protected by manually composing the above elements. Say `Bar` is instead a subclass of `Foo`.

```JavaScript
{
  let xSlot;
  ...
  class Foo {...}
  ...
  class Bar extends Foo {
    &&@y = xSlot;
    ...
    incrY() {
      @y = @y + 1;
    }
  }
}
```
expands to
```JavaScript
{
  let xSlot;
  ...
  class Foo {...}
  ...
  class Bar extends Foo {
    static @t_1 = xSlot;
    ...
    incrFooX() {
      const t_2 = @t_1.get.call(this);
      @t_1.set.call(this, t_2 + 1);
    }
  }
}
```

These uses of `@y` within `Bar` access the `@x` fields of instances of `Foo`, which includes instances of `Bar`. This is like `protected` except that the subclass must be in the scope used to share this friendly access.

In most languages with `protected`, only the superclass exporting access needs to declare that it is doing so. However, in JavaScript we have no static knowledge of the superclass during scope analysis of the text of the subclass. Thus, like module export/import, we require a matching `inherited` declaration in the importing subclass.

```JavaScript
class Foo {
  protected @x = 1;
  ...
  getXSlot() { return &this.@x; }
}

//...elsewhere...

class Bar extends Foo {
  inherited @x;
  ...
  incrX() {
    @x = @x + 1;
  }
}
```
This expands to something like the previous manual use of slots, but with some yet-to-be-designed mechanism by which the superclass communicates the relevant slot to the subclass. With private state, there is a hard security issue at stake. With protected state there is not. Since
  * anyone with access to a class can declare a subclass of that class, and
  * like private visibility, protected visibility is per-class, not per-instance

protected state cannot be denied to anyone with access to the class. If you want selective sharing of access to private state, use the previous explicit friend patterns instead.

## Template literal patterns

TODO(erights): write

```JavaScript
let x, y;

if (LANG`${&x} + ${&y)}`.test(ast)) {
  return LANG`${&x} * ${&y}`.value();
}
```

## Functional reactive programming

TODO(erights): write

```JavaScript
function ObervableSlot(value) {
  const observers = [];
  return Object.freeze({
    get: () => value,
    set: (newValue) => {
      value = newValue;
      for (let observer of observers) {
        Promise.resolve(observer).then(obs => obs());
      }
    },
    observe(observer) {
      observers.push(observer);
    }
  });
}
```
