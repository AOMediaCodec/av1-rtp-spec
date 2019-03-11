RTP Payload Format For AV1 (v0.2.1)
===================================
{:.no_toc }

**Status:** AV1 RTC SG Proposal

## Abstract
{:.no_toc }

This specification defines the encapsulation of the AV1 video codec [AV1] for transport over RTP.  The payload format has wide applicability, from low bit-rate peer-to-peer usage, to
high bit-rate multi-party video conferences. This specification relies on the Dependency Descriptor RTP Header Extension [DD] to describe the temporal and spatial structure of the bitstream.

## Contents
{:.no_toc }

* TOC
{:toc}


## 1. Introduction

This specification describes an RTP payload specification applicable to the transmission
of video streams encoded using the [AV1 video codec][AV1].

In AV1 the smallest individual encoder entity presented for transport is the
Open Bitstream Unit (OBU). This specification allows both for fragmentation and
aggregation of OBUs in the same RTP packet, but explicitly disallows doing so
across frame boundaries.

This specification also provides several mechanisms through which scalability
structures are described. AV1 uses the concept of predefined scalability
structures. These are a set of commonly used picture prediction structures that
can be referenced simply via an indicator value (scalability_mode_idc, residing
in the sequence header). For cases that do not fall in any of the predefined
cases, there is a mechanism for describing the scalability structure. These
bitstream parameters greatly simplify the organization of the corresponding data
at the RTP payload format level.

## 2. Conventions, definitions and acronyms

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC2119].

Coded frame
:   The representation of one frame before the decoding process.

Frame
:   A frame in this document is synonymous to a Coded frame.

**Note:** In contrast, in [AV1], Frame is defined as the representation of video
signals in the spatial domain. {:.alert .alert-info }

**Note:** Multiple frames may be present at the same instant in time. {:.alert
.alert-info }

Open Bitstream Unit (OBU)
:   The smallest bitstream data framing unit in AV1. All AV1 bitstream
    structures are packetized in OBUs.

[Selective Forwarding Unit](SFU)
:   A middlebox that relays streams among transmitting and receiving clients by
    selectively forwarding packets ([RFC7667]).

## 3. Media format description

AV1 has similarities to other video codecs, but introduces a number of key
design changes as well. The AV1 codec can maintain up to eight reference frames,
of which up to seven can be referenced by any new frame. AV1 also allows a frame
to use another frame of a different spatial resolution as a reference frame.
Specifically, a frame may use any references whose width and height are between
1/2 that of the current frame and twice that of the current frame, inclusive.
This allows internal resolution changes without requiring the use of key frames.
These features together enable an encoder to implement various forms of coarse-
grained scalability, including temporal, spatial, and quality scalability modes,
as well as combinations of these, without the need for explicit scalable coding
tools.

Spatial and quality layers define different and possibly dependent
representations of a single input frame. For a given spatial layer, temporal
layers define different frame rates of video. Spatial layers allow a frame to be
encoded at different spatial resolutions, whereas quality layers allow a frame
to be encoded at the same spatial resolution but at different qualities (and
thus with different amounts of coding error). AV1 supports quality layers as
spatial layers without any resolution changes; hereinafter, the term "spatial
layer" is used to represent both spatial and quality layers.

This payload format specification provides for specific mechanisms through which
such temporal and spatial scalability layers can be described and communicated.

Temporal and spatial scalability layers are associated with non-negative integer
IDs. The lowest layer of either type has an ID of 0.

**Note:** Layer dependencies are constrained by the AV1 specification such that
a temporal layer with temporal_id T and spatial layer with spatial_id S are only
allowed to reference previously coded video data of temporal_id T' and
spatial_id S', where T' <= T and S' <= S.
{:.alert .alert-info }

## 4. Payload format

This section describes how the encoded AV1 bitstream is encapsulated in RTP packets. All
integer fields in the specifications are encoded as unsigned integers in network
octet order.

### 4.1 RTP header usage

The general RTP payload format follows the RTP header format [RFC3550] and
generic RTP header extensions [RFC8285], and is shown below.

The AV1 descriptor and AV1 aggregation header are described in this document.
The payload itself is a series of OBUs, each preceded by length information as
detailed later in this document.

