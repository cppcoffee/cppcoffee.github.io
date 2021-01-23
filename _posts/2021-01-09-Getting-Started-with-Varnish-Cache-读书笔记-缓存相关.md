---
layout: post
title:  "《Getting Started with Varnish Cache》读书笔记-缓存相关"
categories: httpcache
---
{{ page.title }}

## Expiration

HTTP has a set of mechanisms in place to decide when a cached object should be removed from cache. Objects cannot live in cache forever: you might run out of cache storage (memory or disk space) and Varnish will have to evict items using an LRU strategy to clear space. Or you might run into a situation where the data you are serving is stale and the object needs to be synchronized with a new response from the backend.

Expiration is all about setting a time-to-live. HTTP has two different kinds of response headers that it uses to indicate that:

### Expires

An absolute timestamp that represents the expiration time.

### Cache-control

The amount of seconds an item can live in cache before becoming stale.

> WARNING
>
> Varnish gives you a heads-up regarding the age of a cached object. The Age header is returned upon every response. The value of this Age header corresponds to the amount of time the object has been in cache. The actual time-to-live is the cache lifetime minus the age value. For that reason, I advise you not to set an Age header yourself, as it will mess with the TTL of your objects.


### The Expires Header

The Expires header is a pretty straight forward one: you just set the date and time when an object should be considered stale. This is a response header that is sent by the backend.

Here’s an example of such a header:

`Expires: Sat, 09 Sep 2017 14:30:00 GMT`

> Important
>
> Do not overlook the fact that the time of an Expires header is based on Greenwich Mean Time. If you are located in another time zone, please express the time accordingly.


### The Cache-Control Header

The Cache-control header defines the time-to-live in a relative way: instead of stating the time of expiration, Cache-control states the amount of seconds until the object expires. In a lot of cases, this is a more intuitive approach: you can say that an object should only be cached for an hour by assigning 3,600 seconds as the time-to-live.

This HTTP header has more features than the Expires header: you can set the time to live for both clients and proxies. This allows you to define distinct behavior depending on the kind of system that processes the header; you can also decide whether to cache and whether to revalidate with the backend.

`Cache-control: public, max-age=3600, s-maxage=86400`

The preceding example uses three important keywords to define the time-to-live and the ability to cache:


#### public

Indicates that both browsers and shared caches are allowed to cache the content.


#### max-age

The time-to-live in seconds that must be respected by the browser.


#### s-maxage

The time-to-live in seconds that must be respected by the proxy.

It’s also important to know that Varnish only respects a subset of the Cache-control syntax. It will only respect the keywords that are relevant to its role as a reverse caching proxy:

- Cache-control headers sent by the browser are ignored
- The time-to-live from an s-maxage statement is prioritized over a max-age statement
- Must-revalidate and proxy-revalidate statements are ignored
- When a Cache-control response header contains the terms private, no-cache, or no-store, the response is not cached


> NOTE
>
> Although Varnish respects the public and private keywords, it doesn’t consider itself a shared cache and exempts itself from some of these rules. Varnish is more like a surrogate web server because it is under full control of the web server and does the webmaster’s bidding.


## Expiration Precedence

Varnish respects both Expires and Cache-control headers. In the Varnish Configuration Language, you can also decide what the time-to-live should be regardless of caching headers. And if there’s no time-to-live at all, Varnish will fall back to its hardcoded default of 120 seconds.

Here’s the list of priorities that Varnish applies when choosing a time-to-live:

1. If beresp.ttl is set in the VCL, use that value as the time-to-live.
2. Look for an s-maxage statement in the Cache-control header.
3. Look for a max-age statement in the Cache-control header.
4. Look for an expires header.
5. Cache for 120 seconds under all other circumstances.

> Important
>
> As you can see, the TTL in the VCL gets the absolute priority. Keep that in mind, because this will cause any other Expires or Cache-control header to be ignored in favor of the beresp.ttl value.


## Conditional Requests

