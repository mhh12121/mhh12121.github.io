---
title: My Understanding towards Promise/A+
date: 2017-01-07 23:00
tags: Javascript
---
Personal comprehension about ES6 promise,including simulated promise implementation
<!--more-->
## For a stupid Instance
There's a scene that you're going to order a cup of coffee.The pretty waitress may say 'Please come to get your coffee later'.So this is a <div style="color:#f92672">Promise</div>.She promised that you can get a coffee soon.Besides, your waiting process is called <div style="color:#4d8fc0">pending</div>.After you sucessfully get your coffee from the waitress,this's called <div style="color:#4d8fc0">resolved(fulfilled)</div>.But it's possible that her coffee beans are out of stock.As a result you can't get your coffee,so this is <div style="color:#4d8fc0">rejected</div>.Furthermore,after you got the promise,the following operation is called  <div style="color:#f92672">then</div>;

## Definition
The Promise object represents the eventual completion (or failure) of an asynchronous operation, and its resulting value.[MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)

Essentially, a promise is a returned object to which you attach callbacks, instead of passing callbacks into a function.(That's why it can avoid callback hell,once you call then() method,it will implicitly <div style="color:#f92672">create a new promise object and return it;</div> ) [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises)

### More advantages:

Unlike old-style passed-in callbacks, a promise comes with some guarantees:

    1. Callbacks will never be called before the completion of the current run of the JavaScript event loop.
    2. Callbacks added with .then even after the success or failure of the asynchronous operation, will be called, as above.
    3. Multiple callbacks may be added by calling .then several times, to be executed independently in insertion order.

But the most immediate benefit of promises is chaining.

For more detail:[Promise A+](https://promisesaplus.com/)

Following my code about a simple promise implementation:
```javascript
```



