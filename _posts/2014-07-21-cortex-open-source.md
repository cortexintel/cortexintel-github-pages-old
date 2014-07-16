---
author: Geoff Harcourt
categories: open-source gems
layout: post
title: 'Cortex in Open Source: parity'
---

At Cortex we have several environments in our workflow. Changes that go through
local development and test environments get pushed up to our cloud CI
environment (currently [Travis](http://travis-ci.com), and then deployed to our
multiple [Heroku](http://heroku.com) environments in the cloud. First, code is
deployed to staging for evaluation by staff. After we've approved the changes on
staging, we push to production and to our demo environment (where we have
special data for demonstrations without exposing client information).

We have found that the [`parity`](http://github.com/croaky/parity) gem has been a
very useful tool for managing git repositories linked to multiple remote Heroku
environments. `parity` creates passthrough shell commands for `production` and
`staging` that pass arguments through to the `heroku` command from Heroku
Toolbelt. So instead of this annoying error message:

{% highlight css %}
>> heroku logs

Multiple apps in folder and no app specified.
Specify app with --app APP.
{% endhighlight %}

when you forget to specify the environment, now you can type:

{% highlight css %}
>> production logs
{% endhighlight %}

and `parity` passes the command through for you to the production instance of
your application. `parity` can also be used to shuttle data from production to
staging or development as a layer on top of Heroku's (free) PG backups addon.
Here's how you can replace your development database with the most recent
production backup:

{% highlight css %}
>> development restore production
{% endhighlight %}

This week my proposed [pull request](https://github.com/croaky/parity/pull/11)
to parity was accepted, adding the capability to expand parity to work with
environments beyond production and staging. We use it for our demo environment,
but author [Dan Croak](http://github.com/croaky) also recommends it for
developer-specific feature servers as well. The new functionality was released
as part of version 0.3.0.

Check out the parity [repo documentation](http://gitub.com/croaky/parity) and
give it a try for yourself.






