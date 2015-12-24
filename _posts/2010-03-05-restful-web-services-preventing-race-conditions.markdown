---
layout:     post
title:      "RESTful Web Services: Preventing Race Conditions"
permalink:  "restful-web-services-preventing-race-conditions"
date:       2010-03-05
comments:   true
tags:
- restful
- http
---
One of the core premisses of RESTful web services is that HTTP should be seen as an application protocol rather than just a transport protocol. It comprises a whole bunch of semantics that allows us to build robust distributed systems. And for some cases, when multiple consumers manipulate the same resource, therefore changing its state, the solution should be robust enough to prevent the system to get into a race condition.

### But how HTTP could prevent that?

HTTP provides a simple but powerful mechanism for aligning resource states by making use of [entity tag or ETag](http://en.wikipedia.org/wiki/HTTP_ETag) and [conditional request headers](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html). An ```ETag``` is anything that uniquely identifies an entity, such as the ID associated with a persisted resource, a checksum of the entity headers and body, etc. If this resource changes—that is, when one or more of its headers, or its entity body, changes—then the entity tag changes accordingly, reflecting this new resource state.

When a response contains an ```ETag``` associated to a resource state and you want to continue working with this same resource, it's recommended to use this tag in subsequent requests (called conditional requests), otherwise the resource state might eventually become out of sync with service one, returning something like a ```409 Conflict```.

Conditional requests happens when the current ```ETag``` is supplied to a conditional request header, such as ```If-Match``` or ```If-None-Match```, when user is requesting to update a resource for example. The service will then check the precondition, by comparing the current resource ```ETag``` with the one provided in the request. If it's satisfied than the server proceeds and process the request, otherwise it concludes that the resource has changed and responds with a ```412 Precondition Failed```.

#### Example

Given an online shop for home goods, where two people— **admin1** and **admin2** —are responsible for administrating its contents. In our scenario both administrators are trying to change the state of the same product (the Weber BBQ), around the same time. **admin1** wants to lower the product price down to $300.00 and **admin2** wants to change its state to "Not Available". Firstly, both administrators ```GET``` the current product state independently of one another by doing the following request:

```javascript
GET /product/1 HTTP/1.1
Host: myshop.com
```

Returning the following resource (product) as response. Note that the service's response contains an ```ETag``` header.

```javascript
HTTP/1.1 200 OK
Content-Length: 265
Content-Type: application/json
ETag: "686897696a7c876b7e"

{
  "name": "WeberFamilyBBQ",
  "description": "Great for parties and cooks a neat roast too.",
  "price": 399,
  "status": "InStock"
}
```

When **admin1** does a conditional PUT, including an ```If-Match``` header with the ```ETag``` value from the previous ```GET```.

```javascript
PUT /product/1 HTTP/1.1
Host: myshop.com
If-Match: "686897696a7c876b7e"

{
  "name": "WeberFamilyBBQ",
  "description": "Great for parties and cooks a neat roast too.",
  "price": 399,
  "status": "InStock"
}
```

And as the product state hasn't changed since the last request, then the request is thus successful! Notice that the response returns an updated ```ETag``` value, reflecting the new product state.

```javascript
HTTP/1.1 204 No Content
ETag: "616898r96a8cy86b8eee11"
```

Little time after **admin1** has updated the product, **admin2** does another ```PUT``` request to the same product, including the same ```If-Match``` header with the ```ETag``` value from the ```GET``` request.

```javascript
PUT /product/1 HTTP/1.1
Host: myshop.com
If-Match: "686897696a7c876b7e"

{
  "name": "WeberFamilyBBQ",
  "description": "Great for parties and cooks a neat roast too.",
  "price": 399,
  "status": "InStock"
}
```

The service then determines that someone is trying to change the same product, using an out-of-date resource representation (```ETags``` are different!), and responds with a ```412 Precondition Failed``` code. No race conditions whatsoever!

```javascript
HTTP/1.1 412 Precondition Failed
```

### Conclusion

Although ETags and conditional request headers make up a powerful mechanism for dealing with concurrency, one thing to keep in mind is that, depending on the amount of computation performed by the server to generate an ```ETag```, response times might increase considerably. So use it only if you need it!

Special thanks to [Jim Webber](http://jim.webber.name) for helping me with this post. For more information on RESTful Web Services, check out his latest book (written together with [Savas Parastatidis](http://savas.me) and [Ian Robinson](http://iansrobinson.com))— [REST in Practice: Hypermedia and Systems Architecture](http://www.amazon.com/gp/product/0596805829).
