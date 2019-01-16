
# RTP Payload Format For AV1 (v0.1.5)

status: AV1 RTC SG Working Draft (WD)

## Abstract

This memo describes an RTP payload format for the AV1 video codec. The payload format has wide applicability, from low bit-rate peer-to-peer usage, to high bit-rate multi-party video conferences. It includes provisions for temporal and spatial scalability.

## Status of This Memo

This document is a working draft of the Real-Time Communications Subgroup.

## 1. Introduction

This memo describes an RTP payload specification applicable to the transmission of video streams encoded using the AV1 video codec [AV1].

In AV1 the smallest individual encoder entity presented for transport is the Open Bitstream Unit (OBU). This specification allows both for fragmentation and aggregation of OBUs in the same RTP packet, but explicitly disallows doing so across frame boundaries.

This specification also provides several mechanisms through which scalability structures are described. AV1 uses the concept of predefined scalability structures. These are a set of commonly used picture prediction structures that can be referenced simply via an indicator value (scalability_mode_idc, residing in the sequence header). For cases that do not fall in any of the predefined cases, there is a mechanism for describing the scalability structure. These bitstream parameters greatly simplify the organization of the corresponding data at the RTP payload format level.


To facilitate the work of selectively forwarding portions of a scalable video bitstream to each endpoint in a video conference, as is done by a Selective Forwarding Unit (SFU), for each packet, several pieces of information are required (e.g., spatial and temporal layer identification). To reduce overhead, highly redundant information can be predefined and sent once. Subsequent packets may index to a template containing predefined information. In particular, when an encoder uses an unchanging (static) prediction structure to encode a scalable bitstream, parameter values used to describe the bitstream repeat in a predictable way. The techniques described in this document provide means to send repeating information as predefined templates that can be referenced at future points of the bitstream. Since a reference index to a template requires fewer bits to convey than the associated structures themselves, header overhead can be substantially reduced.


The techniques also provide ways to describe changing (dynamic) prediction structures. In cases where custom dependency information is required, parameter values are explicitly defined rather than referenced in a predefined template. Typically, even in dynamic structures the majority of frames still follow one of the predefined templates.


## 2. Conventions, Definitions and Acronyms

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119].


  * **Chain** - a sequence of frames for which it can be determined instantly if
    a frame from that sequence has been lost.

  * **Coded frame** - the representation of one frame before the decoding
     process.

  * **Decode target** - the set of frames needed to decode a coded video
    sequence at a given spatial and temporal fidelity.

  * **Referred frame** - a Frame on which the current frame depends.

  * **Decode Target Information (DTI)** - describes the relationship of a frame
    to a Decode target. The DTI indicates four distinct relationships: ‘not
    present, ‘discardable’, ‘switch indication’, and ‘required’.

  * **Discardable** - is an indication for a frame, associated with a given
    Decode target, that it will not be a Referred frame for any frame belonging
    to that Decode target. Note: a Frame belonging to more than one Decode
    target may be Discardable for one Decode target and not for another.

  * **Frame** - a frame in this document is synonymous to a Coded frame. Note:
    in contrast, in [AV1], Frame is defined as the representation of video
    signals in the spatial domain. Note: Multiple frames may be present at the
    same instant in time.

  * **Frame dependency structure** - describes frame dependency information for
    the coded video sequence[a]. The structure includes the number of DTIs, an
    ordered list of Frame dependency templates, and a mapping between Chains and
    Decode targets.

  * **Frame dependency template** - contains frame description information that
    many frames have in common. Includes values for spatial ID, temporal ID,
    DTIs, frame dependencies, and Chain information.

  * **Frame number** - Frame number increases strictly monotonically in decode
    order. Note: Frame number is not the same as Frame ID in
    [AV1 specification].

  * **Instantaneous Decidability of Decodability (IDD)** - the ability to
    decide, immediately upon receiving the very first packet after packet loss,
    if the lost packet(s) contained a packet that is needed to decode the
    information present in that first and following packets.

  * **Open Bitstream Unit (OBU)** - the smallest bitstream data framing unit in
    AV1. All AV1 bitstream structures are packetized in OBUs.

  * **Predefined frame** - a Frame for which the spatial ID, temporal ID, DTIs,
    Referred frames, and Chain information is contained in the
    template_dependency_structure and referenced using
    frame_dependency_template_id.

  * **Required** - an indication for a frame, associated with a given Decode
    target, that it belongs to the Decode target and has neither a Discardable
    nor a Switch indication. Note: a Frame belonging to more than one Decode
    target may be Required for one Decode target and not Required (i.e, either
    Discardable or Switch) for another.

  * **Self-defined frame** - a Frame for which the spatial ID, temporal ID,
    DTIs, Referred frames, and Chain information is present in the packet(s)
    containing the Frame.

  * **Selective Forwarding Unit (SFU)** - a middlebox that relays streams among
    transmitting and receiving clients by selectively forwarding packets
    [RFC 7667].

  * **Switch indication** - an indication associated with a specific Decode
    target that all subsequent frames for that Decode target will be decodable
    if the frame containing the indication is decodable.

  * **Switch request** - a request for the encoder to produce a frame with
    Switch indication that would allow the endpoint to decode a specified Decode
    target.


