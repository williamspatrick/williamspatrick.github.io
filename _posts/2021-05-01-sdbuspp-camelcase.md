---
layout: post
title: "sdbus++: Irritating camelCase from acronyms"
date: 2021-05-01 07:15
categories: blog
tags: [openbmc, sdbusplus, sdbus++, sdbuspp, camelcase]
---

The [sdbus++ generator][1] we use to generate server bindings from
[phosphor-dbus-interfaces][2] has long had an irritating behavior
around how it tries to convert property and method names from YAML
to generated C++ function names.  It had been using a trivial call
to [`inflection.camelize`][3] to create a [`lowerCamelCase`][4] version of
what it found in the YAML.  The problem with this is that many properties,
especially in the networking space, are acronyms.  The `camelize` function
would turn a property like `MACAddress` into an awkward `mACAddress`.

Five years later, I'm getting around to [fixing this behavior][5].  Going
forward the generator will handle acronyms in, what I think is, a reasonable
way.  Some examples:

* `MACAddress` becomes `macAddress`.
* `BMC` becomes `bmc`.
* `IPv6Address` becomes `ipv6Address`.

Generally speaking what the code does is:
1. First turn the name into `UpperCamelCase` using Inflection.
2. Identify if the multiple letters at the beginning are upper case: an
   acronym!
3. Turn everything except the last upper case letter at the beginning to lower
   case.
    - But, there is a special case to handle 'IPv6'.  Any set of upper case
      letters that has a "v" for "version" followed by a number (ie.
      `[A-Z]+v[0-9]+`) is treated as a full acronym.

There are probably a few cases that this doesn't cover in the most ideal way,
but I didn't find any in the current [phosphor-dbus-interfaces][2].  One that
comes to mind is a sequence of multiple acronyms like `DHCPDNSServers` would
become `dhcpdnsServers`, but without a list of known acronyms it would be hard
to figure out `DHCP` and `DNS` are two different acronyms.

There is already quite a bit of code that uses the awkward acronyms when
instantiating instances of generated classes, so I had to define a way to
independently migrate the `sdbus++` code from the instantiating repositories.
What I've done is define a preprocessor constant [`SDBUSPP_NEW_CAMELCASE`][6]
which can be used as a key to use the "new format".

My plan to get this integrate is as follows:

1. Push up to Gerrit the change to sdbus++ for review. (**done**)
2. Find all occurrences I can of old-style acronyms in the openbmc codebase
   and push up fixes utilizing the `SDBUSPP_NEW_CAMELCASE` key.  These
   will be under the Gerrit topic [`sdbuspp_camelcase`][7]. (**done**)
3. After #2's are all merged, make a test commit to [openbmc/openbmc][8] of the
   `sdbus++` changes to ensure I haven't missed something in step 2, and make
   fixes as necessary.  (**done 2021-05-12**)
4. Remove the `#define SDBUSPP_NEW_CAMELCASE` from sdbus++ after #3 is merged.
5. Clean up the `#else` side of the `#ifdef`'s in #2 so that the old-style
   acronyms are no longer present.

This work should happen essentially as fast as reviews from #2 can be done,
so any help to make that speed along would be greatly appreciated!

[1]: https://github.com/openbmc/sdbusplus/blob/master/tools/sdbus%2B%2B
[2]: https://github.com/openbmc/phosphor-dbus-interfaces
[3]: https://inflection.readthedocs.io/en/latest/#inflection.camelize
[4]: https://github.com/openbmc/sdbusplus/blob/ce8a467cade53b2cc88c86b9427a80ea7e526a78/tools/sdbusplus/namedelement.py#L11
[5]: https://gerrit.openbmc-project.xyz/c/openbmc/sdbusplus/+/42823
[6]: https://gerrit.openbmc-project.xyz/c/openbmc/sdbusplus/+/42823/3/tools/sdbusplus/templates/interface.server.hpp.mako#21
[7]: https://gerrit.openbmc-project.xyz/q/topic:sdbuspp_camelcase
[8]: https://github.com/openbmc/openbmc
