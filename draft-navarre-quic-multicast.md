---
title: "Flexicast QUIC: combining unicast and multicast in a single QUIC connection"
abbrev: "FC-QUIC"
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
 -  fullname: Louis Navarre
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
  MULTICAST-QUIC: I-D.jholland-quic-multicast-05
  I-D.pardue-quic-http-mcast:
  RFC1112:
  RFC4607:
  RFC9293:
  DIOT:
    author:
      - ins: C. Diot
      - ins: B. Levine
      - ins: B. Lyles
      - ins: H. Kassem
      - ins: D. Belensiefen
    title: Deployment issues for the IP multicast service and architecture
    seriesinfo: IEEE network, vol. 14, no 1, p. 78-88.
    date: 2000


--- abstract

This document proposes Flexicast QUIC, a simple extension to Multipath
QUIC that enables a source to send the same information to a set of
receivers using a combination of unicast paths and multicast trees.


--- middle

# Introduction


Starting from the initially efforts of Steve Deering {{RFC1112}}, the IETF
has developed a range of IP multicast solutions that enable the efficient
transmission of a single packet to a group of receivers. In this document,
we focus on Source-Specific Multicast for IP {{RFC4607}}, but the solution
proposed could also be applied to other forms of IP Multicast.

Although IP Multicast is not a new solution, it is not as widely used by
applications as transport protocols like TCP {{RFC9293}} or QUIC {{QUIC-TRANSPORT}} do not provide multicast transport support.
Current IP Multicast applications include IP TV distribution in ISP networks
and trading services for financial institutions. Many reasons explain the difficulty
of deploying IP Multicast {{DIOT}}. From the application's viewpoint, a
key challenge with IP Multicast is that even if there is a unicast path
between two hosts, there is no guarantee that it will be possible to create and
maintain a multicast tree between these two hosts to efficiently exchange
data using IP multicast.
Applications must implement multiple protocols in case a recipient does not support
multicast?
For this reason, many applications that send
the same information to a large set of receivers like streaming services
or software updates, still rely on TCP or QUIC. This is inefficient from
the network viewpoint as the network carries the same information multiple
times.

The deployment of QUIC opens an interesting opportunity to reconsider the
utilization of IP Multicast. As QUIC runs above UDP, it could easily use
IP multicast to deliver information along multicast trees. Multicast
extensions to QUIC have already been proposed {{MULTICAST-QUIC}}
{{I-D.pardue-quic-http-mcast}} and {{MULTICAST-QUIC}}.


This document proposes to start from Multipath QUIC {{MULTIPATH-QUIC}}. Thanks
to this recent QUIC extension, a QUIC connection can use several paths
simultaneously to exchange information between two hosts. This document
proposes a simple extension to Multipath QUIC that enables to share an
additional path between multiple receivers. The destination address of
this path can be a multicast IP address and rely on a multicast forwarding
mechanism (e.g., IP Multicast) to transmit the packets.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Flexicast QUIC

Multipath QUIC was designed to enable the simultaneous utilization of multiple
paths inside a single QUIC connection. A Multipath QUIC connection
starts with a handshake like a regular QUIC connection, besides the utilization
of the initial_max_path_id to negotiate the multipath extension. This parameter
specifies the maximum path identifier that an endpoint is willing to maintain
at connection establishment. Multipath QUIC requires the utilization of non-zero
connection identifiers and uses explicit path identifiers and one packet number
space per path. Multipath QUIC assumes that the additional paths are created
by the client using the server address of the initial paths.

Consider as an example a smartphone that uses Multipath QUIC. This smartphone
has a Wi-Fi and a cellular interface. The smartphone has created the QUIC
connection over the cellular interface. After the handshake, the smartphone
and the server have exchanged additional connection identifiers. To create a
new path over the Wi-Fi interface, the smartphone sends a PATH_CHALLENGE over
this path using one of the connection identifiers advertised by the server.
The server replies with a PATH_RESPONSE using one of the connection
identifiers advertised by the smartphone. The smartphone confirms the establishment
of the second path using a PATH_RESPONSE. At this point that smartphone and the
server have two paths to exchange data: the first over the cellular interface
and the second over the Wi-Fi interface. Each path uses a different sequence number
space, but QUIC packets are encrypted using the same connection keys over both paths.
When a QUIC packet needs to be send, the packet scheduler selects the relevant path.
If a QUIC frame is lost over one path, it can be retransmitted over the other
paths.

Flexicast QUIC extends Multipath QUIC by requiring that each path uses different
encryption and decryption keys. As such, multiple clients can share the same additional
path which is encrypted with a different key than their respective unicast path.
The IP destination address of this new, shared path MAY be a multicast address,
so that all receivers with multicast support receive the same packet.

To illustrate the operation of Flexicast QUIC, we consider a scenario with
one source, S, and three receivers: R1, R2 and R3. R1 and R2 are attached to
a network where it is possible to create an IP multicast tree from S to R1 and R2.
R3 is attached to an IP network where unicast works perfectly, but IP multicast
does not. Once the Flexicast QUIC connection will be established, S will be
able to send data along the multicast tree to reach R1 and R2 and using unicast
to reach R3. This is illustrated in {{figfc1}}.


~~~~~~~~~~~~~~~~~~~~~~~~~~~
           IP Multicast tree
  S --------------------+-----> R1
  \                     |
   \                    +-----> R2
    \		Unicast network
     +------------------------> R3

~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #figfc1 title="A Flexicast QUIC connection where source S sends a data packet to R1 and R2 using a multicast tree and to R3 using unicast"}