## 3. Media Format Description

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

Note: Layer dependencies are constrained by the AV1 specification such that a
temporal layer with temporal_id T and spatial layer with spatial_id S are only
allowed to reference previously coded video data of temporal_id T’ and
spatial_id S’, where T’ <= T and S’ <= S.


## 4. Payload Format

This section describes how the encoded AV1 bitstream is encapsulated in RTP. All
integer fields in the specifications are encoded as unsigned integers in network
octet order.


### 4.1 RTP Header Usage

The general RTP payload format follows the RTP header format [RFC3550] and
generic RTP header extensions [RFC8285], and is shown below.

The AV1 descriptor and AV1 aggregation header are described in this document.
The payload itself is a series of OBUs, each preceded by length information as
detailed later in this document.

~~~~~
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
~~~~~


**TODO:** Decide whether to keep the ascii art in this document.


### 4.2  AV1 descriptor

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


Syntax

The syntax for the AV1 descriptor is described in pseudo-code form in this
section.


f(n) - unsigned n-bit number appearing directly in the bitstream.

ns(n) - unsigned encoded integer with maximum number of values n (i.e. output in
range 0..n-1).


(See AV1 specification Section 4 for syntax details including the two functions
(above)


Symbol name
	Value
	Description
	TD_STRUCTURE_INDICATOR
	63
	Value indicating presence of template dependency structure
	MAX_TEMPLATE_ID
	62
	Maximum value for a frame_dependency_template_id to identify a template
	MAX_SPATIAL_ID
	3
	Maximum value for a FrameSpatialId
	MAX_TEMPORAL_ID
	7
	Maximum value for a FrameTemporalId
	Table 1. Syntax constants.



~~~~~
av1_desriptor() {
  base_header()
  if (frame_dependency_template_id_or_structure_indicator ==
TD_STRUCTURE_INDICATOR) {
    frame_dependency_template_id = f(6)
    template_dependency_structure()
  } else {
    frame_dependency_template_id =
      frame_dependency_template_id_or_structure_indicator
  }
  If (frame_dependency_template_id == self_defined_id) {
    self_definition()
  } else {
    pre_definition()
  }
}
~~~~~


~~~~~
base_header() {
  start_of_frame = f(1)
  end_of_frame = f(1)
  frame_dependency_template_id_or_structure_indicator = f(6)
  frame_number = f(16)
}


template_dependency_structure() {
  self_defined_id = f(6)
  PreDefinedOffset = (self_defined_id + 1) % (MAX_TEMPLATE_ID + 1)
  dtis_cnt_minus_one = f(5)
  DtisCnt = dtis_cnt_minus_one + 1
  template_layers()
  template_rbits()
  template_dtis()
  template_fdiffs()
  template_chains()
  populate_decode_target_layer()
}
~~~~~


~~~~~
self_defintion() {
  frame_dtis()
  frame_fdiffs()
  frame_chains()
  populate_frame_layer()
}
~~~~~


~~~~~
template_layers() {
  temporalId = 0
  spatialId = 0
  TemplatesCnt = 0;
  do {
    TemplateSpatialId[TemplatesCnt] = spatialId
    TemplateTemporalId[TemplatesCnt] = temporalId
    TemplatesCnt++
    next_layer_idc = f(2)
    // next_layer_idc == 0 - same sid and tid
    if (next_layer_idc == 1) {
      temporalId++
    } else if (next_layer_idc == 2) {
      temporalId = 0
      spatialId++
    }
  } while (next_layer_idc != 3)
}
~~~~~


~~~~~
template_rbits() {
  for (templateIndex = 0; templateIndex < TemplatesCnt; templateIndex++) {
    template_repeatable[templateIndex] = f(1)
  }
}
~~~~~


~~~~~
template_dtis() {
  for (templateIndex = 0; templateIndex < TemplatesCnt; templateIndex++) {
    for (dtiIndex = 0; dtiIndex < DtisCnt; dtiIndex++) {
      // See table 2 below for meaning of DTI values.
      template_dti[templateIndex][dtiIndex] = f(2)
    }
  }
}
~~~~~


~~~~~
frame_dtis() {
  for (dtiIndex = 0; dtiIndex < DtisCnt; dtiIndex++) {
    // See table 2 below for meaning of DTI values.
    frame_dti[dtiIndex] = f(2)
  }
}
~~~~~


~~~~~
template_fdiffs() {
  templateIndex = 0
  while (templateIndex < TemplatesCnt) {
    fdiffsCnt = 0
    fdiff_follows_flag = f(1)
    while (fdiff_follows_flag) {
      fdiff_minus_one = f(4)
      TemplateFdiff[templateIndex][fdiffsCnt] = fdiff_minus_one + 1
      fdiffsCnt++
      fdiff_follows_flag = f(1)
    }
    TemplateFdiffsCnt[templateIndex] = fdiffsCnt
  }
}
~~~~~


~~~~~
frame_fdiffs() {
  FrameFdiffsCnt = 0
  next_fdiff_size = f(2)
  while (next_fdiff_size) {
    fdiff_minus_one = f(4 * next_fdiff_size)
    FrameFdiff[FrameFdiffsCnt] = fdiff_minus_one + 1
    FrameFdiffsCnt++
    next_fdiff_size = f(2)
  }
}
~~~~~


~~~~~
template_chains() {
  chains_cnt = ns(DtisCnt + 1)
  if (chains_cnt == 0) {
    return
  }
  for (dtiIndex = 0; dtiIndex < DtisCnt; dtiIndex++) {
    // If a decode target is not tracked by a chain, it will have
    // chain id equal to chains_cnt.
    decode_target_protected_by[dtiIndex] = ns(chains_cnt + 1)
  }
  for (templateIndex = 0; templateIndex < TemplatesCnt; templateIndex++) {
    for (chainIndex = 0; chainIndex < chains_cnt; chainIndex++) {
      template_chain_fdiff[templateIndex][chainIndex] = f(4)
    }
  }
}
~~~~~


~~~~~
frame_chains() {
  for (chainIndex = 0; chainIndex < chains_cnt; chainIndex++) {
    frame_chain_fdiff[chainIndex] = f(8)
  }
}
~~~~~


~~~~~
pre_definition() {
  templateIndex = (frame_dependency_template_id + MAX_TEMPLATE_ID + 1 - PreDefinedOffset) % (MAX_TEMPLATE_ID + 1)
  If (templateIndex >= TemplatesCnt) {
    return  // error
  }
  FrameSpatialId = TemplateSpatialId[templateIndex]
  FrameTemporalId = TemplateTemporalId[templateIndex]
  FrameFdiffsCnt = TemplateFdiffsCnt[templateIndex]
  FrameFdiff = TemplateFdiff[templateIndex]
  frame_dti = template_dti[templateIndex]
  frame_chain_fdiff = template_chain_fdiff[templateIndex]
}
~~~~~


~~~~~
populate_decode_target_layer() {
  for (dtiIndex = 0; dtiIndex < DtisCnt; dtiIndex++) {
    spatialId = 0
    temporalId = 0
    for (templateIndex = 0; templateIndex < TemplatesCnt; templateIndex++) {
      if (template_dti[templateIndex][dtiIndex]) {
        spatialId = Max(spatialId, TemplateSpatialId[templateIndex])
        temporalId = Max(temporalId, TemplateTemporalId[templateIndex])
      }
    }
    DecodeTargetSpatialId[dtiIndex] = spatialId
    DecodeTargetTemporalId[dtiIndex] = temporalId
  }
}
~~~~~


~~~~~
populate_frame_layer() {
  FrameSpatialId = MAX_SPATIAL_ID + 1
  FrameTemporalId = MAX_TEMPORAL_ID + 1
  for (dtiIndex = 0; dtiIndex < DtisCnt; dtiIndex++) {
    if (frame_dti[dtiIndex]) {
      FrameSpatialId = Min(FrameSpatialId, DecodeTargetSpatialId[dtiIndex])
      FrameTemporalId = Min(FrameTemporalId, DecodeTargetTemporalId[dtiIndex])
    }
  }
}
~~~~~



Semantics

The semantics pertaining to the AV1 descriptor syntax section above is described in this section.

Base header

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

  * **frame_dependency_template_id**: ID of the Frame dependency template to use
    or a special value to indicate a Self-defined frame. MUST be in the range of
    self_defined_id to (self_defined_id + TemplatesCnt), inclusive. When
    frame_dependency_template_id is equal to self_defined_id, the self_defintion
    MUST be present. Otherwise, self_defintion MUST NOT be present.

    frame_dependency_template_id MUST be the same for all packets of the same
    Frame. Note: values out of the valid range indicate a change of the Frame
    dependency structure.

  * **frame_dependency_template_id_or_structure_indicator**: when equal to
    TD_STRUCTURE_INDICATOR, the template_dependency_structure MUST be present.
    Otherwise, template_dependency_structure MUST NOT be present.


Template dependency structure

  * **self_defined_id**: indicates the value that frame_dependency_template_id
    MUST take when the Frame is self-defined. The value of self_defined_id
    SHOULD be chosen so that the valid frame_dependency_template_id range,
    self_defined_id to self_defined_id + TemplatesCnt, inclusive, of a new
    template_dependency_structure, does not overlap the valid
    frame_dependency_template_id range for the existing
    template_dependency_structure. When self_defined_id of a new
    template_dependency_structure is the same as in the existing
    template_dependency_structure, all fields in both
    template_dependency_structures MUST have identical values.

  * **dtis_cnt_minus_one**: dtis_cnt_minus_one + 1 indicates the number of
    Decode targets present in the coded video sequence.

  * **next_layer_idc**: used to determine spatial ID and temporal ID for the
    next Frame dependency template. Table 3 describes how the spatial ID and
    temporal ID values are determined. A next_layer_idc equal to 3 indicates
    that no more Frame dependency templates are present in the Frame dependency
    structure.

  * **template_repeatable[templateIndex]** equal to 1 indicates that a frame
    using the Frame dependency template with index equal to templateIndex will
    be used with sufficient application-dependent regularity to allow the
    endpoint to avoid sending a Switch request. Note: when the regularity of a
    template is unknown the template_repeatable[templateIndex] bit can safely be
    set to 0.

  * **chains_cnt**: indicates the number of Chains. When set to zero, the Frame
    dependency structure does not utilize protection with Chains.

  * **decode_target_protected_by[dtIndex]**: the index of the Chain that
    protects the Decode target, dtIndex. A value of
    decode_target_protected_by[dtIndex] equal to chains_cnt  indicates that
    Decode target dtIndex is not protected by a Chain. Each Decode target MAY be
    protected by at most one Chain.

  * **template_dti[templateIndex][]**: an array of size dtis_cnt_minus_one + 1
    containing Decode target information for the Frame dependency template
    having index value equal to templateIndex. Table 2 contains a description of
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


DTI
	Value


	Not present
	0
	No payload for this Decode target is present.
	Discardable
	1
	Payload for this Decode target is present and discardable.
	Switch indication
	2
	Payload for this Decode target is present and switch is possible (Switch indication).
	Required
	3
	Payload for this Decode target is present but it is neither discardable nor is it a Switch indication.
	Table 2. DTI values.


Self-defined frame

  * **next_fdiff_size**: indicates the size of following fdiff_minus_one syntax
    elements in 4-bit units. When set to a non-zero value, fdiff_minus_one MUST
    immediately follow; otherwise a value of 0 indicates no more frame
    difference values are present for the current Self-defined frame.

  * **frame_dti[dtiIndex]**: decode target information describing the
    relationship between the current frame and the Decode target having index
    equal to dtiIndex. Table 2 contains a description of the Decode target
    information values.

  * **frame_chain_fdiff[chainIdx]**: indicates the difference between the Frame
    number and the Frame number of the previous frame in the Chain having index
    equal to chainIdx. A value of 0 indicates no previous frames are needed for
    the Chain. For example, when a packet containing
    frame_chain_fdiff[chainIdx]=3 and Frame number=112 the previous frame in the
    Chain with index equal to chainIdx has Frame number=109. The calculation is
    done modulo the size of the frame_number field.


next_layer_idc
	Next Spatial ID And Temporal ID Values
	0
	The next Frame dependency template has the same spatial ID and temporal ID as the current template
	1
	The next Frame dependency template has the same spatial ID and temporal ID plus 1 compared with the current Frame dependency template.
	2
	The next Frame dependency template has temporal ID equal to 0 and spatial ID plus 1 compared with the current Frame dependency template.
	3
	No more Frame dependency templates are present in the Frame dependency structure.
	Table 3. Derivation Of Next Spatial ID And Temporal ID Values.


**TODO:** Add process descriptions and examples.

Implementing IDD with Chains[b]

The frame dependency structure includes a mapping between Decode targets and
chains. The mapping gives an SFU the ability to know the set of chains it needs
to track in order to ensure that the corresponding Decode targets remain
decodable. Every packet includes, for every chain, the Frame number for the
previous frame in that chain. An SFU can instantaneously detect a broken chain
by checking whether or not the previous frame in that chain has been received.
Due to the fact that chain information for all chains is present in all packets,
an SFU can detect a broken chain regardless of whether the first packet received
after a loss is part of that chain.


In order to start/restart chains, a packet may reference its own Frame number as
an indication that no previous frames are needed for the chain. Key frames are
common cases for such ‘(re)start of chain’ indications.


Note: chains may be used for more than just realizing the IDD property.

Switching

An SFU may begin forwarding packets belonging to a new decode target beginning
with a decodable frame containing a switch indication to that decode target. An
SFU may use the R-bit (Repeating) to determine whether or not a switch requires
a layer refresh or key frame. A template with an R-bit value equal to one
indicates Frames using that template appear in the stream in regular, repeating
manner. Presence of a template with an R-bit value equal to one, non-zero DTI
value for the current decode target, and a switch indication to the desired
decode target indicates that a switch would be possible without explicit request
for a layer refresh or key frame.


### 4.3 AV1 aggregation header

The aggregation header is used to indicate if the first and/or last OBU in the
payload is fragmented.

The structure is as follows.

         0 1 2 3 4 5 6 7
        +-+-+-+-+-+-+-+-+
        |Z|Y|-|-|-|-|-|-|
        +-+-+-+-+-+-+-+-+

Z: set to 1 if the first OBU contained in the packet is a continuation of a
previous OBU, 0 otherwise

Y: set to 1 if the last OBU contained in the packet will continue in another
packet, 0 otherwise


### 4.4 Payload Structure

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


Whether or not the first and/or last OBU is fragmented is signaled in the
aggregation header.

**TODO:** Add a paragraph that describes how a temporal unit is mapped into RTP
packets. I think a new temporal unit should start a new RTP packet. You skip the
temporal delimiter OBU, and then packetize the remaining OBUs in one or more
packets, with fragmentation as needed.

* * *

2019/01/15 Notes:

A packet must not include OBUs across a TD
All OBUs in a packet must have the same temporal_id and spatial_id
OBUs with layering information must not be aggregated with OBUs that don’t have layering information (layering information=extension header).
If sequence header is present, it should (not must) be the first OBU in a packet.
Q: May not be needed.
A packet cannot include OBUs from different frames.
Q: for hidden frames, the various frames may all be required anyway, so maybe we allow aggregation of frames with associated hidden frames (of the same temporal unit).

Q: Should we support tile list OBUs?

Packetization Exercises:

TD   SH MD MD(0,0) FH(0,0) TG0(0,0) MD(0,1) FH(0,1) TG0(0,1) …

 X   [.......................................................]
 X   [.. ……..][.........................][.............][.........................................]


FH(0,1) TG0(0,1)  TD SH MD MD(0,0) FH(0,0) TG0(0,0) MD(0,1) FH(0,1) TG0(0,1)

[ ………………….. X   … ] not allowed, SH must be at beginning of packet

FH(0,1) TG0(0,1)  TD MD MD(0,0) FH(0,0) TG0(0,0) MD(0,1) FH(0,1) TG0(0,1)

[ …………………… x  …..]


## 5. Examples

Each example in this section contains a prediction structure figure and a table
describing the associated Frame dependency structure. The Frame dependency
structure table column headings have the meanings listed below. For the DTI-
related columns, Table 2.1 shows the symbol used to represent each DTI value.

  * Idx - template index

  * S - spatial ID

  * T - temporal ID

  * Fdiffs - comma delimited list of TemplateFdiff[Idx] values

  * Chain(s) - template_chain_fdiff[Idx] values for each Chain

  * DTI - template_dti[Idx]

  * R - template_repeatable[Idx]

DTI
Value
Symbol
Not present
0
-
Discardable
1
D
Switch indication
2
S
Required
3
Q

Table 2.1. DTI values.


L1T3 Single spatial layer with 3 temporal layers


Idx
S
T
Fdiffs
Chain
DTI
R
30 fps
15 fps
7.5 fps
1
0
0


0
S
S
S
0
2
0
0
4
4
S
S
S
1
3
0
1
2
2
S
D
-
1
4
0
2
1
1
D
-
-
1
5
0
2
1
3
D
-
-
1
decode_target_protected_by
0
0
0


Note: this example uses one Chain, which includes Frames with temporal ID equal
to 0.

L2T1 Full SVC with occasional switch


Idx
S
T
Fdiffs
Chains
DTI
R
0
1
HD
VGA
1
0
0


0
0
S
S
0
2
0
0
2
2
1
Q
S
1
3
0
0
2
2
1
S
S
0
4
1
0
1
1
1
S
-
0
5
1
0
2,1
1
1
Q
-
1
decode_target_protected_by
1
0


Note: this example uses two Chains. Chain 0 includes Frames with spatial ID
equal to 0. Chain 1 includes all Frames.


L3T3 Full SVC


Idx
S
T
Fdiffs
Chains
DTI
R
0
1
2
HD
30 fps
HD
15 fps
HD
7.5 fps
VGA
30 fps
VGA
15 fps
VGA
7.5fps
QVGA
30 fps
QVGA
15 fps
QVGA
7.5 fps
1
0
0


0
0
0
S
S
S
S
S
S
S
S
S
0
2
0
0
12
12
11
10
Q
Q
Q
Q
Q
Q
S
S
S
1
3
0
1
6
6
5
4
Q
Q
-
Q
Q
-
S
D
-
1
4
0
2
3
3
2
1
Q
-
-
Q
-
-
D
-
-
1
5
0
2
3
9
8
7
Q
-
-
Q
-
-
D
-
-
1
6
1
0
1
1
1
1
S
S
S
S
S
S
-
-
-
0
7
1
0
12,1
1
1
1
Q
Q
Q
S
S
S
-
-
-
1
8
1
1
6,1
7
6
5
Q
Q
-
S
D
-
-
-
-
1
9
1
2
3,1
4
3
2
Q
-
-
D
-
-
-
-
-
1
10
1
2
3,1
10
9
8
Q
-
-
D
-
-
-
-
-
1
11
2
0
1
2
1
1
S
S
S
-
-
-
-
-
-
0
12
2
0
12,1
2
1
1
S
S
S
-
-
-
-
-
-
1
13
2
1
6,1
8
7
6
S
D
-
-
-
-
-
-
-
1
14
2
2
3,1
5
4
3
D
-
-
-
-
-
-
-
-
1
15
2
2
3,1
11
10
9
D
-
-
-
-
-
-
-
-
1
decode_target_protected_by
2
2
2
1
1
1
0
0
0


Note: this example uses three Chains. Chain 0 includes Frames with spatial ID
equal to 0 and temporal ID equal to 0. Chain 1 includes Frames with spatial ID
equal to 0 or 1 and temporal ID equal to 0. Chain 2 includes all Frames with
temporal ID equal to 0.

S3T3 K-SVC with temporal shift


Idx
S
T
Fdiffs
Chains
DTI
R
0
1
2
HD
30 fps
HD
15 fps
HD
7.5 fps
VGA
30 fps
VGA
15 fps
VGA
7.5 fps
QVGA
30 fps
QVGA
15 fps
QVGA
7.5 fps
1
0
0


0
0
0
S
S
S
S
S
S
S
S
S
0
2
0
0
3
3
2
1
-
-
-
-
-
-
S
S
S
0
3
0
0
12
12
8
1
-
-
-
-
-
-
S
S
S
1
4
0
1
6
6
2
7
-
-
-
-
-
-
S
D
-
1
5
0
2
3
3
5
4
-
-
-
-
-
-
D
-
-
0
6
0
2
3
9
5
10
-
-
-
-
-
-
D
-
-
1
7
0
2
3
3
11
4
-
-
-
-
-
-
D
-
-
1
8
1
0
1
1
1
1
S
S
S
S
S
S
-
-
-
0
9
1
0
6
4
6
5
-
-
-
S
S
S
-
-
-
0
10
1
0
12
4
12
5
-
-
-
S
S
S
-
-
-
1
11
1
1
6
10
6
11
-
-
-
S
D
-
-
-
-
1
12
1
2
3
1
3
2
-
-
-
D
-
-
-
-
-
0
13
1
2
3
7
3
8
-
-
-
D
-
-
-
-
-
1
14
1
2
3
1
9
2
-
-
-
D
-
-
-
-
-
1
15
2
0
1
2
1
1
S
S
S
-
-
-
-
-
-
0
16
2
0
12
11
7
12
S
S
S
-
-
-
-
-
-
1
17
2
1
6
5
1
6
S
D
-
-
-
-
-
-
-
1
18
2
2
3
2
4
3
D
-
-
-
-
-
-
-
-
0
19
2
2
3
8
4
9
D
-
-
-
-
-
-
-
-
1
20
2
2
3
2
10
3
D
-
-
-
-
-
-
-
-
1


HD
30 fps
HD
15 fps
HD
7.5 fps
VGA
30 fps
VGA
15 fps
VGA
7.5 fps
QVGA
30 fps
QVGA
15 fps
QVGA
7.5 fps


decode_target_protected_by
2
2
2
1
1
1
0
0
0


Note: this example uses three Chains. Chain 0 includes Frames with spatial ID
equal to 0 and temporal ID equal to 0. Chain 1 includes Frame 100 and Frames
with spatial ID equal to 1 and temporal ID equal to 0. Chain 2 includes Frames
100, 101, and Frames with spatial ID equal to 2 and temporal ID equal to 0.

##6. References


### 6.1 Normative References

  * RFC3550 for rtp header format
  * RFC8285 for generic rtp header extensions
  * RFC 7667 RTP Topologies
  * AV1 Bitstream & Decoding Process Specification

**TODO:** flesh out list of normative references.

6.2 Informative References

**TODO:** list informative references.


