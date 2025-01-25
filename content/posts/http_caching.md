+++
date = '2024-10-23T17:44:47-08:00'
draft = false
title = "Pushing performance with HTTP's Cache-Control header"
+++

In this post I want to explore how HTTP's native caching features can be used to reduce server load and response times for users as an application begins to scale.

The [REST architectural style](), as defined by Roy Fielding, makes caching one of its key constraints. With REST APIs being stateless each request is independent of every other request, enabling us to effectively cache the return data for GET requests on the client, provided that the cache can be updated at appropriate times. But how does the server decide what gets cached and for how long? That's where response headers like `Cache-Control` come into play.

---

## The Cache-Control Header

We'll start by walking through an example request flow with the `Cache-Control` header and then build from there.

### Serving from cache

The flow starts with the client making a `GET` request for data. The server responds with a `200 OK` status, the JSON data in the body of the request, and a `Cache-Control` header indicating the age of the cache, in this case 3600 milliseconds (1 hour).

```bash
GET /api/data
```
```bash
200 OK
Cache-Control: max-age=3600
Content-Type: application/json
{ "data": "content" }
```
The client stores in its local cache the data for the endpoint and the age of the cache.

When the client makes a GET request for the same endpoint and the cache has not yet expired, the response data is served locally from its cache. We avoid making a roundtrip request to the server, significantly reducing latency and bandwidth on the application. This storing and serving from cache is part of the HTTP protocol, and is enacted by the browser when it receives the `Cache-Control` header.

### Cache expiration

After a full hour passes the cache becomes stale. If the client makes another GET request to the same endpoint the browser forces a request to the server. The return data and the cache age is again used to re-populate the cache.

![Request flow for the Cache-Control header](/images/http_caching/cache_control_request_flow.png)

---

## Optimizations with Last-Modified

This is great. We've significantly reduced load on the server. But what if we need to optimize further? Well, with the previous implementation once the cache expires the client will always fetch the data from the server when it needs it again, even if that data hasn't changed. Could we check if the data has changed and only send it back if it has? The `Last-Modified` and `ETag` headers allow us to do just that.

We'll start with Last-Modified. When the client initially requests the data the server sends back a `Last-Modified` header with the datetime of when the resource last changed. The client holds this in cache for the endpoint.

Now once the cache expires and the client needs the data again it will make a conditional request sending the `If-Modified-Since` header holding the value of `Last-Modified` stored for that endpoint.

The server will refetch the resource and compare its `updated_at` field with the value of the `If-Modified-Since` header. If the resource hasn't been updated since the last time it was cached, the server sends back a `304 Not Modified`. The client resets the cache age for that endpoint and continue pulling from local cache.

Only if the resource has been modified the server sends the updated data in the JSON of the response and the updated datetime in the `Last-Modified` header. See the full flow mapped out.

![Request flow for the If-Modified-Since header 1](/images/http_caching/if_modified_since_flow_1.png)
![Request flow for the If-Modified-Since header 2](/images/http_caching/if_modified_since_flow_2.png)

<!-- ## The ETag header



## Library and framework implementations

Explain how Express sets ETag headers automatically.

different hashing algorithms

## Use case considerations

Now that we have a general understanding of how some of these mechanisms work, we'll consider the trade-offs involved and when each one might be a better solution.

ETag is better for large payloads like static assets and large JSON objects for 100KB or more.

Large payloads can consume a lot of bandwidth and take time to download. The overhead of calculating a hash value for the ETag would be small compared to the savings.

Small payloads will have minimal bandwidth and processing costs. The overhead from calculating a hash can outweigh the benefits of cashing for small payloads.

Last-Modified, based on timestamps is better for smaller payloads originating from a database. Database's make it easier to store the last modified timestamp of a resource.

Show an example in code. -->

---

## Further resources

https://stackoverflow.com/questions/24542959/how-does-a-etag-work-in-expressjs