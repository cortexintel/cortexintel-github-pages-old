---
author: Geoff Harcourt
author_twitter: "@geoffharcourt"
categories: open-source gems
layout: post
title: 'Cortex in Open Source: parity'
---

At Cortex our application runs in many environments. Changes are first built and
evaluated in local development and test environments on a laptop. When changes
are considered ready for deployment, they get pushed up to our cloud CI
environment (currently [Travis](http://travis-ci.com)), where they are tested
independently from local developer machines. After the build passes on Travis,
the new code is then deployed to our [Heroku](http://heroku.com)
environments in the cloud. Our team evaluates changes that are ready for team
review on the staging server, and once those changes are approved, we deploy the
code to our production and demonstration environments. (Have you been counting?
That's six environments: development, test, CI, staging, production, and demo.)

<!--break-->

We use the [`parity`](http://github.com/croaky/parity) gem to manage interaction
with our three Heroku environments. Parity creates shell commands for
`production` and `staging` that pass arguments through to the `heroku` command
from Heroku Toolbelt. So instead of this annoying and frequent error message
when you try to read the application logs:

{% highlight css %}
>> heroku logs

Multiple apps in folder and no app specified.
Specify app with --app APP.
{% endhighlight %}

when you forget to specify the environment, you can instead use a passthrough
command and get to the right environment without having to add an `-app myapp`
switch:

{% highlight css %}
>> production logs
{% endhighlight %}

Parity sends the command to the production instance of your application.
The gem can also be used to shuttle data from production to staging or
development as a layer on top of Heroku's (free) PG backups addon. Here's how
you can replace your development database with the most recent production
data:

{% highlight css %}
>> production backup && development restore production
{% endhighlight %}

This week my proposed [pull request](https://github.com/croaky/parity/pull/11)
to Parity was accepted, adding the capability to work with environments beyond
production and staging. At Cortex use it for our demo environment, but author
[Dan Croak](http://github.com/croaky) also recommends it for developer-specific
feature servers as well (developer Jane's personal staging/group test server
could be `myapp-jane`, allowing her to share her work with teammates for browser
testing).

The new multi-environment functionality was released as part of version 0.3.0.
Check out the Parity [repo documentation](http://gitub.com/croaky/parity) and
give it a try for yourself.