*Expiration* is a valuable mechanism for updating the cache. It’s based on the concept of checking the *freshness* of an object at set intervals. These intervals are defined by the time-to-live and are processed by Varnish. The end user doesn’t really have a say in this.

After the expiration, both the headers and the payload are transmitted and stored in cache. This could be a very resource-intensive matter and a waste of bandwidth, especially if the requested data has not changed in that period of time.

Luckily, HTTP offers a way to solve this issue. Besides relying on a time-to-live, HTTP allows you to keep track of the validity of a resource. There are two separate mechanisms for that:

- The Etag response header
- The Last-Modified response header

> NOTE
>
> Most web browsers support conditional requests based on the Etags and Last-Modified headers, but Varnish supports this as well when it communicates with the backend.

### ETag

An Etag is an HTTP response header that is either set by the web server or your application. It contains a unique value that corresponds to the state of the resource.

A common strategy is to create a unique hash for that resource. That hash could be an md5 or a sha hash based on the URL and the internal modification date of the resource. It could be anything as long as it’s unique.

```
HTTP/1.1 200 OK
Host: localhost
Etag: 7c9d70604c6061da9bb9377d3f00eb27
Content-type: text/html; charset=UTF-8

Hello world output
```

As soon as a browser sees this Etag, it stores the value. Upon the next request, the value of the Etag will be sent back to the server in an If-None-Match request header.

```
GET /if_none_match.php HTTP/1.1
Host: localhost
User-Agent: curl/7.48.0
If-None-Match: 7c9d70604c6061da9bb9377d3f00eb27
```

The server receives this If-None-Match header and checks if the value differs from the Etag it’s about to send.

If the Etag value is equal to the If-None-Match value, the web server or your application can return an HTTP/1.1 304 Not Modified response header to indicate that the value hasn’t changed.

```
HTTP/1.0 304 Not Modified
Host: localhost
Etag: 7c9d70604c6061da9bb9377d3f00eb27
```

When you send a 304 status code, you don’t send any payload, which can dramatically reduce the amount of bytes sent over the wire. The browser receives the 304 and knows that it can still output the old data.

If the If-None-Match value doesn’t match the Etag, the web server or your application will return the full payload, accompanied by the HTTP/1.1 200 OK response header and, of course, the new Etag.

This is an excellent way to conserve resources. Whereas the primary goal is to reduce bandwidth, it will also help you to reduce the consumption of memory, CPU cycles, and disk I/O if you implement it the right way.


### Last-Modified

ETags aren’t the only way to do conditional requests; there’s also an alternative technique based on the Last-Modified response header. The client will then use the If-Modified-Since request header to validate the freshness of the resource.

The approach is similar:

1. Let your web server or application return a Last-Modified response header
2. The client stores this value and uses it as an If-Modified-Since request header upon the next request
3. The web server or application matches this If-Modified-Since value to the modification date of the resource
4. Either an HTTP/1.1 304 Not Modified or a HTTP/1.1 200 OK is returned

The benefits are the same: reduce the bytes over the wire and load on the server by avoiding the full rendering of output.

> CAUTION
>
> The timestamps are based on the GMT time zone. Please make sure you convert your timestamps to this time zone to avoid weird behavior.

The starting point in the following example is the web server (or the application) returning a Last-Modified response header:

```
HTTP/1.1 200 OK
Host: localhost
Last-Modified: Fri, 22 Jul 2016 10:11:16 GMT
Content-type: text/html; charset=UTF-8

Hello world output
```

The browser stores the Last-Modified value and uses it as an If-Last-Modified in the next request:

```
GET /if_last_modified.php HTTP/1.1
Host: localhost
User-Agent: curl/7.48.0
If-Last-Modified: Fri, 22 Jul 2016 10:11:16 GMT
```

The resource wasn’t modified, a 304 is returned, and the Last-Modified value remains the same:

```
HTTP/1.0 304 Not Modified
Host: localhost
Last-Modified: Fri, 22 Jul 2016 10:11:16 GMT
```

