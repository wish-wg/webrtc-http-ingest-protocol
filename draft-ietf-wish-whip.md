---
docname: draft-ietf-wish-whip-15
title: WebRTC-HTTP ingestion protocol (WHIP)
abbrev: whip
category: std
ipr: trust200902
area: ART
workgroup: wish
updates: 8842, 8840

keyword: WebRTC

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: S. Murillo
    name: Sergio Garcia Murillo
    organization: Millicast
    email: sergio.garcia.murillo@cosmosoftware.io
 -
    ins: A. Gouaillard
    name: Alexandre Gouaillard
    organization: CoSMo Software
    email: alex.gouaillard@cosmosoftware.io

normative:
  FETCH:
    author:
      org: WHATWG
    title: Fetch - Living Standard
    target: https://fetch.spec.whatwg.org
      
--- abstract

This document describes a simple HTTP-based protocol that will allow WebRTC-based ingestion of content into streaming services and/or CDNs.

This document updates RFC 8842 and RFC 8840.

--- middle

# Introduction

The IETF RTCWEB working group standardized JSEP ({{!RFC9429}}), a mechanism used to control the setup, management, and teardown of a multimedia session. It also describes how to negotiate media flows using the Offer/Answer Model with the Session Description Protocol (SDP) {{!RFC3264}} including the formats for data sent over the wire (e.g., media types, codec parameters, and encryption). WebRTC intentionally does not specify a signaling transport protocol at application level.

Unfortunately, the lack of a standardized signaling mechanism in WebRTC has been an obstacle to adoption as an ingestion protocol within the broadcast/streaming industry, where a streamlined production pipeline is taken for granted: plug in cables carrying raw media to hardware encoders, then push the encoded media to any streaming service or Content Delivery Network (CDN) ingest using an ingestion protocol.

While WebRTC can be integrated with standard signaling protocols like SIP {{?RFC3261}} or XMPP {{?RFC6120}}, they are not designed to be used in broadcasting/streaming services, and there is also no sign of adoption in that industry. RTSP {{?RFC7826}}, which is based on RTP, does not support the SDP offer/answer model {{!RFC3264}} for negotiating the characteristics of the media session.

This document proposes a simple protocol based on HTTP for supporting WebRTC as media ingestion method which:

- Is easy to implement,
- Is as easy to use as popular IP-based broadcast protocols
- Is fully compliant with WebRTC and RTCWEB specs
- Enables ingestion on both classical media platforms and WebRTC end-to-end platforms, achieving the lowest possible latency.
- Lowers the requirements on both hardware encoders and broadcasting services to support WebRTC.
- Is usable both in web browsers and in standalone encoders.

# Terminology

{::boilerplate bcp14-tagged}

# Overview

The WebRTC-HTTP Ingest Protocol (WHIP) is designed to facilitate a one-time exchange of Session Description Protocol (SDP) offers and answers using HTTP POST requests. This exchange is a fundamental step in establishing an Interactive Connectivity Establishment (ICE) and Datagram Transport Layer Security (DTLS) session between the WHIP client, which represents the encoder or media producer, and the media server, the broadcasting ingestion endpoint.

Upon successful establishment of the ICE/DTLS session, unidirectional media data transmission commences from the WHIP client to the media server. It is important to note that SDP renegotiations are not supported in WHIP, meaning that no modifications to the "m=" sections can be made after the initial SDP offer/answer exchange via HTTP POST is completed and only ICE related information can be updated via HTTP PATCH requests as defined in {{ice-support}}.

The following diagram illustrates the core operation of the WHIP protocol for initiating and terminating an ingest session:

~~~~~
                                                                               
 +-------------+    +---------------+ +--------------+ +---------------+
 | WHIP client |    | WHIP endpoint | | Media Server | | WHIP session  |
 +--+----------+    +---------+-----+ +------+-------+ +--------|------+
    |                         |              |                  |       
    |                         |              |                  |       
    |HTTP POST (SDP Offer)    |              |                  |       
    +------------------------>+              |                  |       
    |201 Created (SDP answer) |              |                  |       
    +<------------------------+              |                  |       
    |          ICE REQUEST                   |                  |       
    +--------------------------------------->+                  |       
    |          ICE RESPONSE                  |                  |       
    |<---------------------------------------+                  |       
    |          DTLS SETUP                    |                  |       
    |<======================================>|                  |       
    |          RTP/RTCP FLOW                 |                  |       
    +<-------------------------------------->+                  |       
    | HTTP DELETE                                               |       
    +---------------------------------------------------------->+       
    | 200 OK                                                    |       
    <-----------------------------------------------------------x       
                                                                               
