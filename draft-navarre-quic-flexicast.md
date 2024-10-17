---
title: "Flexicast QUIC: combining unicast and multicast in a single QUIC connection"
abbrev: "FC-QUIC"
category: exp

docname: draft-navarre-quic-flexicast-latest
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
 - flexicast
venue:
  group: "QUIC"
  type: "Working Group"
  mail: "quic@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/quic/"
  github: "louisna/draft-navarre-quic-flexicast"
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
  QUIC-TLS: rfc9001
  MULTIPATH-QUIC: I-D.ietf-quic-multipath

informative:
  I-D.jholland-quic-multicast-05:
  I-D.pardue-quic-http-mcast:
  RFC1112:
  RFC4607:
  RFC9293:
  DIOT: DOI.10.1109/65.819174
  I-D.draft-michel-quic-fec:
  QUIRL: DOI.10.1109/TNET.2024.3453759
  rQUIC: DOI.10.1109/GLOBECOM38437.2019.9013401
  I-D.draft-krose-multicast-security:
  RFC4654:
  RFC9002:

--- abstract

This document proposes Flexicast QUIC, a simple extension to Multipath
QUIC that enables a source to send the same information to a set of
receivers using a combination of unicast paths and multicast distribution trees.


--- middle

# Introduction  {#sec-intro}


Starting from the initially efforts of Steve Deering {{RFC1112}}, the IETF
has developed a range of IP multicast solutions that enable the efficient
transmission of a single packet to a group of receivers. In this document,
we focus on Source-Specific Multicast for IP {{RFC4607}}, but the solution
proposed could also be applied to other forms of IP Multicast.

Although IP Multicast is not a new solution, it is not as widely used by
applications as transport protocols like TCP {{RFC9293}} or QUIC {{QUIC-TRANSPORT}}
do not provide multicast transport support.
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
extensions to QUIC have already been proposed in
{{I-D.pardue-quic-http-mcast}} and {{I-D.jholland-quic-multicast-05}}.


Flexicast QUIC extends Multipath QUIC {{MULTIPATH-QUIC}}. Thanks
to this recent QUIC extension, a QUIC connection can use several paths
simultaneously to exchange information between two hosts. This document
proposes a simple extension to Multipath QUIC that enables to share an
additional path between multiple receivers. The destination address of
this path can be a multicast IP address and rely on a multicast forwarding
mechanism (e.g., IP Multicast) to transmit the packets.
This document defines the core design of Flexicast QUIC starting from Multipath QUIC.
Side documents will further expand this design to add new features.

This document is organized as follows. After having specified some conventions
in {{sec-conventions}}, we provide a brief overview of Flexicast QUIC in
{{sec-flexicast}}. We describe in more details the Flexicast QUIC handshake
in {{sec-handshake}} and then the new QUIC frames in {{sec-frames}}.


# Conventions and Definitions {#sec-conventions}

{::boilerplate bcp14-tagged}

# Flexicast QUIC {#sec-flexicast}

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
    \            Unicast network
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

Flexicast flow-specific information, such as the IP multicast destination address, are advertised
by the source on the unicast path using the new frames defined in this document (see {{sec-frames}}).

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
utilization of a shared unidirectional path, the flexicast flow, as one of the Multipath QUIC paths.
The flexicast flow can be transported on top of an IP multicast distribution tree to reach several receivers simultaneously,
which brings several challenges:

1. An IP multicast tree is unidirectional, in contrast with a Multipath QUIC path.

2. An IP multicast tree is identified by a pair <source address, group address>.

3. All the receivers attached to an IP Multicast tree receive the same packet.

Since an IP multicast tree is unidirectional, it is impossible to use the QUIC
path challenge/response mechanism to create the flexicast flow along an IP multicast tree.
Flexicast QUIC only uses path challenge/response on unicast paths.
Flexicast QUIC replaces the explicit path challenge/response from Multipath QUIC
to create the flexicast flow by an implicit path creation.
Since the destination address of this new "path" is a multicast IP address, it is impossible for
receivers to test its reachability from the source.
An underlying mechanism (e.g., using IGMP) SHOULD provide feedback to the receiver on the
reachability of the flexicast flow at the IP multicast address provided by the source.

Furthermore, the data received over the flexicast flow cannot be acknowledged over this path
since it is unidirectional.
Because Flexicast QUIC uses Multipath QUIC, ACK frames are replaced by MP_ACK frames.
Flexicast QUIC receivers MUST send acknowledgments to the source using MP_ACK frames
sent on their unicast path.

