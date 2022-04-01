---
layout: post
title: "sdbusplus: object_t constructor signals"
date: 2022-04-01 15:46
categories: blog
tags: [openbmc, sdbusplus, dbus, signals]
---

# TL;DR

The [`sdbusplus::server::object_t`][ctor] constructor that takes a
`bool deferSignal` is going away.

```diff
-    object(bus_t& bus, const char* path, bool deferSignal) :
-        object(bus, path,
-               deferSignal ? action::defer_emit : action::emit_object_added)
-    {
-        // Delegate to default ctor
-    }
```

The `bool deferSignal` argument should be replaced with one of the `action`
enums:

```cpp
    enum class action
    {
        /** sd_bus_emit_object_{added, removed} */
        emit_object_added,
        /** sd_bus_emit_interfaces_{added, removed} */
        emit_interface_added,
        /** no automatic added signal, but sd_bus_emit_object_removed on
         *  destruct */
        defer_emit,
        /** no interface signals */
        emit_no_signals,
    };
```

If you were previously using `deferSignal = true` you likely want either
`defer_emit` or `emit_no_signals` but you need to read on to know which one.

[ctor]: https://github.com/openbmc/sdbusplus/blob/4b0d12687aedd528126932bbe6275173ca6cb99e/include/sdbusplus/server/object.hpp#L159

# Background

## Missing InterfaceRemoved signals?
[Lei YU][LeiYU] recently posted to the mailing list about an [issue][mlpost] he
was observing with phosphor-logging where log entries did not always send the
[InterfacesRemoved][signals] signals.  After some discussion between a few of
us in the review of a simple fix Lei YU proposed we realized there is a
fundamental issue in the `sdbusplus::server::object_t` API with respect to these
signals.

[LeiYU]: https://github.com/mine260309
[mlpost]: https://lore.kernel.org/openbmc/CAGm54UHMED4Np0MThLfp4H-i8R24o8pCns2-6MEzy1Me-9XJmA@mail.gmail.com
[signals]: https://dbus.freedesktop.org/doc/dbus-specification.html#standard-interfaces-objectmanager

## sd_bus APIs

At a DBus level there are two signals used to indicate that an object has shown
up on or been removed from the bus, `InterfacesAdded` and `InterfacesRemoved`,
both of which are from the `org.freedesktop.DBus.ObjectManager` interface.  The
systemd sd-bus API provides the following functions:

- `sd_bus_emit_interfaces_added`
- `sd_bus_emit_interfaces_removed`
- `sd_bus_emit_object_added`
- `sd_bus_emit_object_removed`

The interface-level functions take a list of interfaces and create the
corresponding `Interfaces{Added,Removed}` signal.  The object-level functions
are simply helper functions which create the signals for *all* interfaces the
sd-bus `ObjectManager` is aware of having resided at that object-path.

## sdbusplus::server::object_t

sdbusplus has two relevant classes here: `server::object_t` and
`server::interface_t`.  Typically the `interface_t` is only used by the
`sdbus++` generated server bindings and it provides constructor/destructor
hooks to register the interface with the `ObjectManager`, but it also has
`emit_added` and `emit_removed` functions which can be used to explicitly
emit the `Interfaces{Added,Removed}` signals.  These functions are rarely used.
The `object_t` class is much more widely used and what it does is stitch
together multiple generated server classes into a single C++ object.  Often
you will observe an override class such as:

```cpp
class LocalObject : public object_t<InterfaceA, InterfaceB>
{
   // ...

   void methodFromInterfaceA(...) override // method call
   {
       // ...
   }

   size_t propertyFromInterfaceB() override // property get
   {
      // ...
   }
}
```

Originally, the `object_t` class would automatically send the
`Interface{Added,Removed}` signals, and it still does so by default, but a
parameter was added to control the signal behavior.  First the parameter was a
boolean (`deferSignals`) and later it was changed to an enumeration (`action`)
but the boolean form was preserved for backwards compatibility with old code.
It is around this attempted automatic signal behavior that the issue Lei YU
observed stems.

