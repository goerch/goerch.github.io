---
layout: post
title:  "Getting Jekyll Running on Windows: 99 Gems and One Version Mismatch"
date:   2026-04-26 19:11:29 +0200
categories: jekyll update
---
So you want to publish a static blog with Jekyll and GitHub Pages. How hard can it be?

Harder than it should be, it turns out — but not for the reasons you'd expect.

**Starting point**

Fresh Windows install, Ruby 4.0.3 via winget, following the official GitHub Pages documentation. Step one: create a new Jekyll site, add the `github-pages` gem, run `bundle install`. Straightforward enough.

Except it isn't.

**The version wall**

`bundle install` fails immediately with a dependency error: `commonmarker`, somewhere deep in the `github-pages` gem chain, explicitly requires Ruby `>= 2.6, < 4.0`. Ruby 4.0.3 is, technically, not less than 4.0.

The GitHub documentation points you to `pages.github.com/versions.json` for the current gem version but says nothing about Ruby version requirements. To be fair, pinning documentation to fast-moving version constraints is genuinely hard — but a note like "Ruby 3.x recommended" would save a lot of people some time.

The fix: uninstall Ruby 4.0.3, install 3.4.x via RubyInstaller (with Devkit — don't skip that), start over.

**99 gems**

Once the right Ruby is in place, `bundle install` completes cheerfully:

```
Bundle complete! 7 Gemfile dependencies, 99 gems installed.
```

99 gems. For a static site generator. If you know the song, you know the song.

To be fair to Jekyll: the resulting site looks genuinely nice out of the box, Minima is a clean default theme, and the whole thing is now running on localhost exactly as advertised. The friction was entirely in getting there.

**What I'd do differently**

Install Ruby via RubyInstaller from the start, not winget. Pick 3.3.x or 3.4.x. Run `ridk install` when it asks you to. Then follow the docs.

Two hours of dependency archaeology, condensed into one paragraph.
