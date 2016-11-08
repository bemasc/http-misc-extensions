---
title: QPACK - Header Compression for HTTP over QUIC
abbrev: QPACK
docname: draft-bishop-quic-qpack-latest
date: 2016
category: std

ipr: trust200902
area: General
workgroup: QUIC Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: M. Bishop
    name: Mike Bishop
    organization: Microsoft
    email: michael.bishop@microsoft.com

normative:
  RFC7230:
  RFC7231:
  RFC2119:
  RFC7541:
  I-D.hamilton-quic-transport-protocol:
  I-D.shade-quic-http2-mapping:

informative:
  RFC7540:



--- abstract

HTTP/2 [RFC7540] uses HPACK [RFC7541] for header compression.
However, HPACK relies on the in-order message-based semantics
of the HTTP/2 framing layer in order to function.  Messages can
only be successfully decoded if processed by the receiver in the
same order as generated by the sender.  This draft refines HPACK
to loosen the ordering requirements for use over 
QUIC [I-D.hamilton-quic-transport-protocol].

--- middle

# Introduction        {#problems}

HPACK has a number of features that were intended to provide performance 
advantages to HTTP/2, but which don't live well in an out-of-order 
environment such as that provided by QUIC. 

The largest challenge is the fact that elements are referenced by a very 
fluid index. Not only is the index implicit when an item is added to the 
header table, the index will change without notice as other items are 
added to the header table. Static entries occupy the first 61 values, 
followed by dynamic entries. A newly-added dynamic entry would cause 
older dynamic entries to be evicted, and the retained items are then 
renumbered beginning with 62. This means that, without processing all 
preceding header sets, no index into the dynamic table can be 
interpreted, and the index of a given entry cannot be predicted. 

Any solution to the above will almost certainly fall afoul of the memory 
constraints the decompressor imposes. The automatic eviction of entries 
is done based on the compressor's declared dynamic table size, which 
MUST be less than the maximum permitted by the decompressor (and relayed 
using an HTTP/2 SETTINGS value). 

In the following sections, this document proposes a new version of HPACK 
which makes different trade-offs, enabling out-of-order interpretation 
and bounded memory consumption with minimal head-of-line blocking. 


## Terminology          {#Terminology}
In this document, the key words "MUST", "MUST NOT", "REQUIRED",
"SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY",
and "OPTIONAL" are to be interpreted as described in BCP 14, RFC 2119
{{RFC2119}} and indicate requirement levels for compliant STuPiD
implementations.

# QPACK {#QPACK}

## Changes to Static and Dynamic Tables

QPACK uses two tables for associating header fields to indexes. The 
static table is unchanged from [RFC7541]. 

The dynamic table is a map from index to header field. Indices are 
arbitrary numbers greater than the last index of the static table. Each 
insert instruction will specify the index being modified. While any 
index MAY be chosen for a new entry, smaller numbers will yield better 
compression performance. Once an index has been assigned, its value is 
immutable for the lifetime of that dynamic table. 

(Note: Changing or deleting values re-introduces strong ordering 
requirements this draft attempts to eliminate. Suggestions for adding 
this feature without such requirements are welcome.) 

The dynamic table is still constrained to the size specified by the 
receiver. An attempt to add a header to the dynamic table which causes 
it to exceed the maximum size MUST be treated as an error by a decoder. 

Because it is possible for QPACK frames to arrive which reference 
indices which have not yet been defined, such frames MUST wait until 
another frame has arrived and defined the index. In order to guard 
against malicious senders, implementations SHOULD impose a time limit 
and treat expiration of the timer as a decoding error. However, if the 
implementation chooses not to abort the connection, the remainder of the 
header block MUST be decoded and the output discarded. 

### Dynamic table management and phase

No entries are evicted from the dynamic table. Size management is 
achieved using successive phases of dynamic table which evolve from each 
other. 

Each QPACK frame will contain two bits of phase, the sending phase and 
the received phase. When the dynamic table is approaching the maximum 
size, the encoder condenses the entries by sending a PHASE_CHANGE 
command (see {{phase-change}}). The phase change creates a new dynamic 
table containing a subset of entries from the previous phase's dynamic 
table. The encoder then discards the previous phase's dynamic table and 
uses the new table as the basis for the remainder of the current frame and
all future QPACK frames.

The decoder will be unable to process any index references from the new 
phase until it has received the QPACK frame containing the phase-change 
instruction.

