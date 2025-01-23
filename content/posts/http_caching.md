+++
date = '2025-01-22T15:17:33-08:00'
draft = true
title = 'HTTP caching with the Cache-Control header'
+++

Recently I've wanted to dive deeper into the various ways caching can be used to improve the performance of an application. With this post I wanted to share how the Cache-Control header and other related mechanisms can be used to take advantage of HTTP's native caching features to reduce server load and response times as an application begins to scale.

## The Cache-Control Header

Walk through client/server request/response flow with the Cache-Control header.

Show diagram

## Further optimizations with the ETag and Last-Modified headers

Walk through client/server request/response flow with the Cache-Control header and with both ETag and Last-Modified.

## Library and framework implementations

Explain how Express sets ETag headers automatically.

## Use case considerations

Now that we have a general understanding of how some of these mechanisms work, we'll consider the trade-offs involved and when each one might be a better solution.

ETag is better for large payloads like static assets and large JSON objects for 100KB or more.

Large payloads can consume a lot of bandwidth and take time to download. The overhead of calculating a hash value for the ETag would be small compared to the savings.

Small payloads will have minimal bandwidth and processing costs. The overhead from calculating a hash can outweigh the benefits of cashing for small payloads.

Last-Modified, based on timestamps is better for smaller payloads originating from a database. Database's make it easier to store the last modified timestamp of a resource.

Show an example in code.

## Resources

https://stackoverflow.com/questions/24542959/how-does-a-etag-work-in-expressjs