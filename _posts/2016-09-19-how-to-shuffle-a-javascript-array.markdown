---
title: "How to shuffle a JavaScript array"
layout: post
date: 2016-09-19 16:09
image: /assets/images/markdown.jpg
headerImage: false
tag:
- javascript
blog: true
author: pavelkomiagin
description: How to shuffle a JavaScript array
---

Every programmer can face the challenge of array shuffling. Of course sometimes 
it is possible to use a library (e.g. [underscore](http://underscorejs.org/#shuffle)).
But it is useful to have the snippets for this task.

#### Shuffle that modify a given array:

Array isn't a primitive so Arrays are passed by reference. It means that 
if we pass an array as parameter to function and modify this array inside 
this function then this array will be changed.

{% highlight js %}
// Initial array will be changed after shuffle
function mutableShuffle(arr) {
  var j;
  var temp
  
  for (var i = 0, max = arr.length; i < max; i++) {
    j = Math.floor(Math.random() * max);
    temp = arr[i];
    arr[i] = arr[j];
    arr[j] = temp;
  }
  
  return arr;
}
{% endhighlight %}

#### Shuffle that don't modify a given array:

{% highlight js %}
// Initial array will NOT be changed after shuffle
function immutableShuffle(arr) {
  var result = [];
  var j;
  var temp;
  
  for (var i = 0, max = arr.length; i < max; i++) {
    j = Math.floor(Math.random() * max);
    temp = arr[i];
    result[i] = arr[j];
    result[j] = temp;
  }
  
  return result;
}
{% endhighlight %}

#### How to use:

{% highlight js %}
var mutableArr = [1, 2, 3];
console.log(mutableShuffle(mutableArr));
console.log(mutableArr); // will be changed

var immutableArr = [1, 2, 3];
console.log(immutableShuffle(immutableArr));
console.log(immutableArr); // will NOT be changed
{% endhighlight %}

You can try and modify it on [codepen](http://codepen.io/pavel_komiagin/pen/PGGEmK?editors=0011).
