---
author: Geoff Harcourt
categories:
- rails
- rails assets
- bower
- asset pipeline
layout: post
title: Bower in the Asset Pipeline with Rails-Assets
---

[Bower](http://bower.io) is a dependency manager created by Twitter. Like Ruby's
Bundler, Bower takes a list of specified packages and version ranges, calculates
a dependency graph, and determines what set of package versions should be
delivered (working out which versions of which packages are compatible with the
other packages you have installed). While Bundler manages Ruby dependencies,
Bower manages web assets such as JavaScript, CSS, or fonts. 

Bower can
be integrated with the [Rails Asset
Pipeline](http://guides.rubyonrails.org/asset_pipeline.html), making managed
external JavaScript, CSS, or font libraries available with all the conveniences
the asset pipeline provides.

## External assets and the asset pipeline

Since the addition of the Asset Pipeline in Rails, there have been two widely
used strategies for integrating external assets into a Rails application:
vendoring the assets or "gemifying" the assets.

Developers can vendor assets by placing the relevant files in a subfolder of the
app's `vendor/assets` folder (such as `fonts`, `javascripts`, or `stylesheets`).
From there, you include assets using Sprockets require directives from your
`.js` or `.css` files as if they were saved with your application's internal
assets folder. This technique is fine for CSS and Javascript (if you are willing
to update the vendored files whenever you want a newer version), but prior to
Rails 4 there were some inconsistent behaviors related to the treatment of
non-CSS/JavaScript assets such as fonts, necessitating custom configuration.

In Rails 3.2, some developers solved the problem of including assets by building
gems that contained the assets, bringing assets into the application through
Bundler. You could then bundle a library by including `gem
"twitter-bootstrap-rails"` in your Gemfile and get Bootstrap in your Rails
application. I often found this approach frustrating, because gem authors would
frequently fail to update the gems promptly after new versions of dependencies
were released. Compounding the issue, the versioning scheme of gemified
dependencies often did not track in step with the asset versions (so version
2.1.0 of the gem didn't match up with version 2.1.0 of the library), resulting
in unexpected breakages or upgrades without the most pessimistic gemfile
locking.

## Enter Bower

Assets managed by Bower can be updated as soon as the authors published new code
on Github, and bring with them none of the versioning baggage of "gemified"
dependencies, making it a great way to manage non-Ruby dependencies in a Rails
application. Much like Bundler, Bower allows version locking and Github branch
and tag tracking (I like Bower's syntax for specifying a version or range of
versions better than Bundler's). Bower can be configured to store downloaded
assets under `/vendor/assets`, making them available through Sprockets and the
asset pipeline. At Cortex, we initially integrated Bower into our application's
deploy process using a [custom Heroku
buildpack](https://github.com/qnyp/heroku-buildpack-ruby-bower).

This approach ensured that our assets were built on each deploy, and prevented
issues where assets used in development might not be checked into version
control (and therefore would not make it into staging). Assets were downloaded
and assembled on each deploy. Using Bower as a built-in step in our buildpack
also had the advantage of performing automatic updates (for versions within our
locking scheme) whenever we deployed to Heroku.

Working with Bower made inclusion of assets in our application much easier
(definitely the fewest problems I had ever experienced with a complicated Rails
application in production), but I was frustrated by a few aspects of the
integration. Because assets were being built on each deploy and were not
included in source control (only the Bower configuration was checked in), it
wasn't possible to know with absolute certainty what asset versions were active
for a given deploy. If we forgot to run `bower update` locally, it was possible
that our local versions of the Cortex app might not be in sync with what was
deployed in staging or production.  In addition, our CI server had to download
npm and Bower and install all the assets from scratch on every build. (There are
workarounds for this problem such as caching asset manifests on S3, but they
only introduce further environment sync headaches.)

## Rails Assets

We were not the only developers wishing for an easier solution for the
integration of Bower and Rails. The [Rails Assets](http://rails-assets.org) team
has created a very slick application that provides for simple inclusion of Bower
packages into the asset pipeline with the tracking of dependency versions
through source control and consistency of dependency versions between
environments without any programs to install.

Any package on Bower can be built as a Rails Assets gem, where the gem takes the
name `rails-assets-` plus the Bower package name. Gems are versioned by their
Bower package version, controlled by Git tags on the associated Github
repository. Rails Assets takes advantage of Bundler's ability to bring in gems
from multiple sources. You declare Rails Assets as a gem source along with
Rubygems and then list your assets in `Gemfile` along with your Ruby gems.
Here's a snippet to show how Cortex brings the JavaScript visualization library
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

When we deploy our app to Heroku, Bundler and `Gemfile.lock` ensure that the
same version of our Bower dependencies are used in production as were used in
our development and testing environments. As a side benefit, we can dig back in
our source control history and always know exactly what versions of libraries
were included in our application at a given point in time.
