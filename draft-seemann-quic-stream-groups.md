---
title: "QUIC Stream Groups"
abbrev: "QUIC Stream Groups"
docname: draft-seemann-quic-stream-groups-latest
submissiontype: IETF
category: std

ipr: trust200902
area: "Transport"
workgroup: "QUIC"
keyword: QUIC, flow control, stream groups

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: M. Seemann
    name: Marten Seemann
    organization: Protocol Labs
    email: martenseemann@gmail.com

normative:

informative:


--- abstract

QUIC ({{!RFC9000}}) defines a few different mechanism flow control mechanisms:
Stream flow control, connection-level flow control and flow control for the
number of (unidirectional / bidirectional) streams

This allows a single application running on of a QUIC connection to apply backpressure.

However, when multiple independent applications share a single underlying QUIC connection,
these mechanisms are not sufficient to prevent one resource-hungry application from consuming
all the resources (streams and / or connection-level flow control credit) available on the
connection, effectively starving the other application.


--- middle

# Introduction

This document defines a QUIC extension that introduces the concept of stream
groups. Unidirectional and bidirectional streams are added to different
stream groups. Additional limits for the number of streams and the flow
controlled data are applied to each group.

This extension adds another layer of flow control for data sent on streams, by
adding a data limit per stream group. For the number of streams opened, this
extension replaces the mechanism described in ({{!RFC9000}}).

Logically, the flow control mechanisms of QUIC ({{!RFC9000}}) can be regarded
as the degenerate case of this extension: A connection that only has a single
stream group.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Using Stream Groups

This extension doesn't prescribe any specific usage pattern for stream groups.
Applicationsrunning on top of a QUIC connection might check the peer assigns
streams to the correct stream groups, based on some application-defined logic.

# Negotiating this Extension

Endpoints advertise their support of the extension described in this document by
sending at least one of the following transport parameters (Section 7.4 of {{!RFC9000}}).

initial_max_stream_groups (0x2a4c1f8e9b6d2c01):

: The initial maximum stream group is an integer value that contains the
  initial value for the maximum stream group.

initial_max_group_streams_bidi (0x2a4c1f8e9b6d2c02):

: The initial maximum bidirectional streams group parameter is an integer value
  that contains the initial maximum number of bidirectional streams the endpoint
  that receives this transport parameter is permitted to initiate in a new stream
  group. If this parameter is absent or zero, the peer cannot open bidirectional
  streams until a MAX_GROUP_STREAMS frame is sent.

initial_max_group_streams_uni (0x2a4c1f8e9b6d2c03):

: The initial maximum unidirectional streams group parameter is an integer value
  that contains the initial maximum number of unidirectional streams the endpoint
  that receives this transport parameter is permitted to initiate in a new stream
  group. If this parameter is absent or zero, the peer cannot open unidirectional
  streams until a MAX_GROUP_STREAMS frame is sent.

initial_max_group_data (0x2a4c1f8e9b6d2c04):

: The initial maximum data parameter is an integer value that contains the
  initial value for the maximum amount of data that can be sent on a newly
  established stream group. This is equivalent to sending a MAX_GROUP_DATA
  frame for the stream group immediately after establishment of the stream group.

Any of these transport parameters enables the use of this extension. The missing
transport parameters take a default value of 0.

An implementation that understands these transport parameters MUST treat the
receipt of an empty value or a value that is not a QUIC varint for any of
these parameters as a connection error of type TRANSPORT_PARAMETER_ERROR.

When negotiating this extension, the initial_max_streams_bidi (0x08) and the
initial_max_streams_uni (0x09) transport parameters MUST NOT be used. The
receiver MUST treat receipt of any of these parameters as  a connection error
of type TRANSPORT_PARAMETER_ERROR.

TODO: define which of these parameters need to be remembered for 0-RTT


# Limiting the Number of Streams

QUIC ({{!RFC9000}}) identifies a stream by one number, its stream ID. When
using this extension, a stream is identified by the tuple of the stream group
and the stream ID.

## Opening a New Stream Group

A new stream group is initialized implicitly, by sending a STREAM,
RESET_STREAM or STREAMS_BLOCKED frame with a new Stream Group ID. Stream Group
IDs are integers, starting at 0 and are incremented by 1 for every new stream
group.

