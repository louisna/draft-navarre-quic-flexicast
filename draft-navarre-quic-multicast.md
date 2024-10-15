---
title: "Extending QUIC with multicast capabilities"
abbrev: "MC-QUIC"
category: exp

docname: draft-navarre-quic-multicast-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Transport"
workgroup: "QUIC"
keyword:
 - quic
 - multicast
venue:
  group: "QUIC"
  type: "Working Group"
  mail: "quic@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/quic/"
  github: "louisna/draft-navarre-quic-multicast"
  latest:

author:
 -
    fullname: Louis Navarre
    organization: UCLouvain
    email: louis.navarre@uclouvain.be
 -  fullname: Olivier Bonaventure
    organization: UCLouvain
    email: olivier.bonaventure@uclouvain.be

normative:
  RFC2119:
  QUIC-TRANSPORT: rfc9000

informative:
  MULTIPATH-QUIC: I-D.ietf-quic-multipath
  RFC8678:


--- abstract

TODO Abstract


--- middle

# Introduction

TODO Introduction


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# New Frames

All frames defined in this document MUST only be sent in 1-RTT packets.

If an endpoint receives a flexicast-specific frame in a different packet type, it MUST close the connection with an error of type FRAME_ENCODING_ERROR.

Receipt of flexicast-specific frames related to a Flexicast Flow ID that is not added by an endpoint MUST be treated as a connection error of type FC_PROTOCOL_VIOLATION.

If an endpoint receives a flexicast-specific frame with a Flexicast Flow ID that it cannot process anymore (e.g., the flexicast flow might have been abandoned), it MUST silently ignore the frame.

The new frames introduced below are for control-specific purpose only, and MUST NOT be sent on the Flexicast flow. Receipt of any of the following frames on the Flexicast flow MUST be trated as a connection error of type FC_PROTOCOL_VIOLATION.

## FC_ANNOUNCE frame

The FC_ANNOUNCE frame informs the receiver that a Flexicast flow is available or updated.
FC_ANNOUNCE frames MUST NOT be sent by the receiver. A Flexicast QUIC source receiving an FC_ANNOUNCE frame MUST close the connection with a connection error of type FC_PROTOCOL_VIOLATION.

FC_ANNOUNCE frames are formatted as shown in {{fig-fc-announce-frame-format}}.