To join the flexicast flow, the receivers must be informed of the source
and group addresses of this tree. The server uses the FC_ANNOUNCE frame to
advertise the identifier of the multicast tree that the receivers can join.

The Flexicast QUIC packets that are sent over the flexicast flow must
be encrypted and authenticated. However, these packets cannot be encrypted using
the TLS keys derived during the QUIC connection establishment since each unicast
path has its own set of TLS keys. To secure the transmission of Flexicast QUIC
packets along the flexicast flow, the source generates an independent set of
security keys and shares them using the FC_KEY frame (see {{fc-key-frame}}) to all receivers over
the unicast path. Since this frame is sent over the unicast paths, it is authenticated
and encrypted using the TLS keys associated to each unicast path.

{{sec-packet-protection}} gives more details on the changes from {{MULTIPATH-QUIC}} to protect packets sent on the flexicast flow.

# Handshake Negotiation and Transport parameter {#sec-handshake}

This extension defines a new transport parameter, used to negotiate the use of the flexicast extensiong during the connection handshake, as specified in {{QUIC-TRANSPORT}}.
The new transport parameter is defined as follows:

* flexicast_support (current version uses TBD-03): the presence of this transport parameter indicates support of the flexicast extension. The transport parameter contains two boolean values, respectively indicating support of IPv4 and IPv6 for multicast addresses. If an endpoint receives the flexicast_support transport parameter with both IPv4 and IPv6 supports set to false (0), it must close the connection with an error type of FC_PROTOCOL_VIOLATION.

The final support of the flexicast extension is conditioned to the support of the multipath extension, as defined in {{MULTIPATH-QUIC}}.
Since a Flexicast flow is a new multipath path, the support of multipath, with sufficient (e.g., at least 2) path ID, is required, as defined in {{Section 2 of MULTIPATH-QUIC}}.

An endpoint receiving the flexicast_support transport parameter from its peer, without support for multicast MUST ignore the flexicast_support transport parameter, as if the peer does not support the flexicast extension.

The extension does not change the definition of any transport parameter defined in {{Section 18.2 of QUIC-TRANSPORT}}.

# Initialisation of a Flexicast Flow

This section details how a Flexicast QUIC source advertises flexicast flows and how receivers join them.

{{fig-join-flexicast-flow}} illustrates how a Flexicast QUIC source and receiver exchange frames on the unicast path to advertise and join a flexicast flow.

~~~~~~~~~~~~~~~~~~~~~~~~~~~
Source                                        Receiver

QUIC handshake      <------------>      QUIC handshake

FC_ANNOUNCE         ------------->
                    <-------------      FC_STATE(JOIN)
FC_KEY              ------------->
                    <-------------    FC_STATE(LISTEN)
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #fig-join-flexicast-flow title="Flexicast flow announcement and join"}

## Flexicast Flow Announcement

A Flexicast QUIC source announces flexicast flows through the FC_ANNOUNCE frame (see {{fc-announce-frame}}).
Flexicast QUIC replaces the concept of Connection ID with the Flexicast Flow ID. Concretelly, these two terms refer to the same concept.
Since the flexicast flow is unidirectional, refering to this value as a "connection" ID would be quite ambiguous.

Through the FC_ANNOUNCE frame, the source advertises the Flexicast FLow ID of the flexicast flow to listen to.
As the Flexicast Flow ID will serve as a "destination Connection ID" for the flexicast flow, receivers willing to join the flexicast flow
MUST add this Flexicast Flow ID as a new "source Connection ID".
_TODO: how to handle the case were the receiver already has this Flexicast Flow ID as a source Connection ID?_

The FC_ANNOUNCE frame also contains routing information, i.e., the IP multicast destination address carrying flexicast flow packets, and the UDP destination port that the receivers MUST listen to.

The source MAY advertise multiple distinct flexicast flows to the same receivers, i.e., flexicast flows with distinct Flexicast Flow ID.
The source MAY advertise updated information about a specific flexicast flow by sending new FC_ANNOUNCE frames with increased sequence numbers.
Upon reception of a new FC_ANNOUNCE frame updating information of an existing flexicast flow with an increased sequence number compared to the last received FC_ANNOUNCE frame, a receiver MUST update flexicast flow information.
Upon reception of a new FC_ANNOUNCE frame updating information of an existing flexicast flow with a smaller sequence number compared to the last received FC_ANNOUNCE frame, a receiver MUST silently discard the new FC_ANNOUNCE frame.

