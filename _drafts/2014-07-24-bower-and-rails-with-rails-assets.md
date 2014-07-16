---
author: Geoff Harcourt
categories:
- rails
- rails assets
- bower
- asset pipeline
layout: post
---

[Bower](http://bower.io) is a dependency manager created by Twitter. Like Ruby's
Bundler, Bower takes a list of specified packages and version ranges, calculates
a dependency graph, and determines what set of package versions should be
delivered. While Bundler manages Ruby dependencies, Bower manages asset
dependencies such as JavaScript or CSS libraries. Bower can be integrated with
the [Rails Asset Pipeline](http://guides.rubyonrails.org/asset_pipeline.html),
making managed external JavaScript, CSS, or font libraries available with all
the conveniences the asset pipeline provides.

## External assets and the asset pipeline

Since the addition of the Asset Pipeline in Rails, there have been two widely
used strategies for integrating external assets into a Rails application:
vendoring the assets or "gemifying" the assets.

You vendor assets by placing the relevant files in a subfolder of the app's
`vendor/assets` folder (such as `fonts`, `javascripts`, or `stylesheets`). From
there, you can include them with Sprockets require directives from your `.js` or
`.css` files as if they were saved with your application's internal assets in
the `app/assets` folder. This approach worked well for CSS and Javascript, but
prior to Rails 4 there were some inconsistent behaviors related to the treatment
of CSS and JavaScript/CoffeeScript vs. fonts and other assets.

During the Rails 3.2 era, some developers solved the problem of including assets
by building gems that contained the assets, which would bring the assets into
the application through Bundler. You could bundle `twitter-bootstrap-rails` and
get Bootstrap in your Rails application. This approach often resulted in
frustration when gems were not updated as promptly as the assets were by the
original asset authors, and often the versioning scheme of the gems did not
track in step with the asset versions, resulting in unexpected breakages or
undesired upgrades.

## Enter Bower

Assets managed by Bower can be updated as soon as the authors published new code
on Github, and brought with them none of the versioning baggage of "gemified"
dependencies, making it a great way to manage non-Ruby dependencies in a Rails
application. Much like Bundler, Bower allows version locking and Github branch
and tag tracking (I like Bower's version syntax better than Bundler's). Bower
can be configured to store downloaded assets under `/vendor/assets`, making them
available through Sprockets and the asset pipeline. At Cortex, we integrated
Bower into our application's deploy process using a [custom Heroku
buildpack](https://github.com/qnyp/heroku-buildpack-ruby-bower).  

This approach ensured that our assets were built on each deploy, and prevented
issues where assets might not be checked into version control. As the assets
were downloaded and assembled on each deploy, it had the advantage of performing
automatic updates (for versions within our locking scheme) whenever we deployed
to Heroku.

Working with Bower made inclusion of assets in our application much easier, but
I was frustrated by a few aspects of the integration. Because assets were being
built on each deploy and were not included in source control (only the Bower
configuration was checked in), it wasn't possible to know with absolute
certainty what asset versions were active for a given deploy. If we forgot to
run `bower update` locally, it was possible that our local versions of the
Cortex app might not be in sync with what was going on in staging or production.
In addition, our CI server had to download npm and Bower and install all the
assets from scratch on every build. (There are workarounds for this problem such
as caching asset manifests on S3, but they only introduce further environment
sync headaches.)

## Rails Assets

We were not the only developers wishing for an easier solution for integrating
Bower and Rails. The [Rails Assets] team has created a very slick application
that allows for simple integration of Bower packages into the asset pipeline,
tracking of dependency versions through source control, and consistency of
dependency versions between environments without any programs to install.

Any package on Bower can be built as a Rails Assets gem, where gem takes the
name `rails-assets-` plus the Bower package name. Gems are versioned by their
Bower package version (controller by Git tags on the associated Github
repository). Rails Assets takes advantage of Bundler's ability to bring in gems
from multiple sources. You declare Rails Assets as a gem source along with
Rubygems and then list your assets in `Gemfile` along with your Ruby
gems. Here's how Cortex brings the JavaScript visualization library
[d3](http://www.d3js.org) into our application:

{% highlight ruby %}
### Gemfile

source "https://rubygems.org"
source "https://rails-assets.org"

# Gemified Bower packages locked to patch versions
gem "rails-assets-d3", "~> 3.4.10"
gem "rails-assets-jquery", "~> 1.11.0"

# Ruby gems...
gem "rails", "4.1.4"
# ...

{% endhighlight %}

When we deploy our app to Heroku, Bundler, and `Gemfile.lock` ensure that the
same version of our Bower dependencies are used in staging and production as
were used in development and testing.
