---
title: "SIPv3: Session Initiation Protocol over QUIC"
abbrev: "SIPv3"
category: exp

docname: draft-hurst-sip-quic-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: ART
workgroup: Session Initiation Protocol Core
keyword: Internet-Draft
venue:
  group: WG
  type: "Individual Draft"
  mail: WG@example.com
  arch: https://example.com/WG
  github: USER/REPO
  latest: https://example.com/LATEST

author:
 -
    fullname: Sam Hurst
    organization: BBC Research & Development
    email: sam.hurst@bbc.co.uk

normative:
  SIP2.0: RFC3261
  RFC3264: # Offer/Answer
  RFC7639:
  QUIC-TRANSPORT: RFC9000
  HTTP-SEMANTICS: RFC9110
  HTTP3: RFC9114
  QPACK: RFC9204
informative:
  RFC8499: # DNS Terminology
  HTTP1.1: RFC9112
  HTTP2: RFC9113
  QUIC-DATAGRAMS: RFC9221
  QRT:
    title: "QRT: QUIC RTP Tunnelling"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-hurst-quic-rtp-tunnelling-01
    author:
      -
        ins: S. Hurst
        name: Sam Hurst
        org: BBC Research & Development
  RTP-over-QUIC:
    title: "RTP over QUIC"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-ietf-avtcore-rtp-over-quic-00
    author:
      -
        ins: J. Ott
        name: Jörg Ott
        org: Technical University Munich
      -
        ins: M. Engelbart
        name: Mathis Engelbart
        org: Technical University Munich
  QuicR-Arch:
    title: "QuicR - Media Delivery Protocol over QUIC"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-jennings-moq-quicr-arch-01
    author:
      -
        ins: C. Jennings
        name: Cullen Jennings
        org: Cisco
      -
        ins: S. Nandakumar
        name: Suhas Nandakumar
        org: Cisco
  QuicR-Proto:
    title: "QuicR - Media Delivery Protocol over QUIC"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-jennings-moq-quicr-proto-01
    author:
      -
        ins: C. Jennings
        name: Cullen Jennings
        org: Cisco
      -
        ins: S. Nandakumar
        name: Suhas Nandakumar
        org: Cisco
      -
        ins: C. Huitema
        name: Christian Huitema
        org: Private Octopus Inc.
  RUSH:
    title: "RUSH - Reliable (unreliable) streaming protocol"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-kpugin-rush-01
    author:
      -
        ins: K. Pugin
        name: Kirill Pugin
        org: Facebook
      -
        ins: A. Frindell
        name: Alan Frindell
        org: Facebook
      -
        ins: J. Cenzano
        name: Jordi Cenzano
        org: Facebook
      -
        ins: J. Weissman
        name: Jake Weissman
        org: Facebook
  Warp:
    title: "Warp - Segmented Live Media Transport"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-lcurley-warp-01
    author:
      -
        ins: L. Curley
        name: Luke Curley
        org: Twitch
  SDP-QUIC:
    title: "SDP Offer/Answer for RTP using QUIC as Transport"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-dawkins-avtcore-sdp-rtp-quic-00
    author:
      -
        ins: S. Dawkins
        name: Spencer Dawkins
        org: Tencent America LLC
  WebTransH3:
    title: "WebTransport over HTTP/3"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-ietf-webtrans-http3-03
    author:
      -
        ins: A. Frindell
        name: Alan Frindell
        org: Facebook
      -
        ins: E. Kinnear
        name: Eric Kinnear
        org: Apple Inc.
      -
        ins: V. Vasiliev
        name: Victor Vasiliev
        org: Google


--- abstract

This document describes a mapping of Session Initiation Protocol (SIP) semantics over QUIC Transport. It allows the
creation, modification and temination of meida sesions with one or more participants, possibly carried over the same
QUIC transport connection, using RTP/AVP directly, or some mixture of both.

SIP version 3 (SIP/3) enables a more efficient use of network resources by introducing field compression to the header
fields carried in SIP transactions.

--- middle

# Introduction {#introduction}

The Session Initiation Protocol (SIP) {{!RFC3261}} is widely used for managing media sessions over the internet.
Examples of these media sessions include internet telephony services, video conferencing and live streaming of media.

{{SIP2.0}} uses whitespace-delimited text fields to convey SIP messages in a similar format to HTTP/1.1 {{HTTP1.1}},
and may optionally be transported over TLS (so called "SIPS"). SIP/3, as defined by this document, uses a binary
framing layer carried over QUIC streams and is protected by the mandatory TLS encryption afforded by the QUIC
transport connection. A future optional extension may introduce the ability to carry SIP messages in QUIC Datagrams
{{QUIC-DATAGRAMS}}.

## Conventions {#conventions}

{::boilerplate bcp14-tagged}

## Definitions {#definitions}