Initially, the number of stream groups is limited by the value provided in the
initial_max_stream_groups transport parameter. Receiving a MAX_STREAM_GROUP
frame updates this value.

Note that due to packet reordering, MAX_STREAM_GROUP frames might be received
out of order. Endpoints MUST therefore ignore MAX_STREAM_GROUP frames that
decrease the Maximum Stream Group value received so far.

When a peer wishes to initialize a new stream group, but is unable to do so due
to the maximum stream group limit set by the peer, it SHOULD send a
STREAM_GROUP_BLOCKED frame.

## Opening New Streams

Every stream is associated with an existing stream group, or it opens a new
stream group (see previous section).

When an endpoint wishes to open a new stream for an existing stream group, but
is unable to do so due the stream group limit, it SHOULD send a STREAMS_BLOCKED
frame.

When receiving a frame that would cause a new stream to be opened, the receiver
MUST check if the sender was permitted to open this stream in the respective
stream group, and close the connection with a STREAM_LIMIT_ERROR if the peer
violated the limit.


# Flow Control

## Sending and Receiving Stream Data

In addition to the flow control mechanism described in (Section 4.1 of {{!RFC9000}}),
senders need to respect the stream group flow control. In specific, when sending
data on the stream, this means the sender has to check:
1. The stream flow control window
2. The stream group flow control window
3. The connection flow control window

When receiving a frame that advances the highest offset received on stream, the
receiver MUST check that none of the flow control windows was violated. In case
of a violating, it MUST close the connection using a FLOW_CONTROL_ERROR.


# Changes to Existing Frames

When this extension is negotiated, an additional Stream Group varint is added to a number
of QUIC frames:

## STREAM frame

~~~
STREAM Frame {
  Type (i) = 0x08..0x0f,
  Stream Group (i),
  Stream ID (i),
  [Offset (i)],
  [Length (i)],
  Stream Data (..),
}
~~~

## RESET_STREAM frame

~~~
RESET_STREAM Frame {
  Type (i) = 0x04,
  Stream Group (i),
  Stream ID (i),
  Application Protocol Error Code (i),
  Final Size (i),
}
~~~

## STREAMS_BLOCKED frame

~~~
STREAMS_BLOCKED Frame {
  Type (i) = 0x16..0x17,
  Stream Group (i),
  Maximum Streams (i),
}
~~~

# New Frames

The following new frames are defined.

TODO: define eligibility for 0-RTT

## MAX_STREAM_GROUP

A MAX_STREAM_GROUP frame informs the peer of the maximum stream group that
it can create new streams for.

~~~
MAX_STREAM_GROUP Frame {
  Type (i) = 0x1b37fd9a08c2e500,
  Maximum Stream Group (i),
}
~~~

## MAX_GROUP_DATA

A MAX_GROUP_DATA frame is used in flow control to inform the peer of the
maximum amount of data that can be sent on the stream group as a whole.

~~~
MAX_GROUP_DATA Frame {
  Type (i) = 0x1b37fd9a08c2e501,
  Stream Group (i),
  Maximum Data (i),
}
~~~

## GROUP_DATA_BLOCKED

A sender SHOULD send a GROUP_DATA_BLOCKED frame when it wishes to send data but
is unable to do so due to stream group-level flow control. GROUP_DATA_BLOCKED
frames can be used as input to tuning of flow control algorithms;
see (Section 4.2 of {{!RFC9000}}).

~~~
GROUP_DATA_BLOCKED Frame {
  Type (i) = 0x1b37fd9a08c2e520,
  Stream Group (i),
  Maximum Data (i),
}
~~~

## STREAM_GROUP_BLOCKED

A sender SHOULD send a STREAM_GROUP_BLOCKED frame when it wishes to open a
new stream group, but is unable to do so because the maximum stream group limit
set by the peer.

~~~
GROUP_DATA_BLOCKED Frame {
  Type (i) = 0x1b37fd9a08c2e521,
  Stream Group (i),
}
~~~

# Security Considerations

We succeeded at encrypting all the things.
Isn't it wonderful to have a protocol that middleboxes can't mess with?

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
