<!--#region:intro-->
# ECMAScript class property access expressions

Class access expressions seek to simplify access to static members of a class as well as provide
access to static members of a class when that class is unnamed:

```js
class C {
  static f() { ... }
  
  g() {
    class.f();
  }
}
```

* [Stage 0 Presentation](https://docs.google.com/presentation/d/1VXqGgxq_a0byLuc9yl4jSLR6GlopjvhU_Z0OpZnk6l0/edit?usp=sharing)

<!--#endregion:intro-->

<!--#region:status-->
## Status

**Stage:** 1  
**Champion:** Ron Buckton (@rbuckton)  

_For detailed status of this proposal see [TODO](#todo), below._  
<!--#endregion:status-->

<!--#region:authors-->
## Authors

* Ron Buckton (@rbuckton)  
<!--#endregion:authors-->

<!--#region:prior-art-->
<!--
# Prior Art 

> TODO: Add links to similar concepts in existing languages, prior proposals, etc.

* Language: [Concept](url)  
-->
<!--#endregion:prior-art-->

<!--#region:motivations-->
# Motivations

Today, ECMAScript developers can write classes with static members that can be accessed in one of
two ways:

- Via the class name:
  ```js
  class C {
    static x() {}
    y() { C.x(); }
  }
  ```
- Via `this` in a static member:
  ```js
  class C {
    static x() {}
    static y() { this.x(); }
  }
  ```

However, there is no easy mechanism to access class statics in a class that has no name:

```js
export default class {
  static x() { }
  y() { 
    this.constructor.x(); // Not actually guaranteed to be the correct `x`.
  }
}


const x = class { 
  static x() { }
  y() {
    this.constructor.x(); // Not actually guaranteed to be the correct `x`.
  }
}
```

It’s not guaranteed to be the correct `x` in the above examples, because you could be
accessing an overridden member of a subclass.

Also, with current proposal for private static fields and methods, its very easy to
run into errors at runtime when using `this` in a static method:

```js
class C {
  static #x() {}
  static y() { this.#x(); }
}
class D extends C {}
D.y(); // TypeError
```

<!--#endregion:motivations-->

<!--#region:syntax-->
# Syntax

```js
// from a non-static method
class C {
  static f() { }
  g() {
    class.f();
    class["f"]();
  }
}

// from a static method
class C {
  static f() { }
  static g() {
    class.f();
    class["f"]();
  }
}

// with static private members
class C {
  static #f() {}
  static g() {
    class.#f();
  }
}
```
<!--#endregion:syntax-->

<!--#region:semantics-->
# Semantics

- Function Environment Records have a new field in Table 15:
  > | Field Name | Value | Meaning |
  > |:-|:-|:-|
  > | \[\[InitialClassObject]] | Object \| **undefined** | If the associated function has `class` property access and is not an _ArrowFunction_, \[\[InitialClassObject]] is the class that the function is bound to as a method. The default value for \[\[InitialClassObject]] is **undefined**. |
- Function Environment Records have a new method in Table 16:
  > | Method | Purpose |
  > |:-|:-|
  > | GetClassBinding() | Return the object that is the base for `class` property access bound in this Environment Record. |
- ECMAScript Function Objects have a new internal slot in Table 27:
  > | Interanl Slot | Type | Description |
  > |:-|:-|:-|
  > | \[\[InitialClassObject]] | Object \| **undefined** | If the function has `class` property access, this is the object where `class` property lookups begin. |
- During ClassDefinitionEvaluation, the class constructor (<var>F</var>) is set as the \[\[InitialClassObject]] on the method.
- During NewFunctionEnvironment, the \[\[InitialClassObject]] is copied from the method (<var>F</var>) to <var>envRec</var>.\[\[InitialClassObject]].
- Arrow functions use the \[\[InitialClassObject]] of their containing lexical environment (similar to `super` and `this`).
- When evaluating ``ClassProperty: `class` `.` IdentifierName`` we return a new Reference with the following properties:
  - The referenced name component is the StringValue of _IdentifierName_. 
  - The base value component is the \[\[InitialClassObject]] of GetThisEnvironment().
- When evaluating ``ClassProperty: `class` `[` Expression `]` `` we return a new Reference with the following properties:
  - The referenced name component is the result of calling ?ToPropertyKey on the result of calling GetValue on the result of evaluating _Expression_. 
  - The base value component is the \[\[InitialClassObject]] of GetThisEnvironment().
- When evaluating ``ClassProperty: `class` `.` PrivateIdentifier`` we perform the following steps:
  1. Let `fieldNameString` be the StringValue of _PrivateIdentifier_. 
  2. Let `bv` be the \[\[InitialClassObject]] of GetThisEnvironment().
  3. Return ?[MakePrivateReference](https://tc39.es/proposal-class-fields/#sec-makeprivatereference)( `bv`, `fieldNameString` )
- `class` bindings are only available in the following declarations:
  - A method of a class.
  - An accessor of a class.
  - A field initializer of a class.
  - The constructor of a class.
  - A class's `static` intialization block (see http://github.com/tc39/proposal-class-static-block)
  - An *ArrowFunction* in any of these declarations.
- `class` bindings are *not* valid in any of the following declarations:
  - A *FunctionDeclaration* or *FunctionExpression*.
  - A method of an object literal.
  - An accessor of an object literal.
  - Any execution context not nested within a lexical `class` declaration, including module or global scope.
  - *NOTE:* This aligns with the behavior of `super`.

<!--#endregion:semantics-->

<!--#region:examples-->
## Examples

### Property Access

In class methods or the class constructor, getting the value of `class.x` always refers to the value of the property `x` on the 
containing lexical class:

```js
class Base {
    static f() {
        console.log(`this: ${this.name}, class: ${class.name})`);
    }
}
class Sub extends Base {
}

Base.f();                           // this: Base, class: Base
Sub.f();                            // this: Sub, class: Base
Base.f.call({ name: "Other" });     // this: Other, class: Base
```

This behavior provides the following benefits:

- Able to reference static members of the containing lexical class without needing to repeat the class name.
- Able to reference static members of an anonymous class declaration or expression:
  ```js
  export default class {
      static f() { ... }
      g() { class.f(); }
  }
  ```

### Property Assignment

In class methods or the class constructor, setting the value of `class.x` always updates the value of the property `x` on the 
containing lexical class:

```js
function print(F) {
    const { name, x, y } = F;
    const hasX = F.hasOwnProperty("x") ? "own" : "inherited";
    const hasY = F.hasOwnProperty("y") ? "own" : "inherited";
    console.log(`${name}.x: ${x} (${hasX}), ${name}.y: ${y} (${hasY})`);
}

class Base {
    static f() {
        this.x++;
        class.y++;
    }
}

Base.x = 0;
Base.y = 0;

class Sub extends Base {
}

print(Base);                        // Base.x: 0 (own), Base.y: 0 (own)
print(Sub);                         // Sub.x: 0 (inherited), Sub.y: 0 (inherited)

Base.f();

print(Base);                        // Base.x: 1 (own), Base.y: 1 (own)
print(Sub);                         // Sub.x: 1 (inherited), Sub.y: 1 (inherited)

Sub.f();

print(Base);                        // Base.x: 1 (own), Base.y: 2 (own)
print(Sub);                         // Sub.x: 2 (own), Sub.y: 2 (inherited)

Base.f();

print(Base);                        // Base.x: 2 (own), Base.y: 3 (own)
print(Sub);                         // Sub.x: 2 (own), Sub.y: 3 (inherited)
```

This behavior provides the following benefits:

- Assignments always occur on the current lexical class, which should be unsurprising to users.

### Method Invocation

Invoking `class.x()` in a method, an initializer, or in the constructor uses the value of containing lexical class as the receiver:

```js
class Base {
    static f() {
        console.log(`this.name: ${this.name}, class.name: ${class.name})`);
    }
    static g() {
        class.f();
    }
    h() {
        class.f();
    }
}
class Sub extends Base {
}

Base.g();                           // this: Base, class: Base
Sub.g();                            // this: Sub, class: Base
Base.g.call({ name: "Other" });     // this: Other, class: Base

let b = new Base();
let s = new Sub();
b.h();                              // this: Base, class: Base
s.h();                              // this: Sub, class: Base
b.h.call({ name: "Other" });        // this: Other, class: Base
```

### Invalid usage

```js
class C {
  static x = 1;
  constructor() {
    function f() {
      return class.x; // function has no access to `class.`
    }
    f(); // throws TypeError

    const obj = {
      method() {
        return class.x; // method of object literal has no access to `class`.
      }
    };
    obj.method(); // throws TypeError
  }
}
```

<!--#endregion:examples-->

<!--#region:api-->
<!--
# API

> TODO: Provide description of High-level API.
-->
<!--#endregion:api-->

<!--#region:grammar-->
# Grammar

```grammarkdown
MemberExpression[Yield, Await] :
  ClassProperty[?Yield, ?Await]

ClassProperty[Yield, Await] :
  `class` `[` Expression[+In, ?Yield, ?Await] `]`
  `class` `.` IdentifierName
  `class` `.` PrivateIdentifier
```
<!--#endregion:grammar-->

# Relationship to Other Proposals

## Class Fields

This proposal can easily align with the current class fields proposal, providing easier access to static fields without unexpected behavior:

```js
class Base {
    static counter = 0;
    id = class.counter++;       // `Base` is used as `this`
}

class Sub extends Base {
}

console.log(new Base().id);     // 0
console.log(new Sub().id);      // 1
console.log(Base.counter);      // 2
console.log(Sub.counter);       // 2
```


## Class Private Fields

In addition to private methods, this proposal can also align with the current proposals for class private fields, providing 
access to class static private state without introducing TypeErrors due to incorrect `this`:

```js
class Base {
    static #counter = 0;
    static increment() {
        return class.#counter++;
    }
}

class Sub extends Base {
}

console.log(Base.increment());  // 0
console.log(Sub.increment());   // 1
console.log(Base.increment());  // 2
```

## Class Private Methods

One of the benefits of `class` access expressions is that they guarantee the correct reference is used when accessing static private members. Special care must be taken, however, when invoking private and non-private static methods using `class` access expressions, as the `this` binding within the invoked method will be the lexical class declaration:

```js
class Base {
  static #counter = 0;
  static #increment() {
    class.#counter++;
    this.printCounter();
  }
  static doIncrement() {
    class.#increment();
  }
  static printCounter() {
    console.log(class.#counter);
  }
}
class Sub extends Base {
  static printCounter() {
    console.log("Custom Counter");
    super.printCounter();
  }
}

Base.doIncrement(); // prints: 1
Sub.doIncrement(); // prints: 2
```

In the example above, `Sub`'s overriden `printCounter` is never invoked. Such method calls would need to be rewritten:

```js
// option 1:
class Base {
  ...
  static #increment() {
    ...
  }
  static doIncrement() {
    class.#increment.call(this);
  }
}

// option 2:
class Base {
  ...
  static #increment(C) {
    ...
    C.printCounter();
  }
  ...
  static doIncrement() {
    class.#increment(this);
  }
}
```

This is due to the fact that `class.` in this example is essentially a substitute for `Base.`, therefore `Base` becomes the receiver in these method calls.


<!--#region:references-->
<!--
# References

> TODO: Provide links to other specifications, etc.

* [Title](url)  
-->
<!--#endregion:references-->

<!--#region:prior-discussion-->
<!--
# Prior Discussion

> TODO: Provide links to prior discussion topics on https://esdiscuss.org.

* [Subject](https://esdiscuss.org)  
-->
<!--#endregion:prior-discussion-->

<!--#region:todo-->
# TODO

The following is a high-level list of tasks to progress through each stage of the [TC39 proposal process](https://tc39.github.io/process-document/):

### Stage 1 Entrance Criteria

* [x] Identified a "[champion][Champion]" who will advance the addition.  
* [x] [Prose][Prose] outlining the problem or need and the general shape of a solution.  
* [x] Illustrative [examples][Examples] of usage.  
* [x] High-level [API][API].  

### Stage 2 Entrance Criteria

* [x] [Initial specification text][Specification].  
* [ ] [Transpiler support][Transpiler] (_Optional_).  

### Stage 3 Entrance Criteria

* [ ] [Complete specification text][Specification].  
* [ ] Designated reviewers have [signed off][Stage3ReviewerSignOff] on the current spec text.  
* [ ] The ECMAScript editor has [signed off][Stage3EditorSignOff] on the current spec text.  

### Stage 4 Entrance Criteria

* [ ] [Test262](https://github.com/tc39/test262) acceptance tests have been written for mainline usage scenarios and [merged][Test262PullRequest].  
* [ ] Two compatible implementations which pass the acceptance tests: [\[1\]][Implementation1], [\[2\]][Implementation2].  
* [ ] A [pull request][Ecma262PullRequest] has been sent to tc39/ecma262 with the integrated spec text.  
* [ ] The ECMAScript editor has signed off on the [pull request][Ecma262PullRequest].  
<!--#endregion:todo-->

<!--#region:links-->
<!-- The following links are used throughout the README: -->
[Process]: https://tc39.github.io/process-document/
[Proposals]: https://github.com/tc39/proposals/
[Grammarkdown]: http://github.com/rbuckton/grammarkdown#readme
[Champion]: #status
[Prose]: #motivations
[Examples]: #examples
[API]: #api
[Specification]: https://tc39.es/proposal-class-access-expressions

<!-- The following links should be supplied as the proposal advances: -->
[Transpiler]: #todo
[Stage3ReviewerSignOff]: #todo
[Stage3EditorSignOff]: #todo
[Test262PullRequest]: #todo
[Implementation1]: #todo
[Implementation2]: #todo
[Ecma262PullRequest]: #todo
<!--#endregion:links-->
