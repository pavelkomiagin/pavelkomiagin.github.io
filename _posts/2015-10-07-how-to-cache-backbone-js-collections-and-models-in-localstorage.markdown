---
title: "How to cache Backbone.js collections and models in LocalStorage"
layout: post
date: 2015-10-07 14:53
image: /assets/images/markdown.jpg
headerImage: false
tag:
- backbonejs
- javascript
blog: true
author: pavelkomiagin
description: How to cache Backbone.js collections and models in LocalStorage
---

Working on project [Foxford](http://foxford.ru) as a front-end developer I wanted to reduce 
the number of requests to server. For example we have filters containing 
a list of disciplines, a list of learning objectives, and a list of classes. 
They are used on several pages and are changed rarely so I wanted to cache 
the received data. 
Since we use Backbone, I decided to extend the models and the collections 
by adding the method **cachedFetch** like the native **fetch**.

### Step 1

At first I wrote a simple wrapper for caching of JS-objects in the LocalStorage 
with the ability to specify the duration of the cache:

{% highlight js %}

var StoreWithExpiration = function() {
  return {
    set: function(key, value, expireTime) {
      var item = {
        value: value,
        expireTime: expireTime,
        cachedAt: new Date().getTime()
      };
      try {
        localStorage.setItem(key, JSON.stringify(item));
      } catch(e) {}
    },

    get: function(key) {
      try {
        var info = JSON.parse(localStorage.getItem(key));
        if (!info || new Date().getTime() - info.cachedAt > info.expireTime)
          return null;
        return info.value;
      } catch(e) {
        localStorage.removeItem(key);
      }
    }
  };
};

{% endhighlight %}

### Step 2

Then I extended **Backbone.Collection** by the method **cachedFetch**. 
Lifetime of the cache can be set by parameter **expireTime** (default lifetime equals 1 hour). 
The logic is simple:

1. If LocalStorage is unavailable then call normal **fetch()**
2. If there is a valid cache then we use it, otherwise we receive and cache new data.

{% highlight js %}

var expirationStore = new StoreWithExpiration();
Backbone.Collection = Backbone.Collection.extend({

  cachedFetch: function(options) {
    options = options || {};
    var expireTime = options.expireTime || 60 * 60 * 1000;
    var cacheKey = 'bbCollection_' + this.url;

    if (!window.localStorage)
      return this.fetch(options);

    var cachedData = expirationStore.get(cacheKey);
    var cachedModels = cachedData ? cachedData.value : [];
    var success = options.success;

    if (!cachedData) {
      options.success = _.bind(function (resp) {
        if (success)
          success.call(options.context, this, resp, options);

        expirationStore.set(cacheKey, this.models, expireTime);

      }, this);
      return this.fetch(options);

    } else {

      options = _.extend({parse: true}, options);
      var method = options.reset ? 'reset' : 'set';
      this[method](cachedModels, options);

      if (success)
        success.call(options.context, this, cachedModels, options);

      this.trigger('sync', this, cachedModels, options);
      return this.sync('cachedRead', this, options);
    }
  }
});

{% endhighlight %}

And for the models:

{% highlight js %}

Backbone.Model = Backbone.Model.extend({
  cachedFetch: function(options) {
    options = options || {};
    var expireTime = options.expireTime || 60 * 60 * 1000;
    var cacheKey = 'bbModel_' + this.url + '_' + this.id;

    if (!window.localStorage)
      return this.fetch(options);

    var cachedData = expirationStore.get(cacheKey);
    var cachedModel = cachedData ? cachedData.value : {};
    var success = options.success;

    if (!cachedData) {
      options.success = _.bind(function (resp) {
        if (success)
          success.call(options.context, this, resp, options);

        expirationStore.set(cacheKey, this.models, expireTime);

      }, this);
      return this.fetch(options);

    } else {

      options = _.extend({parse: true}, options);
      var serverAttrs = options.parse ? model.parse(cachedModel, options) : cachedModel;

      if (!model.set(serverAttrs, options))
        return false;

      if (success)
        success.call(options.context, this, cachedModel, options);

      this.trigger('sync', this, cachedModel, options);
      return this.sync('cachedRead', this, options);
    }
  }
});

{% endhighlight %}

### Step 3

And finally I patched **Backbone.Sync** to support our caching:

{% highlight js %}

var _sync = Backbone.sync;
Backbone.sync = function(method, model, options) {
  if (method === 'cachedRead') {

    var _getCacheKey = function() {
      return model.collection ? 'bbModel_' + model.url + '_' + model.id : 'bbCollection_' + model.url;
    };

    var _getCachedValue = function() {
      var cachedData = expirationStore.get(_getCacheKey());
      var empty = model.collection ? {} : [];
      return cachedData ? cachedData.value : empty;
    };

    if (options.success)
      options.success(_getCachedValue());

  } else {
    return _sync.apply(this, arguments);
  }
}

{% endhighlight %}

The full source code for this plugin you can find on [github](https://github.com/pavelkomiagin/Backbone.cachedFetch).
