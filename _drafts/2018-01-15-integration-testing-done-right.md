---
layout: post
title:  "Integration Testing Done Right"
date:   2018-01-15 12:00:00 -0800
categories: integration testing
author: Sam Sweeney
---

Let's talk about integration testing.

Your manager has asked you to expose the number of active sessions on your mobile app right now through some API.  You decide to spin up a simple API service named `Middleman`.  When it receives a request, Middleman will make an RPC to the Sessions Service (which can't be called by external parties) to get the number of active sessions and pass that information back to the caller.

The service itself is probably going to look something like this:

{% highlight javascript %}
const http = require('http');
const SessionsClient = require('./sessions-client');
const PORT = 3000;

class MiddlemanApp {
    constructor(opts) {
        this.sessionsClient = new SessionsClient(opts.sessionsClientUrl);
        this.server = http.createServer(this.handler.bind(this));
    }

    start(cb) {
        this.server.listen(PORT, cb);
    }

    handler(req, resp) {
        req.on('data', (chunk) => {
            // no-op
        }).on('end', () => {
            this.sessionsClient.getNumActiveSessions((err, sessionsResp) => {
                resp.statusCode = sessionsResp.statusCode;
                if (sessionsResp.numActiveSessions !== undefined) {
                    resp.setHeader('Content-Type', 'application/json');
                    resp.write(JSON.stringify({
                        numActiveSessions: sessionsResp.numActiveSessions
                    }));
                }
                resp.end();
            });
        });
   }

    shutdown(cb) {
        this.server.close(cb);
    }
}
{% endhighlight %}

With a simple SessionsClient:

{% highlight javascript %}
const request = require('request');

class SessionsClient {
    constructor(url) {
        this.url = url;
    }

    getNumActiveSessions(cb) {
        request(this.url, function onResp(err, resp, body) {
            if (err) {
                return cb(err);
            }

            const parsedBody = body && JSON.parse(body);
            const numActiveSessions = parsedBody &&
                parsedBody.numActiveSessions;
            return cb(null, {
                statusCode: resp.statusCode,
                numActiveSessions: numActiveSessions
            });
        });
    }
}
{% endhighlight %}

So how do we test this service?

[Note: in the upcoming examples, we use [`tape`](https://github.com/substack/tape), a fabulous Javascript testing library.  Tape tests give you an `assert` object you can use to make assertions about your code.  A test is completed once `assert.end` is called.]

## Option 1: Mock O'Clock

Why don't we just mock out the client itself?  That way, we can test how your service handles various responses from the Sessions Service without having to actually make requests to it whenever we want to run our tests.  So we write a mock Sessions Client:

{% highlight javascript %}
class MockSessionsClient {
    setResp(resp) {
        this.resp = resp;
    }

    getNumActiveSessions(cb) {
        cb(null, this.resp);
    }
}
{% endhighlight %}

And use it in our tests, replacing `Middleman's` client with the mock one:

{% highlight javascript %}
const request = require('request');
const test = require('tape');
const App = require('../index.js');
const MockSessionsClient = require('./mock-sessions-client.js');

const testOpts = {
    timeout: 500
};
const APP_URL = 'http://localhost:3000';

let middlemanApp;
let mockSessionsClient;

test('start service', testOpts, function t(assert) {
    middlemanApp = new App({
        sessionsClientUrl: 'http://www.fake-url.com/'
    });
    mockSessionsClient = new MockSessionsClient();
    middlemanApp.sessionsClient = mockSessionsClient;
    middlemanApp.start(assert.end);
});

test('everything ok - 200', testOpts, function t(assert) {
    const resp = {
        statusCode: 200,
        numActiveSessions: 5
    };
    mockSessionsClient.setResp(resp);
    request(APP_URL, function onResp(err, resp, body) {
        assert.notOk(err);
        assert.equal(200, resp && resp.statusCode);
        assert.deepEqual(JSON.parse(body), {
            numActiveSessions: 5
        });
        assert.end();
    });
});

test('everything not ok - 500', testOpts, function t(assert) {
    const resp = {
        statusCode: 500
    };
    mockSessionsClient.setResp(resp);
    request(APP_URL, function onResp(err, resp, body) {
        assert.notOk(err);
        assert.equal(500, resp && resp.statusCode);
        assert.end();
    });
});

test('teardown', testOpts, function t(assert) {
    middlemanApp.shutdown(assert.end);
});
{% endhighlight %}

What's wrong with this test?  Well, for starters, it mocks out an entire class.  We have no guarantee that our mock client actually behaves like our real client.  Sure, we can (and probably should) test the real class separately with a unit test, but the point of the tests we're writing is to cover the end-to-end functionality of `Middleman`.  Back to the drawing board.  

## Option 2: Nock O'Clock

A coworker recently told us about a cool library called [`nock`](https://github.com/node-nock/nock) that lets you intercept HTTP requests and return whatever you want.  Sounds perfect!

We toss the MockSessionsClient out and rewrite our tests with `nock`:

{% highlight javascript %}
const nock = require('nock');
const request = require('request');
const test = require('tape');
const App = require('../index.js');

const testOpts = {
    timeout: 500
};
const APP_URL = 'http://localhost:3000';
const SESSIONS_URL = 'http://localhost:4000';

let middlemanApp;

test('start service', testOpts, function t(assert) {
    middlemanApp = new App({
        sessionsClientUrl: SESSIONS_URL
    });
    middlemanApp.start(assert.end);
});

test('everything ok - 200', testOpts, function t(assert) {
    nock(SESSIONS_URL).get('/').reply(200, {
        numActiveSessions: 5
    });
    request(APP_URL, function onResp(err, resp, body) {
        assert.notOk(err);
        assert.equal(200, resp && resp.statusCode);
        assert.deepEqual(JSON.parse(body), {
            numActiveSessions: 5
        });
        assert.end();
    });
});

test('everything not ok - 500', testOpts, function t(assert) {
    nock(SESSIONS_URL).get('/').reply(500);
    request(APP_URL, function onResp(err, resp, body) {
        assert.notOk(err);
        assert.equal(500, resp && resp.statusCode);
        assert.end();
    });
});

test('teardown', testOpts, function t(assert) {
    nock.cleanAll();
    middlemanApp.shutdown(assert.end);
});
{% endhighlight %}

Better!  Our service is running in our test just the way it would in production.  It's almost too good to be true.  And, actually, it kind of is.

The issue is how `nock` works: by overriding the `request` and `clientRequest` functionality in Node's core `http` library (see [here](https://github.com/node-nock/nock/blob/master/lib/common.js#L103)).  In a lot of ways, `nock` is everything we want: it lets us neatly intercept outgoing requests from our app and test how it handles various responses.

On the other hand, we're breaking a central rule of testing: don't mess around with the internals of your environment if you don't have to, especially when it's going to deviate from your production environment.  Just like we wouldn't run the test suite of a Ruby 2.1.x app in an environment that's running 2.3.x, we should try to avoid monkeypatching a part of Node.JS's core library.

Why?  Mainly because we have no way to guarantee that `nock's` implementation of `http.request` is the same as what it's overriding.  (Which is not a shot against `nock`, an elegantly written library that's served me well many times.)  But even if `nock` is 99% identical to the core `http` library, that 1% will lurk in the background, ready to rear its head when you least expect it.

Writing good tests is often about limiting their seams -- their blind spots, the minor differences between how your code runs in your tests and how it runs in production.  If we can avoid introducing a seam here, we should.

## Option 3: Yo Dawg, I Heard You Liked Services

If we shouldn't mock out the client, and we don't want to mess around with our `http` requests, why don't we let `Middleman` actually make the requests and set up a mock Sessions Service that can handle our requests?

Here's our mock service:

{% highlight javascript %}
const http = require('http');

class MockSessionsService {
    constructor(port) {
        this.port = port;
        this.server = http.createServer(this.handler.bind(this));
        this.responses = [];
    }

    start(cb) {
        this.server.listen(this.port, cb);
    }

    registerResp(resp) {
        this.responses.push(resp);
    }

    handler(req, resp) {
        req.on('data', (chunk) => {
            // no-op
        }).on('end', () => {
            const currentResp = this.responses.pop();
            resp.statusCode = currentResp.statusCode;
            if (currentResp.numActiveSessions !== undefined) {
                resp.setHeader('Content-Type', 'application/json');
                resp.write(JSON.stringify({
                    numActiveSessions: currentResp.numActiveSessions
                }));
            }
            resp.end();
        });
   }

    shutdown(cb) {
        this.server.close(cb);
    }
}
{% endhighlight %}

And the tests that use it:

{% highlight javascript %}
const async = require('async');
const request = require('request');
const test = require('tape');
const App = require('../index.js');
const MockSessionsService = require('./mock-sessions-service.js');

const testOpts = {
    timeout: 500
};
const APP_URL = 'http://localhost:3000';
const SESSIONS_PORT = 4000;
const SESSIONS_URL = 'http://localhost:' + SESSIONS_PORT;

let middlemanApp;
let sessionsService;

test('start services', testOpts, function t(assert) {
    middlemanApp = new App({
        sessionsClientUrl: SESSIONS_URL
    });
    sessionsService = new MockSessionsService(SESSIONS_PORT);
    async.parallel([
        middlemanApp.start.bind(middlemanApp),
        sessionsService.start.bind(sessionsService)
    ], assert.end);
});

test('everything ok - 200', testOpts, function t(assert) {
    sessionsService.registerResp({
        statusCode: 200,
        numActiveSessions: 5
    });
    request(APP_URL, function onResp(err, resp, body) {
        assert.notOk(err);
        assert.equal(200, resp && resp.statusCode);
        assert.deepEqual(JSON.parse(body), {
            numActiveSessions: 5
        });
        assert.end();
    });
});

test('everything not ok - 500', testOpts, function t(assert) {
    sessionsService.registerResp({
        statusCode: 500
    });
    request(APP_URL, function onResp(err, resp, body) {
        assert.notOk(err);
        assert.equal(500, resp && resp.statusCode);
        assert.end();
    });
});

test('teardown', testOpts, function t(assert) {
    async.parallel([
        middlemanApp.shutdown.bind(middlemanApp),
        sessionsService.shutdown.bind(sessionsService)
    ], assert.end);
});
{% endhighlight %}

Pretty good!  The downside to this approach is that we're making actual HTTP requests, which can introduce instability to our tests simply because making network requests can be an unreliable endeavor at times.  But our tests now come just about as close to actual production behavior as we're going to get without actually making requests from our test environment to the production Sessions Service.  (And if you do that, you're going to have a bad time.)

## Conclusion

`Middleman` isn't going to win any awards for innovation.  But hopefully the three approaches to writing integration tests for it illustrate a principle I try to follow when writing integration tests: mock sparingly, and when you do mock something, try to do so in a way that deviates from your code's production behavior as little as possible.

**P.S.** This post was inspired by a testing library written and used at Uber, where I previously worked.  Thanks to all my former teammates, in particular those on the Marketplace team, for teaching me about these kinds of tests and making me a better engineer overall.
