---
layout: post
title:  "JS Promises"
date:   2015-01-16 22:30:00
summary: "Carrying state around in JS when you do not know about bind (or cannot use it)."
tags:
- js
---

In JavaScript, entering [callback hell](http://callbackhell.com/) or the [pyramid of doom](http://survivejs.com/common_problems/pyramid.html) is generally considered to be [a bad idea](https://www.youtube.com/watch?v=Pr-8AP0To4k). Naming functions and keeping code shallow will keep you out of trouble. On it’s own, however, this doesn't change the fact that you have to pass callback arguments as inputs to your functions. [Promises](https://www.promisejs.org/patterns/) solve that problem and more. In particular they compose nicely and help you handle errors with minimal repetition.

Below is a module that exposes three functions. Function `getA` takes no arguments and returns `'a'`. Function `getB` takes the argument `'a'` and returns `'b'`. Function `getC` takes two arguments `'a'` and `'b'` and returns `'c'`. From the perspective of client code, `getA` has no dependencies, `getB` depends on `getA` and `getC` depends on both `getA` and `getB`. If the arguments are not as expected a rejected promise is returned.

{% highlight js %}
/*jslint node:true*/
'use strict';

/* global -Promise */
var Promise = require('promise');

module.exports.getA = function() {
  return Promise.resolve('a');
};

module.exports.getB = function(a) {
  if (a === 'a') return Promise.resolve('b');
  else return Promise.reject(new Error('a required'));
};

module.exports.getC = function(a, b) {
  if (a === 'a' && b === 'b') return Promise.resolve('c');
  else return Promise.reject(new Error('a and b  required'));
};
{% endhighlight %}

Composing `getA` and `getB` is easy assuming all you care about is `'b'`. This is especially true if you use function pointers (you do right?). The code below shows how. And of course `err` is handled just once.

{% highlight js %}
/*jslint node:true*/
'use strict';

var should = require('should');
var functions = require('../lib/functions');

/* global describe, it */
describe('promise', function () {
  it('should compose', function(done) {
    functions.getA().
      then(functions.getB).
      then(function(b) {
        b.should.be.exactly('b');
        done();
      }, function (err) {
        done(err);
      });
  });
});
{% endhighlight %}

Composing `getA`, `getB` and `getC` is a bit harder even assuming all you care about is `'c'`. If it wasn't for the depencencies between the functions then we could just use `Promise#all` and be done with it. But given the dependencies `Promise#all` won't cut it. The code below shows how to do it.

{% highlight js %}
/*jslint node:true*/
'use strict';

var should = require('should');
var functions = require('../lib/functions');

/* global describe, it */

describe('promise', function () {
  it('should compose', function(done) {
    functions.getA().then(function (a) {
      return functions.getB(a).then(function (b) {
        return functions.getC(a, b);
      });
    }).then(function (c) {
      c.should.be.exactly('c');
      done();
    }, function (err) {
      done(err);
    });
  });
});
{% endhighlight %}

Notice, however, that some nesting has crept in and there is a distinct lack of function pointers. What we really want is some function that can wrap `getB` and `getC` whilst carrying over the results of each invocation to the next function. The usage of such a `carry` function is shown below.

{% highlight js %}
/*jslint node:true*/
'use strict';

var should = require('should');
var carry = require('../lib/carry');
var functions = require('../lib/functions');

/* global describe, it */
describe('carry', function () {
  it('should carry results over', function(done) {
    functions.getA().
      then(carry(functions.getB)).
      then(carry(functions.getC)).
      then(carry(function(a, b, c) {
        a.should.be.exactly('a');
        b.should.be.exactly('b');
        c.should.be.exactly('c');
        done();
      }), function (err) {
        done(err);
      });
  });
});
{% endhighlight %}

The `carry` function reduces nesting, allows us to use function pointers and gives us to access `'a'`, `'b'` and `'c'`. The code below shows how to define it.

{% highlight js %}
/*jslint node:true*/
'use strict';

module.exports = function(f) {
  return function(data) {
    if (data && data.constructor === Array) {
      return f.apply(this, data).then(function (result) {
        if (result && result.constructor === Array) {
          return data.concat(result);
        } else {
          return data.concat([result]);
        }
      });
    } else {
      return f(data).then(function (result) {
        return [data, result];
      });
    }
  };
};
{% endhighlight %}

It's worth noting that this isn't the only way to solve this particular problem. In fact, I spent **a lot** of time thrashing about before I settled on this as my favourite pattern.