## Joining a Flexicast Flow {#sec-join-fc-flow}

After receiving an FC_ANNOUNCE frame dvertising a flexicast flow, a receiver MAY decide to join it.
Flexicast flow management is handled with the FC_STATE frame (see {{fc-state-frame}}).
The receiver sends an FC_STATE frame with the Flexicast Flow ID of the flexicast flow it wants to join and the JOIN action.
_TODO: add a finite state machine of the receiver and source status._

At this point, the Flexicast QUIC source MAY still decide to unauthorize the receiver to listen to the flexicast flow.
However, a Flexicast QUIC source having advertised a flexicast flow through an FC_ANNOUNCE SHOULD accept a receiver to join
it after receiving the corresponding FC_STATE frame with the JOIN action.

In the positive case, the source sends an FC_KEY frame, containing the TLS keys necessary for the receiver to decrypt packets
received on the flexicast flow. The frame also contains the first packet number that the receiver SHOULD receive with this key.

Once the receiver received the FC_KEY frame and its underlying multicast network is ready to transmit packets with the IP multicast
address from the FC_ANNOUNCE frame as destination address, it sends an FC_STATE frame to the source with the LISTEN action.

The Flexicast QUIC source SHOULD NOT consider the receiver as a full member of the flexicast flow until it receives an
FC_STATE with the LISTEN action from the receiver.

## Underlying multicast network

After sending the FC_STATE frame with the JOIN action, the receiver MAY decide to notify the underlying multicast routing network
its willingness to receive packets from the flexicast flow that will be sent by the source on the IP multicast address advertised
in the FC_ANNOUNCE frame. The receiver MAY also wait to receive the FC_KEY frame of the corresponding flexicast flow to avoid
receiving packets that it will not be able to process before receiving the required TLS keys.

# Flexicast flow memberhsip management {#sec-leave}

A Flexicast QUIC source exactly knows the set of receivers listening to a specific flexicast flow.
Receivers MAY decide, at any point in time during the communication, to leave the flexicast flow.
However, receivers SHOULD only leave a flexicast flow only if network conditions are not met to
ensure a good quality of experience for the applications.
The exact metrics defining "good enough network conditions" is out of scope of this document,
and MUST be provided by the application.

## Receiver-side management

Receivers leave a flexicast flow membership by sending an FC_STATE frame with the LEAVE action.
Upon reception of this frame, the Flexicast QUIC source MUST NOT consider anymore the receiver
as a member of the flexicast flow, and it MUST continue transmitting data through the unicast path
with the receiver.
The receiver SHOULD drop any path state for the flexicast flow regarding.
The finite-state machine on the receiver-side MUST be reset to the same state as when it received
the FC_ANNOUNCE for the first time.

A receiver that previously left a flexicast flow MAY attempt to join again the same flow by
restarting the phases from {{sec-join-fc-flow}}. Receiving a new FC_KEY is mandatory if
the Flexicast QUIC source uses group-key management systems to ensure backward and forward
secrecy.

Applications SHOULD provide a mechanism to avoid malicious receivers to periodically join and leave
the same flexicast flow to saturate the Flexicast QUIC source.

## Source-side management

At any point in time, the Flexicast QUIC source MAY decide to unilaterally remove a receiver from a flexicast flow, by sending an FC_STATE frame with the LEAVE action.
A malicious receiver might decide to ignore the frame; that is why this decision is done unilaterally, since for the source, the receiver is out of the flexicast flow.

Reasons to decide on the source-side to remove receivers from the flexicast flow include:
- A receiver is a bottleneck on the flexicast flow, i.e., the bit-rate of the flexicast flow becomes too low because of this receiver.
  In that case, the bottleneck receiver is removed from the specific flexicast flow. The source can continue distributing the content either through
  the unicast path with this receiver, or by letting the receiver join another flexicast flow which transmits data at a lower bit-rate.
- A receiver experiences too many losses. Receivers send feedback to the source. Too many losses degrade the quality of experience for the applications.
  To recover from them, there is a need for retransmissions, which can take up to several RTTs. The source SHOULD decive to remove the receiver from the
  flexicast flow and continue receiving data through unicast, even temporarilly. A reason why it would be better to continue through the unicast path
  is that the underlying IP multicast tree may be failing.