The browser does yet another conditional request:

```
GET /if_last_modified.php HTTP/1.1
Host: localhost
User-Agent: curl/7.48.0
If-Last-Modified: Fri, 22 Jul 2016 10:11:16 GMT
```

The resource was modified in the meantime and a full 200 is returned, including the payload and a new Last-Modified_header.

```
HTTP/1.1 200 OK
Host: localhost
Last-Modified: Fri, 22 Jul 2016 11:00:23 GMT
Content-type: text/html; charset=UTF-8

Some other hello world output
```


## How Varnish Deals with Conditional Requests
When Varnish spots an If-Modified-Since or If-None-Match header in the request, it keeps track of the Last-Modified timestamp and/or the Etag. Regardless of whether or not Varnish has the object in cache, a 304 status code will be returned if the Last-Modified or the Etag header matches.

From a client point of view, Varnish reduces the amount of bytes over the wire by returning the 304.

On the other hand, Varnish also supports conditional requests when it comes to backend communication: when an object is considered stale, Varnish will send If-Modified-Since and If-None-Match headers to the backend if the previous response from the backend contained either a Last-Modified timestamp or an Etag.

When the backend returns a 304 status code, Varnish will not receive the body of that response and will assume the content hasn’t changed. As a consequence, the stale data will have been revalidated and will no longer be stale. The Age response header will be reset to zero and the object will live in cache in accordance to the time-to-live that was set by the web server or the application.


## Cache Variations

In general, an HTTP resource is public and has the same value for every consumer of the resource. If data is user-specific, it will, in theory, not be cacheable. However, there are exceptions to this rule and HTTP has a mechanism for this.

HTTP uses the Vary header to perform cache variations. The Vary header is a response header that is sent by the backend. The value of this header contains the name of a request header that should be used to vary on.

> Important
>
> The value of the Vary header can only contain a valid request header that was set by the client. You can use the value of custom X- HTTP headers as a cache variation, but then you need to make sure that they are set by the client.

A very common example is language detection based on the Accept-Language request header. Your browser will send this header upon every request. It contains a set of languages or locales that your browser supports. Your application can then use the value of this header to determine the language of the output. If the desired language is not exposed in the URL or through a cookie, the only way to know is by using the Accept-Language header.

If no vary header is set, the cache (either the browser cache or any intermediary cache) has no way to identify the difference and stores the object based on the first request. If that first request was made in Dutch, all other users will get output in Dutch — regardless of the browser language — for the duration of the cache lifetime.

That is a genuine problem, so in this case, the application returns a Vary header containing Accept-Language as its value. Here’s an example:

The browser language is set to Dutch:

```
GET / HTTP/1.1
Host: localhost
Accept-Language: nl
```

The application sets a Vary header that instructs the cache to keep a separate version of the cached object based on the Accept-Language value of the request.

```
HTTP/1.1 200 OK
Host: localhost
Vary: Accept-Language

Hallo, deze pagina is in het Nederlands geschreven.
```

The cache knows there is a Dutch version of this resource and will store it separately, but it will still link it to the cached object of the main resource. When the next request is sent from a browser that only supports English, the cached object containing Dutch output will not be served. A new backend request will be made and the output will be stored separately.

> WARNING
>
> Be careful when you perform cache variations based on request headers that can contain many different values. The User-Agent and the Cookie headers are perfect examples.
>
> In many cases, you don’t have full control over the cookie value. Tracking cookies set by third-party services can add unique values per user to the cookie. This could result in too many variations, and the hit rate would plummet.
>
> The same applies to the User-Agent: almost every device has its own User-Agent. When using this as a cache variation, the hit rate could drop quite rapidly.

Varnish respects the Vary header and adds variations to the cache on top of the standard identifiers. The typical identifiers for a cached object are the hostname (or the IP if no hostname was set) and the URL.