~~~~~
{: title="WHIP session setup and teardown" #whip-protocol-operation}

The elements in {{whip-protocol-operation}} are described as follows:

- WHIP client: This represents the WebRTC media encoder or producer, which functions as a client of the WHIP protocol by encoding and delivering media to a remote media server.
- WHIP endpoint: This denotes the ingest server that receives the initial WHIP request.
- WHIP endpoint URL: Refers to the URL of the WHIP endpoint responsible for creating the WHIP session.
- Media server: This is the WebRTC media server or consumer responsible for establishing the media session with the WHIP client and receiving the media content it produces.
- WHIP session: This indicates the server handling the allocated HTTP resource by the WHIP endpoint for an ongoing ingest session.
- WHIP session URL: Refers to the URL of the WHIP resource allocated by the WHIP endpoint for a specific media session. The WHIP client can send requests to the WHIP session using this URL to modify the session, such as ICE operations or termination. 

The {{whip-protocol-operation}} illustrates the communication flow between a WHIP client, WHIP endpoint, media server, and WHIP session. This flow outlines the process of setting up and tearing down an ingestion session using the WHIP protocol, involving negotiation, ICE for Network Address Translation (NAT) traversal, DTLS and Secure Real-time Transport Protocol (SRTP) for security, and RTP/RTCP for media transport:

- WHIP client: Initiates the communication by sending an HTTP POST with an SDP Offer to the WHIP endpoint.
- WHIP endpoint: Responds with a "201 Created" message containing an SDP answer.
- WHIP client and media server: Establish an ICE and DTLS sessions for NAT traversal and secure communication.
- RTP/RTCP Flow: Real-time Transport Protocol and Real-time Transport Control Protocol flows are established for media transmission from the WHIP client to the media server, secured by the SRTP profile.
- WHIP client: Sends an HTTP DELETE to terminate the WHIP session.
- WHIP session: Responds with a "200 OK" to confirm the session termination.


# Protocol Operation

## HTTP usage {#http-usage}

Following {{?BCP56}} guidelines, WHIP clients MUST NOT match error codes returned by the WHIP endpoints and resources to a specific error cause indicated in this specification. WHIP clients MUST be able to handle all applicable status codes gracefully falling back to the generic n00 semantics of a given status code on unknown error codes. WHIP endpoints and resources could convey finer-grained error information by a problem statement json object in the response message body of the failed request as per {{?RFC9457}}.

The WHIP endpoints and sessions are origin servers as defined in {{Section 3.6. of !RFC9110}} handling the requests and providing responses for the underlying HTTP resources. Those HTTP resources do not have any representation defined in this specification, so the WHIP endpoints and sessions MUST return a 2XX sucessfull response with no content when a GET request is received.

## Ingest session set up

In order to set up an ingestion session, the WHIP client MUST generate an SDP offer according to the JSEP rules for an initial offer as in {{Section 5.2.1 of !RFC9429}} and perform an HTTP POST request as per {{Section 9.3.3 of !RFC9110}} to the configured WHIP endpoint URL.

The HTTP POST request MUST have a content type of "application/sdp" and contain the SDP offer as the body. The WHIP endpoint MUST generate an SDP answer according to the JSEP rules for an initial answer as in {{Section 5.3.1 of !RFC9429}} and return a "201 Created" response with a content type of "application/sdp", the SDP answer as the body, and a Location header field pointing to the newly created WHIP session. If the HTTP POST to the WHIP endpoint has a content type different than "application/sdp" or the SDP is malformed, the WHIP endpoint MUST reject the HTTP POST request with an appropiate 4XX error response. 

As the WHIP protocol only supports the ingestion use case with unidirectional media, the WHIP client SHOULD use "sendonly" attribute in the SDP offer but MAY use the "sendrecv" attribute instead, "inactive" and "recvonly" attributes MUST NOT be used. The WHIP endpoint MUST use "recvonly" attribute in the SDP answer. 

Following {{sdp-exchange-example}} is an example of an HTTP POST sent from a WHIP client to a WHIP endpoint and the "201 Created" response from the WHIP endpoint containing the Location header pointing to the newly created WHIP session:

~~~~~
POST /whip/endpoint HTTP/1.1
Host: whip.example.com
Content-Type: application/sdp
Content-Length: 1101

v=0
o=- 5228595038118931041 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0 1
a=extmap-allow-mixed
a=ice-options:trickle ice2
m=audio 9 UDP/TLS/RTP/SAVPF 111
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:EsAw
a=ice-pwd:bP+XJMM09aR8AiX1jdukzR6Y
a=fingerprint:sha-256 DA:7B:57:DC:28:CE:04:4F:31:79:85:C4:31:67:EB:27:58:29:ED:77:2A:0D:24:AE:ED:AD:30:BC:BD:F1:9C:02
a=setup:actpass
a=mid:0
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=sendonly
a=msid:d46fb922-d52a-4e9c-aa87-444eadc1521b ce326ecf-a081-453a-8f9f-0605d5ef4128
a=rtcp-mux
a=rtcp-mux-only
a=rtpmap:111 opus/48000/2
a=fmtp:111 minptime=10;useinbandfec=1
m=video 0 UDP/TLS/RTP/SAVPF 96 97
a=mid:1
a=bundle-only
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:10 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=extmap:11 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
a=sendonly
a=msid:d46fb922-d52a-4e9c-aa87-444eadc1521b 3956b460-40f4-4d05-acef-03abcdd8c6fd
a=rtpmap:96 VP8/90000
a=rtcp-fb:96 ccm fir
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtpmap:97 rtx/90000
a=fmtp:97 apt=96

HTTP/1.1 201 Created
ETag: "xyzzy"
Content-Type: application/sdp
Content-Length: 1053
Location: https://whip.example.com/session/id

v=0
o=- 1657793490019 1 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0 1
a=extmap-allow-mixed
a=ice-lite
a=ice-options:trickle ice2
m=audio 9 UDP/TLS/RTP/SAVPF 111
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:38sdf4fdsf54
a=ice-pwd:2e13dde17c1cb009202f627fab90cbec358d766d049c9697
a=fingerprint:sha-256 F7:EB:F3:3E:AC:D2:EA:A7:C1:EC:79:D9:B3:8A:35:DA:70:86:4F:46:D9:2D:CC:D0:BC:81:9F:67:EF:34:2E:BD
a=candidate:1 1 UDP 2130706431 198.51.100.1 39132 typ host
a=setup:passive
a=mid:0
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=recvonly
a=rtcp-mux
a=rtcp-mux-only
a=rtpmap:111 opus/48000/2
a=fmtp:111 minptime=10;useinbandfec=1
m=video 0 UDP/TLS/RTP/SAVPF 96 97
c=IN IP4 0.0.0.0
a=mid:1
a=bundle-only
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:10 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=extmap:11 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
a=recvonly
a=rtpmap:96 VP8/90000
a=rtcp-fb:96 ccm fir
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtpmap:97 rtx/90000
a=fmtp:97 apt=96
~~~~~
{: title="Example of SDP offer/answer exchange done via an HTTP POST" #sdp-exchange-example}

Once a session is set up, consent freshness as per {{!RFC7675}} SHALL be used to detect non-graceful disconnection by full ICE implementations and DTLS teardown for session termination by either side.

To explicitly terminate a WHIP session, the WHIP client MUST perform an HTTP DELETE request to the WHIP session URL returned in the Location header field of the initial HTTP POST. Upon receiving the HTTP DELETE request, the WHIP session will be removed and the resources freed on the media server, terminating the ICE and DTLS sessions.

A media server terminating a session MUST follow the procedures in {{Section 5.2 of !RFC7675}}  for immediate revocation of consent.

The WHIP endpoints MUST support OPTIONS requests for Cross-Origin Resource Sharing (CORS) as defined in {{FETCH}}. The "200 OK" response to any OPTIONS request SHOULD include an "Accept-Post" header with a media type value of "application/sdp" as per {{!W3C.REC-ldp-20150226}}.

## ICE support {#ice-support}

ICE {{!RFC8845}} is a protocol addressing the complexities of NAT traversal, commonly encountered in Internet communication. NATs hinder direct communication between devices on different local networks, posing challenges for real-time applications. ICE facilitates seamless connectivity by employing techniques to discover and negotiate efficient communication paths. 

Trickle ICE {{!RFC8838}} optimizes the connectivity process by incrementally sharing potential communication paths, reducing latency, and facilitating quicker establishment. 

ICE Restarts are crucial for maintaining connectivity in dynamic network conditions or disruptions, allowing devices to re-establish communication paths without complete renegotiation. This ensures minimal latency and reliable real-time communication.

Trickle ICE and ICE restart support are RECOMMENDED for both WHIP sessions and clients.

### HTTP PATCH request usage {#http-patch-usage}

The WHIP client MAY perform trickle ICE or ICE restarts by sending an HTTP PATCH request as per {{!RFC5789}} to the WHIP session URL, with a body containing an SDP fragment with media type "application/trickle-ice-sdpfrag" as specified in {{!RFC8840}} carrying the relevant ICE information. If the HTTP PATCH to the WHIP session has a content type different than "application/trickle-ice-sdpfrag" or the SDP fragment is malformed, the WHIP session MUST reject the HTTP PATCH with an appropiate 4XX error response.


If the WHIP session supports either Trickle ICE or ICE restarts, but not both, it MUST return a "422 Unprocessable Content" error response for the HTTP PATCH requests that are not supported as per {{Section 15.5.21 of !RFC9110}}. 

The WHIP client MAY send overlapping HTTP PATCH requests to one WHIP session. Consequently, as those HTTP PATCH requests may be received out-of-order by the WHIP session, if WHIP session supports ICE restarts, it MUST generate a unique strong entity-tag identifying the ICE session as per {{Section 8.8.3 of !RFC9110}}, being OPTIONAL otherwise. The initial value of the entity-tag identifying the initial ICE session MUST be returned in an ETag header field in the "201 Created" response to the initial POST request to the WHIP endpoint.

WHIP clients SHOULD NOT use entity-tag validation when matching a specific ICE session is not required, such as for example when initiating a DELETE request to terminate a session. WHIP sessions MUST ignore any entity-tag value sent by the WHIP client when ICE session matching is not required, as in the HTTP DELETE request.

Missing or outdated ETags in the PATCH requests from WHIP clients will be answered by WHIP sessions as per {{Section 13.1.1 of !RFC9110}} and {{Section 3 of !RFC6585}}, with a "428 Precondition Required" response for a missing entity tag, and a "412 Precondition Failed" response for a non-matching entity tag.

### Trickle ICE {#trickle-ice}

Depending on the Trickle ICE support on the WHIP client, the initial offer by the WHIP client MAY be sent after the full ICE gathering is complete with the full list of ICE candidates, or it MAY only contain local candidates (or even an empty list of candidates) as per {{!RFC8845}}. For the purpose of reducing setup times, when using Trickle ICE the WHIP client SHOULD send the SDP offer as soon as possible, containing either locally gathered ICE candidates or an empty list of candidates.

In order to simplify the protocol, the WHIP session cannot signal additional ICE candidates to the WHIP client after the SDP answer has been sent. The WHIP endpoint SHALL gather all the ICE candidates for the media server before responding to the client request and the SDP answer SHALL contain the full list of ICE candidates of the media server.

As the WHIP client needs to know the WHIP session URL associated with the ICE session in order to send a PATCH request containing new ICE candidates, it MUST wait and buffer any gathered candidates until the "201 Created" HTTP response to the initial POST request is received.
In order to lower the HTTP traffic and processing time required the WHIP client SHOULD send a single aggregated HTTP PATCH request with all the buffered ICE candidates once the response is received.
Additionally, if ICE restarts are supported by the WHIP session, the WHIP client needs to know the entity-tag associated with the ICE session in order to send a PATCH request containing new ICE candidates, so it MUST also wait and buffer any gathered candidates until it receives the HTTP response with the new entity-tag value to the last PATCH request performing an ICE restart.

WHIP clients generating the HTTP PATCH body with the SDP fragment and its subsequent processing by WHIP sessions MUST follow to the guidelines defined in {{Section 4.4 of !RFC8840}} with the following considerations:

 - As per {{!RFC9429}}, only m-sections not marked as bundle-only can gather ICE candidates, so given that the "max-bundle" policy is being used, the SDP fragment will contain only the offerer-tagged m-line of the bundle group.
 - The WHIP client MAY exclude ICE candidates from the HTTP PATCH body if they have already been confirmed by the WHIP session with a successful HTTP response to a previous HTTP PATCH request.

WHIP sessions and clients that support Trickle ICE MUST make use of entity-tags and conditional requests as explained in {{http-patch-usage}}.

When a WHIP session receives a PATCH request that adds new ICE candidates without performing an ICE restart, it MUST return a "204 No Content" response without a body and MUST NOT include an ETag header in the response. If the WHIP session does not support a candidate transport or is not able to resolve the connection address, it MUST silently discard the candidate and continue processing the rest of the request normally.

~~~~~
PATCH /session/id HTTP/1.1
Host: whip.example.com
If-Match: "xyzzy"
Content-Type: application/trickle-ice-sdpfrag
Content-Length: 576

a=group:BUNDLE 0 1
m=audio 9 UDP/TLS/RTP/SAVPF 111
a=mid:0
a=ice-ufrag:EsAw
a=ice-pwd:P2uYro0UCOQ4zxjKXaWCBui1
a=candidate:1387637174 1 udp 2122260223 192.0.2.1 61764 typ host generation 0 ufrag EsAw network-id 1
a=candidate:3471623853 1 udp 2122194687 198.51.100.2 61765 typ host generation 0 ufrag EsAw network-id 2
a=candidate:473322822 1 tcp 1518280447 192.0.2.1 9 typ host tcptype active generation 0 ufrag EsAw network-id 1
a=candidate:2154773085 1 tcp 1518214911 198.51.100.2 9 typ host tcptype active generation 0 ufrag EsAw network-id 2
a=end-of-candidates

HTTP/1.1 204 No Content
~~~~~
{: title="Example of a Trickle ICE request and response" #trickle-ice-example}

{{trickle-ice-example}} shows an example of the Trickle ICE procedure where the WHIP client sends a PATCH request with updated ICE candidate information and receives a successful response from the WHIP session.

### ICE Restarts {#ice-restarts}

As defined in {{!RFC8839}}, when an ICE restart occurs, a new SDP offer/answer exchange is triggered. However, as WHIP does not support renegotiation of non-ICE related SDP information, a WHIP client will not send a new offer when an ICE restart occurs. Instead, the WHIP client and WHIP session will only exchange the relevant ICE information via an HTTP PATCH request as defined in {{http-patch-usage}} and MUST assume that the previously negotiated non-ICE related SDP information still apply after the ICE restart.

When performing an ICE restart, the WHIP client MUST include the updated "ice-pwd" and "ice-ufrag" in the SDP fragment of the HTTP PATCH request body as well as the new set of gathered ICE candidates as defined in {{!RFC8840}}.
Similar what is defined in {{trickle-ice}}, as per {{!RFC9429}} only m-sections not marked as bundle-only can gather ICE candidates, so given that the "max-bundle" policy is being used, the SDP fragment will contain only the offerer-tagged m-line of the bundle group.
A WHIP client sending a PATCH request for performing ICE restart MUST contain an "If-Match" header field with a field-value "*" as per {{Section 13.1.1 of !RFC9110}}. 

{{!RFC8840}} states that an agent MUST discard any received requests containing "ice-pwd" and "ice-ufrag" attributes that do not match those of the current ICE Negotiation Session, however, any WHIP session receiving an updated "ice-pwd" and "ice-ufrag" attributes MUST consider the request as performing an ICE restart instead and, if supported, SHALL return a "200 OK" with an "application/trickle-ice-sdpfrag" body containing the new ICE username fragment and password and a new set of ICE candidates for the WHIP session. Also, the "200 OK" response for a successful ICE restart MUST contain the new entity-tag corresponding to the new ICE session in an ETag response header field and MAY contain a new set of ICE candidates for the media server. 

As defined in {{Section 4.4.1.1.1 of !RFC8839}} the set of candidates after an ICE restart may include some, none, or all of the previous candidates for that data stream and may include a totally new set of candidates. So after performing a successful ICE restart, both the WHIP client and the WHIP session MUST replace the previous set of remote candidates with the new set exchanged in the HTTP PATCH request and response, discarding any remote ICE candidate not present on the new set. Both the WHIP client and the WHIP session MUST ensure that the HTTP PATCH requests and response bodies include the same 'ice-options,' 'ice-pacing,' and 'ice-lite' attributes as those used in the SDP offer or answer.

If the ICE restart request cannot be satisfied by the WHIP session, the resource MUST return an appropriate HTTP error code and MUST NOT terminate the session immediately and keep the existing ICE session. The WHIP client MAY retry performing a new ICE restart or terminate the session by issuing an HTTP DELETE request instead. In any case, the session MUST be terminated if the ICE consent expires as a consequence of the failed ICE restart as per {{Section 5.1 of !RFC7675}}.

In case of unstable network conditions, the ICE restart HTTP PATCH requests and responses might be received out of order. In order to mitigate this scenario, when the client performs an ICE restart, it MUST discard any previous ICE username and passwords fragments and ignore any further HTTP PATCH response received from a pending HTTP PATCH request. WHIP clients MUST apply only the ICE information received in the response to the last sent request. If there is a mismatch between the ICE information at the WHIP client and at the WHIP session (because of an out-of-order request), the STUN requests will contain invalid ICE information and will be dropped by the receiving side. If this situation is detected by the WHIP client, it MUST send a new ICE restart request to the server.

~~~~~
PATCH /session/id HTTP/1.1
Host: whip.example.com
If-Match: "*"
Content-Type: application/trickle-ice-sdpfrag
Content-Length: 82

a=ice-options:trickle ice2
a=group:BUNDLE 0 1
m=audio 9 UDP/TLS/RTP/SAVPF 111
a=mid:0
a=ice-ufrag:ysXw
a=ice-pwd:vw5LmwG4y/e6dPP/zAP9Gp5k
a=candidate:1387637174 1 udp 2122260223 192.0.2.1 61764 typ host generation 0 ufrag EsAw network-id 1
a=candidate:3471623853 1 udp 2122194687 198.51.100.2 61765 typ host generation 0 ufrag EsAw network-id 2
a=candidate:473322822 1 tcp 1518280447 192.0.2.1 9 typ host tcptype active generation 0 ufrag EsAw network-id 1
a=candidate:2154773085 1 tcp 1518214911 198.51.100.2 9 typ host tcptype active generation 0 ufrag EsAw network-id 2

HTTP/1.1 200 OK
ETag: "abccd"
Content-Type: application/trickle-ice-sdpfrag
Content-Length: 252

a=ice-lite
a=ice-options:trickle ice2
a=group:BUNDLE 0 1
m=audio 9 UDP/TLS/RTP/SAVPF 111
a=mid:0
a=ice-ufrag:289b31b754eaa438
a=ice-pwd:0b66f472495ef0ccac7bda653ab6be49ea13114472a5d10a
a=candidate:1 1 udp 2130706431 198.51.100.1 39132 typ host
a=end-of-candidates
~~~~~
{: title="Example of an ICE restart request and response" #trickle-restart-example}

{{trickle-ice-example}} demonstrates a Trickle ICE restart procedure example. The WHIP client sends a PATCH request containing updated ICE information, including a new ufrag and password, along with newly gathered ICE candidates. In response, the WHIP session provides ICE information for the session after the ICE restart, including the updated ufrag and password, as well as the previous ICE candidate.

## WebRTC constraints

To simplify the implementation of WHIP in both clients and media servers, WHIP introduces specific restrictions on WebRTC usage. The following subsections will explain these restrictions in detail:

### SDP Bundle

Both the WHIP client and the WHIP endpoint SHALL support {{!RFC9143}} and use "max-bundle" policy as defined in {{!RFC9429}}. The WHIP client and the media server MUST support multiplexed media associated with the BUNDLE group as per {{Section 9 of !RFC9143}}. In addition, per {{!RFC9143}} the WHIP client and media server SHALL use RTP/RTCP multiplexing for all bundled media. In order to reduce the network resources required at the media server, both The WHIP client and WHIP endpoints MUST include the "rtcp-mux-only" attribute in each bundled "m=" sections as per {{Section 3 of !RFC8858}}.

### Single MediaStream

WHIP only supports a single MediaStream as defined in {{!RFC8830}} and therefore all "m=" sections MUST contain a "msid" attribute with the same value. The MediaStream MUST contain at least one MediaStreamTrack of any media kind and it MUST NOT have two or more than MediaStreamTracks for the same media (audio or video). However, it would be possible for future revisions of this spec to allow more than a single MediaStream or MediaStreamTrack of each media kind, so in order to ensure forward compatibility, if the number of audio and or video MediaStreamTracks or number of MediaStreams are not supported by the WHIP endpoint, it MUST reject the HTTP POST request with an "422 Unprocessable Content" or "400 Bad Request" error response. The WHIP endpoint MAY also return a problem statement as recommended in {{http-usage}} proving further error details about the failed request.

### No partially successful answers

The WHIP endpoint SHOULD NOT reject individual "m=" sections as per {{Section 5.3.1 of !RFC9429}} in case there is any error processing the "m=" section, but reject the HTTP POST request with an "422 Unprocessable Content" or "400 Bad Request" error response to prevent having partially successful ingest sessions which can be misleading to end users. The WHIP endpoint MAY also return a problem statement as recommended in {{http-usage}} proving further error details about the failed request.

### DTLS setup role and SDP "setup" attribute

When a WHIP client sends an SDP offer, it SHOULD insert an SDP "setup" attribute with an "actpass" attribute value, as defined in {{!RFC8842}}. However, if the WHIP client only implements the DTLS client role, it MAY use an SDP "setup" attribute with an "active" attribute value. If the WHIP endpoint does not support an SDP offer with an SDP "setup" attribute with an "active" attribute value, it SHOULD reject the request with an "422 Unprocessable Content" or "400 Bad Request" error response.

NOTE: {{!RFC8842}} defines that the offerer must insert an SDP "setup" attribute with an "actpass" attribute value. However, the WHIP client will always communicate with a media server that is expected to support the DTLS server role, in which case the client might choose to only implement support for the DTLS client role.

### Trickle ICE and ICE restarts

The media server SHOULD support full ICE, unless it is connected to the Internet with an IP address that is accessible by each WHIP client that is authorized to use it, in which case it MAY support only ICE lite. The WHIP client MUST implement and use full ICE.

Trickle ICE and ICE restarts support is OPTIONAL for both the WHIP clients and media servers as explained in {{ice-support}}.

## Load balancing and redirections

WHIP endpoints and media servers might not be colocated on the same server, so it is possible to load balance incoming requests to different media servers. 

WHIP clients SHALL support HTTP redirections as per {{Section 15.4 of !RFC9110}}. In order to avoid POST requests to be redirected as GET requests, status codes 301 and 302 MUST NOT be used and the preferred method for performing load balancing is via the "307 Temporary Redirect" response status code as described in {{Section 15.4.8 of !RFC9110}}. Redirections are not required to be supported for the PATCH and DELETE requests.

In case of high load, the WHIP endpoints MAY return a "503 Service Unavailable" response indicating that the server is currently unable to handle the request due to a temporary overload or scheduled maintenance as described in {{Section 15.6.4 of !RFC9110}}, which will likely be alleviated after some delay. The WHIP endpoint might send a Retry-After header field indicating the minimum time that the user agent ought to wait before making a follow-up request as described in {{Section 10.2.3 of !RFC9110}}.

## STUN/TURN server configuration

The WHIP endpoint MAY return STUN/TURN server configuration URLs and credentials usable by the client in the "201 Created" response to the HTTP POST request to the WHIP endpoint URL.

A reference to each STUN/TURN server will be returned using the "Link" header field {{!RFC8288}} with a "rel" attribute value of "ice-server". The Link target URI is the server URI as defined in {{!RFC7064}} and {{!RFC7065}}. The credentials are encoded in the Link target attributes as follows:

- username: If the Link header field represents a TURN server, and credential-type is "password", then this attribute specifies the username to use with that TURN server.
- credential: If the "credential-type" attribute is missing or has a "password" value, the credential attribute represents a long-term authentication password, as described in {{Section 9.2 of !RFC8489}}.
- credential-type: If the Link header field represents a TURN server, then this attribute specifies how the credential attribute value should be used when that TURN server requests authorization. The default value if the attribute is not present is "password".

~~~~~
     Link: <stun:stun.example.net>; rel="ice-server"
     Link: <turn:turn.example.net?transport=udp>; rel="ice-server";
           username="user"; credential="myPassword"; credential-type="password"
     Link: <turn:turn.example.net?transport=tcp>; rel="ice-server";
           username="user"; credential="myPassword"; credential-type="password"
     Link: <turns:turn.example.net?transport=tcp>; rel="ice-server";
           username="user"; credential="myPassword"; credential-type="password"
~~~~~
{: title="Example of a STUN/TURN servers configuration"  #stun-server-example}

{{stun-server-example}} illustrates the Link headers included in a 201 Created response, providing the ICE server URLs and associated credentials.

NOTE: The naming of both the "rel" attribute value of "ice-server" and the target attributes follows the one used on the W3C WebRTC recommendation {{?W3C.REC-webrtc-20210126}} RTCConfiguration dictionary in section 4.2.1. "rel" attribute value of "ice-server" is not prepended with the "urn:ietf:params:whip:" so it can be reused by other specifications which may use this mechanism to configure the usage of STUN/TURN servers.

NOTE: Depending on the ICE Agent implementation, the WHIP client may need to call the setConfiguration method before calling the setLocalDescription method with the local SDP offer in order to avoid having to perform an ICE restart for applying the updated STUN/TURN server configuration on the next ICE gathering phase.

There are some WebRTC implementations that do not support updating the STUN/TURN server configuration after the local offer has been created as specified in {{Section 4.1.18 of !RFC9429}}. In order to support these clients, the WHIP endpoint MAY also include the STUN/TURN server configuration on the responses to OPTIONS request sent to the WHIP endpoint URL before the POST request is sent. However, this method is NOT RECOMMENDED to be used by the WHIP clients and, if supported by the underlying WHIP client's webrtc implementation, the WHIP client SHOULD wait for the information to be returned by the WHIP endpoint on the response of the HTTP POST request instead.

The generation of the TURN server credentials may require performing a request to an external provider, which can both add latency to the OPTIONS request processing and increase the processing required to handle that request. In order to prevent this, the WHIP endpoint SHOULD NOT return the STUN/TURN server configuration if the OPTIONS request is a preflight request for CORS as defined in {{FETCH}}, that is, if The OPTIONS request does not contain an Access-Control-Request-Method with "POST" value and the Access-Control-Request-Headers HTTP header does not contain the "Link" value. 

The WHIP clients MAY also support configuring the STUN/TURN server URIs with long term credentials provided by either the broadcasting service or an external TURN provider, overriding the values provided by the WHIP endpoint.

### Congestion control

{{?RFC8836}} defines the congestion control requirements for interactive Real-Time media to be used in WebRTC. These requirements are based on the assumption of the need to provide the data continuously, within a very limited time window (no more delay than hundreds of milliseconds end-to-end). If the latency target is higher, some of the requirements present in RFC8836 could be relaxed to allow more flexible implementations.

## Authentication and authorization

All WHIP endpoints, sessions and clients MUST support HTTP Authentication as per {{Section 11 of !RFC9110}} and in order to ensure interoperability, bearer token authentication as defined in the next section MUST be supported by all WHIP entities. However, this does not preclude the support of additional HTTP authentication schemes as defined in {{Section 11.6 of !RFC9110}}.

### Bearer token authentication

WHIP endpoints and sessions MAY require the HTTP request to be authenticated using an HTTP Authorization header field with a Bearer token as specified in {{Section 2.1 of !RFC6750}}. WHIP clients MUST implement this authentication and authorization mechanism and send the HTTP Authorization header field in all HTTP requests sent to either the WHIP endpoint or session except the preflight OPTIONS requests for CORS.

The nature, syntax, and semantics of the bearer token, as well as how to distribute it to the client, is outside the scope of this document. Some examples of the kind of tokens that could be used are, but are not limited to, JWT tokens as per {{!RFC6750}} and {{!RFC8725}} or a shared secret stored on a database. The tokens are typically made available to the end user alongside the WHIP endpoint URL and configured on the WHIP clients (similar to the way RTMP URLs and Stream Keys are distributed).

WHIP endpoints and sessions could perform the authentication and authorization by encoding an authentication token within the URLs for the WHIP endpoints or sessions instead. In case the WHIP client is not configured to use a bearer token, the HTTP Authorization header field MUST NOT be sent in any request.

## Simulcast and scalable video coding

Simulcast as per {{!RFC8853}} MAY be supported by both the media servers and WHIP clients through negotiation in the SDP offer/answer.

If the client supports simulcast and wants to enable it for ingesting, it MUST negotiate the support in the SDP offer according to the procedures in {{Section 5.3 of !RFC8853}}. A server accepting a simulcast offer MUST create an answer according to the procedures in {{Section 5.3.2 of !RFC8853}}.

It is possible for both media servers and WHIP clients to support Scalable Video Coding (SVC). However, as there is no universal negotiation mechanism in SDP for SVC, the encoder must consider the negotiated codec(s), intended usage, and SVC support in available decoders when configuring SVC.

## Protocol extensions {#protocol-extensions}

In order to support future extensions to be defined for the WHIP protocol, a common procedure for registering and announcing the new extensions is defined.

Protocol extensions supported by the WHIP sessions MUST be advertised to the WHIP client in the "201 Created" response to the initial HTTP POST request sent to the WHIP endpoint.
The WHIP endpoint MUST return one "Link" header field for each extension that it supports, with the extension "rel" attribute value containing the extension URN and the URL for the HTTP resource that will be available for receiving requests related to that extension.

Protocol extensions are optional for both WHIP clients and servers. WHIP clients MUST ignore any Link attribute with an unknown "rel" attribute value and WHIP sessions MUST NOT require the usage of any extension.

Each protocol extension MUST register a unique "rel" attribute value at IANA starting with the prefix: "urn:ietf:params:whip:ext" as defined in {{urn-whip-subspace}}.

For example, considering a potential extension of server-to-client communication using server-sent events as specified in https://html.spec.whatwg.org/multipage/server-sent-events.html#server-sent-events, the URL for connecting to the server-sent event resource for the ingested stream could be returned in the initial HTTP "201 Created" response with a "Link" header field and a "rel" attribute of "urn:ietf:params:whip:ext:example:server-sent-events" (this document does not specify such an extension, and uses it only as an example).

In this theoretical case, the "201 Created" response to the HTTP POST request would look like:

~~~~~
HTTP/1.1 201 Created
Content-Type: application/sdp
Location: https://whip.example.com/session/id
Link: <https://whip.example.com/session/id/sse>;
      rel="urn:ietf:params:whip:ext:example:server-sent-events"
~~~~~
{: title="Example of a WHIP protocol extension" #protocol-extension-example}

{{protocol-extension-example}} shows an example of a WHIP protocol extension supported by the WHIP session, as indicated in the Link header of the 201 Created response.

# Security Considerations

This document specifies a new protocol on top of HTTP and WebRTC, thus, security protocols and considerations from related specifications apply to the WHIP specification. These include:

- WebRTC security considerations: {{!RFC8826}}. HTTPS SHALL be used in order to preserve the WebRTC security model.
- Transport Layer Security (TLS): {{!RFC8446}} and {{!RFC9147}}.
- HTTP security: {{Section 11 of !RFC9112}} and {{Section 17 of !RFC9110}}.
- URI security: {{Section 7 of !RFC3986}}.

On top of that, the WHIP protocol exposes a thin new attack surface specific of the REST API methods used within it:

- HTTP POST flooding and resource exhaustion:
  It would be possible for an attacker in possession of authentication credentials valid for ingesting a WHIP stream to make multiple HTTP POST to the WHIP endpoint.
  This will force the WHIP endpoint to process the incoming SDP and allocate resources for being able to set up the DTLS/ICE connection.
  While the malicious client does not need to initiate the DTLS/ICE connection at all, the WHIP session will have to wait for the DTLS/ICE connection timeout in order to release the associated resources.
  If the connection rate is high enough, this could lead to resource exhaustion on the servers handling the requests and it will not be able to process legitimate incoming ingests.
  In order to prevent this scenario, WHIP endpoints SHOULD implement a rate limit and avalanche control mechanism for incoming initial HTTP POST requests.

- Insecure direct object references (IDOR) on the WHIP session locations:
  If the URLs returned by the WHIP endpoint for the WHIP sessions location are easy to guess, it would be possible for an attacker to send multiple HTTP DELETE requests and terminate all the WHIP sessions currently running.
  In order to prevent this scenario, WHIP endpoints SHOULD generate URLs with enough randomness, using a cryptographically secure pseudorandom number generator following the best practices in Randomness Requirements for Security {{!RFC4086}}, and implement a rate limit and avalanche control mechanism for HTTP DELETE requests.
  The security considerations for Universally Unique IDentifier (UUID) {{!RFC9562, Section 8}} are applicable for generating the WHIP sessions location URL.

- HTTP PATCH flooding: 
Similar to the HTTP POST flooding, a malicious client could also create a resource exhaustion by sending multiple HTTP PATCH request to the WHIP session, although the WHIP sessions can limit the impact by not allocating new ICE candidates and reusing the existing ICE candidates when doing ICE restarts.
In order to prevent this scenario, WHIP endpoints SHOULD implement a rate limit and avalanche control mechanism for incoming HTTP PATCH requests.

# IANA Considerations

This specification adds a new link relation type and a registry for URN sub-namespaces for WHIP protocol extensions.

## Link Relation Type: ice-server

The link relation type below has been registered by IANA per {{Section 4.2 of !RFC8288}}.

Relation Name: ice-server

Description: Conveys the STUN and TURN servers that can be used by an ICE Agent to establish a connection with a peer.

Reference: TBD

## WebRTC-HTTP Ingestion Protocol (WHIP) registry group

IANA  is asked to create a new registry group called "WebRTC-HTTP Ingestion Protocol (WHIP)". This group includes the "WebRTC-HTTP ingestion protocol (WHIP) URNs" and "WebRTC-HTTP ingestion protocol (WHIP) extension URNs" registries described below.

## Registration of WHIP URN Sub-namespace and WHIP registries

IANA is asked to add an entry to the "IETF URN Sub-namespace for Registered Protocol Parameter Identifiers" registry and create a sub-namespace for the Registered Parameter Identifier as per {{!RFC3553}}: "urn:ietf:params:whip".

To manage this sub-namespace, IANA is asked to create the "WebRTC-HTTP ingestion protocol (WHIP) URNs" and "WebRTC-HTTP ingestion protocol (WHIP) extension URNs".

### WebRTC-HTTP ingestion protocol (WHIP) URNs registry {#urn-whip-registry}

The "WebRTC-HTTP ingestion protocol (WHIP) URNs" registry is used to manage entries within the "urn:ietf:params:whip" namespace. The registry descriptions is as follows:

   - Registry group: WebRTC-HTTP ingestion protocol (WHIP)

   - Registry name: WebRTC-HTTP ingestion protocol (WHIP) URNs
     
   - Specification: this document (RFC TBD)
     
   - Registration procedure: Specification Required

   - Field names: URI, description, change controller, reference and IANA registry reference

The registry contains a single initial value:

   - URI: urn:ietf:params:whip:ext
     
   - Description: WebRTC-HTTP ingestion protocol (WHIP) extension URNs

   - Change Controller: IETF
     
   - Reference: this document (RFC TBD) Section {{urn-whip-ext-registry}}

   - IANA registry reference: WebRTC-HTTP ingestion protocol (WHIP) extension URNs registry.

### WebRTC-HTTP ingestion protocol (WHIP) extension URNs registry {#urn-whip-ext-registry}

The "WebRTC-HTTP ingestion protocol (WHIP) Extension URNs" is used to manage entries within the "urn:ietf:params:whip:ext" namespace. The registry descriptions is as follows:

   - Registry group: WebRTC-HTTP ingestion protocol (WHIP)
 
   - Registry name: WebRTC-HTTP ingestion protocol (WHIP) Extension URNs
     
   - Specification: this document (RFC TBD)
     
   - Registration procedure: Specification Required

   - Field names: URI, description, change controller, reference and IANA registry reference
     

## URN Sub-namespace for WHIP {#urn-whip-subspace}

WHIP endpoint utilizes URNs to identify the supported WHIP protocol extensions on the "rel" attribute of the Link header as defined in {{protocol-extensions}}.

This section creates and registers an IETF URN Sub-namespace for use in the WHIP specifications and future extensions.

### Specification Template

Namespace ID:

- The Namespace ID "whip" has been assigned.

Registration Information:

- Version: 1

- Date: TBD

Declared registrant of the namespace:

- Registering organization: The Internet Engineering Task Force.

- Designated contact: A designated expert will monitor the WHIP public mailing list, "wish@ietf.org".

Declaration of Syntactic Structure:

- The Namespace Specific String (NSS) of all URNs that use the "whip" Namespace ID shall have the following structure: urn:ietf:params:whip:{type}:{name}:{other}.

 - The keywords have the following meaning:

     - type: The entity type. This specification only defines the "ext" type.

     - name: A required ASCII string that conforms to the URN syntax requirements (see {{?RFC8141}}) and defines a major namespace of a WHIP protocol extension. The value MAY also be an industry name or organization name.

     - other: Any ASCII string that conforms to the URN syntax requirements (see {{?RFC8141}}) and defines the sub-namespace (which MAY be further broken down in namespaces delimited by colons) as needed to uniquely identify an WHIP protocol extension.

Relevant Ancillary Documentation:

 - None

Identifier Uniqueness Considerations:

- The designated contact shall be responsible for reviewing and enforcing uniqueness.

Identifier Persistence Considerations:

 - Once a name has been allocated, it MUST NOT be reallocated for a different purpose.
 - The rules provided for assignments of values within a sub-namespace MUST be constructed so that the meanings of values cannot change.
 - This registration mechanism is not appropriate for naming values whose meanings may change over time.

Process of Identifier Assignment:

- Namespace with type "ext" (e.g., "urn:ietf:params:whip:ext") is reserved for IETF-approved WHIP specifications.

Process of Identifier Resolution:

 - None specified.

Rules for Lexical Equivalence:

 - No special considerations; the rules for lexical equivalence specified in {{?RFC8141}} apply.

Conformance with URN Syntax:

 - No special considerations.

Validation Mechanism:

 - None specified.

Scope:

 - Global.

## Registering WHIP Protocol Extensions URNs

This section defines the process for registering new WHIP protocol extensions URNs with IANA in the "WebRTC-HTTP ingestion protocol (WHIP) extension URNs" registry (see {{urn-whip-subspace}}). 
   
A WHIP Protocol Extension URNs is used as a value in the "rel" attribute of the Link header as defined in {{protocol-extensions}} for the purpose of signaling the WHIP protocol extensions supported by the WHIP endpoints.
   
WHIP Protocol Extensions URNs have an "ext" type as defined in {{urn-whip-subspace}}.

###  Registration Procedure

   The IETF has created a mailing list, "wish@ietf.org", which can be used
   for public discussion of WHIP protocol extensions proposals prior to registration.
   Use of the mailing list is strongly encouraged. The IESG has
   appointed a designated expert as per {{?RFC8126}} who will monitor the
   wish@ietf.org mailing list and review registrations.

   Registration of new "ext" type URNs (in the namespace "urn:ietf:params:whip:ext") belonging to a WHIP Protocol Extension MUST be documented in a permanent and readily available public specification, in sufficient detail so that interoperability between independent implementations is possible and reviewed by the designated expert as per Section 4.6 of {{?RFC8126}}.
   An Standards Track RFC is REQUIRED for the registration of new value data types that modify existing properties.
   An Standards Track RFC is also REQUIRED for registration of WHIP Protocol Extensions URNs that modify WHIP Protocol Extensions previously documented in an existing RFC.

   The registration procedure begins when a completed registration template, defined in the sections below, is sent to iana@iana.org.
   Decisions made by the designated expert can be appealed to an Applications and Real Time (ART) Area Director, then to the IESG.
   The normal appeals procedure described in {{?BCP9}} is to be followed. 
   
   Once the registration procedure concludes successfully, IANA creates
   or modifies the corresponding record in the WHIP Protocol Extension registry.

   An RFC specifying one or more new WHIP Protocol Extension URNs MUST include the
   completed registration templates, which MAY be expanded with
   additional information. These completed templates are intended to go
   in the body of the document, not in the IANA Considerations section.
   The RFC MUST include the syntax and semantics of any extension-specific attributes that may be provided in a Link header
   field advertising the extension.

### Guidance for Designated Experts

The Designated Expert (DE) is expected to ascertain the existence of suitable documentation (a specification) as described in {{?RFC8126}} and to verify that the document is permanently and publicly available. 

The DE is also expected to check the clarity of purpose and use of the requested registration.

Additionally, the DE must verify that any request for one of these registrations has been made available for review and comment by posting the request to the WebRTC Ingest Signaling over HTTPS (wish) Working Group mailing list. 

Specifications should be documented in an Internet-Draft. Lastly, the DE must ensure that any other request for a code point does not conflict with work that is active in or already published by the IETF.

###  WHIP Protocol Extension Registration Template

A WHIP Protocol Extension URNs is defined by completing the following template:

 -   URN: A unique URN for the WHIP Protocol Extension (e.g., "urn:ietf:params:whip:ext:example:server-sent-events").
 -   Reference: A formal reference to the publicly available specification
 -   Name: A descriptive name of the WHIP Protocol Extension (e.g., "Sender Side events").
 -   Description: A brief description of the function of the extension, in a short paragraph or two
 -   Contact information: Contact information for the organization or person making the registration

# Acknowledgements

The authors wish to thank Lorenzo Miniero, Juliusz Chroboczek, Adam Roach, Nils Ohlmeier, Christer Holmberg, Cameron Elliott, Gustavo Garcia, Jonas Birme, Sandro Gauci, Christer Holmberg and everyone else in the WebRTC community that have provided comments, feedback, text and improvement proposals on the document and contributed early implementations of the spec. 

--- back