The two aforementionned reasons could include malicious receivers whishing to degrade the overall performance of communication through the flexicast flow.
By letting the source unilaterally decide to remove receivers from a flexicast flow, the impact of malicious receivers is limited.

# Reliability

The reliability mechanism of Flexicast QUIC uses the standard QUIC mechanism.
This section only details the reliability mechanism for frames sent on the flexicast flow.
This document does not alter the reliability mechanism of QUIC as defined in {{RFC9002}} for packets sent on the unicast path.

Receivers regularly send MP_ACK frames back to the source on their unicast path. Receivers MUST NOT send MP_ACK on the flexicast flow since this path is unidirectional.
A Flexicast QUIC source receiving an MP_ACK from a receiver on the flexicast flow MUST close the connection with this receiver with an error of type FC_PROTOCOL_VIOLATION.
The MP_ACK frames sent by the receivers on their unicast path include the `space_id` field to refer to the flexicast flow.

A major change between {{RFC9002}}, and to some extend {{MULTIPATH-QUIC}} with this document is that a Flexicast QUIC takes into account acknowledgments from multiple receivers for the same data.
A Flexicast QUIC source MUST wait for the acknowledgment from all receivers listening to the flexicast flow before releasing state for the transmitted data.
An acknowledgment-aggregation mechanism MUST be implemented, at least on the Flexicast QUIC source, to advance its state when all members of the flexicast flow sent MP_ACK frames to acknowledge some packets sent on the flexicast flow.

Bottleneck or malicious receivers may slow down, and even block the transmission of data, thus impacting the quality of service for other receivers.
Similarly to {{RFC9002}}, a Flexicast QUIC source may decide to stop the communication with a specific receiver if packet retransmissions do not trigger MP_ACK frames from this receiver to advance the state of communication.
A Flexicast QUIC source SHOULD remove bottleneck or malicious receivers from the flexicast flow in that case, and MAY continue transmitting data through the unicast path with these receivers instead, thus relying on {{RFC9002}}, in case the severe losses occur because of the underlying multicast distribution tree.

The Flexicast QUIC source is responsible to decide whether to send retransmitted frames on the flexicast flow to all receivers, or specifically to receivers individually on their unicast path.
Receivers are not impacted by this choice since {{MULTIPATH-QUIC}} specification authorizes to retransmit frames on another path than the initial path on which they were sent.
The first case would be more efficient if multiple receivers lost the same packet; the second case is more interesting for isolated losses to avoid healthy receivers receiving duplicate frames.

# FLow and Congestion Control

TODO: find a good mechanism for Flow Control. There are several possibilities, like requiring receivers to send new limits to stay in the group, or assume that the limits are of XXX bytes, and increase this over time unilaterally.

TODO: what do we detail regarding the congestion control?

There are currently several possibilities regarding congestion control for Flexicast QUIC. A first idea is to maintain constant bit-rate delivery and exposing multiple flexicast flows operating at diffeent bit-rates.
As such, if a receiver sees degradation in its communication (e.g., congestion or increased delay in the network), it may switch to a lower bit-rate flexicast flow. However, this idea does not solve the problem of increasing the congestion in the network.

The other idea outlines the possibility to take into account all receiver-specific bit-rate to chose the overall bit-rate on the flexicast flow, e.g., by requiring that the flexicast flow bit-rate is the lowest bit-rate among all receivers listening to this flexicast flow.
Since receivers regularly send MP_ACK frames, it is possible for the source to adjust a per-receiver bit-rate (or congestion window) and use the minimum value as the value for the flexicast flow.
Of course, such method paves the way to malicious receivers decreasing on-purpose the flexicast flow bit-rate. Applications SHOULD provide to a Flexicast QUIC source a "minimum bit-rate" to ensure a minimum quality of service for the receivers.
Malicious or bottleneck receivers with a per-receiver bit-rate below this minimum bit-rate SHOULD be removed from the flexicast flow membership.
Since the Flexicast QUIC source can unilaterally evict such receivers, it can adjust the flexicast flow bit-rate without considering these receivers.

