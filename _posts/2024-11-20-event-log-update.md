---
layout: post
title: Modern event logs in OpenBMC
date: 2024-11-20T16:20-05:00
categories: blog
tags: [openbmc, events, sdbusplus, phosphor-logging]
---

# Current Status

I've been working on implementations of revamped [event log design][design] and
wanted to give an update as to the current implementation progress.

Currently, the definition of the YAML format and event logging has been
implemented through `sdbusplus`, `phosphor-dbus-interfaces`, and
`phosphor-logging`; new errors can be defined in PDI and phosphor-logging can be
called to record them. There is also a handy CLI tool written that allows users
to mock events.

The following items are to be worked in the near future:

- Allow-list / Block-list support in `phosphor-logging`.
- Changing `AdditionalData` to be a dictionary.
- Generation of the Redfish Message Registry.
- Translation in `bmcweb` of `phosphor-logging` events to proper Redfish
  `LogEntry` format, including populating `MessageArgs`.
- Adding 'Hint' support in `Logging.Create`.

[design]: https://github.com/openbmc/docs/blob/master/designs/event-logging.md

# Defining a new Event

Defining a new event requires writing an `events.yaml` file in PDI. Two decently
featured examples are currently available: [Sensor Thresholds][Sensor-Events]
(exclusively using standard Redfish events) and [Leak Detector][Leak-Events]
(defining OpenBMC-specific events).

Rather than doing custom sanity parsing of these YAML files, as was done for the
`sdbusplus` interface definitions, I am leveraging [JSON-Schema][JSON-Schema].
This allows pretty decent explanation when the YAML file doesn't meet
expectations; I may add this capability to the interface YAML files at some
point.

From the YAML, `sdbusplus` will generate a C++ class for an event. This class
can either be thrown, which is useful for sdbusplus servers that want the event
to be sent back as part of the dbus response, or ["committed"][commit-api] to
the `phosphor-logging` record. The commit APIs support both blocking and async
variants.

See this [test-case][logging-test] as an example:

```cpp
        path = co_await lg2::commit(data->client_ctx,
                                    LoggingCleared("NUMBER_OF_LOGS", 6));
```

In the above code it might be noticed that the event metadata is taken as
`{string, data}` pairs. There is compile-time checking to ensure the correct
metadata has been provided.

```
### Missing metadata
../test/log_manager_dbus_tests.cpp:131:32: error: no matching function for call to ‘sdbusplus::event::xyz::openbmc_project::Logging::Cleared::Cleared()’
  131 |         { throw LoggingCleared(); }, LoggingCleared);

### Incorrect metadata name
../subprojects/sdbusplus/include/sdbusplus/utility/consteval_string.hpp: In member function ‘virtual void phosphor::logging::test::TestLogManagerDbus_GenerateSimpleEvent_Test::TestBody()’:
../test/log_manager_dbus_tests.cpp:130:5:   in ‘constexpr’ expansion of ‘sdbusplus::utility::consteval_string<sdbusplus::utility::details::consteval_string_holder<15>{"NUMBER_OF_LOGS"}>("LOGS")’
../subprojects/sdbusplus/include/sdbusplus/utility/consteval_string.hpp:41:25: error: call to non-‘constexpr’ function ‘static void sdbusplus::utility::consteval_string<V>::report_error(const char*) [with consteval_string_holder<...auto...> V = sdbusplus::utility::details::consteval_string_holder<15>{"NUMBER_OF_LOGS"}]’
   41 |             report_error("String mismatch; check parameter name.");

### Incorrect metadata type
../test/log_manager_dbus_tests.cpp:131:50: error: invalid conversion from ‘const char*’ to ‘uint64_t’ {aka ‘long unsigned int’} [-fpermissive]
  131 |         { throw LoggingCleared("NUMBER_OF_LOGS", "string"); }, LoggingCleared);
      |                                                  ^~~~~~~~
      |                                                  |
      |                                                  const char*
```

[Sensor-Events]: https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Sensor/Threshold.events.yaml
[Leak-Events]: https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/State/Leak/Detector.events.yaml
[JSON-Schema]: https://github.com/openbmc/sdbusplus/blob/master/tools/sdbusplus/schemas/events.schema.yaml
[commit-api]: https://github.com/openbmc/phosphor-logging/blob/236d864bfd291cbb4bdbb6202436563dd38c93c8/lib/include/phosphor-logging/commit.hpp
[logging-test]: https://github.com/openbmc/phosphor-logging/blob/236d864bfd291cbb4bdbb6202436563dd38c93c8/test/log_manager_dbus_tests.cpp#L175

# Using the CLI

I also wrote a `log-create` CLI which is useful for testing events, which is now
installed by the `phosphor-logging` package. Running `log-create --list` will
enumerate all of the event types that `phosphor-logging` is aware of:

```
$ log-create --list
Known events:
    xyz.openbmc_project.Logging.Cleared
    xyz.openbmc_project.Sensor.Threshold.InvalidSensorReading
    ...
```

Individual events can be created by specifying the event name and passing the
metadata as JSON (if any metadata is required):

```
$ log-create \
    xyz.openbmc_project.Sensor.Threshold.ReadingAboveUpperSoftShutdownThreshold \
    --json '{ "SENSOR_NAME": "Inlet Temperature", "READING_VALUE": 98.6, "UNITS": "xyz.openbmc_project.Sensor.Value.Unit.DegreesC", "THRESHOLD_VALUE": 40 }'
```

This will result in an event such as (some interfaces and fields pruned):

```
$ busctl --user introspect xyz.openbmc_project.Logging /xyz/openbmc_project/logging/entry/1 -l
xyz.openbmc_project.Logging.Entry           interface -         -                                                                                                                                                                                                                                                                                                            -
.GetEntry                                   method    -         h                                                                                                                                                                                                                                                                                                            -
.AdditionalData                             property  as        8 "READING_VALUE=98.6" "SENSOR_NAME=\"Inlet Temperature\"" "THRESHOLD_VALUE=40.0" "UNITS=\"xyz.openbmc_project.Sensor.Value.Unit.DegreesC\"" "_CODE_FILE=../log_create_main.cpp" "_CODE_FUNC=int generate_event(const std::string&, const nlohmann::json_abi_v3_11_2::json&)" "_CODE_LINE=34" "_PID=1931265" emits-change writable
.EventId                                    property  s         ""                                                                                                                                                                                                                                                                                                           emits-change writable
.Id                                         property  u         1                                                                                                                                                                                                                                                                                                            emits-change writable
.Message                                    property  s         "xyz.openbmc_project.Sensor.Threshold.ReadingAboveUpperSoftShutdownThreshold"                                                                                                                                                                                                                                emits-change writable
.Resolution                                 property  s         ""                                                                                                                                                                                                                                                                                                           emits-change writable
.Resolved                                   property  b         false                                                                                                                                                                                                                                                                                                        emits-change writable
.ServiceProviderNotify                      property  s         "xyz.openbmc_project.Logging.Entry.Notify.NotSupported"                                                                                                                                                                                                                                                      emits-change writable
.Severity                                   property  s         "xyz.openbmc_project.Logging.Entry.Level.Critical"                                                                                                                                                                                                                                                           emits-change writable
.Timestamp                                  property  t         1732137165521                                                                                                                                                                                                                                                                                                emits-change writable
.UpdateTimestamp                            property  t         1732137165521                                                                                                                                                                                                                                                                                                emits-change writable
```
