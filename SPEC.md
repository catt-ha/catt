CATT Specification
==================

Version: 0.0.1

## Purpose

The purpose of this document is to specify a communication standard that can be
used to implement an extensible and scalable IoT network. Current systems either
suffer from being too closed, where users are stuck with whatever a proprietary
hub supports, or, if open-source, too monolithic, where any extensions have to
be written as plugins for the framework. The later, while more extensible, tends
to suffer from its own version of lock-in. Generally, plugins must either be
written in the same language as the framework or must implement a client for its
RESTful interface. Components of the framework are not easily replaceable, such
as rule engines or user-friendly frontends.

Rather than this kitchen-sink-style of home automation framework, a more
loosely-coupled approach may provide a greater degree of flexibility. This
system should, ideally, define only the method by which things communicate,
leaving the number of services, where they run, what language they're written
in, and supported features up to the users and development community.

## Concepts

### The Bus

The bus is the method by which services will communicate. All communication is
done via an MQTT broker. Clients should support authenticated and TLS-encrypted
connections to the broker. A base path should be configurable, such as
`catt/items`, after which the standardized heirarchy
`{{ item name }}/{{ channel}}` will be used. Thus, the overall path heirarchy will be
`{{ base path }}/{{ item name }}/{{ channel }}`

### Items

Items are the basic unit of communication. They have three channels: `state`,
`command`, and `meta`.

#### State

Items should send their state through the `state` channel. Publishing to the state
channel for an item does not necessarily convey that the state has changed - it
is valid to simply send the same state as was sent previously.

For example, when a light comes on, it should publish `ON` to
`catt/items/Light_1/state`. It may also periodically send `ON` to the same path
to keep any new subscribers apprised of its state. It should send `OFF` when it
is turned off.

#### Command

Commands for items can be sent through the `command` channel. Backends should
subscribe to this channel for all of the items that they control, and update
them accordingly. They may then send the updated state for the item to its
`state` channel.

For example, a backend which controls a zwave switch, `Switch_1` should
subscribe to `catt/items/Switch_1/command`. When it receives the `ON` command,
it should turn the switch on, and report `ON` back to
`catt/items/Switch_1/state` if the switch was successfully turned on.

#### Meta

Metadata about the item should be sent to the `meta` channel. This channel is
optional and serves to provide more information about what the item
actually represents, what commands it supports, its health, etc. This would be
of interest to systems implementing auto-discovery of items.

For example, a zwave controller might publish metadata for all of its items
informing subscribers that they're zwave devices, their command class, their
home\_id, their node\_id, etc.

## Message Formats

### State and Commands

State and command messages will have the same format. On the bus, they will
always be represented as valid utf-8 strings. Once received, they will be
interpreted as one of three data types: string, number, or boolean. This will be
done by attempting to convert the string to a boolean value, falling back to
converting to a number if that fails, and finally simply falling back to a
string. If desired, a client may also convert strings to/from raw byte arrays by
base64 en/decoding. On-wire, the messages should always be in encoded form.

* TODO: Full conversion table for data types

### Metadata

Metadata is currently sent as a TOML table. Subscribers can expect it to contain the keys `backend` and `value_type`, identifying the backend service and the data type that it sends/receives. Additional implementation-specific keys may go in the `[ext]` table, such as zwave node_id, home_id, command class, etc.

Example meta message:

```
backend = "zwave"
value_type = "string"

[ext]
home_id = 12345
node_id = 1
command_class = "SwitchBinary"
````