*[malformed]: #

# SIP/3 Protocol Overview {#sip3-overview}

SIP/3 provides a transport for SIP semantics using the QUIC transport protocol and an internal framing layer based on
{{HTTP3}}.

Once a user agent client knows that a user agent server supports SIP/3, it opens a QUIC connection. QUIC provides
protocol negotiation, stream-based multiplexing, and flow control. SIP transactions are multiplexed across QUIC
streams, which is described in {{Section 2 of QUIC-TRANSPORT}}. Each request and response consumes a single QUIC
stream.

Within each stream, the basic unit of SIP/3 communication is a frame as described in {{framing-layer}}. Each frame type
serves a different purpose. The HEADERS and DATA frames form the basis of the offer/answer transaction model described
in {{RFC3264}} and are described in {{sip-transaction-framing}}.

In {{SIP2.0}}, some header fields may be compressed by using abbreviated versions. In SIP/3, all request and response
header fields are compressed for transmission using {{QPACK}}, in which header fields may be mapped to indexed values,
or literal values may be encoded using a static Huffman code. {{QPACK}} uses two tables for its indexed values; the
static table is predefined with common header fields and values, and the dynamic table can be used to encode frequently
used header fields to reduce repetition. Because {{QPACK}}'s static table is designed to work with {{HTTP3}}, this
specification replaces the default static table defined in {{Appendix A of QPACK}} with the table in
{{static-table}}.

## QUIC Transport {#quic-transport}

SIP/3 relies on QUIC version 1 as the underlying transport. The use of other QUIC transport versions with SIP/3 MAY be
defined by future specifications. QUIC version 1 uses TLS version 1.3 or greater as its handshake protocol. SIP/3 user
agents MUST support a mechanism to indicate the target host to the server during the TLS handshake. If the target
destination user agent server is identified by a domain name {{?RFC8499}}, clients MUST send the Server Name Indication
({{?SNI=RFC6066}}) TLS extension unless an alternative mechanism to indicate the target host is used.

QUIC connections are established as described in {{!RFC9000}}. During connection establishment, SIP/3 support is
indicated by selecting the ALPN token "sips/3" in the TLS handshake. Support for other application-layer protocols MAY
be offered in the same handshake.

> **Author's Note:** Perhaps the ALPN string should actually be "s3" instead of "sips/3", similar to h3 vs. HTTP/3?
Should probably mint the ALPN token "sips/2.0" as well, for backwards compatibility?

### Draft Version Identification {#draft-version-indication}

> **RFC Editor's Note:** Please remove this section prior to publication of a final version of this document.

Only implementations of the final, published RFC can identify themselves as "sips/3". Until such an RFC exists,
implementations MUST NOT identify themselves using this string. Implementations of draft versions of the protocol MUST
add the string "-h" and the corresponding draft number to the identifier. For example, draft-hurst-sip-quic-00 is
identified using the string "sips/3-h00".

Non-compatible experiments that are based on these draft versions MUST append the string "-" and an experiment name to
the identifier. For example, an experimental implementation based on draft-hurst-sip-quic-00 which uses QUIC datagrams
instead of QUIC streams to carry SIP messages might identify itself as "sips/3-h00-datagrams". Note that any label MUST
conform to the "token" syntax defined in {{Section 5.6.2 of HTTP-SEMANTICS}}. Experimenters are encouraged to
coordinate their experiments.

## Connection Reuse {#connection-reuse}

SIP/3 connections are persistent across multiple transactions. SIP/3 connections may also be persistent across multiple
established dialogs. A single SIP/3 connection may carry transactions for multiple distinct dialogs simultaneously,
with each dialog individually identified by the `Call-ID` header and `tag=` parameters on the `To` and `From` headers.

# Expressing SIP Semantics in SIP/3 {#sip3-semantics}

## QUIC Clients and Servers {#quic-clients-servers}

As the intention with SIP/3 is to reuse the transport association as much as possible, this is not entirely compatible
with conventional SIP's user agent client and user agent server model. Instead, this specification introduces two new
terms, "transport client" and "transport server", to denote the QUIC client as the initiator of the transport
connection, and the QUIC server as its peer.

## SIP Transaction Framing {#sip-transaction-framing}

SIP transactions begin with a request being sent by a user agent client (UAC) on a request stream, which is a
bidirectional QUIC stream; see {{stream-mapping}}. If the user agent client making the request is accessing the
transport connection from the transport client endpoint, then the request stream is carried on a client-initiated
bidirectional QUIC stream. If the user agent client making the request is accessing the transport connection from the
transport server endpoint, then the request stream is carried on a server-initiated bidirectional QUIC stream.

Each SIP transaction has exclusive use of a request stream. Only one request may be made per request stream. The user
agent server sends zero or more provisional responses on the same stream as the request, followed by one or more final
responses. See {{Section 7.2 of SIP2.0}} for a description of provisional and final responses.