# Object Lifecycles

## Poorly Performing Simple Object

The simplest object life-cycle is that you create an object and it already has
its properties set to a valid initial value, so you want to immediately
advertise it on the dbus and you want it to report its removal when you destruct
it.  This is what `object_t` does by default, with the
`action::emit_object_added`, but it is almost never what you actually want.  Why?

* Rarely are the default property values correct for your object.  So by using
  this you've advertised the object before it is fully initialized.  You are
  either sending a bunch of spurious `PropertiesChanged` signals immediately
  or you lied in the property values of the `InterfacesAdded` signal, or both.
  This reduces the utility of the `InterfacesAdded` signal and/or causes
  performance issues by a flood of `PropertiesChanged` signals.

* On daemon start-up, you shouldn't have claimed a bus-name until all your
  initial state objects are created, so these signals are uselessly sent from
  an anonymous DBus connection (`:1:123` style).

If you are using the default constructor (or explicitly adding
`action::emit_object_added`) your application probably isn't being a good DBus
citizen but it is likely working as-is and other than the lies told in the
`InterfacesAdded` no one is likely to notice.  At a future date I will
probably remove the `default=action::emit_object_added`.

## Proper Simple Dynamic Object

For most use-cases the proper behavior for a simple dynamically created object
is to use the `action::defer_emit` on object construction, set up all the
properties using the `skipSignal=true` property-set overrides, and then call
`this->emit_object_added()`.  This prohibits the initial automatic
`InterfaceAdded` signal, disables all the spurious `PropertiesChanged` signals,
sends a nice (and correct) `InterfacesAdded` signal with the correct initial
property state of your object, and lastly automatically sends
`InterfacesRemoved` when you decide to delete the object.  Perfect!

Except, you'll notice I used the word "dynamic" here.  This means your daemon
has been up and running and you have already claimed a bus-name.  What about the
objects your daemon is creating on start up (or maybe restoring from persistent
storage as in the case of `phosphor-logging`)?  It is a little more complicated
and is the use-case of the issue.  But, first, let's talk about a few other use
cases.

## Permanent Sub-Objects

In the `phosphor-hwmon` repository I noticed an implementation pattern where
a `object_t<Sensor::Value>` is created to hold the primary information about
the Sensor.  In some cases, depending on additional data, something like a
`std::unique_ptr<object_t<Sensor::Threshold::Critical>>` is created at the same
object path.  This critical threshold object I am calling a "permanent
sub-object" because the lifetime of it matches that of the primary object (the
`Sensor::Value` in this case).

The best way to handle this pattern is pretty similar to the Simple Dynamic
Object.  In between setting the primary object's properties and calling
`emit_object_added()` we create any sub-objects and set their properties.  Then,
the primary object's `emit_object_added()` will *include* all the interfaces for
all the sub-objects.  Great!

There is still the question of how do we set the `action` parameter for the
sub-objects.  The answer now is `action::emit_no_signals`.  We *never* want
the sub-object to deal with its own lifetime signals!  If we did they'd also
be the lifetime signals for the parent object because they are all residing at
the same dbus object-path (Uh-oh!).

## Temporary Sub-Objects

Another pattern I've seen is a primary object which contains mostly static
data, but sometimes creates a temporary `object_t<Progress>` sub-object.  This
is temporary in the sense that the lifetime of the `Progress` is shorter than
the primary object; once the operation is done, the `Progress` is eventually
deleted.

In this pattern, on lifetime beginning and end of the `Progress` interface we
*only* want the `Progress` interface included in the `Interfaces{Added,Removed}`
signals.  The way to handle this is to use `action::emit_interfaces_added`.
This will ensure that `sd_bus_emit_interfaces_*` functions are used instead of
`sd_bus_emit_object_*` functions.  When the `Progress` object is destructed it
will similar only send a `InterfacesRemoved` containing itself.

