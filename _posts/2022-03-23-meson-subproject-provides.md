---
layout: post
title: Using Meson subproject [provide]s
date: 2022-03-23 16:10
categories: blog
tags: [openbmc, meson, subprojects]
---

# Specifying dependencies

I [previously introduced][1] how to find dependencies using meson subproject
wrapfiles, but I suggested using the verbose `fallback` argument to
`dependency`.  I also wrote:

> I think there is a method to add the `fallback` directives into the wrap
> file itself but I haven't played with that yet.

I've since learned how to do this and have been updating most of the repos.  It
greatly reduces the noise in `meson.build` dependency expression.

## Depending on a simple library

If the repository you're depending on is just providing a library, then the
`fallback` can be entirely removed.  Instead the wrapfile is the proper way to
specify the fallback information in a `[provide]` section.

As an example, phosphor-networkd had a [change][2] to the `meson.build` to
remove `fallback`:

```diff
- phosphor_logging_dep = dependency(
-   'phosphor-logging',
-   fallback: ['phosphor-logging', 'phosphor_logging_dep'])
+ phosphor_logging_dep = dependency('phosphor-logging')
```

and a change to `phosphor-logging.wrap` to add the same information:

```diff
+ [provide]
+ phosphor-logging = phosphor_logging_dep
```

In the `provide` statement, the left hand side is the dependency name and the
right hand side is the meson variable in the subproject holding the dependency.

## Exposing an executable

Sometimes a repository exposes an executable which is needed as part of a build
process such as for generating code.  The same phosphor-networkd [change][2] had
a fairly complex meson directive for finding some of these executables:

```meson
sdbusplus_dep = dependency('sdbusplus', required: false)
if sdbusplus_dep.found() and sdbusplus_dep.type_name() != 'internal'
  sdbusplusplus_prog = find_program('sdbus++', native: true)
  sdbuspp_gen_meson_prog = find_program('sdbus++-gen-meson', native: true)
else
  sdbusplus_proj = subproject('sdbusplus', required: true)
  sdbusplus_dep = sdbusplus_proj.get_variable('sdbusplus_dep')
  sdbusplusplus_prog = sdbusplus_proj.get_variable('sdbusplusplus_prog')
  sdbuspp_gen_meson_prog = sdbusplus_proj.get_variable('sdbuspp_gen_meson_prog')
endif
```

Instead these can be expressed in the `[provide]` section using the special
`program_names` directive and the simple native meson `find_program` function
can be used.

```ini
[provide]
sdbusplus = sdbusplus_dep
program_names = sdbus++, sdbus++-gen-meson
```

```meson
sdbusplus_dep = dependency('sdbusplus')
sdbusplusplus_prog = find_program('sdbus++', native: true)
sdbuspp_gen_meson_prog = find_program('sdbus++-gen-meson', native: true)
```

[1]: {% post_url 2021-04-18-meson-subprojects %}
[2]: https://github.com/openbmc/phosphor-networkd/commit/3397be3ca10310a264ee0ab9c5fb17add9e0307b