When the decoder has received and processed all QPACK frames generated 
using the previous phase, it discards the previous phase's dynamic table 
and changes the value of its received phase bit. An encoder MUST NOT 
generate a new PHASE_CHANGE command until it has received a QPACK frame 
with a received phase matching the updated sending phase value. 

## Changes to Binary Format

### Literal Header Field Representation

(This section replaces [RFC7541], Section 6.2.1.)

A literal header field with indexing representation results in 
inserting a header field to the decoded header list and inserting it as 
a new entry into the dynamic table. 

~~~~~~~~~~
     0   1   2   3   4   5   6   7
   +---+---+---+---+---+---+---+---+
   | 0 | 1 |    New Index (6+)     |
   +---+---+-----------------------+
   |          Name Index (8+)      |
   +---+---------------------------+
   | H |     Value Length (7+)     |
   +---+---------------------------+
   | Value String (Length octets)  |
   +-------------------------------+
~~~~~~~~~~
{: title="Literal Header Field with Indexing -- Indexed Name"}

~~~~~~~~~~
     0   1   2   3   4   5   6   7
   +---+---+---+---+---+---+---+---+
   | 0 | 1 |    New Index (6+)     |
   +---+---+-----------------------+
   |               0               |
   +---+---+-----------------------+
   | H |     Name Length (7+)      |
   +---+---------------------------+
   |  Name String (Length octets)  |
   +---+---------------------------+
   | H |     Value Length (7+)     |
   +---+---------------------------+
   | Value String (Length octets)  |
   +-------------------------------+
~~~~~~~~~~
{: title="Literal Header Field with Indexing -- New Name"}

A literal header field with incremental indexing representation starts 
with the '01' 2-bit pattern, followed by the new index of the header 
represented as an integer with a 6-bit prefix. This value is always 
greater than the number of entries in the static table. 

If the header field name matches the header field name of an entry 
stored in the static table or the dynamic table, the header field name 
can be represented using the index of that entry. In this case, the 
index of the entry is represented as an integer with an 8-bit prefix 
(see Section 5.1). This value is always non-zero. 

Otherwise, the header field name is represented as a string literal 
(see Section 5.2). A value 0 is used in place of the 8-bit index, 
followed by the header field name. 

Either form of header field name representation is followed by the 
header field value represented as a string literal (see Section 5.2). 


### Dynamic Table Phase Change {#phase-change}

(This section replaces [RFC7541], Section 6.3.)

A "Phase Change" instruction creates a new dynamic table consisting of entries
from the previous phase's dynamic table.  The new dynamic table begins empty,
and the insertion index is placed at the first index after the static table.
The instructions are then processed in order to populate the new dynamic table.

The Phase Change instruction begins with the count of QPACK frames
which have already been encoded using the previous phase, followed
by a sequence of instructions for generating the new dynamic table as follows:

~~~~~~~~~~
     0   1   2   3   4   5   6   7
   +---+---+---+---+---+---+---+---+
   | 0 | 0 | 1 |  Phase Count (5+) |
   +---+---------------------------+
   | Instructions...               |
   +---+---------------------------+   
~~~~~~~~~~
{: title="Phase Change"}

The count of preceding frames, given as a 5-bit prefix integer, does not include
the frame containing the current Phase Change instruction or the frame containing
the previous Phase Change instruction.

#### Insert single entry from previous phase

~~~~~~~~~~
     0   1   2   3   4   5   6   7
   +---+---+---+---+---+---+---+---+
   | 1 |        Index (7+)         |
   +---+---------------------------+
~~~~~~~~~~
{: title="Single Entry Insertion"}

The entry with the specified entry from the previous phase's dynamic table
is inserted into the new phase's dynamic table at the current insertion point
and the insertion index is incremented.

#### Insert multiple entries from previous phase

~~~~~~~~~~
     0   1   2   3   4   5   6   7
   +---+---+---+---+---+---+---+---+
   | 0 | 1 |      Count (6+)       |
   +---+---------------------------+
   |        Starting Index (8+)    |
   +---+---------------------------+
~~~~~~~~~~
{: title="Multiple Entry Insertion"}

A number of contiguous entries, given by Count, are read from the previous
phase's dynamic table and included at the insertion point in the new phase's dynamic
table, beginning at Starting Index.

#### Complete phase change

~~~~~~~~~~
     0   1   2   3   4   5   6   7
   +---+---+---+---+---+---+---+---+
   |               0               |
   +---+---+---+---+---+---+---+---+