~~~
FC_ANNOUNCE Frame {
    Type (i) = TBD-00,
    Length (8),
    Flexicast Flow ID (8..160),
    Sequence number (i),
    IP Version (8),
    Source IP (32, 128),
    Group IP (32, 128),
    UDP Port (16),
    Ack delay timer (64),
}
~~~
{: #fig-fc-announce-frame-format title="FC_ANNOUNCE Frame Format"}

FC_ANNOUNCE frames contain the following fields:

Length: An 8-bit unsigned internet containing the length of the Flexicast Flow ID. Values less than 1 and greated than 20 are invalid and MUST be treated as a connection error of type FRAME_ENCODING_ERROR.

Flexicast Flow ID: A Flexicast Flow ID of the specified length.

Sequence number: The monotically increasing sequence number related to the advertised Flexicast Flow ID.

IP Version: An 8-bit unsigned integer containing the version of IP used to advertise the Source IP and Group IP. Values different than 4 (for IPv4) and 6 (IPv6) are invalid and MUST be treated as a conneciton error of type FRAME_ENCODING_ERROR.

Source IP: The IP address of the multicast source.

Group IP: The IP address of the multicast group. The address MUST be a multicast address.

UDP Port: The UDP destination port.

Ack delay timer: A 64-bit unsigned integer containing the delay, in ms, between two acknowledgments from a receiver.

FC_ANNOUNCE frames are ack-eliciting. If a packet containing an FC_ANNOUNCE frame is considered lost, the peer SHOULD repeat it.

Sources are allowed to send multiple times FC_ANNOUNCE frames with an increasing sequence number for the same Flexicast Flow ID.
New FC_ANNOUNCE frames MAY contain updated information, e.g., a new Ack delay timer.

Sources are allowed to advertise multiple Flexicast flows by sending multiple parallel FC_ANNOUNCE frames with distinct Flexicast Flow IDs.
The Sequence number is linked to a specific Flexicast Flow ID. The same Sequence number can be used for two distinct Flexicast Flow IDs.

TODO: the frame should also contain a `Status` field to indicate if the flexicast flow is retired?

## FC_STATE frame

The FC_STATE frame informs the endpoint of the state of the Flexicast receiver in the Flexicast flow.
FC_STATE frames MAY be sent by both endpoints (i.e., receiver and source).

FC_STATE frames are formatted as shown in {{fig-fc-state-frame-format}}.

~~~
FC_STATE frame {
    Type (i) = TDB-01,
    Length (8),
    Flexicast Flow ID (8..160),
    Sequence number (i),
    Action (u64),
}
~~~
{: #fig-fc-state-frame-format title="FC_STATE Frame Format"}

FC_STATE frames contain the following fields:

Length: An 8-bit unsigned internet containing the length of the Flexicast Flow ID. Values less than 1 and greated than 20 are invalid and MUST be treated as a connection error of type FRAME_ENCODING_ERROR.

Flexicast Flow ID: The Flexicast Flow ID of the Flexicast flow that this frame relates to.

Sequence number: The monotically increasing sequence number related to the advertised Flexicast Flow ID.

Action: The bit-encoded action, defined in Section {{fc-state-action}}.

FC_STATE frames are ack-eliciting. If a packet containing an FC_STATE frame is considered lost, the peer SHOULD repeat it.

For a same Flexicast flow (i.e., identical Flexicast Flow ID), both endpoints use their own Sequence number.

A receiver sending an FC_STATE frame indicates the source its status regarding the Flexicast flow indicated by the Flexicast Flow ID.
The source MAY also send FC_STATE frames to a receiver to unilaterally change the status of the receiver within the Flexicast flow indicated by the Flexicast Flow ID.

### FC_STATE actions {#fc-state-action}

This section lists the defined Actions encoded in an FC_STATE frame. An endpoint receiving an unknown value MUST treat it as a connection error of type FC_PROTOCOL_VIOLATION.

The three following actions are receiver-specific. These actions MUST NOT be sent inside an FC_STATE frame sent by the source. A receiver receiving an FC_STATE frame with any of the following actions MUST treat it as a connection error of type FC_PROTOCOL_VIOLATION.

JOIN (0x01): The receiver joins the Flexicast flow.

LEAVE (0x02): The receiver leaves the Flexicast flow.

READY (0x03): The receiver is ready to receive content on the Flexicast flow.

The following action is source-specific. This action MUST NOT be sent inside an FC_STATE frame sent by the receiver. A source receiving an FC_STATE frame with the following action MUST treat it as a connection error of type FC_PROTOCOL_VIOLATION.

MUST_LEAVE (0x04): The source unilaterally decides that the receiver MUST leave the Flexicast flow.

## FC_KEY frame

The FC_KEY frame informs a receiver with the decryption key of a Flexicast flow joined by the receiver.
FC_KEY frames MUST NOT be sent by the receiver. A Flexicast QUIC source receiving an FC_KEY frame MUST clone the connection with a connection error of type FC_PROTOCOL_VIOLATION.

FC_KEY frames are formatted as shown in {{fig-fc-key-frame-format}}.

~~~
FC_KEY Frame {
    Type (i) = TDB-02,
    Length(8),
    Flexicast Flow ID(8..160),
    Sequence number (i),
    Key length (i),
    Key (..),
    Algorithm (64),
}
~~~
{: #fig-fc-key-frame-format title="FC_KEY Frame Format"}

FC_KEY frames contain the following fields:

Length: An 8-bit unsigned internet containing the length of the Flexicast Flow ID. Values less than 1 and greated than 20 are invalid and MUST be treated as a connection error of type FRAME_ENCODING_ERROR.

Flexicast Flow ID: The Flexicast Flow ID of the Flexicast flow that this frame relates to.

Sequence number: The monotically increasing sequence number related to the advertised Flexicast Flow ID.

Key length: A var-int indicating the length of the decryption key.

Key: Byte-sequence of the decryption key of the Flexicast flow.

Algorithm: The bit-encoded algorithm used for decryption, defined in Section {{fc-key-algorithm}}.

FC_KEY frames are ack-eliciting. If a packet containing an FC_STATE frame is considered lost, the peer SHOULD repeat it.

The source MAY send new FC_KEY frames with an increased sequence number to notify a new decryption key.
This mechanism can be used to provide backward and forward secrecy with dynamic Flexicast groups.

### FC_KEY algorithms {#fc-key-algorithm}

TODO: define the bit-encoded algorithms.

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