A mix of both approaches is also possible. A Flexicast QUIC source may expose multiple flexicast flow, each operating at different bit-rate windows. For example, a first window would operate at a minimum bit-rate of 2 Mbps and a maximum bit-rate of 5 Mbps; another flexicast flow would operate between 5 Mbps and 10 Mbps,...
Receivers falling below the 2 Mbps bit-rate SHOULD be evicted from flexicast delivery; a Flexicast QUIC source MAY decide to continue the delivery through the unicast path.

# Packet protection {#sec-packet-protection}

Packet protection for QUIC version 1 is specified in {{Section 5 of QUIC-TLS}}.
{{Section 4 of MULTIPATH-QUIC}} further expands this design when multiple paths with different packet number spaces are used.
Because the flexicast flow is shared among multiple receivers, this document modifies the design from {{Section 4 of MULTIPATH-QUIC}}.

## Protection Keys

This document extends the design from {{MULTIPATH-QUIC}} by requiring that all flexicast flows use different TLS protection keys. Of course, these protections keys MUST be different from the protection keys derived during the handshake between the source and any receiver.

The per-flexicast flow protection keys are derived by the Flexicast QUIC source. The required material, e.g., master secret and used algorithm, are transmitted to admitted received through the FC_KEY {{fc-key-frame}} on the unicast path.

Since no explicit path probing phase is used in Flexicast QUIC, the receiver can use the dedicated protection key context whenever receiving a packet from the flexicast flow directly after receiving the FC_KEY frame from the source.

## Nonce Calculation

{{Section 4 of MULTIPATH-QUIC}} further expands the computation of the Nonce from {{Section 5.3 of QUIC-TLS}} to integrate the least significant 32 bits of the Path ID to guarantee its uniqueness.
A flexicast flow is shared among multiple receivers simultaneously, thus requiring that all receivers share the same Path ID for the same flexicast flow. Since receivers are allowed to dynamically change the flexicast flows they listen, it is impossible to ensure that all receivers use the same Path ID for the same flexicast flow.

TODO: add example?

However, since each flexicast flow uses its own set of TLS keys, the computation of the Nonce is decorelated between any pair of flexicast flows, and between any flexicast flow and any unicast path with a receiver. It is therefore not mandatory anymore to use the Path ID to ensure the uniqueness of the Nonce.

To remain compliant with {{Section 4.1 of MULTIPATH-QUIC}}, Flexicast QUIC still uses the Path ID to compute the Nonce. However, to ensure that all receivers use the same Path ID no matter the flexicast flow they listen, the Path ID used for the nonce is always set to 1.

## Key Update

TODO

# New Frames {#sec-frames}

All frames defined in this document MUST only be sent in 1-RTT packets.

If an endpoint receives a flexicast-specific frame in a different packet type, it MUST close the connection with an error of type FRAME_ENCODING_ERROR.

Receipt of flexicast-specific frames related to a Flexicast Flow ID that is not added by an endpoint MUST be treated as a connection error of type FC_PROTOCOL_VIOLATION.

If an endpoint receives a flexicast-specific frame with a Flexicast Flow ID that it cannot process anymore (e.g., the flexicast flow might have been abandoned), it MUST silently ignore the frame.

The new frames introduced below are for control-specific purpose only, and MUST NOT be sent on the Flexicast flow. Receipt of any of the following frames on the Flexicast flow MUST be trated as a connection error of type FC_PROTOCOL_VIOLATION.

## FC_ANNOUNCE frame {#fc-announce-frame}

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

Source IP: The IP address of the multicast source, used for Single-Source Multicast.

Group IP: The IP address of the multicast group. The address MUST be a multicast address.

UDP Port: The UDP destination port.

Ack delay timer: A 64-bit unsigned integer containing the delay, in ms, between two acknowledgments from a receiver.

FC_ANNOUNCE frames are ack-eliciting. If a packet containing an FC_ANNOUNCE frame is considered lost, the peer SHOULD repeat it.

Sources are allowed to send multiple times FC_ANNOUNCE frames with an increasing sequence number for the same Flexicast Flow ID.
New FC_ANNOUNCE frames MAY contain updated information, e.g., a new Ack delay timer.

Sources are allowed to advertise multiple Flexicast flows by sending multiple parallel FC_ANNOUNCE frames with distinct Flexicast Flow IDs.
The Sequence number is linked to a specific Flexicast Flow ID. The same Sequence number can be used for two distinct Flexicast Flow IDs.

TODO: the frame should also contain a `Status` field to indicate if the flexicast flow is retired?

## FC_STATE frame {#fc-state-frame}

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

