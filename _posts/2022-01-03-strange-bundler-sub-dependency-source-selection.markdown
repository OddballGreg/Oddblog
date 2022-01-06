---
layout: single
title:  "Strange Bundler sub-dependency source selection"
date:   2022-01-03 16:38:00 +0200
categories: bundler
---

Bundler implicit sub-dependencies prefer to pull from their highest declared non-default source, and it's an absolute pain...

## Context

If you've done any heavy enterprise Ruby work with shared features across multiple systems, logic that multiple systems need to be aware of simultaneously, or alternatively built some kind of internal cli tool in ruby, then you've probably built a gem in the past. There's a further chance that you might not have been ready to opensource that particular gem for some reason, and thus opted to host it on some form of alternative gem registry. Gem Fury, Artifactory, or a simple `gem server` probably.

Around the beginning of 2021 it was realized that Bundler was vulnerable to something referred to as [dependency confusion](https://medium.com/@alex.birsan/dependency-confusion-4a5d60fec610). While I won't go too far into depth about about this attack, the TL:DR version is that when declaring a Gemfile, in the past Bundler would prioritize gems found on the default source `rubygems.org`. If you then added an additional source for some kind of internal gem, someone could release a opensource gem using the same name as your internal gem, which Bundler would then install instead of pulling your gem from your specified source.

While there were protections in place on `rubygems.org` to prevent malicious gems from being uploaded, this was still a hazardous outcome from the perspective of being, at worst as possible RCE, at best, still a rather massive nuisance.

So Bundler was [patched](https://bundler.io/blog/2021/02/15/a-more-secure-bundler-we-fixed-our-source-priorities.html) to make it prefer the explicitly declared gem source for a particular gem over the default gem source. Great! In almost all cases, this is probably exactly the behaviour you want. But, it comes with a caveat...

## The Problem

Bundler is a magnificent piece of software for helping ensure versions are locked at their intended versions, matched according to their dependency version specifications, and pulled from their effective sources, all without needing to be needlessly verbose about every single little thing that your app needs to run. If you've ever taken a peek at the `Gemfile.lock` file in a fresh rails app you might have been surprised to find close to or even over 100 dependencies, despite the default Gemfile declaring 15 or so. The reason for this is when a gem is created, a gemspec file is written up containing gem dependencies and versions for those dependencies that the gem should be able to work with. This is the information which Bundler combines with the declared version constraints within your project's Gemfile in order to grab any dependencies for the gems you declare to make sure they work as intended, without you needing to explicitly declare the sub-dependencies of your dependencies yourself in your Gemfile. Brilliant right?

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

Well, with the above behaviour, because `work-logic` is sourced from `https://internal-gem-repo.awesome`, and `rails` is a dependency of `work-logic`, bundler is going to try source `rails` from `https://internal-gem-repo.awesome`. And if you couldn't be bothered to mirror `rails` from your gem server, then it's going to fail.

"So just declare the rails source as `rubygems.com` explictly then..." you're probably thinking. 

{% highlight ruby %}
source "https://rubygems.org"

gem "rspec"
gem "rails"
gem "work-logic", source: "https://internal-gem-repo.awesome"
{% endhighlight %}

{% include figure image_path="/assets/images/2021/01/03/dependencies 3.png" alt="Source Graph for above Gemfile, with rails as dependency, explicitly declared" caption="Rails as a dependency of work-logic, explicitly sourced from rubygems.org" %}

And that does indeed work, `rails` is then explicitly sourced from `rubygems.org`, `work-logic` is explicitly sourced from `https://internal-gem-repo.awesome`, and all sounds swell until we hit the next reason bundler fails. Rails has dependencies. Lots of them. So, in a situation where `work-logic` depends on `rails` but is sourced from `https://internal-gem-repo.awesome`, and `rails`, explicitly requested from `rubygems.org`, has all of it's own implicit dependencies like `active-record`; where would you expect `rails`'s dependencies to be sourced from? Intuitively, one would probably say "well, those are rails dependencies, so those should come from `rubygems.org`." Except that's not what Bundler does. Bundler sees the sub dependencies of `rails` in this situation as part of the initial `work-logic` dependency tree. So even though we've explicitly requested `rails` be sourced from `rubygems.org`, bundler attempts to source `rails`'s dependencies from `https://internal-gem-repo.awesome`. Since there are mirrored versions of those gems on the internal gem repo, this technically works, but is not the desired outcome of our configuration.

{% include figure image_path="/assets/images/2021/01/03/dependencies 5.png" alt="Source Graph for above Gemfile, with rails as an explicitly declared dependency, but it has it's own dependencies that still come from the internal gem repo" caption="Rails as an explicitly declared dependency of work-logic but sourced from ruby gems, but it's implicit gems still source from our internal gem repo." %}

As it stands, there's only 2 ways to fix this, some dependency jiggery pokery and potentially some manual modification of your `Gemfile.lock` to try force these sub-dependencies to source from `rubygems.org`, or explicitly declare every single gem that `rails` depends on in your Gemfile and it's source as `rubygems.org`, eschewing much of the simplicity the initial Gemfile would have offered.

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

## The (Rough) Solution

Simply, in the process of generating the dependency graph and figuring out the versions and sources of gems, the closest explicitly declared parent of a gem should be used to determine it's source. So as per the above context, the moment we declare `rails` explicitly, as being sourced from `rubygems.org`, any implicit dependencies of `rails` should then be sourced from `rubygems.org` as well, unless a more explicit directive on one of those lower dependencies is given.

I spent... **way** too much time reading the bundler source code to figure out why this was happening, but I think I've tracked down the problem to the [SourceMap implementation](https://github.com/rubygems/rubygems/blob/1c8c02b45db892a834d517eb64474d52311ff4c1/bundler/lib/bundler/source_map.rb). 

You can delve into the depths of Bundler if you're interested, but the issue seems to boil down to the `Bundler:SourceMap#all_requirements` method's solution for resolving the source of indirect dependencies after the first pass of direct dependency resolution.

{% highlight ruby %}
def all_requirements
  requirements = direct_requirements.dup

  unmet_deps = sources.non_default_explicit_sources.map do |source|
    (source.spec_names - pinned_spec_names).each do |indirect_dependency_name|
      previous_source = requirements[indirect_dependency_name]
      if previous_source.nil?
        requirements[indirect_dependency_name] = source
      else
        no_ambiguous_sources = Bundler.feature_flag.bundler_3_mode?

        msg = ["The gem '#{indirect_dependency_name}' was found in multiple relevant sources."]
        msg.concat [previous_source, source].map {|s| "  * #{s}" }.sort
        msg << "You #{no_ambiguous_sources ? :must : :should} add this gem to the source block for the source you wish it to be installed from."
        msg = msg.join("\n")

        raise SecurityError, msg if no_ambiguous_sources
        Bundler.ui.warn "Warning: #{msg}"
      end
    end

    source.unmet_deps
  end

  sources.default_source.add_dependency_names(unmet_deps.flatten - requirements.keys)

  requirements
end

def direct_requirements
  @direct_requirements ||= begin
    requirements = {}
    default = sources.default_source
    dependencies.each do |dep|
      dep_source = dep.source || default
      dep_source.add_dependency_names(dep.name)
      requirements[dep.name] = dep_source
    end
    requirements
  end
end
{% endhighlight %}

To contextualize the snippet, the SourceMap defines the fully realized set of dependencies and where they need to be pulled from for Bundler to ultimately resolve and pull your gems. `direct_requirements` looks at the list of dependencies you explicitly defined in your Gemfile, and if the Gem has an explicitly defined source by virtue of the `source` kwarg or a `source` block wrapping it's definition in your Gemfile, then the source is defined. This dsl is evaluated to create a bundler [Definition](https://github.com/rubygems/rubygems/blob/1c8c02b45db892a834d517eb64474d52311ff4c1/bundler/lib/bundler/definition.rb), which prior to resolution, generates a SourceMap using the above snippet to determine which source your respective gem will come from.

The problem however, is in these few lines:

{% highlight ruby %}
def all_requirements
  requirements = direct_requirements.dup

  unmet_deps = sources.non_default_explicit_sources.map do |source|
    (source.spec_names - pinned_spec_names).each do |indirect_dependency_name|
      previous_source = requirements[indirect_dependency_name]
      if previous_source.nil?
        requirements[indirect_dependency_name] = source
      else
  ...
{% endhighlight %}

Bundler is grabbing unmet dependencies by cycling through your explicitly declared, non-default sources. Which is to say any gem source you've declared at all, that's not RubyGems. It then goes through the spec list from that source, excluding any gems you explicitly declared in your Gemfile, and if that gem was already declared with a source elsewhere (such as your Gemfile or Gemfile.lock), it will just use that source. Otherwise it will just select the source it's currently looking at as the source for your gem. This isn't an illogical choice. If your gem is coming from this alternative repository, and your gem's dependencies are present on this mirror, it's probably reasonable you want your gem's dependencies to come from there too.

The problem is, per our example above, we've explicitly declared `rails` with the expectation that it's dependencies should be sourced from `rubygems.org`, but because some version of `rails`'s dependencies exist on `https://internal-gem-repo.awesome`, it's prefering our explicitly declared gem source anyways.

So step one to fixing this bizarre behaviour, is to make Bundler explicitly prefer to fetch a gems dependencies from it's closest explicitly declared parent:

{% highlight ruby %}
def all_requirements
  requirements = direct_requirements.dup
  direct_dependency_parents = @dependencies.map{|dep| dep.to_spec.dependencies.map{|sub_dep| {sub_dep.name => dep.name} } }.compact.flatten.reduce(&:merge)

  unmet_deps = sources.non_default_explicit_sources.map do |source|
    (source.spec_names - pinned_spec_names).each do |indirect_dependency_name|
      requirements[indirect_dependency_name] = requirements[direct_dependency_parents[indirect_dependency_name]]
{% endhighlight %}

Without modifying bundler in crazy ways, the most straightforward way to do this is to take the list of explicit dependencies we're trying to configure a SourceMap for and map through them to create a hash of the sub_dependencies as keys, and their values being the gem that depends on them, so that we can establish the parent of that gem. The `requirements` variable is a hash list of our explicitly declared gems and what their source is, already made by the SourceMap class. So we leverage this with our new method of determing the parent gem for our current gem, and use that to determine the source.

This largely resolves our issue, as now any gems that `rails` depends on are sourced from `rubygem.org` despite `rails` being a dependency of our `work-logic` gem. However, now that we're here, we may go a step further. Our `work-logic` gem has some other dependencies that we don't actually want to be sourced from `https://internal-gem-repo.awesome` as these are other open source gems we don't have any good reason to mirror either. So I added an additional option to the `gem` command of the Gemfile dsl, `preffered_spec_source`. This required 3 particular changes.

First we add our preferred_spec_source as a variable to the Dependency object Bundler uses to track dependencies.

{% highlight ruby %}
  class Dependency < Gem::Dependency
    attr_reader :autorequire
    attr_reader :groups, :platforms, :gemfile, :git, :github, :branch, :ref, :preferred_spec_source    

    ...

    def initialize(name, version, options = {}, &blk)
      type = options["type"] || :runtime
      super(name, version, type)

      @autorequire    = nil
      @groups         = Array(options["group"] || :default).map(&:to_sym)
      @source         = options["source"]
      @preferred_spec_source = options["preferred_spec_source"]
      @git            = options["git"]
      @github         = options["github"]
      @branch         = options["branch"]
      @ref            = options["ref"]
      @platforms      = Array(options["platforms"])
      @env            = options["env"]
      @should_include = options.fetch("should_include", true)
      @gemfile        = options["gemfile"]

      @autorequire = Array(options["require"] || []) if options.key?("require")
    end
{% endhighlight %}

Then we add `preferred_spec_source` as a VALID_KEY for the `gem` command in the DSL.

{% highlight ruby %}
module Bundler
  class Dsl
    include RubyDsl

    def self.evaluate(gemfile, lockfile, unlock)
      builder = new
      builder.eval_gemfile(gemfile)
      builder.to_definition(lockfile, unlock)
    end

    VALID_PLATFORMS = Bundler::Dependency::PLATFORM_MAP.keys.freeze

    VALID_KEYS = %w[group groups git path glob name branch ref tag require submodules
                    platform platforms type source install_if gemfile preferred_spec_source].freeze
{% endhighlight %}

And then we need to modify the SourceMap implementation one last time so that, in the event the gem we're looking at has an explicit parent, we need to check if the explicit parent declared a `preferred_spec_source`, and use that instead of anything we might have determined elsewise. 
{% highlight ruby %}
    def all_requirements
      requirements = direct_requirements.dup
      direct_dependency_parents = @dependencies.map{|dep| dep.to_spec.dependencies.map{|sub_dep| {sub_dep.name => dep.name} } }.compact.flatten.reduce(&:merge)

      unmet_deps = sources.non_default_explicit_sources.map do |source|
        (source.spec_names - pinned_spec_names).each do |indirect_dependency_name|
          previous_source = requirements[indirect_dependency_name]
          if previous_source.nil?
            explicit_parent = @dependencies.find{ |dependency| dependency.name == direct_dependency_parents[indirect_dependency_name] } 
            requirements[indirect_dependency_name] =  if explicit_parent && explicit_parent.preferred_spec_source
              sources.rubygems_sources.find{|source| source.name == "rubygems repository #{explicit_parent.preferred_spec_source}"}
            else
              requirements[direct_dependency_parents[indirect_dependency_name]]
            end
          else
{% endhighlight %}

This particular implementation is probably flaky as it does no validation that your `preferred_spec_source` isn't garbage. (If it is, it seems to just ignore it.) With some refactoring and testing though,this would likely handle this particular situation robustly. I'd need to understand the bundler ecosystem far more than I do after my 2 day deep dive to do that though.

## Conclusion

Figuring out how this functionality worked without ever having looked at how Bundler works internally has been quite the adventure. I'd like to take a moment to thank [binding.pry](https://github.com/pry/pry) for being the best developer tool I could ever recommend, as figuring this out ultimately required me to pull the bundler source code and just start jumping into various locations during the execution to determine the origin of my undesired source mappings. I've definitely gained a massive appreciation for the sheer complexity of what it is Bundler does so that we can, under 99% of all cases, just pull a bunch of correctly versioned and platformed gems from the right place, and move on with our lives without trudging through [dependency hell](https://en.wikipedia.org/wiki/Dependency_hell). 

These fixes are likely very naive in some regards, and will probably need to be refactored and tested to truly be useable in the wild so modify bundler at your own risk. I might consider opening a PR for bundler with a request for some guidance on any potential issues these changes could cause and how to prevent them.