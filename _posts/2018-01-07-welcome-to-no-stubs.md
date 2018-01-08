---
layout: post
title:  "Welcome to No Stubs"
date:   2018-01-07 21:00:00 -0800
categories: welcome testing personal
author: Sam Sweeney
---

I first learned how to program at [App Academy](https://www.appacademy.io/), a three month coding bootcamp.  I had an incredible experience there, but we probably spent a day, maybe two, learning about testing.  As a result, the `spec` [directory](https://github.com/shubik22/BetterNote/tree/master/spec) of my final project, a Rails/Backbone clone of Evernote, is full of files like [this](https://github.com/shubik22/BetterNote/blob/master/spec/controllers/notes_controller_spec.rb):

{% highlight ruby %}
require 'spec_helper'

describe NotesController do
end
{% endhighlight %}

Sad!  Of all the things that I learned at App Academy, the value of testing was not among them.  That's why I feel so lucky to have ended up at [Wealthfront](https://www.wealthfront.com/), an online financial advisor, as my first job as a software engineer.

Wealthfront loves testing.  And, starting at Wealtfront as more or less a blank canvas, I learned to love testing too.  I don't consider myself a testing ideologue today and hopefully come Monday you won't find me at work lecturing my coworkers about the purity of their unit tests.  But tests are the first thing I think about whenever I'm starting on almost any software engineering task these days and I consider what I learned about testing at Wealthfront to have been the most valuable part of my education as a software engineer to this day.

At App Academy, to the extent that we thought about testing at all, we thought of it as a chore, something with minimal value that you did before you get back to building, creating, doing the stuff that made you want to write code in the first place.  Maybe that's true when your ultimate goal is to throw together an app by yourself in two weeks and the move on.

But at Wealthfront, I first confronted one of the central problems of software development.  Writing code, modifying code, maintaining code, collaborating on code are difficult, error-prone, unpredictable endeavors.  And yet, you need to produce reliable, stable, maintainable software.  How?

Wealthfront's answer was automated testing:

  * Nearly any change to the codebase, even a seemingly trivial one, was expected to have test coverage.
  * If you wanted to enforce a practice throughout the codebase, getting the team to agree with you wasn't enough -- you also needed to find a way to enforce the practice in an automated way.
  * Deployment was as simple as clicking a button in the aptly named Deployment Manager, which would kick off a series of tests and, if they passed, make your changes live.  Engineers were allowed to deploy more or less [at will](https://www.wealthfront.com/engineering).
  * Code comments were frowned upon; there's no way to reliably ensure they stay in sync with the code, and anyways your code is already documented -- by your tests.
  * Manual QA was non-existent, having been offloaded to a [lovable rodent](https://github.com/teamcapybara/capybara).

For the most part, the system worked.  Only after I left Wealthfront did I learn what a "code freeze" was, or experience the tedium of manually tapping through an app to verify the latest release whenever I was on call, or have to explain (twice!) to a senior engineer why merging changes that broke the build wasn't ok, no matter how urgently he needed to ship his feature.

Of course, it wasn't perfect.  All testing strategies have flaws, which will inevitably become painfully apparent at the most inopportune time.  And for every 10 comments on a PR that just read "test?" that made me write a useful test, there'd be one that would lead me to write a test that was about as valuable as the one at the top of this post.

But today, I think of testing as a core part of any software development I do.  Testing philosophy and culture is near the top of my list of criteria for companies I consider working at.  During my last job search, the top of my resume read "Writing code for my tests, and not the other way around," a joke I stole from I'm not sure where that wasn't really a joke.

Which leads to me starting this blog.  As important as testing is to successful software development, it's also really complicated and tough to do well.  So there's a lot to say about it, I think.

My goal with this blog is to write about all things testing-related: stories from places I've worked, libraries I like, patterns that I think are interesting, and ideas that I think are worth talking about.  I doubt all, or even much, of it will be new or revolutionary but I'm sure the act of writing it will be fun for me, and hopefully reading it will be fun for some of you too.
