#### Identify Sources

- To identify sources, you must go through trial and error. It involves following these steps

##### 1. 

- Inject a property via the query string, URL fragment, and any JSON input:

```http
vulnerable-website.com/?__proto__[foo]=bar
```
##### 2.

- In the browser console, inspect `Object.prototype` to see if it was polluted:

```javascript
Object.prototype.foo
// "bar" means pollution has worked.
```
##### 3.

- Try with different techniques

```http
vulnerable-website.com/?__proto__.foo=bar
```
##### 4.

- Repeat for each potential source

### This is tedious, use DOM-Invader!

- Open a burp browser, and go to extensions
- Enable DOM-Invader and go to attack types and select prototype pollution
- Reload the page 
- Right-click -> Inspect -> DOM Invader. 
- When it finds something, you can scan for gadgets. When it finds gadgets you can exploit it.

- This can lead to DOM-based XSS!

#### Identify Gadgets

- Then we need to identify Gadgets. To do so, go to the `Sources` Tab inside of developer tools and look through the Javascript Code.
- Try to find dangerous functions like `eval()` which take in variables which have not been defined. That means that we can overwrite them with arbitrary values like `alert` with our prototype pollution.
- You can always set breakpoints and debug this!!!

- This can also be done with DOM-Invader!

#### Blocked `__proto__` constructor

- Each object has a `constructor` property which tells you what function was used to create it.
- Each `constructor` is also a object, meaning it has a `prototype` property!

-> `myObject.constructor.prototype` is the same as `myObject.__proto__`

#### Stripped `__proto__` constructor

- When it is stripped only once, it can be inserted inside of another one. Meaning:

```http
vulnerable-website.com/?__pro__proto__to__.gadget=payload
```

- This call, will result in this:

```http
vulnerable-website.com/?__proto__.gadget=payload
```

## Pollution via JavaScript APIs

- A good example is the `fetch()` API. It takes to arguments, the URL to send a request, and options like method, headers, ...

```javascript
fetch('https://normal-website.com/my-account/change-email', { 
	method: 'POST', 
	body: 'user=carlos&email=carlos%40ginandjuice.shop'
})
```

- Options like `headers` are undefined though. If the `Object.prototype` is polluted with the value `headers`, this will be passed into the `fetch()` call as default.