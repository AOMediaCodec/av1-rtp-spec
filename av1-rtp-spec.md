﻿
RTP Payload Format For AV1 (v0.2.1)
===================================
{:.no_toc }

**Status:** AV1 RTC SG Working Draft (WD)


## Abstract
{:.no_toc }

This document describes an RTP payload format for the [AV1 video codec][AV1]. The
payload format has wide applicability, from low bit-rate peer-to-peer usage, to
high bit-rate multi-party video conferences. It includes provisions for temporal
and spatial scalability.


## Status of this document
{:.no_toc }

This document is a working draft of the Real-Time Communications Subgroup.


## Contents
{:.no_toc }

* TOC
{:toc}


## 1. Introduction

This document describes an RTP payload specification applicable to the transmission
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
: The representation of one frame before the decoding process.

Frame
: A frame in this document is synonymous to a Coded frame.

**Note:** In contrast, in [AV1], Frame is defined as the representation of
video signals in the spatial domain.
{:.alert .alert-info }

**Note:** Multiple frames may be present at the same instant in time.
{:.alert .alert-info }

Media-Aware Network Element (MANE
: A middlebox that relays streams among transmitting and receiving clients
by selectively forwarding packets and which may have access to the media ([RFC6184]).

OBU element
: An OBU, or a fragment of an OBU, contained in an RTP packet.

Open Bitstream Unit (OBU)
: The smallest bitstream data framing unit in AV1. All AV1 bitstream structures
  are packetized in OBUs.

Selective Forwarding Unit (SFU)
: A middlebox that relays streams among transmitting and receiving clients by
  selectively forwarding packets without requiring access to the media ([RFC7667]).


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

This section describes how the encoded AV1 bitstream is encapsulated in RTP. All
integer fields in the specifications are encoded as unsigned integers in network
octet order.


### 4.1 RTP header usage

The general RTP payload format follows the RTP header format [RFC3550] and
generic RTP header extensions [RFC8285], and is shown below.

The dependency descriptor and AV1 aggregation header are described in this document. The payload itself is a series of OBU elements, preceded by length information as detailed later in this document.

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
|          dependency descriptor (hdr_length #octets)           |
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

### 4.2  RTP header Marker bit (M)

The RTP header Marker bit MUST be set equal to 0 if the packet is not the last packet of the temporal unit, it SHOULD be set equal to 1 otherwise.

**Note:** It is possible for a receiver to receive the last packet of a temporal unit without the marker bit being set equal to 1, and a receiver should be able to handle this case. The last packet of a temporal unit is also indicated by the next packet, in RTP sequence number order, having an incremented timestamp.

### 4.3  Dependency Descriptor RTP Header Extension

To facilitate the work of selectively forwarding portions of a scalable video
bitstream, as is done by a Selective Forwarding Unit (SFU), certain information
needs to be provided for each packet. The appendix of this specification defines
how this information is communicated.


### 4.4 AV1 aggregation header

The aggregation header is used to indicate if the first and/or last OBU element in the
payload is a fragment of an OBU.

The structure is as follows.

<pre><code>
 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+
|Z|Y| W |N|-|-|-|
+-+-+-+-+-+-+-+-+
</code></pre>

Z: set to 1 if the first OBU element is an OBU fragment that is a continuation of an OBU fragment from the previous packet, 0 otherwise.

Y: set to 1 if the last OBU element is an OBU fragment that will continue in the next packet, 0 otherwise.

W: two bit field that describes the number of OBU elements in the packet. This field MUST be set equal to 0 or equal to the number of OBU elements contained in the packet. If set to 0, each OBU element MUST be preceded by a length field. If not set to 0 the last OBU element MUST NOT be preceded by a length field. Instead, the length of the last OBU element contained in the packet can be calculated as follows:

<pre><code>
Length of the last OBU element = length of the RTP payload - length of aggregation header - length of previous OBU elements including length fields
</code></pre>

N: set to 1 if the packet is the first packet of a coded video sequence,
0 otherwise.
**Note:** if N equals 1 then Z must equal 0.

### 4.5 Payload structure

The smallest high-level syntax unit in AV1 is the OBU. All AV1 bitstream
structures are packetized in OBUs. Each OBU has a header, which provides
identifying information for the contained data (payload).

The payload contains a series of one or more OBU elements. The design allows for a combination of aggregation and fragmentation of OBUs, i.e., a set of OBU elements in which the first and/or last element is a fragment of an OBU.

The length field is encoded using leb128. Leb128 is defined in the AV1
specification, and provides for a variable-sized, byte-oriented encoding of non-
negative integers where the first bit of each (little-endian) byte indicates if
additional bytes are used in the representation (AV1, Section 4.10.5).

Whether or not the first and/or last OBU element is a fragment of an OBU is signaled in the aggregation header. Fragmentation may occur regardless of how the W field is set.

The AV1 specification allows OBUs to have an optional size field called 
obu_size (also leb128 encoded), signaled by the obu_has_size_field flag 
in the OBU header. To minimize overhead, the obu_has_size_field flag SHOULD 
be set to zero in all OBUs.

The following figure shows an example payload where the length field is shown as
taking two bytes for the first and second OBU elements and one byte for the last (N) OBU element.

<pre><code>
0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Z|Y|0 0|N|-|-|-|  OBU element 1 size (leb128)  |               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+               |
:                                                               :
:                      OBU element 1 data                       :
:                                                               :
|                                                               |
|                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               |  OBU element 2 size (leb128)  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
:                                                               :
:                       OBU element 2 data                      :
:                                                               :
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
:                                                               :
:                              ...                              :
:                                                               :
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|OBU e... N size|                                               |
+-+-+-+-+-+-+-+-+       OBU element N data      +-+-+-+-+-+-+-+-+
|                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
</code></pre>

The following figure shows an example payload containing two OBU elements where the last OBU element omits the length field (and the W field is set to 2). The size of the last OBU element can be calculated given the formula described in section 4.3.

<pre><code>
OBU element example size calculation:
Total RTP payload size    = 303 bytes
AV1 aggregation header    = 1 byte
OBU element 1 size        = 2 bytes
OBU element 1 data        = 200 bytes
OBU element 2 data        = 303 - 1 - (2 + 200) = 100 bytes

0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Z|Y|1 0|N|-|-|-|  OBU element 1 size (leb128)  |               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+               |
|                                                               |
:                                                               :
:                      OBU element 1 data                       :
:                                                               :
|                                                               |
|                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               |                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               |
|                                                               |
:                                                               :
:                      OBU element 2 data                       :
:                                                               :
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
</code></pre>

## 5. Packetization rules

Each RTP packet MUST contain OBUs that belong to a single temporal unit.

The temporal delimiter OBU, if present, SHOULD be removed when transmitting, and
MUST be ignored by receivers.

If a sequence header OBU is present in an RTP packet and operating_points_cnt_minus_1 > 0 then for any number i where 0 <= i < operating_points_cnt_minus_1 the following MUST be true: (operating_point_idc[i] & operating_point_idc[i+1]) == operating_point_idc[i+1].

A sender MAY produce a sequence header with operating_points_cnt_minus_1 = 0 and operating_point_idc[0] = 0xFFF and seq_level_idx[0] = 0. In such case, seq_level_idx[0] does not reflect the level of the operating point.

**Note:** The intent is to disable OBU dropping in the decoder. To ensure a decoder’s capabilities are not exceeded, OBU filtering should instead be implemented at the system level (e.g., SFU).

If more than one OBU contained in an RTP packet has an OBU extension header then
the values of the temporal_id and spatial_id must be the same in all such OBUs
in the RTP packet.

If a sequence header OBU is present in an RTP packet it	SHOULD be the first OBU
in the packet. OBUs that are not associated with a particular layer (and thus do not have
an OBU extension header) SHOULD be in the beginning of a packet, following the
sequence header OBU if present.

A sequence header OBU SHOULD be included in the base layer when scalable encoding is used,
and it SHOULD be aggregated with each spatial layer in the case of simulcast.

Tile list OBUs are not supported. They SHOULD be removed when transmitted, and MUST be ignored by receivers.

### 5.1 Examples

The following are example packetizations of OBU sequences. A two-letter notation is used to identify the OBU type: FH frame header, TG - tile group, FR - frame, SH - sequence header, TD - temporal delimitered, MD - metadata. Parentheses after the type indicate the temporal_id and spatial_id combination. For example "TG(0,1)" indicates a tile group OBU with temporal_id equal to 0 and spatial_id equal to 1.

The following is an example coded video sequence:

<pre><code>
    TD SH MD MD(0,0) FH(0,0) TG0(0,0) MD(0,1) FH(0,1) TG(0,1)
</code></pre>

This sequence could be packetized as follows. First, the TD OBU is dropped. Then, the following packetization
grouping (indicated using square brackets) may be used.

<pre><code>
    [ SH MD MD(0,0) FH(0,0) TG(0,0) ] [ MD(0,1) FH(0,1) TG(0,1) ]
</code></pre>

It is also possible to send each OBU in its own RTP packet:

<pre><code>
    [ SH ] [ MD ] [ MD(0,0) ] [ FH(0,0) ] [ TG(0,0) ] ...
</code></pre>

The following packetization grouping would not be allowed, since it combines data from different spatial layers
in the same packet.

<pre><code>
    [ SH MD MD(0,0) FH(0,0) TG(0,0) MD(0,1) FH(0,1) TG(0,1) ]
</code></pre>


## 6. MANE and SFU Behavior

If a packet contains an OBU with an OBU extension header then the entire packet
is considered associated with the layer identified by the temporal_id and spatial_id combination that are indicated in the extension header.
If a packet does not contain any OBU with an OBU extension header, then it is considered
to be associated with all operating points.

A MANE or SFU performs its function based on a target operating point. A packet MUST be forwarded if it is associated with the target
operating point, or with all operating points. A packet SHOULD NOT be forwarded if it is associated with an operating point different than the
target operating point.

SFUs can operate using end-to-end encryption, i.e., with encrypted payload, using the RTP header extension defined in
Appendix A. The extension exposes the layer information of a packet so that the SFU can make the appropriate forwarding
decision.

### 6.1. Simulcast

The [AV1 Bitstream & Decoding Specification][AV1] enables multiple simulcast encodings to be provided within
a single bitstream. Within the RTP payload defined in this specification, simulcast encodings can be transported
each on a separate RTP stream [I-D.ietf-avtext-rid] with Session Description Protocol (SDP) signaling as
described in [I-D.ietf-mmusic-sdp-simulcast][I-D.ietf-mmusic-rid]. Alternatively, simulcast encodings can
be transported on a single RTP stream, in which case Restriction Identifiers (RIDs) are not used. In either
case, simulcast transport MUST only be used to convey multiple encodings from the same source.

## 7. Payload Format Parameters

This payload format has three optional parameters.

### 7.1. Media Type Definition


* Type name:
   * **video**
* Subtype name:
  * **AV1**
* Required parameters:
  * None.
* Optional parameters:
  * These parameters are used to signal the capabilities of a receiver implementation. If the implementation is willing to receive media, **profile** and **level-idx** parameters MUST be provided. These parameters MUST NOT be used for any other purpose.
    * **profile**: The value of **profile** is an integer indicating highest AV1 profile supported by the receiver. The range of possible values is identical to **seq_profile** syntax element specified in [AV1]
    * **level-idx**: The value of **level-idx** is an integer indicating the highest AV1 level supported by the receiver. The range of possible values is identical to **seq_level_idx** syntax element specified in [AV1]
    * **tier**: The value of **tier** is an integer indicating tier of the indicated level.  The range of possible values is identical to **seq_tier** syntax element specified in [AV1]. If parameter is not present, level's tier is to be assumed equal to 0

* Encoding considerations:
  * This media type is framed in RTP and contains binary data; see Section 4.8 of [RFC6838].
* Security considerations:
  * See Section 10.
* Interoperability considerations:
  * None.
* Published specification:
  * AV1 bitstream format [AV1]
* Applications which use this media type:
  * Video over IP, video conferencing.
* Fragment identifier considerations:
  * N/A.
* Additional information:
  * None.
* Person & email address to contact for further information:
  * TODO
* Intended usage:
  * COMMON
* Restrictions on usage:
  *  This media type depends on RTP framing, and hence is only defined for transfer via RTP [RFC3550].
* Author:
  * TODO
* Change controller:
  * AoMedia Codec Group, RTC sub-group

### 7.2 SDP Parameters
The receiver MUST ignore any fmtp parameter unspecified in this document.

#### 7.2.1 Mapping of Media Subtype Parameters to SDP
The media type video/AV1 string is mapped to fields in the Session Description Protocol (SDP) [RFC4566] as follows:
* The media name in the "m=" line of SDP MUST be video.
* The encoding name in the "a=rtpmap" line of SDP MUST be AV1 (the media subtype).
* The clock rate in the "a=rtpmap" line MUST be 90000.
* The parameters "**profile**", and "**level-idx**", MUST be included in the "a=fmtp" line of SDP if SDP is used to declare receiver capabilities. These parameters are expressed as a media subtype string, in the form of a semicolon separated list of parameter=value pairs.
* Parameter "**tier**" COULD be included alongside "**profile**" and "**level-idx** parameters in "a=fmtp" line if indicated level supports tier different to 0.

#### 7.2.2  Usage with the SDP Offer/Answer Model

When AV1 is offered over RTP using SDP in an Offer/Answer model [RFC3264] for negotiation for unicast usage, the following limitations and rules apply:
  *  The media format configuration is identified by **level-idx**, **profile** and **tier**.  Answerer SHOULD maintain all parameters. These media configuration parameters are asymmetrical and answerer COULD declare its own media configuration if answerer capabilities are different to offerer.
     *  The  profile to use in the offerer-to-answerer direction MUST be lesser or equal to the profile the answerer supports for receiving, and the profile to use in the answerer-to-offerer direction MUST be lesser or equal to the profile the offerer supports for receiving.
     *  The level to use in the offerer-to-answerer direction MUST be lesser or equal to the level the answerer supports for receiving, and the level to use in the answerer-to-offerer direction MUST be lesser or equal to the level the offerer supports for receiving.
     *  The tier to use in the offerer-to-answerer direction MUST be lesser or equal to the tier the answerer supports for receiving, and the tier to use in the answerer-to-offerer direction MUST be lesser or equal to the tier the offerer supports for receiving.

#### 7.2.3  Usage in Declarative Session Descriptions

 When AV1 over RTP is offered with SDP in a declarative style, as in Real Time Streaming Protocol (RTSP) [RFC2326] or Session Announcement Protocol (SAP) [RFC2974], the following considerations apply.
 * All parameters capable of indicating both stream properties and receiver capabilities are used to indicate only stream properties. In this case, the parameters **profile**, **level-idx** and **tier** declare only the values used by the stream, not the capabilities for receiving streams.  
 * A receiver of the SDP is required to support all parameters and values of the parameters provided; otherwise, the receiver MUST reject (RTSP) or not participate in (SAP) the session. It falls on the creator of the session to use values that are expected to be supported by the receiving application.

### 7.3 Example
An example of media representation in SDP is as follows:

* m=video 49170 RTP/AVPF 98
* a=rtpmap:98 AV1/90000
* a=fmtp:98 profile=2; level-idx=8; tier=1;

In the following example, the offer is accepted with level upgrading. The level to use in the offerer-to-answerer
direction is Level 2.0, and the level to use in the answerer-to-offerer direction is Level 3.0/Tier 1.  The answerer is allowed to send at
any level up to and including Level 2.0, and the offerer is allowed to send at any level up to and including Level 3.0/Tier 1:

Offer SDP:
* m=video 49170 RTP/AVPF 98
* a=rtpmap:98 AV1/90000
* a=fmtp:98 profile=0; level-idx=0;
*
Answer SDP:
* m=video 49170 RTP/AVPF 98
* a=rtpmap:98 AV1/90000
* a=fmtp:98 profile=0; level-idx=4; tier=1;

In the following example, an offer is made by a conferencing server to receive 3 simulcast streams.  The answerer agrees to send 3 simulcast streams at different resolutions.

Offer SDP:
*   m=video 49170 UDP/TLS/RTP/SAVPF 98
*   a=mid:0
*   a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
*   a=extmap:2 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
*   a=extmap:3 urn:3gpp:video-orientation
*   a=extmap:4 https://aomediacodec.github.io/av1-rtp-spec/#dependency-descriptor-rtp-header-extension
*   a=sendrecv
*   a=rtcp-mux
*   a=rtcp-rsize
*   a=rtpmap:98 AV1/90000
*   a=fmtp:98 profile=2; level-idx=8; tier=1;
*   a=rtcp-fb:98 ccm fir
*   a=rtcp-fb:98 nack
*   a=rtcp-fb:98 nack pli
*   a=rid:f recv
*   a=rid:h recv
*   a=rid:q recv
*   a=simulcast:recv f;h;q

Answer SDP:
*   m=video 48120 UDP/TLS/RTP/SAVPF 98
*   a=mid:0
*   a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
*   a=extmap:2 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
*   a=extmap:3 urn:3gpp:video-orientation
*   a=extmap:4 https://aomediacodec.github.io/av1-rtp-spec/#dependency-descriptor-rtp-header-extension
*   a=sendrecv
*   a=rtcp-mux
*   a=rtcp-rsize
*   a=rtpmap:98 AV1/90000
*   a=fmtp:98 profile=2; level-idx=8; tier=1;
*   a=rtcp-fb:98 ccm fir
*   a=rtcp-fb:98 nack
*   a=rtcp-fb:98 nack pli
*   a=rid:f send
*   a=rid:h send
*   a=rid:q send
*   a=simulcast:send f;h;q

In the following example, an offer is made by a browser to send a single RTP stream to a conferencing server.
This single stream could include include any AV1 scalability mode, including "S" modes involving simulcast.

Offer SDP:
*   m=video 49170 UDP/TLS/RTP/SAVPF 98
*   a=mid:0
*   a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
*   a=extmap:3 urn:3gpp:video-orientation
*   a=extmap:4 https://aomediacodec.github.io/av1-rtp-spec/#dependency-descriptor-rtp-header-extension
*   a=sendrecv
*   a=rtcp-mux
*   a=rtcp-rsize
*   a=rtpmap:98 AV1/90000
*   a=fmtp:98 profile=2; level-idx=8; tier=1;
*   a=rtcp-fb:98 ccm fir
*   a=rtcp-fb:98 nack
*   a=rtcp-fb:98 nack pli

Answer SDP:
*   m=video 48120 UDP/TLS/RTP/SAVPF 98
*   a=mid:0
*   a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
*   a=extmap:3 urn:3gpp:video-orientation
*   a=extmap:4 https://aomediacodec.github.io/av1-rtp-spec/#dependency-descriptor-rtp-header-extension
*   a=sendrecv
*   a=rtcp-mux
*   a=rtcp-rsize
*   a=rtpmap:98 AV1/90000
*   a=fmtp:98 profile=2; level-idx=8; tier=1;
*   a=rtcp-fb:98 ccm fir
*   a=rtcp-fb:98 nack
*   a=rtcp-fb:98 nack pli

## 8. Feedback Messages

## 8.1.  Full Intra Request (FIR)

   The Full Intra Request (FIR) [RFC5104] RTCP feedback message allows a
   receiver to request a Decoder Refresh Point of an encoded stream.

   Upon reception of FIR, the AV1 sender MUST send a new coded video sequence
   (see section 7.5 of the [AV1] bitstream specification) as soon as possible.
   
   **Note** If simulcast is used and simulcast streams are transported using multiple SSRCs,
   an AV1 sender must send new coded video sequences for every SSRC indicated in the FIR message
   and might send new coded video sequences for other simulcast streams. If an AV1 bitstream
   contains several spatial layers without inter-layer dependencies, an AV1 sender must send
   a new coded video sequence for each independent spatial layer.


## 8.2.  Layer Refresh Request (LRR)

   The Layer Refresh Request [I-D.ietf-avtext-lrr] allows a receiver to
   request a single layer of a spatially or temporally encoded stream to
   be refreshed, i.e, allow it to be decodable using information currently
   available in a decoder, without necessarily affecting the stream's other
   layers.

               +---------------+---------------+
               |0|1|2|3|4|5|6|7|0|1|2|3|4|5|6|7|
               +---------------+-----------+---+
               |   RES   | TID |   RES     |SID|
               +---------------+-----------+---+

                                 Figure 4

   AV1 streams MUST use the Layer Reefresh Request format defined for VP9 [I-D.ietf-payload-vp9] 
   (see Section 5.3), with the high order bit of its three-bit SID fieldset to 0. Figure 4 shows the 
   format for AV1 streams. Notice that SID here is two bits.  SID is associated with AV1's spatial_id
   and TID is associatd with AV1's temporal_id. See Sections 2, 5.3.3, and 6.2.3 of the AV1 bitstream
   specification [AV1] for details on the temporal_id (TID) and spatial_id (SID) fields.
   
   Both "RES" fields MUST be set to 0 on transmission and ignored on reception. 
   
   Identification of a layer refresh frame may be performed by examining the 
   coding dependency structure of the coded video sequence it belongs to.
   This may be provided by the scalability metadata (Sections 5.8.5 and 6.7.5
   of [AV1]), either in the form of a pre-defined scalability mode or through a
   scalability structure (Sections 5.8.6 and 6.7.6 of [AV1]). Alternatively,
   the Dependency Descriptor RTP header extension that is specified in Appendix
   A of this document may be used.

## 9. IANA Considerations

   Upon publication, a new media type, as specified in Section 7.1 of this document, will be
   registered with IANA.

## 10. Security Considerations

   RTP packets using the payload format defined in this document are subject
   to the security considerations discussed in the RTP specification [RFC3550]
   and in any appropriate RTP profile.
   This implies that confidentiality of the media streams is achieved by
   encryption, for example, through the application of SRTP [RFC3711].
   A potential denial-of-service threat exists for data
   encodings using compression techniques that have non-uniform
   receiver-end computational load.  The attacker can inject
   pathological datagrams into the stream that are complex to decode and
   that cause the receiver to be overloaded. Therefore, the usage of data origin
   authentication and data integrity protection of at least the RTP
   packet is RECOMMENDED, for example, with SRTP [RFC3711]. End-to-end encryption
   helps mitigate these attacks [perc-double].

   Note that the appropriate mechanism to ensure confidentiality and
   integrity of RTP packets and their payloads is very dependent on the
   application and on the transport and signaling protocols employed.
   Thus, although SRTP is given as an example above, other possible
   choices exist [RFC7202].

   Decoders MUST exercise caution with respect to the handling of
   reserved OBU types and reserved metadata OBU types, particularly if
   they contain active elements, and MUST restrict their domain of
   applicability to the presentation containing the stream. The safest
   way is to simply discard these OBUs.

   When integrity protection is applied to a stream, care MUST be taken
   that the stream being transported may be scalable; hence a receiver
   may be able to access only part of the entire stream.

   End-to-end security with either authentication, integrity, or
   confidentiality protection will prevent a MANE from performing media-
   aware operations other than discarding complete packets. The use of the
   Dependency Descriptor RTP extension described in Appendix A allows discarding
   of packets in a media-aware way even when confidentiality protection is used.
   Repacketization by a MANE requires access to the media payload.


## 11. References


### 11.1 Normative references

 
  * [AV1](https://aomediacodec.github.io/av1-spec/av1-spec.pdf "AV1 1.0.0 with Errata1") **AV1 Bistream & Decoding Process Specification, Version 1.0.0 with Errata 1**, January 2019.

  * [RFC3550](https://tools.ietf.org/html/rfc3550 "RFC3550") **RTP: A Transport Protocol for Real-Time Applications**, H. Schulzrinne, S. Casner, R. Frederick, and V. Jacobson, July 2003.
  
  * [RFC5104](https://tools.ietf.org/html/rfc5104 "RFC5104") **Codec Control Messages in the RTP Audio-Visual Profile with Feedback (AVPF)**, S. Wenger, U. Chandra, M. Westerlund, and B. Burman, February 2008. 

  * [RFC6184](https://tools.ietf.org/html/rfc6184 "RFC6184") **RTP Payload Format for H.264 Video**, Y.-K. Wang, R. Even, T. Kristensen, and R. Jesup, May 2011.

  * [RFC8285](https://tools.ietf.org/html/rfc8285 "RFC8285") **A General Mechanism for RTP Header Extensions for generic RTP header extensions**, D. Singer, H. Desineni, and R. Even, October 2017.
  
  * [RFC7667](https://tools.ietf.org/html/rfc7667 "RFC7667") **RTP Topologies**, M. Westerlund and S. Wenger, November 2015.
  
 * [I-D.ietf-avtext-lrr](https://tools.ietf.org/html/draft-ietf-avtext-lrr-07 "ietf-avtext-lrr")  **The Layer Refresh Request (LRR) RTCP Feedback Message**, J. Lennox, D. Hong, J. Uberti, S. Holmer, and M. Flodman, June 29, 2017.
 
  * [I-D.ietf-avtext-rid](https://tools.ietf.org/html/draft-ietf-avtext-rid "ietf-avtext-rid")  ** RTP Stream Identifier Source Description (SDES)**, A. Roach, S. Nandakumar and P. Thatcher, October 06, 2016.
  
  * [I-D.ietf-mmusic-rid](https://tools.ietf.org/html/draft-ietf-mmusic-rid "ietf-mmusic-rid")  ** RTP Payload Format Restrictions**, A. Roach, May 15, 2018.
  
  * [I-D.ietf-mmusic-sdp-simulcast](https://tools.ietf.org/html/draft-ietf-mmusic-sdp-simulcast "ietf-mmusic-sdp-simulcast")  ** Using Simulcast in SDP and RTP Sessions**, B. Burman, M. Westerlund, S. Nandakumar and M. Zanaty, March 5, 2019.


### 11.2 Informative references

* [RFC3711](https://tools.ietf.org/html/rfc3711 "RFC3711") **The Secure Real-time Transport Protocol (SRTP)**, M. Baugher, D. McGrew, M. Naslund, E. Carrara, and K. Norrman, March 2004.

* [RFC7202](https://tools.ietf.org/html/rfc7202 "RFC7202") **Securing the RTP Framework: Why RTP Does Not Mandate a Single Security Solution**, C. Perkins and M. Westerlund, April 2014.

* [perc-double](https://tools.ietf.org/html/draft-ietf-perc-double-10 "perc-double") **SRTP Double Encryption Procedures**, C. Jennings, P. Jones, R. Barnes, and A. Roach, October 17, 2018



## Appendix

### Dependency Descriptor RTP Header Extension

#### A.1 Introduction

This appendix describes the Dependency Descriptor (DD) RTP Header extension for
conveying information about individual video frames and the dependencies between
these frames. The Dependency Descriptor includes provisions for both temporal
and spatial scalability.


In the DD, the smallest unit for which dependencies are described is an RTP Frame.
An RTP frame contains one complete coded video frame and may also contain additional
information (e.g., metadata). This specification allows for fragmentation of RTP
Frames over multiple packets. RTP frame aggregation is explicitly disallowed.


To facilitate the work of selectively forwarding portions of a scalable video
bitstream to each endpoint in a video conference, as is done by a Selective
Forwarding Unit (SFU), for each packet, several pieces of information are
required (e.g., spatial and temporal layer identification). To reduce overhead,
highly redundant information can be predefined and sent once. Subsequent packets
may index to a template containing predefined information. In particular, when
an encoder uses an unchanging (static) prediction structure to encode a scalable
bitstream, parameter values used to describe the bitstream repeat in a
predictable way. The techniques described in this document provide means to send
repeating information as predefined templates that can be referenced at future
points of the bitstream. Since a reference index to a template requires fewer
bits to convey than the associated structures themselves, header overhead can be
substantially reduced.


The techniques also provide ways to describe changing (dynamic) prediction
structures. In cases where custom dependency information is required, parameter
values are explicitly defined rather than referenced in a predefined template.
Typically, even in dynamic structures the majority of frames still follow one of
the predefined templates.


#### A.2 Conventions, definitions and acronyms

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC2119].


Chain
: A sequence of frames for which it can be determined instantly if a frame from
  that sequence has been lost.

Decode target
: The set of frames needed to decode a coded video sequence at a given spatial
  and temporal fidelity.

Decode Target Information (DTI)
: Describes the relationship of a frame to a Decode target. The DTI indicates
  four distinct relationships: 'not present', 'discardable', 'switch',
  and 'required'.

Discardable indication
: An indication for a frame, associated with a given Decode target, that it will
  not be a Referred frame for any frame belonging to that Decode target.

**Note:** A Frame belonging to more than one Decode target may be discardable
for one Decode target and not for another.
{:.alert .alert-info }

Frame dependency structure
: Describes frame dependency information for the coded video sequence. The
  structure includes the number of DTIs, an ordered list of Frame dependency
  templates, and a mapping between Chains and Decode targets.

Frame dependency template
: Contains frame description information that many frames have in common.
  Includes values for spatial ID, temporal ID, DTIs, frame dependencies, and
  Chain information.

Frame number
: Frame number increases strictly monotonically in decode order.

Instantaneous Decidability of Decodability (IDD)
: The ability to decide, immediately upon receiving the very first packet after
  packet loss, if the lost packet(s) contained a packet that is needed to decode
  the information present in that first and following packets.

Not present indication
: An indication for a frame, that it is not associated with a given Decode target.

Referred frame
: A Frame on which the current frame depends.

Required indication
: An indication for a frame, associated with a given Decode target, that it
  belongs to the Decode target and has neither a Discardable indication nor a Switch
  indication.

**Note:** A Frame belonging to more than one Decode target may be required
for one Decode target and not required (i.e, either discardable or switch)
for another.
{:.alert .alert-info }

Switch indication
: An indication associated with a specific Decode target that all subsequent
  frames for that Decode target will be decodable if the frame containing the
  indication is decodable.

#### A.3 Media stream requirements

A bitstream conformant to this extension must adhere to the following statement(s).

A frame for which all Referred frames are decodable MUST itself be decodable.

**Note:** dependencies are not limited to motion compensated prediction, other
relevant information such as entropy decoder state also constitute dependencies.


#### A.4 Dependency Descriptor format

To facilitate the work of selectively forwarding portions of a scalable video
bitstream, as is done by a selective forwarding unit (SFU), for each packet, the
following information is made available (even though not all elements are
present in every packet).

  * spatial ID
  * temporal ID
  * DTIs
  * Frame number of the current frame
  * Frame numbers of the Referred frames
  * Frame numbers of last frame in each Chain


##### A.4.1 Syntax

The syntax for the descriptor is described in pseudo-code form in this section.

  * **f(n)** - unsigned n-bit number appearing directly in the bitstream.
  * **ns(n)** - unsigned encoded integer with maximum number of values n (i.e.
    output in range 0..n-1).

[f(n) and ns(n) need to be defined. A definition may be found in the AV1 specification Section 4.]


| Symbol name               | Value | Description                                         |
| ------------------------- | ----- | --------------------------------------------------- |
| MAX_TEMPLATE_ID           | 63    | Maximum value for a frame_dependency_template_id to identify a template
| MAX_SPATIAL_ID            | 3     | Maximum value for a FrameSpatialId
| MAX_TEMPORAL_ID           | 7     | Maximum value for a FrameTemporalId
{:.table .table-sm .table-bordered }

Table A.1. Syntax constants
{: .caption }


<pre><code>
dependency_descriptor( sz ) {
  mandatory_descriptor_fields()
  if (sz > 3) {
    extended_descriptor_fields()
  } else {
    no_extended_descriptor_fields()
  }
  frame_dependency_definition()
}
</code></pre>


<pre><code>
mandatory_descriptor_fields() {
  <b>start_of_frame</b> = f(1)
  <b>end_of_frame</b> = f(1)
  <b>frame_dependency_template_id</b> = f(6)
  <b>frame_number</b> = f(16)
}
</code></pre>


<pre><code>
extended_descriptor_fields() {
  <b>frame_dependency_template_id</b> = f(6)
  <b>template_dependency_structure_present_flag</b> = f(1)
  <b>active_decode_targets_present_flag</b> = f(1)
  <b>custom_dtis_flag</b> = f(1)
  <b>custom_fdiffs_flag</b> = f(1)
  <b>custom_chains_flag</b> = f(1)

  if (template_dependency_structure_present_flag) {
    template_dependency_structure()
    active_decode_targets_bitmask = (1 << DtisCnt) - 1
  }

  if (active_decode_targets_present_flag) {
    <b>active_decode_targets_bitmask</b> = f(DtisCnt)
  }
}
</code></pre>


<pre><code>
no_extended_descriptor_fields() {
  custom_dtis_flag = 0
  custom_fdiffs_flag = 0
  custom_chains_flag = 0
}
</code></pre>


<pre><code>
template_dependency_structure() {
  <b>template_id_offset</b> = f(6)
  <b>dtis_cnt_minus_one</b> = f(5)
  DtisCnt = dtis_cnt_minus_one + 1
  template_layers()
  template_dtis()
  template_fdiffs()
  template_chains()
  <b>resolutions_present_flag</b> = f(1)
  if (resolutions_present_flag) {
    render_resolutions()
  }
}
</code></pre>


<pre><code>
frame_dependency_definition() {
  templateIndex = (frame_dependency_template_id + (MAX_TEMPLATE_ID + 1) -
                   template_id_offset) % (MAX_TEMPLATE_ID + 1)
  If (templateIndex >= TemplatesCnt) {
    return  // error
  }
  FrameSpatialId = TemplateSpatialId[templateIndex]
  FrameTemporalId = TemplateTemporalId[templateIndex]

  if (custom_dtis_flag) {
    frame_dtis()
  } else {
    frame_dti = template_dti[templateIndex]
  }

  if (custom_fdiffs_flag) {
    frame_fdiffs()
  } else {
    FrameFdiffsCnt = TemplateFdiffsCnt[templateIndex]
    FrameFdiff = TemplateFdiff[templateIndex]
  }

  if (custom_chains_flag) {
    frame_chains()
  } else {
    frame_chain_fdiff = template_chain_fdiff[templateIndex]
  }

  if (resolutions_present_flag) {
    FrameMaxWidth = max_render_width_minus_one[FrameSpatialId] + 1
    FrameMaxHeight = max_render_height_minus_one[FrameSpatialId] + 1
  }
}
</code></pre>


<pre><code>
template_layers() {
  temporalId = 0
  spatialId = 0
  TemplatesCnt = 0;
  MaxTemporalId = 0
  do {
    TemplateSpatialId[TemplatesCnt] = spatialId
    TemplateTemporalId[TemplatesCnt] = temporalId
    TemplatesCnt++
    <b>next_layer_idc</b> = f(2)
    // next_layer_idc == 0 - same sid and tid
    if (next_layer_idc == 1) {
      temporalId++
      if (temporalId > MaxTemporalId) {
        MaxTemporalId = temporalId
      }
    } else if (next_layer_idc == 2) {
      temporalId = 0
      spatialId++
    }
  } while (next_layer_idc != 3)
  MaxSpatialId = spatialId[c]
}
</code></pre>


<pre><code>
render_resolutions() {
  for (spatial_id = 0; spatial_id <= MaxSpatialId; spatial_id++) {
    <b>max_render_width_minus_1[spatial_id]</b> = f(16)
    <b>max_render_height_minus_1[spatial_id]</b> = f(16)
  }
}
</code></pre>


<pre><code>
template_dtis() {
  for (templateIndex = 0; templateIndex < TemplatesCnt; templateIndex++) {
    for (dtiIndex = 0; dtiIndex < DtisCnt; dtiIndex++) {
      // See table A.2 below for meaning of DTI values.
      <b>template_dti[templateIndex][dtiIndex]</b> = f(2)
    }
  }
}
</code></pre>


<pre><code>
frame_dtis() {
  for (dtiIndex = 0; dtiIndex < DtisCnt; dtiIndex++) {
    // See table A.2 below for meaning of DTI values.
    <b>frame_dti[dtiIndex]</b> = f(2)
  }
}
</code></pre>


<pre><code>
template_fdiffs() {
  templateIndex = 0
  while (templateIndex < TemplatesCnt) {
    fdiffsCnt = 0
    <b>fdiff_follows_flag</b> = f(1)
    while (fdiff_follows_flag) {
      <b>fdiff_minus_one</b> = f(4)
      TemplateFdiff[templateIndex][fdiffsCnt] = fdiff_minus_one + 1
      fdiffsCnt++
      <b>fdiff_follows_flag</b> = f(1)
    }
    TemplateFdiffsCnt[templateIndex] = fdiffsCnt
  }
}
</code></pre>


<pre><code>
frame_fdiffs() {
  FrameFdiffsCnt = 0
  <b>next_fdiff_size</b> = f(2)
  while (next_fdiff_size) {
    <b>fdiff_minus_one</b> = f(4 * next_fdiff_size)
    FrameFdiff[FrameFdiffsCnt] = fdiff_minus_one + 1
    FrameFdiffsCnt++
    <b>next_fdiff_size</b> = f(2)
  }
}
</code></pre>


<pre><code>
template_chains() {
  <b>chains_cnt</b> = ns(DtisCnt + 1)
  if (chains_cnt == 0) {
    return
  }
  for (dtiIndex = 0; dtiIndex < DtisCnt; dtiIndex++) {
    // If a decode target is not tracked by a chain, it will have
    // chain id equal to chains_cnt.
    <b>decode_target_protected_by[dtiIndex]</b> = ns(chains_cnt + 1)
  }
  for (templateIndex = 0; templateIndex < TemplatesCnt; templateIndex++) {
    for (chainIndex = 0; chainIndex < chains_cnt; chainIndex++) {
      <b>template_chain_fdiff[templateIndex][chainIndex]</b> = f(4)
    }
  }
}
</code></pre>


<pre><code>
frame_chains() {
  for (chainIndex = 0; chainIndex < chains_cnt; chainIndex++) {
    <b>frame_chain_fdiff[chainIndex]</b> = f(8)
  }
}
</code></pre>

##### A.4.2 Semantics

The semantics pertaining to the dependency descriptor syntax section above is
described in this section.

**Mandatory Descriptor Fields**

  * **start_of_frame**: MUST be set to 1 if the first payload octet of the RTP
    packet is the beginning of a new Frame, and MUST be set to 0 otherwise. Note
    that this frame might not be the first frame of a temporal unit.

  * **end_of_frame**: MUST be set to 1 for the final RTP packet of a Frame, and
    MUST be set to 0 otherwise. Note that, if spatial scalability is in use,
    more frames from the same temporal unit may follow.

  * **frame_number**: is represented using 16 bits and increases strictly
    monotonically in decode order. frame_number MAY start on a random number,
    and MUST wrap after reaching the maximum value. All packets of the same
    Frame MUST have the same frame_number value.

    **Note:** Frame number is not the same as Frame ID in [AV1 specification].
    {:.alert .alert-info }

  * **frame_dependency_template_id**: ID of the Frame dependency template to use.
    MUST be in the range of template_id_offset to
    (template_id_offset + TemplatesCnt - 1), inclusive.

    frame_dependency_template_id MUST be the same for all packets of the same
    Frame.

    **Note:** values out of the valid range indicate a change of the Frame
    dependency structure.
    {:.alert .alert-info }

**Extended Descriptor Fields**

  * **template_dependency_structure_present_flag**: indicates the presence the
    template_dependency_structure. When the
    template_dependency_structure_present_flag is set to 1,
    template_dependency_structure MUST be present; otherwise
    template_dependency_structure MUST NOT be present.
    template_dependency_structure_present_flag MUST be set to 1 for the first
    packet of a coded video sequence, and MUST be set to 0 otherwise.

  * **active_decode_targets_present_flag**: indicates the presence of
    active_decode_targets_bitmask. When set to 1, active_decode_targets_bitmask
    MUST be present, otherwise, active_decode_targets_bitmask MUST NOT be present.

  * **active_decode_targets_bitmask**: contains a bitmask that indicates which
    decode targets are available for decoding. Bit i is equal to 1 if
    decode target i is available for decoding, 0 otherwise.

  * **custom_dtis_flag**: indicates the presence of frame_dtis. When set to 1,
    frame_dtis MUST be present. Otherwise, frame_dtis MUST NOT be present.

  * **custom_fdiffs_flag**: indicates the presence of frame_fdiffs. When set to
    1, frame_fdiffs MUST be present. Otherwise, frame_fdiffs MUST NOT be present.

  * **custom_chains_flag**: indicates the presence of frame_chain_fdiff. When
    set to 1, frame_chain_fdiff MUST be present. Otherwise, frame_chain_fdiff MUST NOT
    be present.

**Template dependency structure**

  * **template_id_offset**: indicates the value of the
    frame_dependency_template_id having templateIndex=0. The value of
    template_id_offset SHOULD be chosen so that the valid
    frame_dependency_template_id range, template_id_offset to
    template_id_offset + TemplatesCnt - 1, inclusive, of a new
    template_dependency_structure, does not overlap the valid
    frame_dependency_template_id range for the existing
    template_dependency_structure. When template_id_offset of a new
    template_dependency_structure is the same as in the existing
    template_dependency_structure, all fields in both
    template_dependency_structures MUST have identical values.

  * **dtis_cnt_minus_one**: dtis_cnt_minus_one + 1 indicates the number of
    Decode targets present in the coded video sequence.

  * **resolutions_present_flag**: indicates the presence of render_resolutions.
    When the resolutions_present_flag is set to 1, render_resolutions MUST be
    present; otherwise render_resolutions MUST NOT be present.

  * **next_layer_idc**: used to determine spatial ID and temporal ID for the
    next Frame dependency template. Table A.3 describes how the spatial ID and
    temporal ID values are determined. A next_layer_idc equal to 3 indicates
    that no more Frame dependency templates are present in the Frame dependency
    structure.

  * **max_render_width_minus_1[spatial_id]**: indicates the maximum render width
    minus 1 for frames with spatial ID equal to spatial_id.

  * **max_render_height_minus_1[spatial_id]**: indicates the maximum render
    height minus 1 for frames with spatial ID equal to spatial_id.

  * **chains_cnt**: indicates the number of Chains. When set to zero, the Frame
    dependency structure does not utilize protection with Chains.

  * **decode_target_protected_by[dtIndex]**: the index of the Chain that
    protects the Decode target, dtIndex. A value of
    decode_target_protected_by[dtIndex] equal to chains_cnt  indicates that
    Decode target dtIndex is not protected by a Chain. Each Decode target MAY be
    protected by at most one Chain.

  * **template_dti[templateIndex][]**: an array of size dtis_cnt_minus_one + 1
    containing Decode target information for the Frame dependency template
    having index value equal to templateIndex. Table A.2 contains a description of
    the Decode target information values.

  * **template_chain_fdiff[templateIndex][]**: an array of size chains_cnt
    containing chain-FDIFF values for the Frame dependency template having index
    value equal to templateIndex. In a template, the values of chain-FDIFF can
    be in the range 0 to 15, inclusive.

  * **fdiff_follows_flag**: indicates the presence of a frame difference value.
    When the fdiff_follows_flag is set to 1, fdiff_minus_one MUST immediately
    follow; otherwise a value of 0 indicates no more frame difference values are
    present for the current Frame dependency template.

  * **fdiff_minus_one**: the difference between Frame number and the Frame
    number of the Referred frame minus one.


| DTI                    | Value |                                                        |
| ---------------------- | ----- | ------------------------------------------------------ |
| Not present indication | 0     | No payload for this Decode target is present.
| Discardable indication | 1     | Payload for this Decode target is present and discardable.
| Switch indication      | 2     | Payload for this Decode target is present and switch is possible (Switch indication).
| Required indication    | 3     | Payload for this Decode target is present but it is neither discardable nor is it a Switch indication.
{:.table .table-sm .table-bordered }

Table A.2. DTI values.
{: .caption }


**Frame dependency defintion**

  * **next_fdiff_size**: indicates the size of following fdiff_minus_one syntax
    elements in 4-bit units. When set to a non-zero value, fdiff_minus_one MUST
    immediately follow; otherwise a value of 0 indicates no more frame
    difference values are present.

  * **frame_dti[dtiIndex]**: decode target information describing the
    relationship between the current frame and the Decode target having index
    equal to dtiIndex. Table A.2 contains a description of the Decode target
    information values.

  * **frame_chain_fdiff[chainIdx]**: indicates the difference between the Frame
    number and the Frame number of the previous frame in the Chain having index
    equal to chainIdx. A value of 0 indicates no previous frames are needed for
    the Chain. For example, when a packet containing
    frame_chain_fdiff[chainIdx]=3 and Frame number=112 the previous frame in the
    Chain with index equal to chainIdx has Frame number=109. The calculation is
    done modulo the size of the frame_number field.


| next_layer_idc | Next Spatial ID And Temporal ID Values                   |
| -------------- | -------------------------------------------------------- |
| 0              | The next Frame dependency template has the same spatial ID and temporal ID as the current template
| 1              | The next Frame dependency template has the same spatial ID and temporal ID plus 1 compared with the current Frame dependency template.
| 2              | The next Frame dependency template has temporal ID equal to 0 and spatial ID plus 1 compared with the current Frame dependency template.
| 3              | No more Frame dependency templates are present in the Frame dependency structure.
{:.table .table-sm .table-bordered }

Table A.3. Derivation Of Next Spatial ID And Temporal ID Values.
{: .caption }


**TODO:** Add process descriptions and examples.
{:.alert .alert-danger }


##### A.4.3 Implementing IDD with Chains

The Frame dependency structure includes a mapping between Decode targets and
Chains. The mapping gives an SFU the ability to know the set of Chains it needs
to track in order to ensure that the corresponding Decode targets remain
decodable. Every packet includes, for every Chain, the Frame number for the
previous Frame in that Chain. An SFU can instantaneously detect a broken Chain
by checking whether or not the previous Frame in that Chain has been received.
Due to the fact that Chain information for all Chains is present in all packets,
an SFU can detect a broken Chain regardless of whether the first packet received
after a loss is part of that Chain.

In order to start/restart Chains, a packet may reference its own Frame number as
an indication that no previous Frames are needed for the Chain. Key frames are
common cases for such '(re)start of Chain' indications.

**Note:** Chains may be used for more than just realizing the IDD property.
{:.alert .alert-info }


##### A.4.4 Switching

An SFU may begin forwarding packets belonging to a new Decode target beginning
with a decodable Frame containing a Switch indication to that Decode target.

An SFU may change which Decode targets it forwards. Similarly, a sender may
change the Decode targets that are currently being produced. In both cases, not
all Decode targets may be available for decoding. Such changes SHOULD be
signaled to the receiver using the active_decode_targets_bitmask and SHOULD be
signaled to the receiver in a reliable way.

When not all Decode targets are active, the active_decode_targets_bitmask MUST
be sent in every packet where the template_dependency_structure_present_flag is
equal to 1.

**Note:** One way to achieve reliable delivery is to include the
active_decode_targets_bitmask in every packet until a receiver report
acknowledges a packet containing the latest active_decode_targets_bitmask.
Alternately, for many video streams, reliable delivery may be achieved by
including the active_decode_targets_bitmask on every chain in the first packet
after a change in active decode targets.
{:.alert .alert-info }

Chains protecting no active decode targets MUST be ignored.

**Note:** To increase the chance of using a predefined template, chains
protecting no active decode targets may refer to any frame, including an RTP
frame that was never produced.
{:.alert .alert-info }

#### A.5 Examples

Each example in this section contains a prediction structure figure and a table
describing the associated Frame dependency structure. The Frame dependency
structure table column headings have the meanings listed below. For the DTI-
related columns, Table A.4 shows the symbol used to represent each DTI value.

  * Idx - template index
  * S - spatial ID
  * T - temporal ID
  * Fdiffs - comma delimited list of TemplateFdiff[Idx] values
  * Chain(s) - **template_chain_fdiff[Idx]** values for each Chain
  * DTI - **template_dti[Idx]**


| DTI                    | Value | Symbol   |
| ---------------------- | ----- | -------- |
| Not present indication | 0     | -        |
| Discardable indication | 1     | D        |
| Switch indication      | 2     | S        |
| Required indication    | 3     | R        |
{:.table .table-sm .table-bordered }

Table A.4. DTI values
{: .caption }


##### A.5.1 L1T3 Single spatial layer with 3 temporal layers

<figure class="figure center-block" style="display: table; margin: 1.5em auto;">
  <img alt="" src="assets/images/L1T3.png" class="figure-img img-fluid">
</figure>


<table class="table-sm table-bordered" style="margin-bottom: 1.5em;">
<tbody><tr>
<th colspan='1' rowspan='2' >Idx</th><th colspan='1' rowspan='2' >S</th><th colspan='1' rowspan='2' >T</th><th colspan='1' rowspan='2' >Fdiffs</th><th colspan='1' rowspan='2' >Chain</th><th colspan='3' rowspan='1' >DTI</th>
</tr>
<tr>
<th colspan='1' rowspan='1' >30 fps</th><th colspan='1' rowspan='1' >15 fps</th><th colspan='1' rowspan='1' >7.5 fps</th>
</tr>
<tr>
<td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' ></td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >4</td><td colspan='1' rowspan='1' >4</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >D</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >4</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >D</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >5</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >D</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='5' rowspan='1' ><b><tt>decode_target_protected_by</tt></b></td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >0</td>
</tr>
</tbody></table>


**Note:** This example uses one Chain, which includes Frames with temporal ID
equal to 0.
{:.alert .alert-info }


##### A.5.2 L2T1 Full SVC with occasional switch

<figure class="figure center-block" style="display: table; margin: 1.5em auto;">
  <img alt="" src="assets/images/L2T1.png" class="figure-img img-fluid">
</figure>


<table class="table-sm table-bordered" style="margin-bottom: 1.5em;">
<tbody><tr>
<th colspan='1' rowspan='2' >Idx</th><th colspan='1' rowspan='2' >S</th><th colspan='1' rowspan='2' >T</th><th colspan='1' rowspan='2' >Fdiffs</th><th colspan='2' rowspan='1' >Chains</th><th colspan='2' rowspan='1' >DTI</th>
</tr>
<tr>
<th colspan='1' rowspan='1' >0</th><th colspan='1' rowspan='1' >1</th><th colspan='1' rowspan='1' >HD</th><th colspan='1' rowspan='1' >VGA</th>
</tr>
<tr>
<td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' ></td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >R</td><td colspan='1' rowspan='1' >S</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >4</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >5</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >2,1</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >R</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='6' rowspan='1' ><b><tt>decode_target_protected_by</tt></b></td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >0</td>
</tr>
</tbody></table>


**Note:** This example uses two Chains. Chain 0 includes Frames with spatial ID
equal to 0. Chain 1 includes all Frames.
{:.alert .alert-info }


##### A.5.3 L3T3 Full SVC

<figure class="figure center-block" style="display: table; margin: 1.5em auto;">
  <img alt="" src="assets/images/L3T3.png" class="figure-img img-fluid">
</figure>


<table class="table-sm table-bordered" style="margin-bottom: 1.5em;">
<tbody><tr>
<th colspan='1' rowspan='2' >Idx</th><th colspan='1' rowspan='2' >S</th><th colspan='1' rowspan='2' >T</th><th colspan='1' rowspan='2' >Fdiffs</th><th colspan='3' rowspan='1' >Chains</th><th colspan='9' rowspan='1' >DTI</th>
</tr>
<tr>
<th colspan='1' rowspan='1' >0</th><th colspan='1' rowspan='1' >1</th><th colspan='1' rowspan='1' >2</th><th colspan='1' rowspan='1' >HD30 fps</th><th colspan='1' rowspan='1' >HD15 fps</th><th colspan='1' rowspan='1' >HD7.5 fps</th><th colspan='1' rowspan='1' >VGA30 fps</th><th colspan='1' rowspan='1' >VGA15 fps</th><th colspan='1' rowspan='1' >VGA7.5fps</th><th colspan='1' rowspan='1' >QVGA30 fps</th><th colspan='1' rowspan='1' >QVGA15 fps</th><th colspan='1' rowspan='1' >QVGA7.5 fps</th>
</tr>
<tr>
<td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' ></td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >12</td><td colspan='1' rowspan='1' >12</td><td colspan='1' rowspan='1' >11</td><td colspan='1' rowspan='1' >10</td><td colspan='1' rowspan='1' >R</td><td colspan='1' rowspan='1' >R</td><td colspan='1' rowspan='1' >R</td><td colspan='1' rowspan='1' >R</td><td colspan='1' rowspan='1' >R</td><td colspan='1' rowspan='1' >R</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >6</td><td colspan='1' rowspan='1' >6</td><td colspan='1' rowspan='1' >5</td><td colspan='1' rowspan='1' >4</td><td colspan='1' rowspan='1' >R</td><td colspan='1' rowspan='1' >R</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >R</td><td colspan='1' rowspan='1' >R</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >D</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >4</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >R</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >R</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >D</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >5</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >9</td><td colspan='1' rowspan='1' >8</td><td colspan='1' rowspan='1' >7</td><td colspan='1' rowspan='1' >R</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >R</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >D</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >6</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >7</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >12,1</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >R</td><td colspan='1' rowspan='1' >R</td><td colspan='1' rowspan='1' >R</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >8</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >6,1</td><td colspan='1' rowspan='1' >7</td><td colspan='1' rowspan='1' >6</td><td colspan='1' rowspan='1' >5</td><td colspan='1' rowspan='1' >R</td><td colspan='1' rowspan='1' >R</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >D</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >9</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >3,1</td><td colspan='1' rowspan='1' >4</td><td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >R</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >D</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >10</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >3,1</td><td colspan='1' rowspan='1' >10</td><td colspan='1' rowspan='1' >9</td><td colspan='1' rowspan='1' >8</td><td colspan='1' rowspan='1' >R</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >D</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >11</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >12</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >12,1</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >13</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >6,1</td><td colspan='1' rowspan='1' >8</td><td colspan='1' rowspan='1' >7</td><td colspan='1' rowspan='1' >6</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >D</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >14</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >3,1</td><td colspan='1' rowspan='1' >5</td><td colspan='1' rowspan='1' >4</td><td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >D</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >15</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >3,1</td><td colspan='1' rowspan='1' >11</td><td colspan='1' rowspan='1' >10</td><td colspan='1' rowspan='1' >9</td><td colspan='1' rowspan='1' >D</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='7' rowspan='1' ><b><tt>decode_target_protected_by</tt></b></td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >0</td>
</tr>
</tbody></table>


**Note:** This example uses three Chains. Chain 0 includes Frames with spatial
ID equal to 0 and temporal ID equal to 0. Chain 1 includes Frames with spatial
ID equal to 0 or 1 and temporal ID equal to 0. Chain 2 includes all Frames with
temporal ID equal to 0.
{:.alert .alert-info }


##### A.5.4 S3T3 K-SVC with temporal shift

<figure class="figure center-block" style="display: table; margin: 1.5em auto;">
  <img alt="" src="assets/images/S3T3.png" class="figure-img img-fluid">
</figure>


<table class="table-sm table-bordered" style="margin-bottom: 1.5em;">
<tbody><tr>
<th colspan='1' rowspan='2' >Idx</th><th colspan='1' rowspan='2' >S</th><th colspan='1' rowspan='2' >T</th><th colspan='1' rowspan='2' >Fdiffs</th><th colspan='3' rowspan='1' >Chains</th><th colspan='9' rowspan='1' >DTI</th>
</tr>
<tr>
<th colspan='1' rowspan='1' >0</th><th colspan='1' rowspan='1' >1</th><th colspan='1' rowspan='1' >2</th><th colspan='1' rowspan='1' >HD 30 fps</th><th colspan='1' rowspan='1' >HD 15 fps</th><th colspan='1' rowspan='1' >HD 7.5 fps</th><th colspan='1' rowspan='1' >VGA 30 fps</th><th colspan='1' rowspan='1' >VGA 15 fps</th><th colspan='1' rowspan='1' >VGA 7.5 fps</th><th colspan='1' rowspan='1' >QVGA 30 fps</th><th colspan='1' rowspan='1' >QVGA 15 fps</th><th colspan='1' rowspan='1' >QVGA 7.5 fps</th>
</tr>
<tr>
<td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' ></td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >12</td><td colspan='1' rowspan='1' >12</td><td colspan='1' rowspan='1' >8</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >4</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >6</td><td colspan='1' rowspan='1' >6</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >7</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >D</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >5</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >5</td><td colspan='1' rowspan='1' >4</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >D</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >6</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >9</td><td colspan='1' rowspan='1' >5</td><td colspan='1' rowspan='1' >10</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >D</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >7</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >11</td><td colspan='1' rowspan='1' >4</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >D</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >8</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >9</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >6</td><td colspan='1' rowspan='1' >4</td><td colspan='1' rowspan='1' >6</td><td colspan='1' rowspan='1' >5</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >10</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >12</td><td colspan='1' rowspan='1' >4</td><td colspan='1' rowspan='1' >12</td><td colspan='1' rowspan='1' >5</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >11</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >6</td><td colspan='1' rowspan='1' >10</td><td colspan='1' rowspan='1' >6</td><td colspan='1' rowspan='1' >11</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >D</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >12</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >D</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >13</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >7</td><td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >8</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >D</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >14</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >9</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >D</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >15</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >16</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >12</td><td colspan='1' rowspan='1' >11</td><td colspan='1' rowspan='1' >7</td><td colspan='1' rowspan='1' >12</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >17</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >6</td><td colspan='1' rowspan='1' >5</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >6</td><td colspan='1' rowspan='1' >S</td><td colspan='1' rowspan='1' >D</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >18</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >4</td><td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >D</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >19</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >8</td><td colspan='1' rowspan='1' >4</td><td colspan='1' rowspan='1' >9</td><td colspan='1' rowspan='1' >D</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='1' rowspan='1' >20</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >10</td><td colspan='1' rowspan='1' >3</td><td colspan='1' rowspan='1' >D</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td><td colspan='1' rowspan='1' >-</td>
</tr>
<tr>
<td colspan='7' rowspan='1' ></td><td colspan='1' rowspan='1' >HD 30 fps</td><td colspan='1' rowspan='1' >HD 15 fps</td><td colspan='1' rowspan='1' >HD 7.5 fps</td><td colspan='1' rowspan='1' >VGA 30 fps</td><td colspan='1' rowspan='1' >VGA 15 fps</td><td colspan='1' rowspan='1' >VGA 7.5 fps</td><td colspan='1' rowspan='1' >QVGA 30 fps</td><td colspan='1' rowspan='1' >QVGA 15 fps</td><td colspan='1' rowspan='1' >QVGA 7.5 fps</td>
</tr>
<tr>
<td colspan='7' rowspan='1' ><b><tt>decode_target_protected_by</tt></b></td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >2</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >1</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >0</td><td colspan='1' rowspan='1' >0</td>
</tr>
</tbody></table>


**Note:** This example uses three Chains. Chain 0 includes Frames with spatial
ID equal to 0 and temporal ID equal to 0. Chain 1 includes Frame 100 and Frames
with spatial ID equal to 1 and temporal ID equal to 0. Chain 2 includes Frames
100, 101, and Frames with spatial ID equal to 2 and temporal ID equal to 0.
{:.alert .alert-info }


#### A.6 References


##### A.6.1 Normative references
 
 * [RFC2119](https://tools.ietf.org/html/rfc2119 "RFC2119") **Key words for use in RFCs to Indicate Requirement Levels, S. Bradner, March 1997. 
 
  * [RFC3550](https://tools.ietf.org/html/rfc3550 "RFC3550") **RTP: A Transport Protocol for Real-Time Applications**, H. Schulzrinne, S. Casner, R. Frederick, and V. Jacobson, July 2003.

   * [RFC8285](https://tools.ietf.org/html/rfc8285 "RFC8285") **A General Mechanism for RTP Header Extensions for generic RTP header extensions**, D. Singer, H. Desineni, and R. Even, October 2017.

   * [I-D.ietf-payload-vp9](https://tools.ietf.org/html/draft-ietf-payload-vp9 "I-D.ietf-payload-vp9") **RTP Payload Format for VP9 Video**, J. Uberti, S. Holmer, M. Flodman, D. Hong, and J. Lennox, January 2020.
   
##### A.6.2 Informative references

* [AV1](https://aomediacodec.github.io/av1-spec/av1-spec.pdf "AV1 1.0.0 with Errata1") **AV1 Bistream & Decoding Process Specification, Version 1.0.0 with Errata 1**, January 2019.

* [RFC3264](https://tools.ietf.org/html/rfc3264 "RFC3264") **An Offer/Answer Model with the Session Description Protocol (SDP)**, J. Rosenberg and H. Schulzrinne, June 2002. 

* [RFC4566](https://tools.ietf.org/html/rfc4566 "RFC4566") **SDP: Session Description Protocol**, M. Handley, V. Jacobson, and C. Perkins, July 2006.

* [RFC4585](https://tools.ietf.org/html/rfc4585 "RFC4585") **Extended RTP Profile for Real-time Transport Control Protocol (RTCP)-Based Feedback (RTP/AVPF)**, J. Ott, S. Wenger, N. Sato, C. Burmeister, and J. Rey, July 2006. 

* [RFC6838](https://tools.ietf.org/html/rfc6838 "RFC6838") **Media Type Specifications and Registration Procedures**, N. Freed, J. Klensin, and T. Hansen, January 2013.

 * [RFC7667](https://tools.ietf.org/html/rfc7667 "RFC7667") **RTP Topologies**, M. Westerlund and S. Wenger, November 2015.
 
    * [I-D.ietf-avttext-lrr](https://datatracker.ietf.org/doc/draft-ietf-avtext-lrr/ "I-D.ietf-avtext-lrr") **The Layer Refresh Request (LRR) RTCP Feedback Message**, J. Lennox, D. Hong, J. Uberti, S. Holmer, and M. Flodman, June 2017.