A Flexicast QUIC connection starts with a handshake like a regular QUIC
connection. R1, R2 and R3 establish a connection to S using the flexicast_support
and the initial_max_path_id transport parameters.
The initial path between each receiver and the source are individually secured using
the keys derived at connection establishement. They remain open during the whole
communication and constitute a unicast bidirectional channel between the source and
each receiver. This path is used for two very different purposes.
First, it acts as a control channel and enables the source to
exchange control information with each receiver. Second, it can be used to transmit
or retransmit data that cannot be delivered along the multicast tree.

In addition, Flexicast QUIC receivers MAY create additional paths with shared keys to receive
content through multicast delivery.
This is illustrated in {{fig-flexicast-2}}.
Receivers R1 and R2 open a new path and use the same decryption key to read incoming packets.
The new path uses an IP multicast destination address and is unidirectional from the source
to the receivers. The remaining of this document refers to this shared, unidirectional path as the
__flexicast flow__.
The Flexicast QUIC source uses the flexicast flow to efficiently sends data to multiple receivers
simultaneously (R1 and R2 in {{fig-flexicast-2}}).
R1 and R2 use their unicast path, created at connection establishment, to send acknowledgment to
the source and to receive retransmitted frames.

In this example, R3 lies in a non-multicast-capable network, and cannot receive content through the
flexicast flow. Instead, it relies on its unicast path, created at connection establishment to receive
data from the source.

~~~~~~~~~~~~~~~~~~~~~~~~~~~
+----------------------------------------+       =======> flexicast flow
|   +----------------------------+       |
|   |                            |       |
|   |     .......................|.......|...
|   |     .  Multicast-network   |       |  .     <------> unicast path
|   |     .                      |       |  .
|   v     .  239.239.23.35:4433  v       |  .
+-> S ==============+==========> R1      |  .
    ^     .         \\                   v  .
    |     .          +=================> R2 .
    |     ...................................
    |
    |     ...................................
    |     . Unicast-only network            .
    +----------------------------> R3       .
          .                                 .
          ...................................
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #fig-flexicast-2 title="A Flexicast QUIC connection is composed of two types of paths: (1) one bidirectional, unique, unicast path between the source and each receiver and (2) a flexicast flow from the source to a set of receivers relyinf on a multicast distribution tree"}

At any point in time, if the network condition of a receiver listening to the Flexicast flow degrades, e.g., because the underlying multicast tree fails or because the receiver moves into a non-multicast network, it MAY leave the Flexicast flow and continue receiving content through its unicast path, similarly to R3.
The seamless transition between the flexicast flow and unicast path is possible because these are Multipath QUIC paths.

## Extensions to Multipath QUIC.

Flexicast QUIC modifies Multipath QUIC is several ways to enable the
utilization of a shared unidirectional flow as one of the Multipath QUIC paths.
The shared unidirectional flow can be transported on top of an IP multicast distribution tree,
which brings several challenges:

1. An IP multicast tree is unidirectional, in contrast with a Multipath QUIC path.

2. An IP multicast tree is identified by a pair <source address, group address>.

3. All the receivers attached to an IP Multicast tree receive the same packet.

Since an IP multicast tree is unidirectional, it is impossible to use the QUIC
path challenge/response mechanism to create the flexicast flow along an IP multicast tree.
Flexicast QUIC only uses path challenge/response on unicast paths. Furthermore,
the data received over the multicast path cannot be acknowledged over this path
since it is unidirectional. Flexicast QUIC receivers return acknowledgments
using MP_ACK frames over the unicast path to provide feedback to the server.

To join an IP multicast tree, the receivers must be informed of the source
and group addresses of this tree. The server uses the MC_ANNOUNCE frame to
advertise the identifier of the multicast tree that the receivers can join.

The Flexicast QUIC packets that are sent over an IP multicast tree must
be encrypted and authenticated. However, these packets cannot be encrypted using
the TLS keys derived during the QUIC connection establishment since each unicast
path has its own set of TLS keys. To secure the transmission of Flexicast QUIC
packets along an IP multicast tree, the source generates an independent set of
security keys and shares them using the MC_ANNOUNCE frame to all receivers over
the unicast path. Since this frame is sent over the unicast paths, it is authenticated
and encrypted using the TLS keys associated to each unicast path.

# Handshake Negotiation and Transport parameter

This extension defines a new transport parameter, used to negotiate the use of the flexicast extensiong during the connection handshake, as specified in {{QUIC-TRANSPORT}}.
The new transport parameter is defined as follows:

* flexicast_support (current version uses TDB-03): the presence of this transport parameter indicates support of the flexicast extension. The transport parameter contains two boolean values, respectively indicating support of IPv4 and IPv6 for multicast addresses. If an endpoint receives the flexicast_support transport parameter with both IPv4 and IPv6 supports set to false (0), it must close the connection with an error type of FC_PROTOCOL_VIOLATION.

The final support of the flexicast extension is conditioned to the support of the multipath extension, as defined in {{MULTIPATH-QUIC}}.
Since a Flexicast flow is a new multipath path, the support of multipath, with sufficient (e.g., at least 2) path ID, is required, as defined in {{Section 2 of MULTIPATH-QUIC}}.

An endpoint receiving the flexicast_support transport parameter from its peer, without support for multicast MUST ignore the flexicast_support transport parameter, as if the peer does not support the flexicast extension.

The extension does not change the definition of any transport parameter defined in {{Section 18.2 of QUIC-TRANSPORT}}.

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
