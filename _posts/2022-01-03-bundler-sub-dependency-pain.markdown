---
layout: single
title:  "Bundler Sub Dependency Pain"
date:   2022-01-03 16:38:00 +0200
categories: bundler
---

Bundler implicit sub-dependencies perfer to pull from their highest declared non-default source, and it's an absolute pain...

## Context

If you've done any heavy enterprise Ruby work with features across multiple systems, logic that multiple systems need to be aware of simultaneous, or alternatively built some kind of internal cli tool in ruby, then you've probably built a gem in the past. There's a further chance that you might not have been ready to opensource that particular gem for some reason, and have thus opted to host it on some form of alternative gem registry. Gem Fury, Artifactory, or a simple `gem server`.

Around the beginning of 2021 it was realized that Bundler was vulnerable to something referred to as [dependency confusion](https://medium.com/@alex.birsan/dependency-confusion-4a5d60fec610). While I won't go too far into depth about about this attack, the TL:DR version is that when declaring a Gemfile, in the past Bundler would prioritize gems found on the default source `rubygems.org`. If you then added an additional source for some kind of internal gem, someone could release a opensource gem using the same name as your internal gem, which Bundler would then install instead of pulling your gem from your specified source.

While there were protections in place on `rubygems.org` to prevent malicious gems from uploading, this was still a hazardous outcome from the perspective of being at worst as possible RCE, at best, still a nuisance.

So Bundler was [patched](https://bundler.io/blog/2021/02/15/a-more-secure-bundler-we-fixed-our-source-priorities.html) to make it prefer the explicitly declared gem source for a particular gem over the default gem source. Great! In almost all cases, this is probably exactly the behaviour you want. But, it comes with a caveat...

## The Problem

Bundler is a magnificent piece of software for helping ensure versions are locked at their intended versions, matched to their needed compatibilities, and pulled from their effective sources, all without needing to be needlessly verbose about every single little thing that your app needs to run. If you've ever taken a peak at the `Gemfile.lock` file in a fresh rails app you might have been surprised to find close to 100 dependencies, despite the default Gemfile declaring 15 or so. The reason for this is when a gem is created, a gemspec file is written up containing gem dependencies and versions for those dependencies that the gem should be able to work with. This is the information which Bundler combines with the declared version constraints within your projects Gemfile in order to grab any dependencies for the gems you declare to make sure they work as intended, without you needing to explicitly declare the dependencies of your dependencies yourself in your Gemfile. Brilliant right?

Now, here's the question. If I declare a gem in my Gemfile, and configure it to come from a non-standard source, where do you expect that gem's dependencies to come from?

{% highlight ruby %}
source "https://rubygems.org"

gem "rspec"
gem "work-logic", source: "https://internal-gem-repo.awesome"
{% endhighlight %}

Intuitively, you might have guessed that the logical source of a gem's dependencies would the same place as the gem itself right? So if I have a gem called, for example, `work-logic`, declared to come from `https://internal-gem-repo.awesome`, we would expect any dependencies of `work-logic` to similarly be sourced from `https://internal-gem-repo.awesome`. Makes perfect sense.

{% include figure image_path="/assets/images/2021/01/03/dependencies 1.png" alt="Source Graph for above Gemfile" caption="rspec from rubygems, work-logic from internal gem repo." %}

Now, what if one of the dependencies of my internal `work-logic` gem is a public gem? Perhaps the `work-logic` gem is actually a Rails engine, and thereby actually depends on `rails`.

{% include figure image_path="/assets/images/2021/01/03/dependencies 2.png" alt="Source Graph for above Gemfile, with rails a dependency" caption="Rails as a dependency of work-logic" %}

Well, with the above behaviour, because `work-logic` is sourced from `https://internal-gem-repo.awesome`, and `rails` is a dependency of `work-logic`, bundler is going to try source rails from `https://internal-gem-repo.awesome`. And if you couldn't be bothered to mirror rails from your gem server, then it's going to fail.


"So just declare rails's source as `rubygems.com` explictly then..." you're probably thinking. 

{% highlight ruby %}
source "https://rubygems.org"

gem "rspec"
gem "rails"
gem "work-logic", source: "https://internal-gem-repo.awesome"
{% endhighlight %}

{% include figure image_path="/assets/images/2021/01/03/dependencies 3.png" alt="Source Graph for above Gemfile, with rails as dependency, explicitly declared" caption="Rails as a dependency of work-logic, explicitly sourced from rubygems.org" %}

And that does indeed work, rails is then explicitly sourced from `rubygems.org`, `work-logic` is explicitly sourced from `https://internal-gem-repo.awesome`, and all sounds swell until we hit the next reason bundler fails. Rails has dependencies. Lots of them. So, in a situation where `work-logic` depends on `rails` but is sourced from `https://internal-gem-repo.awesome`, and `rails`, explicitly requested from `rubygems.org` has all of it's own implicit dependencies like `active-record`; where would you expect `rails`'s dependencies to be sourced from? Intuitively, one would probably say "well, those are rails dependencies, so those should come from `rubygems.org`.". Except that's not what Bundler does. Bundler sees the sub dependencies of `rails` in this situation as part of the initial `work-logic` dependency tree. So even though we've explicitly requested `rails` be sourced from `rubygems.org`, bundler attempts to source `rails`'s dependencies from `https://internal-gem-repo.awesome`, which, much like `rails` itself, isn't there in this situation, and fails.

{% include figure image_path="/assets/images/2021/01/03/dependencies 5.png" alt="Source Graph for above Gemfile, with rails as an explicitly declared dependency, but it has it's own dependencies that still come from the internal gem repo" caption="Rails as an explicitly declared dependency of work-logic but sourced from ruby gems, but it's implicit gems still source from our internal gem repo." %}

As it stands, there's only 2 ways to fix this, some dependency jiggery pokery and potentially manual modification of your gemfile to try force these sub-dependencies to source from `rubygems.org`, or explicitly declare every single gem that rails depends on in your Gemfile and it's source as `rubygems.org`, eschewing much of the simplicity the initial Gemfile would have offered.

{% highlight ruby %}
source "https://rubygems.org"

gem "rspec"
gem "rails"
gem "active-record"
gem "action-text"
gem "active-support"
... etc
gem "work-logic", source: "https://internal-gem-repo.awesome"
{% endhighlight %}

Ew...

## The Solution?

Simply, in the process of generating the dependency graph and figuring out the versions and sources of gems, the closest explicitly declared parent of a gem should be used to determine it's source. So as per the above context, the moment we declare `rails` explicitly, as being sourced from `rubygems.org`, and implicit dependencies of `rails` should then be sourced from `rubygems.org` as well, unless a more explicit directive on one of those dependencies is given.

I'm not familiar with the internal mechanics of bundler however, so I don't know how complicated this would be to implement, but I think this would be far more intuitive.