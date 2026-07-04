- Its good to note down what endpoints do:

```java
|Home Page|
/
|This page contains a summary of what Acme IT Support does with a company photo of their staff.|

|Latest News|
/news
|This page contains a list of recently published news articles by the company, and each news article has a link with an id number, i.e. /news/article?id=1|

|News Article|
/news/article?id=1
|Displays the individual news article. Some articles seem to be blocked and reserved for premium customers only.|
```

## View Page Source

- HTML Comments may reveal info
- May reveal hidden (traversable?) directories in `href`s
- May reveal the used framework (CVEs?)

## Inspector

- May have `hidden` artifacts, which can be made visible!
- Edit things to make them visible/invisible

## Debugger

- May reveal info during processing
- Set breakpoint in interesting function, step through to get info (see what atobs do maybe?)

## Network

- See what resources are fetched during posts
- See the response