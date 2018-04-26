# SCOPE AND CLOSURES

## WHAT IS SCOPE

Scope is the set of rules that determine where and how a variable can be looked-up.

* **LHS (Left-Hand Side):**

  Assignment of value to the variable

* **RHS (Right-Hand Side):**

  Retrieve the value of the variable.

## LEXICAL SCOPE

Scope is defined by decisions of where functions are declared.

Compilation consists of three steps:

1. Tokens
1. Parsing
1. Code generation

## FUNCTION VS BLOCK SCOPE

Units of scope:

* **Function**:

  Any variable declared inside a function is hidden from any enclosing scope.

* **Block**

  Any variable declared inside a block `{}`

It's a good practice to hide code in function scopes from other scopes to avoid collision and scope polution. Instead of declaring a function and then invoking it, use **IIFE (Inmediately Invoked Function Expression):**

```JS
(function foo () {
  ...
})();
```

* `var` always belongs to the enclosing scope/function scope.
* `let` is block scoped.
* `const` is block scoped but with a fixed value.

## HOISTING

Declaration of variables occur in compilation and assignment of values at execution time. It's as if declarations are moved to the top of their respective scope. `const` and `let` are not hoisted.

```JS
for (var i=1; i<=5; i++) {
  setTimeOut(function timer() {
    console.log(i);
  }, i*1000);
}    // Prints 6 5 times

for (var i=1; i<=5; i++) {
  (function () {
    var j = i;
    setTimeout(function timer() {
      console.log(j);
      }, j*1000);
  })();
}    //The IIFE creates a scope for j to be declared and immediately assigns it's value by being immediately invoked

for (let i=1; i<=5; i++) {
  setTimeout(function timer() {
    console.log(i);
  }, i*1000);
}    //The variable is declared for each iteration of the loop
```

## CLOSURE

It's when a function can remember and access it's lexical scope even when invoked outside of it's lexical scope.

```JS
function foo() {
  var a = 2;

  function bar() {
    console.log(a);
  }

  return bar;
}

var baz = foo();

baz();    //2 --Whoa, closure was just observed, man. bar() still has a reference to it's lexical scope.
```

Closueres enable a pattern called modules, which require:

1. An outer wrapping function being invoked, to create the enclosing scope.
1. The return value must include reference to at least one inner function that has closure over the private inner scope of the wrappiong function.

# `this` AND OBJECT PROTOTYPES

## `this`

`this` is a binding made when a function is invoked and what it points to is determined entirely by the call-site.

* **callstack:**

  The stack of functions that have been called to get to the current moment in execution.