When Varnish notices a cache variation, it will create a cache object for that version. Cache variations can expire separately, but when the main object is invalidated, the variations are gone, too.

> Important
>
> You have to find a balance between offering enough cache variations and a good hit rate. Choose the right request header to vary on and look for balance.


## When Is a Request Considered Cacheable?

When Varnish receives a request, it has to decide whether or not the response can be cached or even served from cache. The rules are simple and based on idempotence and state.

A request is cacheable when:

- The request method is GET or HEAD
- There are no cookies being sent by the client
- There is no authorization header being sent

When these criteria are met, Varnish will look the resource up in cache and will decide if a backend request is needed, or if the response can be served from cache.


### How Does Varnish Identify an Object?

Once we decide that an object is cacheable, we need a way to identify the object in order to retrieve it from cache. A hash key is composed of several values that serve as a unique identifier.

1. If the request contains a Host header, the hostname will be added to the hash.
2. Otherwise, the IP address will be added to the hash.
3. The URL of the request is added to the hash.

Based on that hash, Varnish will retrieve the object from cache.


### When Does Varnish Cache an Object?

If an object is not stored in cache or when it’s considered stale, a backend connection is made. Based on the backend response, Varnish will decide if the returned object will be stored in cache or if the cache is going to be bypassed.

A response will be stored in cache when:

- The time-to-live is more than zero.
- The response doesn’t contain a Set-Cookie header.
- The Cache-control header doesn’t contain the terms no-cache, no-store, or private.
- The Vary header doesn’t contain *, meaning vary on all headers.


### What Happens if an Object Is Not Stored in Cache?

If after the backend response Varnish decides that an object will not be stored in cache, it puts the object on a “blacklist” — the so-called hit-for-pass cache.

For a duration of 120 seconds, the next requests will immediatly connect with the backend, directly serving the response, without attempting to store the response in cache.

After 120 seconds, upon the next request, the response can be re-evaluated and a decision can be made whether or not to store the object in cache.


### How Long Does Varnish Cache an Object?

Once an object is stored in cache, a decision must be made on the time-to-live. I mentioned this before, but there’s a list of priorities that Varnish uses to decide which value it will use as the TTL.

Here’s the prioritized list:

- If beresp.ttl is set in the VCL, use that value as the time-to-live.
- Look for an s-maxage statement in the Cache-control header.
- Look for a max-age statement in the Cache-control header.
- Look for an Expires header.
- Cache for 120 seconds under all other circumstances.

When the object is stored in the hit-for-pass cache, it is cached for 120 seconds, unless you change the value in VCL.


## Varnish’s Built-In VCL

As a quick reminder, this is what the preceding code does:

- It does not support the PRI method and throws an HTTP 405 error when it is used.
- Request methods that differ from GET, HEAD, PUT, POST, TRACE, OPTIONS, and DELETE are not considered valid and are piped directly to the backend.
- Only GET and HEAD requests can be cached, other requests are passed to the backend and will not be served from cache.
- When a request contains a cookie or an authorization header, the request is passed to the backend and the response is not cached.
- If at this point the request is not passed to the backend, it is considered cacheable and a cache lookup key is composed.
- A cache lookup key is a hash that is composed using the URL and the hostname or IP address of the request.
- Objects that aren’t stale are served from cache.
- Stale objects that still have some grace time are also served from cache.
- All other objects trigger a miss and are looked up in cache.
- Backend responses that do not have a positive TTL are deemed uncacheable and are stored in the hit-for-pass cache.
- Backend responses that send a Set-Cookie header are also considered uncacheable and are stored in the hit-for-pass cache.
- Backend responses with a no-store in the Surrogate-Control header will not be stored in cache either.
- Backend responses containing no-cache, no-store, or private in the Cache-control header will not be stored in cache.
- Backend responses that have a Vary header that creates cache variations on every request header are not considered cacheable.
- When objects are stored in the hit-for-pass cache, they remain in that blacklist for 120 seconds.
- All other responses are delivered and stored in cache.



