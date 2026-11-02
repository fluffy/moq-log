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

normative:
  MOQT: I-D.ietf-moq-transport
  RFC5424:
  RFC8259:

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

This document specifies how to send syslog RFC5424 type information over
the Media Over QUIC Transport (MOQT) {{MOQT}}.

--- middle


# Introduction 

The idea is each device that was logging would publish each log message
as an MOQT object. The devices or systems publishing the logs are
referred to as resources and have a unique ResourceID. The URLs for the
objects would be set up such that a subscriber could subscribe to each
resource creating logs separately, and could pick the log priority level
in the subscriptions. The log collector would subscribe to the logs from
the appropriate Resources that the collector wished to monitor.

The data model used is consistent with the "OpenTelemetry Specification"
{{OTEL}} (see https://opentelemetry.io/docs/specs/otel/logs/data-model/)
and a superset of the {{RFC5424}} data model for logging.

{{RFC5424}} specifies a layered architecture that provides for support
of any number of transport layer mappings for transmitting syslog
messages.  This document describes the MOQT transport mapping for the
syslog protocol.


# Terminology

## Resource ID {#sec-res-id}

Each Resource that creates logs has a unique ResourceID. This is created
by taking the MAC address of the primary network interface in binary,
computing the SHA1 hash of it, then truncating to lower 64 bits. Note
the SHA1 does not provide any security properties, it is just a hash
that is widely implemented in hardware. If this is not possible, any
other random stable 64 bit identifier may be used. The advantage of
using a MAC address is that many other management systems use this
address and using it makes it easier to correlate with other
systems. The disadvantage is that it reveals the MAC address.


# Naming {#sec-naming}

The TrackNamespace consists of following tuples (represented in string
format for ease of readability):

~~~
 "(moq://moq-syslog.arpa/logs-v1/),(ResourceID)" 
~~~

The TrackName tuple is a single byte that has the log priority level in
binary. Following the pattern:

~~~
 <log_level>
~~~

The MOQT Group ID is timestamp (explained in the next section) in the
message truncated to a 62 bit binary integer.

The MOQT Object ID is zero unless more than one message is produced in
the same microseconds in which case they each will get their own Object
ID.


# Object Data  {#sec-obj-data}

The object payload is a JSON {{RFC8259}} object with the following
optional fields:

* severity: As defined in {{RFC5424}}. Encoded as string "Emergency",
  "Alert", ... "Debug". This is called ServerText in {{OTEL}}.

* timestamp: single integer with number of microseconds since "1 Jan
  1972" using NTP Era zero conventions.

* pri: As defined in {{RFC5424}}. Numeric value from 0 to 23 and default
  is 1 if not present.

* hostname: As defined in {{RFC5424}}. Note this might not be a
  hostname.

* appname: As defined in {{RFC5424}}.

* procid: As defined in {{RFC5424}}.

* msgid: As defined in {{RFC5424}}.

* msg: As defined in {{RFC5424}}. This is a UTF-8 string.

Any other fields are treated as structured data as defined in
{{RFC5424}} and include:

* TraceID: Used in {{OTEL}} and defined in
  {{CRD-trace-context-2-20240328}}.

* SpanID: As defined in {{OTEL}}.

* InstrumentationScope: As defined in {{OTEL}}.

Any other fields are treated as "Attributes" when mapped to {{OTEL}}.

# IANA {#sec-iana}

TBD

# Security Considerations {#sec-security}

TBD

# Examples {#sec-examples}

On 31 Dec 1999 UTC a server produces the log message "shutting down for
Y2K" with severity INFO.  The timestamp for this would be
3,155,587,200. The JSON data would be:

~~~
Group 1740807280, Object ID 0
{
   "timestamp":3155587200,
   "severity":"Info",
   "msg":"shutting down for Y2K"
}
~~~

--- back

# Acknowledgments
{:numbered="false"}

Thanks to Suhas Nandakumar and Tim Evens for contributions and
suggestions to this specification.
