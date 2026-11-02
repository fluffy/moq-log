---
title: "Logging over Media over QUIC Transport"
abbrev: "moqt-log"
docname: draft-jennings-moq-log-latest
category: info
submissiontype: IETF
ipr: trust200902
area: transport
workgroup: ""
keyword:
 - realtime
 - quicr

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: C. Jennings
    name: Cullen Jennings
    organization: Cisco
    email: fluffy@iii.ca
    country: Canada
 -
    ins: S. Nandakumar
    name: Suhas Nandakumar
    organization: Cisco
    email: snandaku@cisco.com
    country: USA

normative:
  MOQT: I-D.draft-ietf-moq-transport-22
  RFC5424:

informative:
  OTEL:
    title: "OpenTelemetry Specification 1.34.0"
    target: https://opentelemetry.io/docs/specs/otel/logs/
    author:
      -
        ins: A. Ruech
        name: Armin Ruech
    date: 2024-06-11
  CRD-trace-context-2-20240328:
    title: "Trace Context Level 2"
    target: https://www.w3.org/TR/2024/CRD-trace-context-2-20240328/
    author:
      -
        ins: S. Kanzhelev
        name: Sergey Kanzhelev
      -
        ins: D. Dyla
        name: Daniel Dyla
      -
        ins: Y. Shkuro
        name: Yuri Shkuro
      -
        ins: J. K. Sundaram
        name: J. Kalyana Sundaram
      -
        ins: B. Krol
        name: Bastian Krol
    date: 2024-03-28
    seriesinfo:
      W3C: CRD-trace-context-2-20240328

--- abstract

Real time systems often run into the problems where the network
bandwidth for logging is shared with the real time media and impacts the
media quality. There is a desire to transport the logging data at an
appropriate priority level over the same transport as the media. This
allows the logging data to take advantage of times when the media
bitrate is below the peak rate while not impacting the peak rate
available for media.

This document specifies how to send syslog {{RFC5424}} type information
over Media over QUIC Transport (MOQT) {{MOQT}}.

--- middle


# Introduction

Each device that logs publishes its log messages as MOQT objects. The
devices or systems publishing the logs are referred to as resources and
have a unique ResourceID. Track names are set up such that a subscriber
can subscribe to the logs of each resource separately, and can pick the
log severity levels it wants. A log collector subscribes to the logs of
the resources it wishes to monitor, and MOQT relays cache logs so that
they can be retrieved later, when needed.

The data model used is consistent with the OpenTelemetry logs data model
{{OTEL}} and a superset of the {{RFC5424}} data model for logging.

