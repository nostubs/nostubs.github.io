---
layout: post
title:  "Integration Testing Done Right"
date:   2018-01-09 12:00:00 -0800
categories: integration testing
author: Sam Sweeney
---

Let's talk about integration testing.

Your manager has asked you to expose the number of active sessions on your mobile app right now through some API.  You decide to spin up a new service that you name `Middleman`.  It's gonna be a simple API.  When it receives a request, Middleman will make an RPC to the Sessions Service (which can't be called by external parties) to get the number of active sessions and pass that information back to the caller.

You figure the service itself is going to look something like this:

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

So how do you test this service?

[Note: we use [`tape`](https://github.com/substack/tape), a great Javascript testing libraries in the upcoming examples.  Tape tests give you an `assert` object you can use to make assertions about your code.  A test is completed once `assert.end` is called.]

## Option 1: Gotta Mock It All

Your first thought: why don't we just mock out the client itself?  That way, you can test how your service handles various responses from the Sessions Service without having to actually make requests to it whenever you want to run your tests.  You write a mock Sessions Client:

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

And use it in your tests:

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


{% highlight javascript %}
{% endhighlight %}

