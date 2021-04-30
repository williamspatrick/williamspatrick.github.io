---
layout: post
title: phosphor-logging now using Meson.
date: 2021-04-30 09:00
categories: blog
tags: [openbmc, meson, phosphor-logging]
---

As of [this commit][1], [phosphor-logging][2] is now enabled to be built with
meson instead of autotools and it can be imported as a [meson-subproject][3].
This means that now all the fundamental libraries used in OpenBMC are
available as meson subprojects, so you should be able to structure any
meson-based openbmc repository such that it can compile easily on any typical
Linux development system outside of an OE-SDK.

I plan to work today on getting the corresponding Yocto recipes updated as well.

[1]: https://github.com/openbmc/phosphor-logging/commit/f06056bc92258c8f578f3cb4cc5edc84443ca321
[2]: https://github.com/openbmc/phosphor-logging
[3]: http://www.stwcx.xyz/blog/2021/04/18/meson-subprojects.html