~~~~~~~~~~
{: title="End of Phase Change"}

The encoder has discarded the previous dynamic table state and will use 
the newly-expressed state of the dynamic table for the remainder of this 
QPACK frame and for all future QPACK frames. The decoder should likewise 
discard the previous dynamic table once it has processed the specified 
number of frames using the previous phase. 

# HTTP over QUIC Mapping Changes

[I-D.shade-quic-http2-mapping] refers to QUIC Stream 3 as carrying
"headers," but more accurately, it carries a nearly-complete HTTP/2 session,
complete with framing and multiplexing.  The mapping deletes certain elements
of HTTP/2's framing layer which can be delegated to the QUIC transport layer.

This was done in large part for expediency, reusing HTTP/2 code in place
anywhere no QUIC-specific approach had yet been added.  A primary driver of
this is the need for in-order reliable delivery of frames carrying HPACK data
(HEADERS, CONTINUATION, PUSH_PROMISE).

QPACK would permit header data to be on-stream with the request/response bodies,
but some framing is still required.  It would be possible (and perhaps desirable)
to introduce a simplified version of HTTP/2's framing on each QUIC stream.

## On-Stream Framing Definition

Many framing concepts from HTTP/2 can be elided away on QUIC, because the transport
deals with them.  Because these frames would already be on a stream, they can omit the stream number.
Because the frames do not block multiplexing (QUIC's multiplexing occurs below this layer),
they can be as long as needed.  Because stream termination is handled by QUIC, END_STREAM is not required.

We would also prefer to minimize the overhead of common frame types, 
since this is yet another layer of framing to be processed, so the frame 
type space has been reduced. If more frame types are needed, a type can 
be used to define a frame which contains a longer type field. (We could 
conceivably reduce the type to two bits and define that sub-type up 
front.)

On QUIC streams other than Stream 1, the format is as follows:

~~~~~~~~~~
     0   1   2   3   4   5   6   7
   +---+---+---+---+---+---+---+---+
   |          Length (16)          |
   |                               |
   +---+---+---+---+---+---+---+---+
   |    Type (4)   |     Flag(4)   |
   +---+---+---+---+---+---+---+---+
   |        Frame Payload        ...
   +---+---+---+---+---+---+---+---+
~~~~~~~~~~
{: title="HTTP/QUIC frame format"}

The frames currently defined are described in this section:

### DATA

DATA frames (type=0x0) convey arbitrary, variable-length sequences of 
octets which make up the HTTP request or response payloads. No padding 
is defined, as QUIC provides facilities for this. No flags are defined. 

### HEADERS

The HEADERS frame (type=0x1) is used to carry part of a header set,
compressed using QPACK ({QPACK}).

No padding is defined.  The flags defined are:

 - Sending phase (0x1): The QPACK phase in use by the sender's encoder 
when it began generating this frame. 

- Receiving phase (0x2): The oldest QPACK phase still in use by the 
sender's decoder when it began generating this frame. 

- End Header Block (0x4): This frame concludes a header block. 

The next frame after a HEADERS frame without the EHB flag set MUST be 
another HEADERS frame. A receiver MUST treat the receipt of any other 
type of frame as a connection error. (Note that QUIC may intersperse 
data from other streams between frames, or even during transmission of 
frames, so multiplexing is not blocked.) 

A full header block is contained in a sequence of zero or more HEADERS 
frames without EHB set, followed by a HEADERS frame with EHB set. 

HEADERS frames from various streams may be processed by the QPACK 
decoder in any order, completely or partially. It is not necessary to 
withhold decoding results until the end of the header block has arrived. 
However, depending on the contents, the processing of a frame might not 
complete until other QPACK frames have arrived. The results of decoding 
MUST be emitted in the same order as the HEADERS frames were placed on 
the stream.

### PUSH_PROMISE

The PUSH_PROMISE frame (type=0x02) is used to carry a request header set 
from server to client, as in HTTP/2. It contains the same flags as 
HEADERS, plus: 

- Promised Stream Size (0x08): Indicates whether Promised Stream ID is 
16 or 32-bits long.

The payload contains a QPACK headers block encoding 
the request whose response is promised, preceded by a 16-bit or 32-bit 
long Stream ID indicating the QUIC stream on which the response headers 
and body will be sent.

### Other frames not mentioned

Still need a way to express connection-level ideas like PRIORITY and 
SETTINGS. PRIORITY really is talking *about* a stream, not data on it. 
Perhaps these remain on Stream 3? Extension frames, likewise.

Being on QUIC stream 3 has the same semantics as being on HTTP/2 stream 0;
these frames likewise could omit the stream number from the framing.  We
might reuse the frame type space and have alternate interpretations when
on the "connection management" stream.

TBD. 

## HTTP Message Exchanges 

A client sends an HTTP request on a new QUIC stream. A server sends an 
HTTP response on the same stream as the request. 

An HTTP message (request or response) consists of: 

1. for a response only, zero or more header blocks (a sequence of 
HEADERS frames with End Header Block set on the last) containing the 
message headers of informational (1xx) HTTP responses (see [RFC7230], 
Section 3.2 and [RFC7231], Section 6.2), 

2. one header block containing the message headers (see [RFC7230], 
Section 3.2), 

3. zero or more DATA frames containing the payload body (see [RFC7230], 
Section 3.3), and 

4. optionally, one header block containing the trailer-part, if present 
(see [RFC7230], Section 4.1.2). 

Following the last frame in the sequence, the QUIC stream is half-closed 
in the sender's direction. 

DATA frames are used to carry message payloads. The "chunked" transfer 
encoding defined in Section 4.1 of [RFC7230] MUST NOT be used. 

Trailing header fields are carried in a header block following the body. 
Such a header block is a sequence of HEADERS frames with End Header 
Block set on the last frame. Header blocks after the first but before 
the end of the stream are invalid and MUST be discarded (after decoding, 
to maintain QPACK decoder state). 

A header block can only appear at the start or end of a stream. An 
endpoint that receives a HEADERS frame after receiving a final (non- 
informational) status code MUST treat the corresponding request or 
response as malformed (Section 8.1.2.6). 

An HTTP request/response exchange fully consumes a single stream. After 
sending a request, a client closes the stream for sending; after sending 
a response, the server closes its stream for sending and the QUIC stream 
is fully closed. A response starts with a HEADERS frame and ends with a 
frame bearing END_STREAM, which places the stream in the "closed" state. 

A server can send a complete response prior to the client sending an 
entire request if the response does not depend on any portion of the 
request that has not been sent and received. When this is true, a server 
MAY request that the client abort transmission of a request without 
error by sending a RST_STREAM with an error code of NO_ERROR after 
sending a complete response and closing its stream. Clients MUST NOT 
discard responses as a result of receiving such a RST_STREAM, though 
clients can always discard responses at their discretion for other 
reasons. 

# Performance Considerations

While QPACK is designed to minimize head-of-line blocking between 
streams on header decoding, there are some situations in which lost or 
delayed packets can block decoding of subsequent frames: 

- References to indexed entries will block if the frame containing the 
entry definition is lost or delayed.

- If the frame initiating a phase change is lost or delayed, the decoder 
cannot process any indexed entries from the new phase until it arrives. 

Encoders MAY choose to avoid references to these entries until the 
packet containing the definiting frame has been acknowledged by the 
decoder. In both cases, the resulting literal values used instead will 
reduce compression efficiency, but avoid blocking either the encoder or 
decoder. 

If frames from the previous phase have been lost or delayed, the decoder 
will delay acknowledging and completing a phase change. The encoder 
can find itself in a state where no new entries can be added to the 
dynamic table. The encoder is not blocked, but will need to use literal 
values until the phase change completes. 

Memory-constrained implementations MAY delay processing of *all* QPACK 
frames from the new phase until the previous phase has completed, in 
order to avoid maintaining two tables in parallel. Doing so would 
introduce a connection-wide delay, so encoders SHOULD minimize the 
frequency of phase changes. 

# Security Considerations

The security considerations for QPACK are believed to be the same
as for HPACK, with one addition.  An encoder could maliciously or
mistakenly claiming to have encoded more frames than it has
actually sent while initiating
a phase-change.  Decoders SHOULD defend themselves against such 
implementations by considering it a connection error if the phase
change cannot be completed within a reasonable number of RTTs,
or if the transport reports that there is no unreceived data
still pending.

# IANA Considerations

This document currently makes no request of IANA, but probably should.

# Acknowledgements {#ack}

This draft draws heavily on the text of [RFC7540], [RFC7541],
and [I-D.shade-quic-http2-mapping].  The indirect input
of those authors is gratefully acknowledged, as well as ideas
stolen from:

  - Jana Iyengar
  - Patrick McManus
  - Martin Thomson
  - Charles 'Buck' Krasic

--- back