SIP/3 does not use any unidirectional QUIC streams.

On a given request stream, receipt of multiple requests MUST be treated as malformed.

A SIP message (request or response) consists of:

1. the header section, including message control data, sent as a single HEADERS frame,
2. optionally, the message body, if present, sent as a series of DATA frames, and
3. optionally, if the initial request used the `INVITE` method, a single DIALOG_TOKEN frame.

Headers are described in {{Section 7.3 of SIP2.0}}. Message bodies are described in {{Section 7.4 of SIP2.0}}.

Receipt of an invalid sequence of frames MUST be treated as a connection error of type SIP3_FRAME_UNEXPECTED. In
particular, a DATA frame before any HEADERS frame is considered invalid. Other frame types, especially unknown frame
types, may be permitted subject to their own rules, see {{extensions}}.

The HEADERS frame might reference updates to the QPACK dynamic table. While these updates are not directly part of the
message exchange, they MUST be received and processed before the message can be consumed.

{::comment}
The HTTP/3 spec has something about Transfer-Encoding here - SIP/2.0 has the Content-Transfer-Encoding header, should
this be also banned here?
{:/comment}

After sending a request, the client MUST close the stream for sending (see {{Section 3.4 of QUIC-TRANSPORT}}). After
sending the last final response, the server MUST close the stream for sending. At this point, the QUIC stream is fully
closed.

When a stream is closed, this indicates the completion of the SIP transaction. If a stream terminates without enough of
the request to provide a complete response, the server SHOULD abort the stream with the error code
SIP3_REQUEST_INCOMPLETE.

{::comment}
The HTTP/3 spec has a section about the server being allowed to respond before the client has finished sending an
entire request. Is this something to consider banning here too?
{:/comment}

### Request Cancellation and Rejection {#cancel-request}

Once a request stream has been opened, the request MAY be cancelled by either endpoint. A client SHOULD gracefully
cancel a request by making a new request stream and sending a cancellation request as described in
{{Section 9 of SIP2.0}}, or alternatively using the CANCEL frame.

An endpoint MAY abruptly cancel any request by resetting both the sending and receiving parts of the streams by
sending a `RESET_STREAM` frame (see {{Section 19.4 of QUIC-TRANSPORT}}) and a `STOP_SENDING` frame (see
{{Section 19.5 of QUIC-TRANSPORT}}). A user agent client may wish to do this if it has run out of new streams to open
in order to send a new cancellation request, or does not wish to use the CANCEL frame.

When the user agent server cancels a request without perfoming any application processing, the request is considered
"rejected". The server SHOULD abort its response stream with the error code SIP3_REQUEST_REJECTED. In this context,
"processed" means that some data from the request stream was passed to some higher layer of software that might have
taken some action as a result. The user agent client can treat requests rejected by the user agent server as though
they had never been sent at all, and may be retried later.

User agent servers MUST NOT use the SIP3_REQUEST_REJECTED error code for requests that were partially or fully
processed. When a server abandons a response after partial processing, it SHOULD abort its response stream with
the error code SIP3_REQUEST_CANCELLED.

User agent clients SHOULD use the error code SIP3_REQUEST_CANCELLED to cancel requests. Upon receipt of this error
code, user agent servers MAY abruptly terminate the response using the error code SIP3_REQUEST_REJECTED if no
processing was performed. User agent clients MUST NOT use the SIP3_REQUEST_REJECTED error code, except when a user
agent server has requested closure of the request stream with this error code.

### Malformed requests and responses {#malformed}

A malformed request or response is one that is an otherwise valid sequence of frames but is invalid due to:

* the presence of prohibited header fields or pseudo-header fields,
* the absense of mandatory pseudo-header fields,
* invalid values for pseudo-header fields,
* pseudo-header fields after header fields,
* an invalid sequence of SIP messages, such as a message body being present before a header section,
* the inclusion of uppercase header field names,
* the inclusion of invalid characters in field names or values.

A request or response that is defined as having content when it contains a `Content-Length` header field (see
{{Section 18.3 of SIP2.0}}) is malformed if the value of the `Content-Length` header field does not equal the sum of
the DATA frame lengths received.

Intermediaries that process SIP requests or responses such as a proxy server MUST NOT forward a malformed request or
response. Malformed requests or responses that are detected MUST be treated as a stream error of type
SIP3_MESSAGE_ERROR.

For malformed requests, a user agent server MAY send an HTTP response indicating the error prior to closing or
resetting the stream. Clients MUST NOT accept a malformed response.


## SIP Header Fields {#header-fields}