<pre><code>
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|V=2|P|X|  CC   |M|     PT      |       sequence number         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           timestamp                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           synchronization source (SSRC) identifier            |
+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
|            contributing source (CSRC) identifiers             |
|                             ....                              |
+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
|         0x100         |  0x0  |       extensions length       |
+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
|   0x1(ID)     |  hdr_length   |                               |
+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+                               |
|                                                               |
|              AV1 descriptor (hdr_length #octets)              |
|                                                               |
|                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               | Other rtp header extensions...|
+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
| AV1 aggr hdr  |                                               |
+-+-+-+-+-+-+-+-+                                               |
|                                                               |
|                   Bytes 2..N of AV1 payload                   |
|                                                               |
|                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               :    OPTIONAL RTP padding       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
</code></pre>


**TODO:** Decide whether to keep the ascii art in this document.
{:.alert .alert-danger }

### 4.2  Dependency Descriptor RTP Header Extension

To facilitate the work of selectively forwarding portions of a scalable video bitstream, as is done by a Selective Forwarding Unit (SFU), certain information needs to be provided for each packet. The [generic descriptor](generic-descriptor.md) defines how this information is communicated.

### 4.3 AV1 aggregation header

The aggregation header is used to indicate if the first and/or last OBU in the
payload is fragmented.

The structure is as follows.

<pre><code>
 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+
|Z|Y|-|-|-|-|-|-|
+-+-+-+-+-+-+-+-+
</code></pre>

Z: set to 1 if the first OBU contained in the packet is a continuation of a
previous OBU, 0 otherwise

Y: set to 1 if the last OBU contained in the packet will continue in another
packet, 0 otherwise


### 4.4 Payload structure

The smallest high-level syntax unit in AV1 is the OBU. All AV1 bitstream
structures are packetized in OBUs. Each OBU has a header, which provides
identifying information for the contained data (payload).

This specification allows to packetize:

  * a single OBU;
  * an OBU fragment (initial, middle, or trailing part); or
  * a set of OBUs.

The design allows combination of aggregation and fragmentation, i.e., allow for
a set of OBUs in which the first and/or last one is fragmented.

It is not allowed, however, to aggregate OBUs across frame boundaries. In other
words, it is not allowed to have OBUs from different frames in the same RTP
packet. OBUs in a temporal unit that precede the first frame are considered part
of that first frame.

The payload contains a series of one or more OBUs (with the first and/or last
possibly being fragments). Each OBU (or OBU fragment) is preceded by a length
field. The length field is encoded using leb128. Leb128 is defined in the AV1
specification, and provides for a variable-sized, byte-oriented encoding of non-
negative integers where the first bit of each (little-endian) byte indicates if
additional bytes are used in the representation (AV1, Section 4.10.5).

The following figure shows an example payload where the length field is shown as
taking two bytes for the first and second OBU and one byte for the last (N) OBU.

<pre><code>
0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      OBU 1 size (leb128)      |                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               |
|                                                               |
|                  OBU 1 data                                   |
|                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               |      OBU 2 size (leb128)      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                                                               |
:                        OBU 2 data                             :
:                                                               :
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   OBU N size  |                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+  OBU N data  +-+-+-+-+-+-+-+-+
|                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
</code></pre>

Whether or not the first and/or last OBU is fragmented is signaled in the
aggregation header.

**TODO:** Add a paragraph that describes how a temporal unit is mapped into RTP
packets. I think a new temporal unit should start a new RTP packet. You skip the
temporal delimiter OBU, and then packetize the remaining OBUs in one or more
packets, with fragmentation as needed.
{:.alert .alert-danger }

* * *

<div class="alert alert-danger" markdown="1">

2019/01/15 Notes:

A packet must not include OBUs across a TD
All OBUs in a packet must have the same temporal_id and spatial_id
OBUs with layering information must not be aggregated with OBUs that don't have layering information (layering information=extension header).
If sequence header is present, it should (not must) be the first OBU in a packet.
Q: May not be needed.
A packet cannot include OBUs from different frames.
Q: for hidden frames, the various frames may all be required anyway, so maybe we allow aggregation of frames with associated hidden frames (of the same temporal unit).

Q: Should we support tile list OBUs?

Packetization Exercises:

TD   SH MD MD(0,0) FH(0,0) TG0(0,0) MD(0,1) FH(0,1) TG0(0,1) ...

 X   [.......................................................]
 X   [.. ........][.........................][.............][.........................................]


FH(0,1) TG0(0,1)  TD SH MD MD(0,0) FH(0,0) TG0(0,0) MD(0,1) FH(0,1) TG0(0,1)

[ ....................... X   ... ] not allowed, SH must be at beginning of packet

FH(0,1) TG0(0,1)  TD MD MD(0,0) FH(0,0) TG0(0,0) MD(0,1) FH(0,1) TG0(0,1)

[ ........................ x  .....]
</div>

## 5. References

### 5.1 Normative references

  * [RFC3550] for RTP header format
  * [RFC8285] for generic RTP header extensions
  * [Dependency Descriptor Extension](generic-descriptor.md) rtp header extension.
  * [RFC7667] RTP Topologies
  * [AV1 Bitstream & Decoding Process Specification][AV1]


**TODO:** flesh out list of normative references.
{:.alert .alert-danger }


### 5.2 Informative references

**TODO:** list informative references.
{:.alert .alert-danger }


[AV1]: https://aomedia.org/av1-bitstream-and-decoding-process-specification/
[RFC2119]: https://tools.ietf.org/html/rfc2119
[RFC3550]: https://tools.ietf.org/html/rfc3550
[RFC4585]: https://tools.ietf.org/html/rfc4585
[RFC7667]: https://tools.ietf.org/html/rfc7667
[RFC8285]: https://tools.ietf.org/html/rfc8285
[Selective Forwarding Unit]: https://webrtcglossary.com/sfu/