JOIN (0x01): The receiver joins the Flexicast flow.

LEAVE (0x02): The receiver leaves the Flexicast flow.

READY (0x03): The receiver is ready to receive content on the Flexicast flow.

The actions JOIN and READY are receiver-specific. These actions MUST NOT be sent inside an FC_STATE frame sent by the source. A receiver receiving an FC_STATE frame with any of the following actions MUST treat it as a connection error of type FC_PROTOCOL_VIOLATION.
The action LEAVE MAY be sent by both the receiver and the source, as detailed in {{sec-leave}}.

## FC_KEY frame {#fc-key-frame}

The FC_KEY frame informs a receiver with the decryption key of a Flexicast flow joined by the receiver.
FC_KEY frames MUST NOT be sent by the receiver. A Flexicast QUIC source receiving an FC_KEY frame MUST clone the connection with a connection error of type FC_PROTOCOL_VIOLATION.

FC_KEY frames are formatted as shown in {{fig-fc-key-frame-format}}.

~~~
FC_KEY Frame {
    Type (i) = TDB-02,
    Length(8),
    Flexicast Flow ID(8..160),
    Sequence number (i),
    Packet number (i),
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

Packet number: The first packet number that the receiver must receive with this key,

Key length: A var-int indicating the length of the decryption key.

Key: Byte-sequence of the decryption key of the Flexicast flow.

Algorithm: The bit-encoded algorithm used for decryption, defined in Section {{fc-key-algorithm}}.

FC_KEY frames are ack-eliciting. If a packet containing an FC_STATE frame is considered lost, the peer SHOULD repeat it.

The source MAY send new FC_KEY frames with an increased sequence number to notify a new decryption key.
This mechanism can be used to provide backward and forward secrecy with dynamic Flexicast groups.

### FC_KEY algorithms {#fc-key-algorithm}

TODO: define the bit-encoded algorithms.

# Discussion {#sec-further}


This document has defined a simple extension to Multipath QUIC that enables a
QUIC connection to simulatenously use unicast paths and multicast trees
to deliver the same data to a set of receivers. The proposed protocol can be extended
in different ways and improvements will be proposed in separate documents. We
briefly describe some of these possible extensions.

This version of Flexicast QUIC uses the existing QUIC mechanisms to retransmit
lost frames. It is well known that Forward Erasure Correction can improve the
performance of multicast transmission when losses occur. Several authors have
proposed techniques to add Forward Erasure Correction to QUIC {{QUIRL}}, {{rQUIC}},
{{I-D.draft-michel-quic-fec}}. FEC can be sent a priori to enable receivers
to recover from different packet losses without having to wait for retransmissions or
a posteriori by sending a repair symbol that enables different receivers to recover
different lost frames.

This version of Flexicast QUIC uses a key shared between the source and all receivers
to authenticate and encrypt the data sent by the source over the multicast tree.
A malicious receiver who has received the shared keys could inject fake data over
the multicast tree provided that it can spoof the IP address of the source. Techniques
have been proposed to authenticate the frames sent by the source {{I-D.draft-krose-multicast-security}}.
Subsequent documents will detail how Flexicast QUIC can be extended to support such
techniques.

Multipath QUIC assumes that the congestion control mechanism operates per path. For
Flexicast QUIC, a different congestion control mechanism will be required for the
unicast paths and the multicast tree. For the former, the QUIC congestion control
mechanisms {{RFC9002}} are applicable. For the latter, multicast specific congestion
control mechanisms such as {{RFC4654}} will be required. The current prototype
uses a variant of the CUBIC congestion control.



# Security Considerations

To be provided in the next version of this document.


# IANA Considerations {#sec-iana}

IANA is requested to assign three new QUIC frame types from the
"QUIC Frame Types" registry available at
https://www.iana.org/assignments/quic/quic.xhtml#quic-frame-types

 - TBD-00 for the FC_ANNOUNCE frame defined in {{fc-announce-frame}}
 - TBD-01 for the FC_STATE frame defined in {{fc-state-frame}}
 - TBD-02 for the FC_KEY frame defined in {{fc-key-frame}}



IANA is requested to assign a new QUIC transport parameter
from the "QUIC Transport Parameters" registry available at
https://www.iana.org/assignments/quic/quic.xhtml#quic-transport


 - TBD-03 for the flexicast_support transport parameter defined in {{sec-handshake}}


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
