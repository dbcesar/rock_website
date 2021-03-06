---
layout: documentation
title: Adding new packages
sort_info: 30
---

# Adding new Packages
{:.no_toc}

- TOC
{:toc}

  
As we already mentioned [in the introduction](index.html), within Rock, none of
the packages should actually rely on Autoproj, Rock's build system. C++ Rock
packages for instance rely on common build systems such as CMake or autotools
to build, and pkg-config or CMake mechanisms to handle cross-package
dependencies. Autoproj is only handling the scheduling of the configuration and
build of each the packages, so that build systems and environment variables are
set up as required, and dependencies are handled properly.

As such, autoproj mainly needs to know two things about a package:

- [what build system it uses](#autobuild). This is done in a Ruby DSL in a file that lies in
  the package set, with a `.autobuild` extension. One usually starts with a
  `packages.autobuild` file. The list of declarations in these autobuild files make up the list
  of package names Autoproj knows about.
- [how to download it](#version_control). This is done in YAML in the package
  set's `source.yml` file we created when we created the package set. When
  looking for a package's version control information. Autoproj allows to
  overlay version control information (for e.g. change the branch that is being
  checked out). As such, it follows [resolution
  rules](#version_control_resolution) to list all version control entries that
  match the package name, and compute the corresponding version control setup.

**Note** for simplicity, it is possible to add autobuild files and define
version control information straight into the main build configuration (the
`autoproj/` folder). The version control information in this case is stored in
the `overrides.yml` file instead of `source.yml`.
{: .tip}

## Creating a package

The first step in creating a package is [to pick a name](conventions.html#naming)

If you create a package from scratch, Rock provides a set of command-line tools
to generate package scaffolds for you:

- `rock-create-lib` for a [C++ library](../integrating_functionality/cpp_libraries.html)
- `rock-create-rubylib` for [Ruby libraries](../integrating_functionality/ruby_libraries.html)
- `rock-create-orogen` for [an oroGen component](../integrating_functionality/components.html)
- `rock-create-bundle` for [a bundle](../basics/getting_started.html)

If you are integrating a package that already exists, it should be easy enough
provided that the package uses widespread build systems and follows common
conventions (such as having a separate source and build folder, and having an
install step).

## Package Description and Dependencies {#manifest_xml}

For documentation reason, the packages are expected to provide metadata such as
a description of its purpose, the author(s), license, … Additionally, it is
common for packages to depend on each other, meaning that a package needs
another package to be there first, for it to build and/or run successfully. The
dependency may be another package declared in autoproj or a package provided by
the underlying operating system through [the osdep system we will see
later](os_dependencies.html).

All this information is stored in a XML file whose format follows. If the
package has been created for Rock specifically, it is saved as `manifest.xml`
file directly at the root of the package. For packages that already exist but
are being integrated in Rock, the file should be saved in the package set under
`manifests/package/name.xml` (e.g. `simulation/gazebo`'s manifest is saved in
`manifests/simulation/gazebo.xml`).  {:

~~~xml
<?xml version="1.0"?>

<package>
  <description brief="one line of text">
    long description goes here, 
    <em>XHTML is allowed</em>
  </description>
  <author>Alice/alice@somewhere.bar</author>
  <author>Bob/bob@nowhere.foo</author>
  <license>BSD</license>
  <url>http://rock-robotics.org/</url>

  <depend package="pkgname"/>
  <depend package="common"/>
  <!-- depend can handle both source and osdep packages -->
  <depend package="ruby-dev" />
  <!-- a dependency that will only be installed for testing -->
  <test_depend package="minitest" />
  <!-- a dependency that can be ignored if it is not available -->
  <depend_optional package="gui/vizkit3d" />
</package>
~~~

The `<test_depend …>` tag is used for dependencies that are specific to [the
package's test suite](../basics/day_to_day.html#test). The `<depend_optional
…>` allows to [avoid building some dependencies within some builds](managing.html).


## Declaring a package {#autobuild}

A package declaration has two functions: to tell autoproj what packages exist,
and to tell it how it is meant to be configured, built and installed. These
definitions are done in a Ruby DSL, within files with the `.autobuild` extension
in the package set or in the main build configuration. While the file names can
be arbitrary, one usually would start with `packages.autobuild`.

What we will see in this section is what type of packages exist, how each of
them are declared and the principal build configuration options.

### CMake packages {#cmake}

A CMake package is defined with

~~~ ruby
cmake_package "package_name"
~~~

More complex tweaking is achieved with

~~~ ruby
cmake_package "package_name" do |pkg|
  [modify the pkg object]
end
~~~

In particular, CMake build options are given with

~~~ ruby
cmake_package "package_name" do |pkg|
  pkg.define "VAR", "VALUE"
end
~~~

The above snippet being equivalent to calling `cmake -DVAR=VALUE`

### Autotools packages {#autotools}

~~~ ruby
autotools_package "package_name"
autotools_package "package_name" do |pkg|
    pkg.configureflags << "--enable-feature" << "VAR=VALUE"
    # additional configuration
end
~~~

Since autotools (and specifically, automake) environments are unfortunately
not so reusable, autoproj tries to regenerate the autotools scripts forcefully.
This can be disabled by setting the some flags on the package:

 * using\[:aclocal]: set to false if aclocal should not be run
 * using\[:autoconf]: set to false if the configure script should not be touched
 * using\[:automake]: set to false if the automake part of the configuration
   should not be touched
 * using\[:libtool]: set to false if the libtool part of the configuration should
   not be touched

For instance, one would add

~~~ ruby
autotools_package "package_name" do |pkg|
    pkg.configureflags << "--enable-feature" << "VAR=VALUE"
    # Do regenerate the autoconf part, but no the automake part
    pkg.using[:automake] = false
end
~~~

### Ruby packages

~~~ ruby
ruby_package "package_name"
ruby_package "package_name" do |pkg|
    # additional configuration
end
~~~

This package handles pure ruby libraries that do not need to be installed at
all. Autoproj assumes that the directory layout of the package follows the following
convention:

 * programs are in bin/
 * the library itself is in lib/

If a Rakefile is present in the root of the source package, its `default`
task will be called during the build. The `rake_setup_task` overrides this default
setting, and to `nil` to avoid running a setup task at all.

~~~ ruby
ruby_package "package_name" do |pkg|
    pkg.rake_setup_task = "setup"
end
~~~

### oroGen packages {#orogen}

~~~ ruby
orogen_package "package_name"
orogen_package "package_name" do |pkg|
    # customization code
end
~~~

oroGen packages are the packages where Rock components are implemented. The
oroGen handler runs the code generation, configuration and build/install steps.

### Simply checking out packages

The importer package type only checks it out, but does not perform any additional steps.

~~~ruby
import_package 'package_name'
~~~

### Custom package building

Post-processing steps can be performed after the checkout using `post_install`:

~~~ ruby
import_package "package_name" do |pkg|
    pkg.post_install do
        # add commands to build and install the package
    end
end
~~~

## Version Control Resolution {#version_control_resolution}

The general format of version control entries is:

~~~ yaml
package_name:
  type: IMPORTER_TYPE
  url: IMPORTER_URL
  <importer specific control options>
~~~

The package set that defines a package must have at least one matching entry in
the `version_control` section of its `source.yml` file. 

~~~yaml
version_control:
  - package_name:
    type: ...
~~~

Follow-up package sets (following the order of the build configuration's
manifest) might also have matching entries in their `source.yml` files, in the
`overrides:` section. These `overrides` apply only to packages from other
package sets. Only entries in `version_control` apply to the packages from the
same package set.

~~~yaml
overrides:
  - package_name:
    type: ...
~~~

Additional overrides may be specified in files in the `autoproj/overrides.d/`
folder. The YAML files in this folder do not specify sections:

~~~yaml
- package_name:
  type: ...
~~~

Once all matching entries are found, Autoproj resolves the final configuration by:

- if the `type` field is not specified, updating existing settings
- if the `type` field is set, clearing existing settings and using the new ones.

At any time, matching entries as well as the resolved version control
configuration for a package can be displayed with `autoproj show`.

For instance, `autoproj show tools/roby` in the rock-gazebo build configuration
as of today gives:

~~~
source definition
  type: git
  url: https://github.com/rock-core/tools-syskit.git
  branch: syskit2-test-refactoring
  push_to: git@github.com:/rock-core/tools-syskit.git
  repository_id: github:/rock-core/tools-syskit.git
  retry_count: 10
  first match: in rock.core (…/autoproj/remotes/rock.core/source.yml)
    branch: $ROCK_BRANCH
    github: rock-core/tools-$PACKAGE_BASENAME
  overriden in main configuration (…/autoproj/overrides.d/01-syskit2.yml)
    branch: syskit2-test-refactoring
~~~

The `$VARIABLE` syntax will be clarified [later](managing.html#configuration_options).  We will now see what version control
systems are supported by autoproj.

## Available Version Control Systems {#version_control}

The package name may contain regular expression characters, in which case the
package name will be used to match against the package name(s).

Note that in all cases, the operations *must* succeed without any password -
the import will fail if a password is needed.
{: .callout .callout-warning}

### Git

The general setup for git imports is:

~~~ yaml
package_name:
  type: git
  url: repository_url_or_path
  push_to: repository_url_or_path
  branch: branch_name
  tag: tag_name # it is branch OR tag
  commit: commit_id # it is tag OR commit ID
  with_submodules: false # true to checkout and update submodules
~~~

Autoproj will maintain an 'autobuild' remote on the checked out repository:
i.e., it will make sure that the URL of this remote is always linked to the
right URL from the config file, and will update the relevant branch on update
(beware: the other branches won't get updated).

Moreover, autoproj will make sure that updating your local repository always
resolves as a fast-forward. An update that would require a merge will fail.

### GitHub

Autoproj allows to define shortcut handlers, that is to define custom entries
that are expanded into full version control configurations. The
`autoproj/git_server_configuration`, which is required by the Rock build
configuration, defines one for github.

This means that one may use the `github:` handler to check a package out from
github, rather than defining the `type` and `url` manually. This has the benefit
of setting the `push_to` automatically to the SSH URL (by default) while the plain
URL is using https (usually faster).

~~~yaml
package_name:
  github: rock-core/buildconf
  … rest of options same as git …
~~~

### Tar archives

~~~ yaml
package_name:
  type: archive
  url: http://sourceforge.net/blablabla.tar.gz?option=value
  filename: blabla.tar.gz # Optional: if the file name cannot be inferred from the URL
  no_subdirectory: false # Optional. Set to true if there is not a leading directory in the archive
  update_cached_file: false # Optional. Set to false to disable automatic updates
~~~

The importer expects that there is a leading subdirectory in the archive, under
which all files. If that is not the case, i.e. if all files are in the root of
the archive, do not forget to set the no\_subdirectory option.

Autoproj tries to guess what is the name of the downloaded file by extracting it
out of the URL. Sometimes, this does not work as the URL does not fit the
expected scheme -- in these cases you will get a tar error on update. To
override this autodetection behaviour, set the filename option to the actual
downloaded file name.

By default, autoproj will check if the downloaded file has been updated on the
server, and will download it again if it is the case. If you are downloading
release tarballs, this is unneeded as the archive should not be updated. In that
case, set the update\_cached\_file option to false to save the time needed to
check for the update (can be long on, for instance, sourceforge). The source
will of course be updated if you change the URL (i.e. to download a new release
of the same software).

### Subversion

The general setup for subversion imports is:

~~~ yaml
package_name:
  type: svn
  url: repository_url_or_path
~~~

### CVS

The general setup for CVS imports is:

~~~ yaml
package_name:
  type: cvs
  url: cvs_root
  module: modulename
~~~

In case a :pserver: is used, it must be quoted - YAML would interpret it
the wrong way otherwise:

~~~ yaml
package_name:
  type: cvs
  url: ":pserver:cvs@blabla:/"
  module: modulename
~~~

### Patching after checkout or update {#patch}

It is possible to apply patches after a given package (imported by any of the
importer types) has been checked out/updated. To do so, simply add the option
`patches:` to the importer configuration and list the patches which should be
applied:

~~~ yaml
package_name:
  type: archive
  url: http://sourceforge.net/blablabla
  patches:
    - $AUTOPROJ_SOURCE_DIR/blablabla-01.patch
    - $AUTOPROJ_SOURCE_DIR/blablabla-02.patch
~~~

Note that in the example above, the patch is saved in the package set's folder
(the value of AUTOPROJ_SOURCE_DIR). This is a highly recommended practice.

The provided patches are by default applied with a patch level of 0 (passed to
patch through the -p option). This can be overriden on a patch-per-patch basis
by making the patch name an array as well:

~~~ yaml
package_name:
  type: archive
  url: http://sourceforge.net/blablabla
  patches:
    - [$AUTOPROJ_SOURCE_DIR/blablabla-01.patch, 1]
    - $AUTOPROJ_SOURCE_DIR/blablabla-02.patch
~~~

**Next**: let's get into [using packages from the underlying
OS or from language-specific handlers](os_dependencies.html)
{: .next-page}

