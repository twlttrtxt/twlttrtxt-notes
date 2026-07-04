- has objects which are a collection of `key:value` pairs:

```javascript
const user = { username: "wiener", userId: 01234, isAdmin: false }
```

- They are accessible like this:

```javascript
user.username // "wiener"
user['userId'] // 01234
```

- The values may also be executable functions:

```javascript
const user = { username: "wiener", userId: 01234, checkAdmin: function(){...} }
```

- This is an `object literal`. That means it was created using curly braces `{...}` to declare the properties and values. Almost everything in javascript is an object.

- Every object in JS is linked to another object `-> its prototype!` By default, JS automatically gives new objects a built-in prototype. Examples:

```javascript
let myObject = {}; 
Object.getPrototypeOf(myObject); // ---> Object.prototype
let myString = ""; 
Object.getPrototypeOf(myString); // ---> String.prototype 
let myArray = []; 
Object.getPrototypeOf(myArray); // ---> Array.prototype 
let myNumber = 1; 
Object.getPrototypeOf(myNumber); // ---> Number.prototype
```

- Objects inherit all properties of the assigned prototype unless they define them with the same keys. For example, when creating a `String`, the object automatically inherits the `toLowerCase()` Method unless defining it itself.
- When trying to access a property of an object, the JS engine looks into the object itself to find it. If there is no matching property, it looks into the prototype instead!
- To see this behaviour in action, open the browser console and create an empty object:

```javascript
let myObject = {};
```

- When typing `myObject.` after creation, the console prompts you to select from a list of properties. Those were not defined during creation, but inherited from the builtin `Object.prototype`

- Each objects prototype is another object, which also has another prototype. Every prototype chain leads to the top-level `Object.prototype`, which has the prototype `null`.
- This means that if you create a `string` Object, it inherits all properties and method from the prototype chain, so `String.prototype`, and `Object.prototype`.

- Each object has a property which can be used to access the prototype. The standard used by most browsers is essentially `__proto__`. This serves as a getter and setter for the objects prototype. These `__proto__` accesses can also be chained:

```javascript
username.__proto__ // String.prototype
username.__proto__.__proto__ // Object.prototype
username.__proto__.__proto__.__proto__ // null
```

- It is possible to modify JS built-in prototypes. This allows for the customisation of the behaviour of the built-in functions.
- Developers sometimes added a function `removeWhitespace` to remove trailing whitespaces:

```javascript
String.prototype.removeWhitespace = function(){ 
// remove leading and trailing whitespace 
}
// Each string would have access to this function now
let searchTerm = " example ";
searchTerm.removeWhitespace(); // "example"
```

