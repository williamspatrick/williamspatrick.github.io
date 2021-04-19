---
layout: post
title: Using Meson subprojects with OpenBMC
date: 2021-04-18 21:20
categories: blog
tags: [openbmc, meson, subprojects]
---

# Motivation

For OpenBMC code repositories, we have a number of different ways you can use
to compile the code:

1. Using `bitbake` to build the recipe and/or a complete flash image.
1. Using `devtool` inside an `oe-init-build-env`.
1. Using the bitbake SDK from `bitbake <image> -c populate_sdk`.
1. Using the openbmc [`run-unit-test-docker`][1] script, which uses Docker to
  build all the required dependencies and launch the same unit-test
  framework used by Jenkins (maybe with my [launch aid][2]).

In the last few days I've been working to ensure that projects I maintain can
also be compiled entirely isolated, using basic Linux distro packages and Meson
subprojects to fill in everything else. You might wonder why we would support
yet another build workflow.  The primary reason is that for speed in both my
"start the day" and "develop / compile / test" operations.
<!--more-->

The first 3 workflows are all low-support effort for the project as they are
provided by the upstream Yocto support and we have to maintain our recipes
anyhow to put them into an image.  For these workflows either the
"start the day" (3) or "DCT" (1,2) operations can be a big time sink.

Similarly, the 4th workflow is slow.  If any of the base packages have changed
since you last ran the unit-test framework, it will rebuild a number of Docker
packages from those sources, which costs 5-10 minutes even on a relatively
fast desktop (I did spend a bunch of time in 2021Q1 greatly improving the
caching of those Docker containers, so it is no longer ~30 minutes).  If the
Docker image doesn't need to be rebuilt it still takes another 5 minutes to
run through all of the testing that framework provides for even a relatively
simple repository like [openbmc/sdbusplus][3].  To be fair, this script is
doing a number of different compile types with unit-tests, code coverage, and
static analysis and automated code formatting, but it just isn't something I
need most of the time when I just want to know if the change I made compiles.

Conversely, using a pure meson workflow I can build and unit-test the same
repository starting from nothing in exactly 20s (including one test that
takes 18.02s to run)!  I'm currently in the process of converting
[openbmc/phosphor-logging][4] to meson, from a very complex Autotools setup,
and it takes 21 seconds to build from scratch and run the unit-tests while
picking up 11 subprojects.

``` sh
rm -rf builddir
meson builddir
meson compile -C builddir
meson test -C builddir
```

# Getting Started

Enabling meson subprojects, assuming all your dependencies are well set up for
it, is pretty easy.  You create a simple `subprojects/<project>.wrap` file to
correspond to any `dependency('<project')` in your `meson.build` and make a
minor change to the `dependency` directive.

``` ini
[wrap-git]
url = https://github.com/openbmc/sdbusplus.git
revision = HEAD
```

``` diff
- dependency('sdbusplus')
+ dependency('sdbusplus', fallback: ['sdbusplus', 'sdbusplus_dep'])
```

_I think there is a method to add the `fallback` directives into the wrap
 file itself but I haven't played with that yet._

This [pending commit][5] creates and enables the wrap files for the
[openbmc/phosphor-virtual-sensor][6] repository as an example.

# Workflow Enhancement

The first time you build the meson project with the wrap files, it will
download all of the source into the repository `subprojects` directory along
side the wrap files.  This is problematic for two reasons:

1. You probably already have a copy of many of the openbmc repositories on
   your system and you don't need yet another one.
2. The subproject copies don't update automatically unless you run `meson
   subprojects update` so your subprojects can get out of date\*.

(\*) I'm assuming you have a common subdirectory of many openbmc code repos and
some [script][7] to keep them updated.

What I do to combat this is to create a symlink from my 'common' repo location
into the `subprojects` directory for each wrap file prior to running `meson`
for the first time.

``` sh
cd subprojects; for w in *.wrap; do ln -s ../../$(basename -s .wrap $w) .; done
```

# Potential Issues

In trying to import some dependencies using this subproject support, I've ran
into a few issues.  These are things that need to be fixed in the upstream
repository (and I've been contributing fixes as I've ran into them).

## Missing or poorly named library dependency

Typically when you provide a library, you provide a `pkgconfig` for it so that
other code using your library knows how to compile and link against it.  When
using a library as a subproject, Meson doesn't use the `pkgconfig`, so the
library also needs to provide a Meson `declare_dependency` in a variable that
the using-project can use.  An example `declare_dependency` from
[openbmc/phosphor-dbus-interfaces][8] is:

``` meson
phosphor_dbus_interfaces_dep = declare_dependency(
    include_directories: include_directories('gen'),
    link_with: libphosphor_dbus,
    dependencies: sdbusplus_dep,
    variables: ['yamldir=' + meson.project_source_root()],
)
```

The Meson team has set [conventions][9] for these dependency variables to be
`<project_name>_dep` and I've found that in many cases the project has an
incorrect name.  This means you have to look up in the project's `meson.build`
for the correct variable name to use.  I've been changes back, as I find them,
so that the upstream project follows these Meson conventions.

## Different header layout from install layout

If you have code that is already using a dependency, it likely has an `#include`
directive somewhere to get a header file from that dependency.  Often these
headers are installed somewhere like `/usr/include/<project>/<header>`.  What
makes a project very difficult to use as a Meson subproject is if the in-repo
layout of their header files do not match the install layout.  By this I mean
if the repository has a header in `<src_root>/<subdir>/<header>` where
`<subdir>` is not `<project>`.  This would require your code to have a
different `#include` directive depending on if the header was used from the
system or from the subproject location.  These should be fixed in the
dependency's repository such as by rearranging the source tree or providing
[symlinks][10].

## Using global meson functions rather than project functions.

Meson has two sets of functions `global_*` and `project_*` and often if a
repository has never been used as a sub-project before it will be using the
`global_*` variety because it makes no difference in that case.  When the
repository is used as a sub-project, the `global_*` functions modify behavior
for every other project and sub-project being built which is almost always
wrong.  Meson itself flags this behavior in many cases to let you know it is
happening and needs to be fixed.

I have ran into this with [`source_root`][11] (which is now split into
`global_source_root` and `project_source_root` starting in Meson 0.56) and
[`global_arguments`][12] (which should be `project_arguments`).

[1]: https://github.com/openbmc/openbmc-build-scripts/blob/17c69eda1542e82c537268ad2a4ce84caf569938/run-unit-test-docker.sh
[2]: https://github.com/williamspatrick/dotfiles/blob/4d7a33b34325f62a0f1ea5ff8a0ff108e3f4813d/env/30_linux/lfopenbmc.zsh#L65
[3]: https://github.com/openbmc/sdbusplus
[4]: https://github.com/openbmc/phosphor-logging
[5]: https://gerrit.openbmc-project.xyz/c/openbmc/phosphor-virtual-sensor/+/42349
[6]: https://github.com/openbmc/phosphor-virtual-sensor
[7]: https://github.com/williamspatrick/dotfiles/blob/4d7a33b34325f62a0f1ea5ff8a0ff108e3f4813d/bin/20_all/obmcsrc-update
[8]: https://github.com/openbmc/phosphor-dbus-interfaces
[9]: https://mesonbuild.com/Subprojects.html#naming-convention-for-dependency-variables
[10]: https://gerrit.openbmc-project.xyz/c/openbmc/pldm/+/42403/1
[11]: https://github.com/openbmc/sdbusplus/commit/d9bb33e26b109881afb04c8aefde7ed170814c2d
[12]: https://gerrit.openbmc-project.xyz/c/openbmc/pldm/+/42404/1
