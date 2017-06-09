---
layout: post
title: "ES6 Tail Recursion in Node.js >= 6.6.0"
author: karl
date: 2017-06-09
comment: true
categories: [Articles]
published: true
noindex: false
---
Since Node.js 6.6.0 it is possible to write tail recursive functions.
However tail recursion is not supported by default.

## What is Tail Recursion?
Tail recursion is an optimisation to recursion where the last statement in a function is a call to itself.
With nothing else to process after the call to itself there is no state that needs to be placed on the call stack.
Therefore there can be no stack overflow errors when calling a tail recursive function no matter how many iterations take place.
 
Some languages recognise when a function can be optimised for tail recursion and perform tail call optimisation (TCO).
 
## Enabling TCO in Node.js
To enable TCO in Node.js the following needs to be done:

* Use strict mode
* Run Node.js with the `--harmony_tailcalls` flag

## Tail Call Function
Here's an example of a tail call recursive function to calculate the nth fibonacci number:

```
"use strict";

const fib = n => {
    const fibTail = (n, a, b) => {
        if (n === 0)
            return a;
        else
            return fibTail(n - 1, b, (a + b));
    };
    return fibTail(n, 0, 1);
};

console.log("Result: " + fib(6));
```

## Running Code with TCO
To run the above code execute the command:

`node --harmony_tailcalls fib.js`

You should receive the output:

`Result: 8`

The source code for this post can be found [here](https://github.com/karlkyck/node-es6-tail-recursion).