## Start-up Objects

At the beginning I mentioned one of the flaws of using
`action::emit_object_added` was:

> On daemon start-up, you shouldn’t have claimed a bus-name until all your
> initial state objects are created, so these signals are uselessly sent from
> an anonymous DBus connection (:1:123 style).

So, there is still an issue of what to do on application start-up where you have
a set of initial objects.

(You don't want to grab the bus-name early because then `mapper` is aware of
your application and you then need to send all the `InterfacesAdded` signals,
which slows everything down.)

The ideal sequence is something like:

1. Create objects without sending any signals.
2. Claim bus-name.
3. Process forever...
    1. Send `InterfacesRemoved` on original objects if they are ever destructed.

We can get away with (1) because `mapper` listens for
[`org.freedesktop.DBus.NameOwnerChanged`][noc] signals and then queries the
daemon for everything it has on the dbus.

The problem is getting an implementation for (3.1) and the source of the
original issue.  In `phosphor-logging`, the `defer_emit` was set but
[`emit_object_added`][eoa] was never called, so the [destructor][dtor] didn't
use to do anything.  There was actually no way to specify the behavior from
(1) + (3.1)!

[noc]: https://dbus.freedesktop.org/doc/dbus-specification.html#bus-messages-name-owner-changed
[eoa]: https://github.com/openbmc/sdbusplus/blob/4b0d12687aedd528126932bbe6275173ca6cb99e/include/sdbusplus/server/object.hpp#L232
[dtor]: https://github.com/openbmc/sdbusplus/blob/4b0d12687aedd528126932bbe6275173ca6cb99e/include/sdbusplus/server/object.hpp#L216

# Solution

We use to have applications using `action::defer_emit` to attempt to do both
the "Permanent Sub-Object" and "Start-up Object" patterns and we had no way
to differentiate them.  Many applications are still using the `boolean
deferSignal = true` constructor parameter, which is the same as `defer_emit`.
We need some way to be able to deduce which pattern desired so we are going to
change the API to be explicit.

* The action `emit_no_signals` is being added, which should be used for the
  "Permanent Sub-Object" pattern to indicate "this C++ object should never
  deal with its own signals; they are covered elsewhere."

* The action `defer_emit` is being changed in the destruct path to *always*
  send `InterfacesRemoved` signals.  This covers the "Start-up" problem and
  all uses of `defer_emit` in the codebase already desired this behavior.
  `defer_emit` now means "I don't want the `InterfacesAdded` signal now and
  maybe I'll send them later, but I definitely want `InterfacesRemoved`".

* The `boolean deferSignal` constructor overload is being deleted.
    - All applications will need to be fixed up to be explicit in `action`.
    - This gives us an avenue to review all uses of the `deferSignal = true`
      pattern and fix them before we change `defer_emit`'s behavior.

I have implemented the `sdbusplus` changes for this:
- Add [`emit_no_signals`][ens] **[merged]**
- Change [`defer_emit`][ded] destructor behavior
- Remove `boolean` [constructor][rbc]

I have reviewed all cases of `defer_emit` in the openbmc organization and
they are all compatible with the destructor change.

I am working through all repositories pulled into the CI-tested machines and
ensuring they compile with the "Remove `boolean` constructor" change.  If you
are a maintainer and see a commit titled "sdbusplus: object: don't use 'bool'
argument constructor" this is the proposed fix.  The important thing to do here
is to review the additions of `defer_emit` or `emit_no_signals` and ensure the
code aligns with one of the earlier patterns (and I've chosen the right one).

[ens]: https://gerrit.openbmc-project.xyz/c/openbmc/sdbusplus/+/52501
[ded]: https://gerrit.openbmc-project.xyz/c/openbmc/sdbusplus/+/52495
[rbc]: https://gerrit.openbmc-project.xyz/c/openbmc/sdbusplus/+/52496
