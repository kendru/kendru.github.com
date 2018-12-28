---
layout: post
title: "Rate Limiting in Javascript with a Token Bucket"
description: "A simple algorithm for rate limiting requests of any sort"
category: Javascript
tags: ["javascript", "algorithm"]
---

Recently I was looking into options to add rate limiting to a specific endpoint
in an application at work. Most endpoints are only exposed internally, and we
are careful to not make more requests than the system can handle. However, in one
case, the endpoint is open to our customers, and it runs some pretty intensive
database operations, so we wanted to limit the rate at which clients can make
requests. This functionality is available in pretty much every API gateway out
there as well as in a lot of [reverse proxies](https://www.nginx.com/blog/rate-limiting-nginx/).
In our case, application updates are easier to make than config updates, so we
opted for a simple solution that we could deploy as part of our Node.js app.

Enter the *Token Bucket*.

A token bucket is an algorithm that allows _tokens_ to be accumulated over time
at a specific rate. These tokens can then be "redeemed" to execute some action.
If there are no tokens available, the action cannot be taken. Imagine that we
have a bucket that holds some number of balls, say 100. When there are fewer than
100 balls in the bucket, a machine will automatically refill the bucket at a rate
of 1 ball per second until it is full again. We can take as many balls as we want
as quickly as we want, but once the bucket is empty, we have to wait for it to start
filling up again before we can take more.

If we use a token bucket to rate limit an API, then it allows us to set a request
rate (the rate at which tokens are added to the bucket) with the ability to *burst*
above this rate for a short period (until we have drained the capacity of the bucket).
Let's take a first pass at implementing a token bucket.

<script async src="//pagead2.googlesyndication.com/pagead/js/adsbygoogle.js"></script>
<ins class="adsbygoogle"
     style="display:block; text-align:center;"
     data-ad-layout="in-article"
     data-ad-format="fluid"
     data-ad-client="ca-pub-6265787006533161"
     data-ad-slot="3706397953"></ins>
<script>
(adsbygoogle = window.adsbygoogle || []).push({});
</script>

#### Initial TokenBucket Implementation

```javascript
class TokenBucket {

    constructor(capacity, fillPerSecond) {
        this.capacity = capacity;
        this.tokens = capacity;
        setInterval(() => this.addToken(), 1000 / fillPerSecond);
    }

    addToken() {
        if (this.tokens < this.capacity) {
            this.tokens += 1;
        }
    }

    take() {
        if (this.tokens > 0) {
            this.tokens -= 1;
            return true;
        }

        return false;
    }
}
```

We could then use this in a Node.js/express application to limit the
number of requests made to a particular endpoint:

#### Rate Limiting with TokenBucket

```javascript
const express = require('express');
const app = express();

function limitRequests(perSecond, maxBurst) {
    const bucket = new TokenBucket(maxBurst, perSecond);

    // Return an Express middleware function
    return function limitRequestsMiddleware(req, res, next) {
        if (bucket.take()) {
            next();
        } else {
            res.status(429).send('Rate limit exceeded');
        }
    }
}


app.get('/',
    limitRequests(5, 10), // Apply rate limiting middleware
    (req, res) => {
        res.send('Hello from the rate limited API');
    }
);

app.listen(3000, () => console.log('Server is running'));
```

In this example, the `/` endpoint is restricted to serving 5 requests per
second across all clients. If we wanted to have a per-client limit , then we
could keep a map of IP address (or API keys) to token bucket, creating a
new token bucket every time we encounter a new client, as in the following
example:

#### Rate Limiting by IP

```javascript
function limitRequests(perSecond, maxBurst) {
    const buckets = new Map();

    // Return an Express middleware function
    return function limitRequestsMiddleware(req, res, next) {
        if (!buckets.has(req.ip)) {
            buckets.set(req.ip, new TokenBucket(maxBurst, perSecond));
        }

        const bucketForIP = buckets.get(req.ip);
        if (bucketForIP.take()) {
            next();
        } else {
            res.status(429).send('Client rate limit exceeded');
        }
    }
}
```

Using this approach, we would need to be careful, since a large number of
distinct IPs could create quite a bit of overhead in terms of both memory and
timers to refill buckets. In practice, we would probably want to remove the token
buckets after some time, and we would also want to defer adding tokens until
they are requested, which will eliminate the need for JavaScript timers. Here is
our new timer-free `TokenBucket` implementation:

#### Timer-Free TokenBucket

```javascript
class TokenBucket {
    constructor(capacity, fillPerSecond) {
        this.capacity = capacity;
        this.fillPerSecond = fillPerSecond;

        this.lastFilled = Math.floor(Date.now() / 1000);
        this.tokens = capacity;
    }

    take() {
        // Calculate how many tokens (if any) should have been added since the last request
        this.refill();

        if (this.tokens > 0) {
            this.tokens -= 1;
            return true;
        }

        return false;
    }

    refill() {
        const now = Math.floor(Date.now() / 1000);
        const rate = (now - this.lastFilled) / this.fillPerSecond;

        this.tokens = Math.min(this.capacity, this.tokens + Math.floor(rate * this.capacity));
        this.lastFilled = now;
    }
}
```

This implementation should be have the same, but it only does work when `take()` is called,
which should be more efficient in most cases.

Please leave a comment to let me know if this post was useful!

<script async src="//pagead2.googlesyndication.com/pagead/js/adsbygoogle.js"></script>
<ins class="adsbygoogle"
     style="display:block; text-align:center;"
     data-ad-layout="in-article"
     data-ad-format="fluid"
     data-ad-client="ca-pub-6265787006533161"
     data-ad-slot="3706397953"></ins>
<script>
(adsbygoogle = window.adsbygoogle || []).push({});
</script>