---
author: Geoff Harcourt
author_twitter: "@geoffharcourt"
categories: ruby tools
layout: post
title: 'Source-diving with Github and gem browse'
---

One of the things that has strengthened my ability to troubleshoot Ruby issues
has been getting comfortable with source-diving. Source-diving is reading the
source code of your language or libraries to follow the behavior implemented in
the software written by other authors that interacts in your software. Rather
than treating third-party libraries as a mysterious black box, reading a
library's source allows you to both troubleshoot unexpected issues in your use of
others' code and to learn from the authors' techniques.

<!--break-->

When I encounter a problem that I've traced back to a gem I'm using in my
application, I like to do two things: browse the issues filed against the gem on
Github to see if someone else has reported a similar problem, and to read the
source code itself to see if I can implement a workaround without having to wait
for an issue to be fixed (or to fix it myself!). There are some awesome
time-saving tools you can use to help make these two tasks fast.

If you want to browse the source code for a gem, you should install Tim Pope's
excellent [gem-browse](https://github.com/tpope/gem-browse) gem. When this gem
is installed, you get some subcommands for Rubygems that help you work with gem
source code. The first is `gem browse gemname`, which opens a web browser at the
official website for the Gem, which is often the gem's Github repository page.

In addition to `gem browse`, the gem comes with some other useful commands. `gem
open gemname` opens the gem in your default text editor. If you are using Tim
Pope's (notice a trend here?) [gem-ctags](https://github.com/tpope/gem-ctags),
you can quickly navigate the gem's files to to the code of interest. You won't
want to edit code here unless you're testing a local fix, as changes here will
affect the gem throughout your development environment. If you're interested in
submitting a pull request to fix an issue you've found, you can quickly clone
the gem's source code in a new folder outside of your gem path with `gem clone
gemname`, which clones the repository from Github.
