+++
date = '2025-01-23T17:44:47-08:00'
draft = true
title = "Pushing performance with HTTP's Cache-Control header"
+++

<!-- TODO: potentially rework intro -->
Have you ever wondered why some applications feel lightning-fast, while others lag behind? The secret often lies in how effectively they use caching. In this post I'll discuss how HTTP's Cache-Control header and other related mechanisms can be used to take advantage of HTTP's native caching features. This will allow us to reduce server load and response times as an application begins to scale.

The [REST architectural style](), as defined by Roy Fielding, makes caching one of its key constraints. REST API's are stateless, meaning each request is independent of every other request. This enables us to effectively cache the return data for GET requests on the client, granted the cache can be updated at appropreate times. But how does the server decide what gets cached and for how long? That's where response headers, like Cache-Control, come into play.

---

## The Cache-Control Header

We'll start by walking through an example request flow with the Cache-Control header and then we'll build from there.

### Serving from cache

The flow starts with the client making a `GET` request for data. The server responds with a `200 OK` status, the JSON data in the body of the request, and a `Cache-Control` header indicating the age of the cache, in this case 3600 milliseconds (1 hour).

```bash
GET /api/data
```
```bash
200 OK
Cache-Control: max-age=3600
Content-Type: aplication/json
{ "data": "content" }
```
The client stores in its local cache the data for the endpoint and the age of the cache.

When the client makes a GET request for the same endpoint and the cache has not yet expired, the response data is served locally from its cache. We avoid making a roundtrip request to the server, significantly reducing latency and bandwidth on the application. This storing and serving from cache is part of the HTTP protocol, and is enacted by the browser when it receives the Cache-Control header.

### Cache expiration

After a full hour passes the cache becomes stale. If the client makes another GET request to the same endpoint the browser will force a request to the server. The return data and the cache age is again used to populate the cache.

<!-- TODO - insert image -->
![Request flow for the Cache-Control header]()

## Further optimizations with the ETag and Last-Modified headers
keyword: Conditional request

Walk through client/server request/response flow with the Cache-Control header and with both ETag and Last-Modified.

## Library and framework implementations

Explain how Express sets ETag headers automatically.

different hashing algorithms

## Use case considerations

Now that we have a general understanding of how some of these mechanisms work, we'll consider the trade-offs involved and when each one might be a better solution.

ETag is better for large payloads like static assets and large JSON objects for 100KB or more.

Large payloads can consume a lot of bandwidth and take time to download. The overhead of calculating a hash value for the ETag would be small compared to the savings.

Small payloads will have minimal bandwidth and processing costs. The overhead from calculating a hash can outweigh the benefits of cashing for small payloads.

Last-Modified, based on timestamps is better for smaller payloads originating from a database. Database's make it easier to store the last modified timestamp of a resource.

Show an example in code.

## Resources

https://stackoverflow.com/questions/24542959/how-does-a-etag-work-in-expressjs