SIP messages carry metadata as a series of key-value pairs called "SIP header fields"; see {{Section 7.3 of SIP2.0}}.
For a listing of registered SIP header fields, see the "Session Initiation Protocol (SIP) Parameters - Header Fields
Registry" maintained at <https://www.iana.org/assignments/sip-parameters/sip-parameters.xhtml#sip-parameters-2>.

In SIP/3, header fields are compressed and decompressed by {{QPACK}}, including the control data present in the header
section. The static table defined in {{Appendix A of QPACK}} is designed for use with HTTP, and as such contains
header fields that are of little to no interest to SIP endpoints. {{static-table}} in this document defines a
replacement static table that MUST be used with SIP/3.

The abbreviated forms of SIP header fields described in {{Section 7.3.3 of SIP2.0}} MUST NOT be used with
SIP/3.

SIP/3 does not use the `CSeq` header field (see {{Section 20.16 of SIP2.0}}) because the correct order of requests can
be inferred by the QUIC stream identifier as described in {{stream-mapping}}. Intermediaries that convert and forward
SIP/3 messages as earlier versions of SIP are responsible for defining the value carried in the `CSeq` header field for
those messages, and the mapping of those values back to the requisite SIP/3 request stream.

### Contact Header Field Version Extension {#contact-extension}

This document defines an additional `Contact` header field parameter "v". The "v" parameter carries a comma-separated
list of SIP versions which the given SIP endpoint supports in descending preference order. The ABNF (Augmented
Backus-Naur Form) syntax for the "v" parameter is given below. It follows the same syntax as defined in
{{Section 2.2 of RFC7639}}.