* **call-site:**

  The location in code where a function is called (not where it's declared). For `this` binding, the call-site we care about is in the invocation before the currentl executing function.

### THE 4 RULES

* **DEFAULT BINDING:**

  Standalone function invocation, when no other rules apply. Resolves to purely locating the call-site for `this` binding.

  ```JS
  function foo() {
    console.log(this.a);
  }

  var a = 2;

  foo();    //2
  ```

* **IMPLICIT BINDING:**

  The call-site has a context object, called "owning/containing object". The call-site uses the object context to reference a function, `this` is binded to that object.

  ```JS
  function foo() {
    console.log(this.a);
  }

  var obj = {
    a: 2,
    foo: foo
  };

  obj.foo();    //2
  ```

  **IMPLICITLY LOST:**

  Implicitly bound functions can lose their binding, so they fallback to the default binding, global object or undefined, in two cases:

  * Function reference/alias to the "containing object" function:

    ```JS
    function foo() {
    console.log(this.a);
    }

    var obj = {
    a: 2,
    foo: foo
    };

    var bar = obj.foo; //function reference/alias!

    var a = "oops, global"; //a also property on global object

    bar(); //"oops, global"
    ```

    > Note: `bar` is only a reference to `foo` and the call-site is `bar()` so the default binding applies.

  * Callbacks (either our own callbacks or built-in to the language):

    ```JS
    function foo() {
      console.log(this.a);
    }

    function doFoo(fn) {
      //fn is just another reference to foo
      fn();    // <-* call-site!
    }

    var obj = {
      a: 2,
      foo: foo
    };

    var a = "oops, global"; //a also property on global object

    doFoo(obj.foo);    //"oops, global"
    ```

    >Note: Parameter passing is just an implicit assignment, so the end result is the same as the previos case.

* **EXPLICIT BINDING**

  The function utilises either call, apply or bind, taking as their first parameter the object to use for `this`.

  ```JS
  function foo() {
    console.log(this.a);
  }

  var obj = {
    a: 2
  };

  foo.call(obj);    //2
  ```

  But this still doesn´t fix the problem of a function losing it's binding, that's why we need a variaton of it:

  **Hard Binding**

  Create a function which internally manually explicitly calls the function to be bound, thereby invoking the function with that object always bound.

  ```JS
  function foo() {
    console.log(this.a);
  }

  var obj = {
    a: 2
  };

  var bar = function() {
    foo.call(obj);
  };

  bar();    //2
  setTimeout(bar, 100);    //2

  //bar hard binds foo's this to obj so that it can't be overriden
  bar.call(window);    //2
  ```

  The most typical way to wrap a function with a hard binding is creating a re-usable helper, which creates a pass-thru of any arguments passed and any return value recived:

  ```JS
  function foo(something) {
    console.log(this.a, something);
    return this.a + something;
  }

  //simple bind helper
  function bind(fn, obj) {
    return function() {
      return fn.apply(obj, arguments);
    };
  }

  var obj = {
    a: 2
  };

  var bar = bind(foo, obj);

  var b = bar(3);    //2 3
  console.log(b);    //5
  ```

  Since hard binding is such a common pattern, it's provided with ES5, returning a new function that is hard-coded to call the original function with `this` context set as specified:

  ```JS
  function foo(something) {
    console.log(this.a, something);
    return this.a + something;
  }

  var obj = {
    a:2
  };

  var bar = foo.bind(foo, obj);

  var b = bar(3);    //2 3
  console.log(b);    //5
  ```

  Many libraries functions and built-in functions in JS (like array functions), provide an optional parameter, which is designed as a work-around for not having to use bind, but internally they use explicit binding.

* **NEW BINDING**

  In JS constructors are just functions that happen to be called with the word `new` in front of them, they are not attached to classes, nor instantiating a class. Pretty much any function can be called with the word `new`, making it a constructor call of a function. When this happens:

  1. A new object is created (constructed).
  1. The new object is [[Prototype]] linked.
  1. The new object is set as the `this` binding of that function call.
  1. Unless the function returns it's own alternate object, the new object is returned.

  ```JS
  function foo(a) {
    this.a = a;
  }

  var bar = new foo(2);
  console.log(bar.a);    //2
  ```

### ORDER OF THE 4 RULES

1. Is the function called with `new` (new binding)? If so, `this` is the newly constructed object.

    ```JS
    var bar = new foo();
    ```

1. Is the function called with call or apply (expliciti binding), even hidden inside a bind hard binding? If so, `this` is the explicitly specified object.

    ```JS
    var bar = foo.call(obj2);
    ```

1. Otherwise, default the `this` (default binding). If in strict mode, pick undefiend, otherwise pick the global object.

    ```JS
    var bar = foo();
    ```

### CURRYING

The primary reason for new > explicit, is to create a function that ignores the explicit `this` binding but presents the function's arguments. One of the capabilites of `bind` is that any arguments passed after `this` are defaulted as arguments for the underlying function.

```JS
function foo(p1, p2) {
  this.val = p1 + p2;
}

//using null here because we don´t care about the this hard-binding in this scenario, and it will be overriden by the new call anyway
var bar =  foo.bind(null, "p1");
var baz = new bar("p2");
baz.val;    //p1p2
```

### IGNORED `this`

In explicit binding you can ignore `this` by passing `null`, this is common for spreading arrays or currying:

```JS
function foo(a, b) {
  console.log("a:" + a + ", b:" + b);
}

//spreading out array as parameters
foo.apply(null, [2, 3]);    //a:2, b:3

//currying with `bind(..)`
var bar = foo.bind(null, 2);
bar(3);    //a:2, b:3
```

There's a danger of passing `null`, in some cases (like third parties) can make a reference to `this`, applying the default binding and making a reference to the global object. Which is why it's better to use, instead of `null`, something like this:

```JS
var ø = Object.create(null);
```

### LEXICAL `this`

Arrow functions don't abide by the 4 rules, instead they adopt the 'this' binding from the enclosing scope:

```JS
function foo() {
  //return an arrow function
  return (a) => {
    //`this` here is lexically adopted from `foo()`
    console.log(this.a);
  };
}

var obj1 = {
  a: 2
};

var obj2 = {
  a: 3
};

var bar = foo.call(obj1);
bar.call(obj2); //2, not 3!
```

The arrow function created by foo captures `this` at call-time. Since foo's `this` was bound to obj1, bar's `this` will too. Lexical binding of arrow functions cannot be overriden by any of the 4 rules.

The most common case of arrow functions are callbacks:

```JS
function fo() {
  setTimeout(() => {
    //`this` here is lexically adopted from `foo()`
    console.log(this.a);
  }, 100);
}

var obj = {
  a: 2
};

foo.call(obj); //2
```

Which is basically:

```JS
function fo() {
  var self = this; //lexical capture of `this`

  setTimeout(function () {
    console.log(self.a);
  }, 100);
}

var obj = {
  a: 2
};

foo.call(obj); //2
```

## OBJECTS

Can be created in two forms:

* **Literal:**

  ```JS
  var myObj = {
    key: value
    //...
  };
  ```

* **Constructed form:**

  ```JS
  var myObj = new Object();
  myObj.key = value;
  ```

## TYPES

There are 6 primary types in JS:

* string
* number
* boolean
* null
* undefined
* object

The first 5 are called simple primitives and they are not objects.

There are a few special object sub-types, called complex primitives:

* **function**

  A "callable object". Functions in JS are "first calss", which means they can bhe handle like any other plain object.
* **arrays**

  Objects with extra behavior. The organization of contents in arrays is more structured than general objects.

### BUILT-IN OBJECTS

There are other object subtypes (note first letter is uppercase):

* String
* Number
* Boolean
* Object
* Function
* Array
* Date
* RegExp
* Error

These have the appearane of being actual types, even classes. But in JS they are just built-in constructors (function call with a `new` operator) with the result beign a newly constructed object of the subtype in question:

```JS
var strPrimitive = "I am a string";
typeof strPrimitive; // "string"
strPrimitive instanceof String; // false

var strObject = new String("I am a string");
typeof strObject: // "object"
strObject instanceof String; // true
```

To perform operations on them (like checking length) a String object is required. Lucikly JS automatically creates these from the "string" object when necessary.

## CONTENTS

The contents of an boject consist of values stroed at specifically named locations, called properties.

To access a vlaue at a location, we can use:

* **Property access**

  ```JS
  myObject.a
  ```

* **Key access**

  ```JS
  myObject["a"]
  ```

Property access requires an Identifier compatible property and key access can take basically any string as the name of the property. So with key access we can programatically build up the value of the string property name.

  ```JS
  var wantA = true;
  var myObject = {
    a: 2
  };

  var idx;

  if (wantA) {
    idx = "a";
  }

  //later

  console.log(myObject[idx]); //2
  ```

In objects, property names are always strings. So if you use any other value other than a string, JS will first convert the value to string.

  ```JS
  var myObject = {};

  myObject[true] = "foo";
  myObject[3] = "bar";
  myObject[myObject] = "baz";

  myObject["true"]; //"foo"
  myObject["3"];  //"bar"
  myObject["[object Object]"];  //"baz"
  ```

## COMPUTED PROPERTY NAMES

ES6 adds the ability to specify an expression, surronded by [], in the property name position of an object literal declaration:

  ```JS
  var prefix = "foo";
  ```

## ARRAYS

Arrays are objects that have better organization for how and where values are stored:

  ```JS
  var myArray = [ "foo", 42, "bar" ];
  myArray.length; // 3
  myArray[0]; // "foo"
  myArray[2]; // "bar"
  ```

Since they are objects, although not recommended, you can add properites to them without changing the array length:

  ```JS
  var myArrary = [ "foo", 42, "bar" ];
  myArray.baz = "baz";
  myArray.length; // 3
  myArray.baz;  // "baz"
  ```

## DUPLICATING OBJECTS

For objects that are JSON-safe (can be serialized to JSON string and then back to JSON with the same structure and vlues) you can use:

  ```JS
  var newObj = JSON.parse(JSON.stringify(someObj));
  ```

In cases when you can't ensure JSON-safe, you can create a shallow copy (objects and functions assigned to properties are just references to these objects/functions):

  ```JS
  var newObj = Object.assign({}, myObject);

  newObj.a; // 2
  newObj.b === anotherObject; // true
  newObj.c === anotherArray;  // true
  newObj.d === anotherFunction; // true
  ```

## PROPERTY DESCRIPTORS

These are characteristics every object property has:

  ```JS
  var myObject = {
    a:2
  };

  Object.getOwnPropertyDescriptor( myObject, "a");
  // {
  //   value: 2,
  //   writable: true,
  //   enumerable: true,
  //   configurable: true
  // }
  ```

Those are the default values, but we can modify them (if it's configurable):

  ```JS
  var myObject = {};

  Object.defineProperty(myObject, "a", {
    value: 2,
    writable: true,
    configurable: true,
    enumerable: true
  });

  myObject.a; //2
  ```

There are 3 (excluding value) characteristics:

* **Writable**

  If you can change the value of a property.

* **Configurable**

  If you can modify the propertie's descriptor definition, using again defineProperty(..). IF a property is not configurable, you can still change writable from ture to false, but not from false to true. It also prevents from using the delete operator on the property.

* **Enumerable**

  If a property will show up in certain object-property enumerations, like `for..in` loop.

## IMMUTABILITY

To make properties or objects that cannot be changed. All of these approaches create shallow immutability, meaning they only affect the object and direct properties. If an object has a reference to another object (array, object, function, etc.) the contents of that object remain mutable.

* **OBJECT CONSTANT**

  Combine `writable: false` and `configurable: false`. It creates a constant (cannot be changed, redefined or deleted).

    ```JS
    var myObject = {};

    Object.defineProperty(myObject, "FAVORITE_NUMBER", {
      value: 42,
      writable: false,
      configurable: false
    });
    ```

* **PREVENT EXTENSIONS**

  Prevents an object from having new properties added to it, but leaves the rest of the properties alone.

  ```JS
  var myObject = {
    a: 2
  };

  Object.preventExtensions(myObject);

  myObject.b = 3;
  myObject.b; // undefined
  ```

* **SEAL**

  Object.seal(..) esentially calls Object.preventExtensions(..) but also marks all existing properties as `configurable:false`. So you cannot add more properties, and also cannot reconfigure or delete existing properties. But you can modify their vlaues.

* **FREEZE**

  Object.freeze(..) esentially calls Object.seal(..) but also marks all properties as `writable:false`.

## [[GET]]

When you access an object property, it doesn't just look in the object for the property. It actually performs a [[Get]] operation (like a function call). This function first inspects the object for the property name and if it can find it then returns the value, if not the [[Get]] algorithm defines other important behavior and returns undefined (as opossed to a variable that cannot be resolved, where ReferenceError is thrown).

  ```JS
  var myObject = {
    a: 2
  };

  myObject.b; // undefined
  ```

## [[PUT]]

How this operation behaves differs based on a number of factors, mainly if the property is already present or not.

If already present:

1. Is the property an accessor descriptor (see below)? Then call the setter.
1. Is the property a data descriptor with `writable:false`? Then silently fail in non-strict-mode, or throw TypeError in strict-mode.
1. Otherwise, set the value of the property as normal.

## GETTERS & SETTERS

ES5 introduces a way to override part of the default operations of [[Get]] and [[Put]] at a per-property level. When a property has a getter and/or setter, the definiton becomes an accessor descriptor (as opposed to data descriptor). For accessor descriptors, JS ignores the value and writable characteristics, and considers set, get, configurable and enumerable.

  ```JS
  var myObject = {
    // define a getter for `a`
    get a() {
      return 2;
    }
  };

  Object.defineProperty(
    myObject, // target
    "b",  // property name
    {
      //define a getter for `b`
      get: function () { return this.a * 2 },
      //make sure `b` shows up as an object property
      enumerable: true
    }
  );

  myObject.a; // 2
  myObject.b; // 4
  myObject.a = 3;
  myObject.a; // 2
  ```

Thus creating a property on the object that doesn't actually hold a value, but whose access automatically results in a hidden function call. And even if we try to set a value for `a` later or there's a valid setter, the custom getter is hard-coded to return 2.

It's recommended to always define both a getter and a setter:

  ```JS
  var myObject = {
    // define a getter for `a`
    get a() {
      return this._a_
    },
    //define a setter for `a`
    set a(val) {
      this._a_ = val *2;
    }
  };

  myObject.a = 2;
  myObject.a; //4
  ```

## EXISTENCE

You can ask an object if it has a certain property without asking to get that property's value:

  ```JS
  var myObject = {
    a: 2
  };

  ("a" in myObject);  // true
  ("b" in myObject);  // false

  myObject.hasOwnProperty("a"); // true
  myObject.hasOwnProperty("b"); //false
  ```

The `in` operator checks if the property is in the object, or if it exists at any higher level of the [[Proptotype]] chain. `hasOwnProperty(..)` only checks if the object has the property or not.

`hasOwnProperty(..)` is accessible via delegation to `Object.prototype`. BUt it is possible to create an object that doesn't link to `Object.prototype`. So there's a more robust way to check this:

  ```JS
  Object.prototype.hasOwnProperty.call(myObject, "a");
  ```

### ENUMERATION

There are properties that exist but they won't show up in some loops (`for..in`) or operations. That's because enumerable basically means "will be included if the object's properties are iterated through".

  ```JS
  var myObject = {};

  Object.defineProperty(
    myObject,
    "a",
    //make `a` enumerable, as normal
    { enumerable: true, value: 2 }
  );

  Object.defineProperty(
    myObject,
    "b",
    //make `b` NON-enumerable
    { enumerable: false, value: 3 }
  );

  // ...

  for (var k in myObject) {
    console.log(k, myObject[k]);
  }
  //  "a" 2

  myObject.propertyIsEnumerable("a"); // true
  myObject.propertyIsEnumerable("b"); // false

  Object.keys(myObject);  // ["a"]
  Object.getOwnPropertyNames(myObject); // ["a", "b"]
  ```

`Object.keys(..)` returns an array of all enumerable properties, whereas `Object.getOwnPropertyNames(..)` returns an array of all properties. Both inspect only the direct object specified, not references to other objects it may include.