{{RFC5424}} specifies a layered architecture that provides for support
of any number of transport layer mappings for transmitting syslog
messages. This document describes the MOQT transport mapping for the
syslog protocol.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Architecture Overview {#arch}

This section describes two deployment models for logging over MOQT. In
both, resources publish logs as MOQT tracks (see {{naming}}) to a relay,
and log collectors subscribe to them via relays. Relays cache logs so
that collectors can retrieve them later.

## Logs Sharing a Session with Media {#arch-shared}

In this model, an endpoint such as a realtime conferencing client
publishes its media and its logs over the same MOQT session, and thus
the same QUIC connection, to its ingress relay. Media and logs share the
congestion control context of that connection, and MOQT priorities
(see {{priorities}}) determine what is sent when the available bandwidth
is limited. At the relay, media and logs are forwarded to different
destinations: media to the other participants, and logs to a log
collector, possibly through other relays.

~~~ ascii-art
                  one QUIC connection
              (shared congestion control)
+------------+                   +---------+  media  +--------------+
|  Endpoint  |      media        |         |-------->| Participants |
| +--------+ |==================>|         |         +--------------+
| | Media  | |                   |  Relay  |
| +--------+ |      logs         |         |  logs   +--------------+
| |  Logs  | |==================>| (cache) |-------->|     Log      |
| +--------+ |   (pri 128..240)  |         |         |  Collector   |
+------------+                   +---------+         +--------------+
~~~

When congestion occurs on the endpoint's connection, log messages of the
higher severity levels are the first to be delayed and, with
OBJECT_DELIVERY_TIMEOUT, dropped, while media is unaffected. The relay
continues to cache what it has received, and a collector can retrieve
logs for a time window of interest later (see {{retrieval}}).

## Logs as the Primary Data {#arch-dedicated}

In this model, the endpoints, such as servers, network devices or
sensors, send logs as their main data over MOQT. Each endpoint publishes
its log tracks to a relay, which may aggregate many endpoints, and one or
more log collectors subscribe to them.

~~~ ascii-art
+------------+
| Endpoint 1 |---logs----+
+------------+           |
                         v
+------------+      +---------+       +---------+     +-----------+
| Endpoint 2 |----->|  Relay  |------>|  Relay  |---->|    Log    |
+------------+      | (cache) |       | (cache) |     | Collector |
                    +---------+       +---------+     +-----------+
+------------+           ^                 |
| Endpoint N |---logs----+                 |          +-----------+
+------------+                             +--------->| Dashboard |
                                                      +-----------+
~~~

Collectors discover endpoints and subscribe to the severity levels of
interest (see {{discovery}}), while the remaining levels stay in relay
caches until needed (see {{retrieval}}).


# Resource ID {#sec-res-id}

Each Resource that creates logs has a unique ResourceID. This is created
by taking the MAC address of the primary network interface in binary,
computing the SHA-1 hash of it, then truncating to lower 64 bits. Note
that SHA-1 does not provide any security properties here; it is just a
hash that is widely implemented in hardware. If this is not possible, any
other random stable 64-bit identifier MAY be used. The advantage of using
a MAC address is that many other management systems use this address and
using it makes it easier to correlate with other systems. The
disadvantage is that it reveals the MAC address.

When carried in MOQT names, the ResourceID is encoded as exactly 16
lowercase hexadecimal ASCII characters.


# Logs and MOQT {#mapping}

## Track Naming {#naming}

Each resource publishes its logs under the Track Namespace consisting of
the following three Track Namespace Fields, each encoded as ASCII:

~~~
("logs.moq.arpa", "v1", <resourceID>)
~~~

The Track Namespace Prefix `("logs.moq.arpa", "v1")` is common to all
resources, which allows a collector to discover all resources (see
{{discovery}}). Within a resource's namespace there is one track per
severity level. The Track Name is a single ASCII digit identifying the
{{RFC5424}} severity of the log messages it carries:

| Severity      | Track Name | Default Publisher Priority |
|:--------------|:----------:|:--------------------------:|
| Emergency     | `0`        | 128                        |
| Alert         | `1`        | 144                        |
| Critical      | `2`        | 160                        |
| Error         | `3`        | 176                        |
| Warning       | `4`        | 192                        |
| Notice        | `5`        | 208                        |
| Informational | `6`        | 224                        |
| Debug         | `7`        | 240                        |
{: title="Severity levels"}

In this document, "lower levels" refers to Emergency (0) towards Error (3),
and "higher levels" to Warning (4) towards Debug (7).

Using the serialization format in {{Section 8.8 of MOQT}}, the
Informational track of the resource with ResourceID `8f2c4a1b0e9d7c63` is
rendered as:

~~~
logs.2emoq.2earpa-v1-8f2c4a1b0e9d7c63--6
~~~

## Groups and Objects {#groups}

Each group in a log track holds the log messages produced by the resource
during one second. The Group ID is the start of that second, in seconds
since the UNIX Epoch (00:00:00 UTC on 1 January 1970). This satisfies the
requirement that Group IDs increase with time ({{Section 2.3.1 of MOQT}})
and allows subscribers to express time ranges directly as Location
filters (see {{retrieval}}).

Each log message is one object, carrying a Log Record ({{record}}). Object
IDs start at 0 in each group and increase by one for each log message in
the order they are produced. Each object is self-contained, so a
subscriber can use any object without other objects of the group.

All objects of a group MUST be sent in a single subgroup with Subgroup ID
0. The publisher sends each object as soon as the log message is
produced, keeping the subgroup open until the end of the second. The
publisher MUST then end the group with an object with the End of Group
Object Status ({{Section 11.1.1 of MOQT}}) and an Object ID one greater
than the last log message, and close the subgroup. The Datagram delivery
mode MUST NOT be used.

Seconds without log messages have no group. The publisher SHOULD include
the Prior Group ID Gap Object Property ({{Section 10.8 of MOQT}}) on the
first object of each group, set to the number of seconds since the
previous group minus one. This allows relays and fetches to know that the
intervening groups will never exist, rather than leaving them in the
unknown state.

## Log Record {#record}

The payload of each log message object is a Log Record, encoded as
follows using the notation of {{Section 1.3 of !RFC9000}} and the
variable-length integer (vi64) of {{Section 8.1 of MOQT}}.

~~~
Log Record {
  Timestamp Offset (vi64),
  Facility (vi64),
  Hostname Length (vi64),
  Hostname (..),
  App Name Length (vi64),
  App Name (..),
  Proc ID Length (vi64),
  Proc ID (..),
  Msg ID Length (vi64),
  Msg ID (..),
  Message Length (vi64),
  Message (..),
  Number of Attributes (vi64),
  Attribute (..) ...,
}

Attribute {
  Key Length (vi64),
  Key (..),
  Value Length (vi64),
  Value (..),
}
~~~
{: title="Log Record"}

* Timestamp Offset: The nanoseconds to add to the Group ID, in seconds,
  to obtain the time the log message was produced. The value is less
  than 1000000000.

* Facility: The facility as defined in {{RFC5424}}, from 0 to 23.

* Hostname, App Name, Proc ID, Msg ID: The HOSTNAME, APP-NAME, PROCID
  and MSGID fields as defined in {{RFC5424}}, as ASCII strings. An empty
  field corresponds to the {{RFC5424}} NILVALUE.

* Message: The MSG as defined in {{RFC5424}}, as a UTF-8 string. This is
  the Body in {{OTEL}}.

* Attribute: A key-value pair, where Key and Value are UTF-8 strings.
  Attributes carry {{RFC5424}} structured data. The keys "TraceID" and
  "SpanID", defined in {{CRD-trace-context-2-20240328}}, and
  "InstrumentationScope" have the meaning defined in {{OTEL}}. Other
  attributes map to Attributes in {{OTEL}}.

The severity of a log message is given by the track that carries it and
is not repeated in the Log Record. Receivers MUST ignore any bytes
following the fields defined in this document, to allow future extension.

## Track Properties {#track-properties}

Publishers SHOULD include the following Track Properties
({{Section 8.4 of MOQT}}) on each log track:

* DEFAULT_PUBLISHER_PRIORITY: see {{priorities}}.

* DEFAULT_PUBLISHER_GROUP_ORDER: Ascending (0x1), so that collectors
  receive log messages in order.

* MAX_CACHE_DURATION: the retention time for which relays may serve
  cached log messages. Lower levels are typically retained longer than
  higher levels.

* OBJECT_DELIVERY_TIMEOUT: for higher levels, a timeout after which a
  stale log message is dropped rather than delivered late.

## Priorities {#priorities}

Logs are expected to yield to the primary application under congestion.
The publisher SHOULD set the DEFAULT_PUBLISHER_PRIORITY Track Property of
each log track to the value in {{naming}}, or another value that keeps
lower levels ahead of higher levels. These values are in the lower
priority half of the range, behind the MOQT default of 128 typically used
for media, with Emergency at par with the default.

Subscribers can adjust the relative importance of logs against other
subscriptions in the session using the SUBSCRIBER_PRIORITY parameter,
which is considered before publisher priority.

## Discovery and Subscription {#discovery}

A collector discovers resources by sending SUBSCRIBE_NAMESPACE with the
Track Namespace Prefix `("logs.moq.arpa", "v1")`. The relay responds with a
NAMESPACE message for each resource namespace, and NAMESPACE_DONE when a
resource stops publishing.

For each discovered resource, the collector subscribes by name to the
tracks of the levels it wants to receive continuously, for example levels
0 to 3. Resources that start publishing later are learned from subsequent
NAMESPACE messages. Tracks of other levels are not subscribed to and
remain available from the relay cache via FETCH (see {{retrieval}}).

Collectors can further select log messages using MOQT Object Property
range filters ({{Section 3.3.2 of MOQT}}) on application-defined Object
Properties, which are outside the scope of this document. Range Filters
are only available when the relay advertises a non-zero
MAX_FILTER_RANGES, so collectors MUST be able to process unfiltered
tracks.

## Retrieval of Past Logs {#retrieval}

Since Group IDs are times in seconds, a time range of logs corresponds
directly to a range of Locations:

* A collector retrieves past log messages for a time window `[T1, T2]` by
  sending FETCH with a Location filter from {T1, 0} to {T2, max}. The
  request is served from the relay cache, subject to MAX_CACHE_DURATION.

* A collector that wants recent history followed by live log messages
  sends a SUBSCRIBE with a Location filter starting at the Next Object,
  together with FILL_PARAMETERS whose Location filter selects the
  starting point of the history. The relay delivers the history on a fill
  fetch stream and live log messages on the subscription, without gaps or
  duplicates ({{Section 3.4 of MOQT}}).

* A collector that only needs live log messages subscribes with a
  Location filter starting at the Next Object.

This enables "just in time" collection of logs: resources publish all
levels to a nearby relay, while collectors subscribe to the lower levels
and fetch the higher levels for the time windows of interest, for example
around an error.

## Examples {#examples}

On 31 December 1999, a server publishes two Informational log messages
in the second starting at 23:59:59 UTC (Group 946684799), and one more
five seconds later (Group 946684805).

~~~ ascii-art
Track "6", Group 946684799

 Obj  Content                         Meaning
 ---  ------------------------------  ----------------------------
  0   Log Record                      23:59:59.250000000
        offset 250000000 ns           user facility
        facility 1, app "y2kd"        "preparing for Y2K"
        msg "preparing for Y2K"
  1   Log Record                      23:59:59.900000000
        offset 900000000 ns           user facility
        facility 1, app "y2kd"        "shutting down for Y2K"
        msg "shutting down for Y2K"
  2   End of Group                    no further log messages

Track "6", Group 946684805

 Obj  Content                         Meaning
 ---  ------------------------------  ----------------------------
  0   Log Record                      2000-01-01 00:00:05.000120000
        offset 120000 ns              "back online"
        facility 1, app "y2kd"
        msg "back online"
        [Prior Group ID Gap 5]        no groups since previous one
  1   End of Group                    no further log messages

[ ] denotes an Object Property.
~~~
{: title="Log track example"}


# Use with MSF {#msf}

An MOQT Streaming Format (MSF) {{?MSF=I-D.ietf-moq-msf}} catalog can
direct a client to publish logs using a `publishTracks` entry with a
`packaging` of `"moqlog"` and a `role` of `"log"`. In such an entry:

* `namespace` is the Track Namespace of {{naming}} rendered using
  {{Section 8.8 of MOQT}}, with `%resourceId%` as a placeholder for the
  client's ResourceID: `logs.2emoq.2earpa-v1-%resourceId%`.

* `name` is the track for a severity level ({{naming}}). A catalog can
  include one entry per level it wants the client to publish.


# IANA Considerations {#sec-iana}

This document has no IANA actions.


# Security Considerations {#sec-security}

TODO


--- back

# Changes from the -03 Version
{:numbered="false"}

* Aligned with draft-ietf-moq-transport-22.

* Added an Architecture Overview describing logs sharing a session with
  media, and logs as the primary data.

* Track Namespace is now the tuple ("logs.moq.arpa", "v1", resourceID),
  replacing the moq:// URL form, and the Track Name is the severity level
  as an ASCII digit.

* Each group holds the log messages of one second, sent on a single
  subgroup and ended with an End of Group object, instead of one group
  per log message.

* Log Records use a binary encoding following the MOQT notation instead
  of JSON, with timestamps relative to the Group ID.

* Added the use of Prior Group ID Gap, Track Properties for priority,
  group order, caching and delivery timeouts, SUBSCRIBE_NAMESPACE for
  discovery, and FETCH and fill for time range retrieval.

* Added use with MSF.

# Acknowledgments
{:numbered="false"}

Thanks to Tim Evens for contributions and suggestions to this
specification.