~~~
contact-version = "v=" DQUOTE version-list DQUOTE
version-list    = 1#protocol-id
protocol-id     = token ; percent-encoded ALPN protocol identifier
~~~
{: #fig-contact-version-abnf title="ABNF syntax for the \"v\" parameter"}

An example `Contact:` header using this extension is given below.

~~~
Contact: "Mr. Watson" <sip:watson@example.com>;
    q=0.7; expires=3600; v="sips%2F3,sips%2F2.0,sip%2F2.0",
    "Mr. Watson" <mailto:watson@example.com>; q=0.1
~~~
{: #fig-contact-version-extension-example title="Example usage of the \"v\" Contact header field parameter"}

### SIP Control Data

Similar to {{HTTP2}} and {{HTTP3}}, SIP/3 employs a series of pseudo-header fields where the field name begins with
the `:` character (ASCII 0x3a). These pseudo-header fields convey message control data, which replaces the
`Request-Line` described in {{Section 7.1 of SIP2.0}}.

Pseudo-header fields are not SIP header fields. Endpoints MUST NOT generate pseudo-header fields other than those
defined in this document. However, an extension could negotiate a modification of this restriction; see {{extensions}}.

Pseudo-header fields are only valud in the context in which they are defined. Pseudo-header fields defined for requests
MUST NOT appear in responses; pseudo-header fields defined for responses MUST NOT appear in requests. Pseudo-header
fields MUST NOT appear in trailer sections. Endpoints MUST treat a request or response that contains undefined or
invalid pseudo-header fields as {{malformed}}.

All pseudo-header fields MUST appear in the header section before regular header fields. Any request or response that
contains a pseudo-header field that appears in a header section after a regular header field MUST be treated as
{{malformed}}.

#### Request Pseudo-header fields

The following pseudo-header fields are defined for requests:

":method": Contains the SIP method. See {{methods}} to understand SIP/3-specific usages of SIP methods.

":request-uri": Contains the SIPS URI as described in {{Section 19.1 of SIP2.0}}.

All SIP/3 requests MUST include exactly one value for the `:method` and `:request-uri` pseudo-header fields. The
`SIP-Version` element of the `Request-Line` structure in {{Section 7.1 of SIP2.0}} is omitted, as the SIP version is
given by the negotiated ALPN version string as described in {{quic-transport}}, and as such all SIP/3 requests
implicitly have a protocol version of "3.0".

A SIP request that omits any mandatory pseudo-header fields or contains invalid values for those pseudo-header fields
is {{malformed}}.

# Compatibility With Earlier SIP Versions {#compatiblity}

~~~
+-----------+       +-------+         +-------+         +---------+
|           |       |       |         |       |         |         |
|           <=SIP/3=> Proxy <-SIP/2.0-> Proxy <-SIP/2.0->         |
|           |       |   A   |         |   B   |         |         |
| Initiator |       +-------+         +-------+         | Invitee |
|   (UAC)   |                                           |  (UAS)  |
|           <==================SIP/3====================>         |
|           |                                           |         |
+-----------+                                           +---------+
~~~
{: #fig-mixed-sip-versions title="Example showing mixed SIP versions"}

In the above example, the proxy initiator, invitee and proxy server identified as "Proxy A" all support SIP/3, but the
proxy server identified as "Proxy B" does not, and only supports SIP/2.0 over UDP. When Proxy A attempts to connect to
Proxy B, it may have previous knowledge of the lack of support for SIP/3 on Proxy B, or the DNS SRV record {{?RFC2782}}
may have indicated that the server only supports `_sips` services over TCP, thereby implying SIP/2.0.

If Proxy B only supported unencrypted SIP over UDP, then Proxy A MUST NOT forward messages from the secure SIP/3 over
an unencrypted protocol, as this could constitute a downgrade attack. Instead, if the designated invitee cannot be
contacted by a means other than via Proxy B, then Proxy A would return a response of `502 Bad Gateway` to the initiator
for that transaction.

When initiating direct communication with an invitee after the conclusion of the initial `INVITE`, the descision to use
SIP/3 SHOULD be performed as follows:

* If the response from the invitee includes a DIALOG_TOKEN, or
* If the DNS SRV record for the SIPS URI indicates that the invitee supports SIPS over UDP, or
* If the `Contact:` header field carries the "v" parameter described in {{contact-extension}} and indicates a
preference for SIP/3.

## Transactions

In {{SIP2.0}}, messages pertaining to a given SIP transaction are identified as such using the branch parameter on the
`Via:` header fields carried in requests and responses. In SIP/3, individual transactions are tracked using the QUIC
streams that are used to carry them. SIP/3 endpoints MAY omit this parameter. For intermediaries converting between
SIP/3 and other versions of SIP, these endpoints SHOULD insert missing branch parameters, which MAY simply be a textual
representation of the stream IDs used.

## Dialogs

In {{SIP2.0}}, dialogs are tracked by use of the `Call-ID:` header field and the `tag=` parameter on the `To:` and
`From:` header fields. This document introduces a dialog token in {{methods}} which is used only to replace the `ACK`
request sent after the `INVITE` request. This document does not introduce any additional means for tracking dialogs,
and as such the `Call-ID:` and `tag=` values MUST be used in SIP/3.

# Stream Mapping and Usage {#stream-mapping}

A QUIC stream provides reliable and in-order delivery of bytes on that stream, but makes no guarantees about order of
delivery with regard to bytes on other streams. The semantics of the QUIC stream layer is invisible to the SIP framing
layer. The transport layer buffers and orders received stream data, exposing a reliable byte stream to the application.
Although QUIC permits out-of-order delivery within a stream, SIP/3 does not make use of this feature.

{::comment}
HTTP/3 uses unidirectional streams to carry the SETTINGS frame - should there be an analog of that here?
{:/comment}

QUIC streams can be either unidirectional, carrying data only from initiator to receiver, or bidirectional, carrying
data in both directions. SIP/3 makes no direct use of unidirectional streams, and bidirectional streams are used
exclusively for all SIP requests and responses. A bidirectional stream ensures that the response can be readily
correlated with the request. These streams are referred to as request streams.

{{SIP2.0}} is designed to run over unreliable transports such as UDP. As QUIC guarantees reliability, some of the
features of SIP/2.0 are no longer required. For example, usage of the `ACK` method is prohibited by SIP/3 (see
{{methods}}). In addition, user agents SHOULD NOT send the `CSeq` header field in requests and responses, as the
messages are already associated with a QUIC stream. Intermediaries that convert SIP/3 to SIP/2.0 and earlier versions
when forwarding message are responsibile for handing `ACK` requests and mapping of the `CSeq` header field to
individual transactions.

> **Author's note:** The author invites feedback as to whether the SHOULD NOT in relation to the `CSeq` header should
be increased to an outright prohibitation, or whether there is a valid use case that I have not identified that means
this restriction should be relaxed.

If the {{QPACK}} dynamic table is used, then the unidirectional encoder and decoder streams described in
{{Section 4.2 of QPACK}} will be in operation in a SIP/3 connection.

# SIP Methods {#methods}

The `REGISTER` and `BYE` methods as described in {{SIP2.0}} continue to operate in SIP/3 as they did in earlier
versions of the protocol.

The `INVITE` method works in mostly the same way, with some differences. In the case of the initial `INVITE`, the
invitee may indicate in the `contact:` header field an address for the initiator to communicate directly with the
invitee. In SIP/2.0, the first request sent directly to the new endpoint would be an `ACK` request, confirming the
direct communication. In SIP/3, the response by the invitee may contain a DIALOG_TOKEN frame.

If a `contact:` header is present in the response header block and a DIALOG_TOKEN frame is present in the response,
then a new SIP/3 connection SHOULD be established directly between the two user agents as described in the relevant
`contact:` header fields exchanged.

If a `contact:` header field is present in the header block but the response does not include a DIALOG_TOKEN frame,
then the invitee wishes for a SIP/2.0 session to be established directly between the two user agents. The `contact:`
header MUST indicate a secure `sips:` URI. A response that indicates a `contact:` header with an insecure `sip:` URI is
{{malformed}}.

> **Author's note:** What about upgrades from SIP/2.0 to SIP/3? Is it enough to assume that specificying a `sips:` URI
in the SIP/2.0 `Contact:` header field, and then getting SIP/3 from ALPN when negotiating would be a valid upgrade
path?

The `ACK` method is prohibited in SIP/3. In the case where this SIP/3 session is acknowledging an offer/answer
transaction on another SIP/3 connection, such as one that goes via one or more SIP/3 proxy servers, then the new
connection is associated with that previous offser/answer transaction using the REDEEM_TOKEN frame.

User agents should use the CANCEL frame instead of a `CANCEL` method request.

> **Author's note:** I have not done a comprehensive review of all SIP/2.0 extensions and their applicability to this
document, so I invite feedback on any other methods that may be problematic.

# SIP Framing Layer {#framing-layer}

SIP/3 frames are carried on QUIC streams, as described in {{stream-mapping}}. SIP/3 defines a single stream type: the
request stream. This section describes SIP/3 frame formats; see {{frame-types}} for an overview.

| Frame | Section |
|:------|:--------|
| DATA  | {{data-frame}} |
| HEADERS | {{headers-frame}} |
| CANCEL | {{cancel-frame}} |
| DIALOG_TOKEN | {{dialog-token-frame}} |
| REDEEM_TOKEN | {{redeem-token-frame}} |
{: #frame-types "SIP/3 Frames"}

*[DATA]: #data-frame
*[HEADERS]: #headers-frame
*[CANCEL]: #cancel-frame
*[DIALOG_TOKEN]: #dialog-token-frame
*[REDEEM_TOKEN]: #redeem-token-frame

Note that, unlike QUIC frames, SIP/3 frames can span multiple QUIC or UDP packets.

## Frame Layout {#frame-layout}

All frames have the following format:

~~~
SIP/3 Frame Format {
  Type (i),
  Length (i),
  Frame Payload (...)
}
~~~
{: #fig-sip-frame-format title="SIP/3 Frame Format"}

A frame includes the following fields:

* Type: A variable-length integer that identifies the frame type.

* Length: A variable-length integer that describes the length in bytes of the Frame Payload.

* Frame Payload: A payload, the semantics of which are determined by the Type field.

Each frame's payload MUST contain exactly the fields identified in its description. A frame payload that contains
additional bytes after the identified fields of a frame payload that terminates before the end of the idenfitifed
fields MUST be treated as a connection error of type SIP3_FRAME_ERROR.

When a stream terminates cleanly, if the last frame on the stream was truncated, this MUST be treated as a connection
error of type SIP3_FRAME_ERROR. Streams that terminate abruptly may be reset at any point in a frame.

## Frame Definitions {#frame-definitions}

### DATA {#data-frame}

`DATA` frames (type=`0x00`) convey arbitrary, variable-length sequences of bytes aassociated with the SIP request or
response content.

`DATA` frames MUST be associated with a SIP request or response.

~~~
DATA Frame {
  Type (i) = 0x00,
  Length (i),
  Data (..)
}
~~~
{: #fig-sip-data-frame-format title="DATA Frame"}

### HEADERS {#headers-frame}

`HEADERS` frames (type=`0x01`) are used to carry the collection of SIP header fields that are associated with a SIP
request or response as described in {{header-fields}} that is encoded using {{QPACK}}.

~~~
HEADERS Frame {
  Type (i) = 0x01,
  Length (i),
  Encoded Field Section (..)
}
~~~
{: #fig-sip-headers-frame-format title="HEADERS Frame"}

### CANCEL {#cancel-frame}

### DIALOG_TOKEN {#dialog-token-frame}

`DIALOG_TOKEN` frames (type=`0x33`) are used to carry an opaque token that are used to associate dialogs across QUIC
connections, for use when two endpoints currently communicating via one or more proxy servers wish to communicate
directly.

~~~
DIALOG_TOKEN Frame {
  Type (i) = 0x33,
  Length (i),
  Token ID Section (..)
}
~~~
{: #fig-sip-dialog-token-frame-format title="DIALOG_TOKEN Frame"}

The payload consists of:

Token ID Section: An opaque identifier that identifies the dialog for this transaction. The size and contents of this
token is implementation-dependent.

`DIALOG_TOKEN` frames MUST only be sent with a SIP response to a request with a method of type `INVITE`.

> **Author's Note:** Would it be a good idea to indicate somewhere which end should initiate the QUIC connection,
seeing as I've decoupled that above. For example, an endpoint that doesn't have access to STUN or TURN but is still
stuck behind a NAT gateway could indicate it wants to begin the connection to the remote peer. I'd need to to some
more thinking around perhaps making this more of an exchange?

### REDEEM_TOKEN {#redeem-token-frame}

`REDEEM_TOKEN` frames (type=`0x34`) are used to carry an opaque token previously given to the transport client in an
extant dialog with its peer via another delivery path, such as communicating via proxy servers. This frame type
replaces the `ACK` request described in {{Section 17.1.1.3 of SIP2.0}}.

~~~
REDEEN_TOKEN Frame {
  Type (i) = 0x34,
  Length (i),
  Token ID Length (i),
  Token ID Section (..),
  Encoded Field Section (..)
}
~~~
{: #fig-sip-redeem-token-frame-format title="REDEEM_TOKEN Frame"}

The payload consists of:

Token ID Length: The length of the token that was previously supplied in a DIALOG_TOKEN frame.

Token ID Section: The opaque identifier that identifies the dialog for a transaction in another QUIC transport session.

Encoded Field Section: {{QPACK}}-encoded request header fields, akin to the header fields delivered in the `ACK`
request as described in {{Section 17.1.1.3 of SIP2.0}}.

# Error Handling {#error-handling}

## SIP/3 Error Codes {#error-codes}

The following error codes are defined for use when abruptly terminating streams, aborting reading of streams, or
immediately closing SIP/3 connections.

SIP3_NO_ERROR (0x0300):
: No error. This is used when the connection or stream needs to be closed, but there is no error to signal.
  {: anchor="SIP3_NO_ERROR"}

SIP3_GENERAL_PROTOCOL_ERROR (0x0301):
: Peer violated protocol requirements in a way that does not match a more specific error code or endpoint declines to
use a more specific error code.
  {: anchor="SIP3_GENERAL_PROTOCOL_ERROR"}

SIP3_INTERNAL_ERROR (0x0302):
: An internal error has occured in the SIP stack.
  {: anchor="SIP3_INTERNAL_ERROR"}

SIP3_FRAME_ERROR (0x0305):
: A frame that fails to satisfy layout requirements or with an invalid size was received.
  {: anchor="SIP3_FRAME_ERROR"}

SIP3_FRAME_UNEXPECTED (0x0306):
: A frame was received that was not permitted in the current state or on the current stream.
  {: anchor="SIP3_FRAME_UNEXPECTED"}

SIP3_REQUEST_INCOMPLETE (0x030d):
: An endpoint's stream terminated without containing a fully formed request.
  {: anchor="SIP3_REQUEST_INCOMPLETE"}

SIP3_REQUEST_REJECTED (0x030b):
: A server rejected a request without performing any application processing.
  {: anchor="SIP3_REQUEST_REJECTED"}

SIP3_REQUEST_CANCELLED (0x030c):
: The request or its response is cancelled.
  {: anchor="SIP3_REQUEST_CANCELLED"}

SIP3_MESSAGE_ERROR (0x030e):
: A SIP message was {{malformed}} and cannot be processed.
  {: anchor="SIP3_MESSAGE_ERROR"}

*[SIP3_NO_ERROR]: #
*[SIP3_GENERAL_PROTOCOL_ERROR]: #
*[SIP3_INTERNAL_ERROR]: #
*[SIP3_FRAME_ERROR]: #
*[SIP3_FRAME_UNEXPECTED]: #
*[SIP3_REQUEST_INCOMPLETE]: #
*[SIP3_REQUEST_REJECTED]: #
*[SIP3_REQUEST_CANCELLED]: #
*[SIP3_MESSAGE_ERROR]: #

# Extensions to SIP/3 {#extensions}

# Future Carriage of Media Sessions {#media-sessions}

Future versions of this specification may include support for carrying media sessions within the same QUIC transport
connection as SIP/3, with the intention being that they will be negotiated using the SDP offer/answer mechanism.

There already exists several attempts to define carriage of media over QUIC transport, such as {{QRT}},
{{RTP-over-QUIC}}, {{QuicR-Arch}}, {{RUSH}} and {{Warp}}.

In the case of media carried in QUIC datagrams, a user agent cannot propose sending media using this mechanism unless
its peer has indicated its support for receiving datagrams by means of the `max_datagram_frame_size` parameter as
described in {{Section 3 of QUIC-DATAGRAMS}}.

In the case of media carried in QUIC streams, if the media streams are transmitted using unidirectional streams, then
new stream types will need to be defined. This document reserves the stream type value 0x04 for this, see
{{unidirectional-streams}}. In the unlikely case where media streams are to be transmitted using bidirectional streams,
the stream type mechanism will need to be extended to cover bidirectional streams, as SIP/3 currently assumes that SIP
messages have exclusive use of the bidirectional streams.

## Carriage of RTP in a QUIC Transport Session

Both {{QRT}} and {{RTP-over-QUIC}} define ways to carry RTP and RTCP messages over QUIC DATAGRAMs, and with SIP and SDP
already closely aligned with RTP media sessions it stands to reason that adapting SIP/3 to coexist within the same QUIC
transport connection would save at least a round trip.

{{QRT}} attempts to define SDP attributes to allow the negotiation of QRT sessions in SIP. {{SDP-QUIC}} also describes
a different set of SDP attributes to perform a similar task.

Future versions of this document or the above documents may specify a mechanism for signalling that a given media
session will be carried in the same QUIC connection that the SIP/3 session is going to be carried in.

## Carriage of non-RTP media streaming protocols in a QUIC Transport Session

{{RUSH}} does not specify a means to discover the presence of a RUSH streaming session, nor a mechanism for negotiating
the encoding parameters of media that is being exchanged. It is possible that this could be performed by SIP/3.

{{Warp}} specifies that sessions are established using HTTP/3 WebTransport ({{WebTransH3}}). However, to the author's
best knowledge WebTransport does not yet contain any signalling or media negotiation similar to how WebRTC would use
SDP offer/answer exchanges, so some form of session establishment mechanism like SIP/3 could be used.

{{QuicR-Arch}} is openly hostile to the usage of SDP, and {{QuicR-Proto}} defines the QuicR Manifest for advertising
media sessions and endpoint capabilities, and as such SIP/3 probably isn't required.

# Security Considerations

TODO Security

# IANA Considerations

TODO IANA actions

--- back

# Acknowledgments

TODO acknowledge.

# QPACK Static Table {#static-table}

| Index | Name                | Value           |
|:------|:--------------------|:----------------|
| 0     | :request-uri        |                 |
| 1     | from                |                 |
| 2     | to                  |                 |
| 3     | call-id             |                 |
| 4     | via                 |                 |
| 5     | :method             | REGISTER        |
| 6     | :method             | INVITE          |
| 7     | :method             | ACK             |
| 8     | :method             | BYE             |
| 9     | :method             | CANCEL          |
| 10    | :method             | UPDATE          |
| 11    | :method             | REFER           |
| 12    | :method             | OPTIONS         |
| 13    | :method             | MESSAGE         |
| 14    | :status             | 100             |
| 15    | :status             | 180             |
| 16    | :status             | 200             |
| 17    | :status             | 301             |
| 18    | :status             | 302             |
| 19    | :status             | 400             |
| 20    | :status             | 401             |
| 21    | :status             | 404             |
| 22    | :status             | 407             |
| 23    | :status             | 408             |
| 24    | contact             |                 |
| 25    | content-type        | application/sdp |
| 26    | content-type        | text/html       |
| 27    | content-disposition | session         |
| 28    | content-disposition | render          |
| 29    | content-length      |                 |
| 30    | accept              | application/sdp |
| 31    | accept-encoding     | gzip            |
| 32    | accept-language     |                 |
| 33    | alert-info          |                 |
| 34    | allow               | REGISTER        |
| 35    | allow               | INVITE          |
| 36    | allow               | ACK             |
| 37    | allow               | BYE             |
| 38    | allow               | CANCEL          |
| 39    | allow               | UPDATE          |
| 40    | allow               | REFER           |
| 41    | allow               | OPTIONS         |
| 42    | allow               | MESSAGE         |
| 43    | authentication-info |                 |
| 44    | authorization       |                 |
| 45    | call-info           |                 |
| 46    | content-encoding    |                 |
| 47    | content-language    |                 |
| 48    | cate                |                 |
| 49    | error-info          |                 |
| 50    | expires             |                 |
| 51    | in-reply-to         |                 |
| 52    | max-forwards        |                 |
| 53    | min-expires         |                 |
| 54    | mime-version        |                 |
| 55    | organization        |                 |
| 56    | priority            | Non-urgent      |
| 57    | priority            | Normal          |
| 58    | priority            | Urgent          |
| 59    | priority            | Emergency       |
| 60    | proxy-authenticate  |                 |
| 61    | proxy-authorization |                 |
| 62    | proxy-require       |                 |
| 63    | record-route        |                 |
| 64    | reply-to            |                 |
| 65    | require             |                 |
| 66    | retry-after         |                 |
| 67    | route               |                 |
| 68    | server              |                 |
| 69    | subject             |                 |
| 70    | supported           |                 |
| 71    | timestamp           |                 |
| 72    | unsupported         |                 |
| 73    | user-agent          |                 |
| 74    | warning             | 300             |
| 75    | warning             | 301             |
| 76    | warning             | 302             |
| 77    | warning             | 303             |
| 78    | warning             | 304             |
| 79    | warning             | 305             |
| 80    | warning             | 306             |
| 81    | warning             | 307             |
| 82    | warning             | 330             |
| 83    | warning             | 331             |
| 84    | warning             | 370             |
| 85    | warning             | 399             |
| 86    | www-authenticate    |                 |
{: #fig-static-table title="Static Table"}
