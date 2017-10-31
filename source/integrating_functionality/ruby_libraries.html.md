---
layout: documentation
title: Ruby Library Packages
sort_info: 15
---

# Ruby Library Packages
{:.no_toc}

- TOC
{:toc}

On the C++ side, we have already discussed the separation between the bulk of
the implementation in C++ libraries and their framework integration in
components.

The same can be applied on the Ruby side of the system. A lot of the
functionality needed to e.g. look at diagnostics data streams from the
components can be made framework-independent (i.e. Syskit independent). It is
somehow weaker in the Ruby case though, as in the end most of the data
processing is done in C++ in a Rock system.

Generally speaking, **do not do too many things within the Syskit process**.
Syskit is in charge of coordinating the system's components, and having long
computations within the Syskit process will increase Syskit's reaction times.

## Conventions

The Ruby packages are expected to follow the de-factor standard Ruby package
layout set forth by RubyGems. The best way to create a new ruby package is to
use [`bundle gem`](https://bundler.io/v1.15/guides/creating_gem.html) and add
Rock's [`manifest.xml`](../workspace/add_packages.html) to it.

`autoproj` runs `rake` within the package during the build, which means that
it runs the `Rakefile`'s `default` task.

External gems can be installed by autoproj using [the osdeps
mechanism](../workspace/os_dependencies.html). For now, autoproj does not know
how to look at a package's gemspec (which defines the gem's dependencies), so
you will have to duplicate dependencies between the gemspec and the
`manifest.xml`.

## Tests

This is 2017 (or later). Testing is now an integral part of modern development
process, and Rock provides support to integrate unit testing in the development
workflow.

Ruby packages are expected to provide a `test` target in their `Rakefile` to run
the tests. The `Rakefile` generated by `bundle gem` has one.

## Type System Plugins

In order to smooth the interface between the Rock type system and the Ruby
layers, one [can provide a
`typelib_plugin.rb`](../type_system/types_in_ruby.html) file within a Ruby
package. Just put the file under your package's main directory (e.g.
`lib/mypackage/typelib_plugin.rb`) to have it automatically loaded.

Let's go now to the Rock specific part of functionality integration, [components](components.html)
{: .next-page}