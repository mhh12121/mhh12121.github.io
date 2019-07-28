---
title: From SetTimeout to Js Scope
date: 2017-01-01 21:48
tags: javascript
---
There's a chinese word '温故而知新'.
<!--more-->
Today I saw a code:
```javascript

for (var i=1;i<=5;i++){
    setTimeout(function timer(){
        console.log(i);

    },i*1000);
}
```
And the result is:

```
34//a random number,it's different any time after u reopen your browser
6//run 5 times
```
Directly go to [Solution](#The-1stQ-solution) !!!!!

To explain above,We begin with some of the characteristics of [setTimeout](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/setTimeout)
##Syntax
```
var timeoutID = scope.setTimeout(function[,delay,param1,param2]);
```
## Return value
we should know about the setTimeout() return a **timeoutID**.
1. this is an positive integer value
2. this value identifies the timer created by setTimeout()(this is the reason why it's different all the time)
3. setTimeout() and setInterval() share the same pool of IDs,which means setInterval() may effect the timer if u use them at the same object(window or a worker);


## "This" problem
Let's first see a simpler example:
```javascript
(function() {
    this.fuck = false;
    Console.log("OMG!");
    var timer = setTimeout((function() {
        this.fuck = true;
        Console.log("timeout Fuck!");
    }),3000);
})
```
The problem of above example is :
The <span style="color:#f92672">this</span> always refers to the 
<span style="color:#f92672">this</span> of the current scope,which changes any time I wrap something in <span style="color:#f92672">function(){}</span>


So there is 
###one solution:
```javascript
(function() {
    var symbol = this;//now refers to the Window object 
    symbol.fuck = false;
    Console.log("OMG!");
    var timer = setTimeout((function() {
        symbol.fuck = true;//still refers to the Window Object
        Console.log("timeout Fuck!");
    }),3000);
})
```

###Second
ES5 Function.prototype.bind()
```javascript
(function() {
    this.fuck = false;
    Console.log("OMG!");
    var timer = setTimeout((function() {
        this.fuck = true;
        Console.log("timeout Fuck!");
    }).bind(this),3000);//bind the scope
})
```

###Third
ES6 [Arrow Function()]
```javascript
(function() {
    this.fuck = false;
    Console.log("OMG!");
    var timer = setTimeout(()=> {
        this.fuck = true;
        Console.log("timeout Fuck!");
    },3000);//no binding in Arrow Functions!!!
})
```
I think now it's clear that why the first code resulted in that,
The solution:

###The 1stQ solution
```javascript
for (var i=1;i<=5;i++){
    
    (function(i){//1.set the self-invoking anonymous function to set the scope
        setTimeout(function timer(){
            console.log(i);
        },i*1000);
    })(i)//2.Then call the value of i 
}
```

Also u can try this,result is the same:
```javascript
for (var i=1;i<=5;i++){
    setTimeout((function timer(i){
        return function(){//set return value as an anonymous function
            console.log(i);
        }
    })(i),i*1000);
}
```


 

