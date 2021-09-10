---
docname: draft-ietf-wish-whip-00
title: WebRTC-HTTP ingestion protocol (WHIP)
abbrev: whip
category: std
ipr: trust200902
area: ART
workgroup: wish

keyword: WebRTC

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: S. Murillo
    name: Sergio Garcia Murillo
    organization: CoSMo Software
    email: sergio.garcia.murillo@cosmosoftware.io
 -
    ins: A. Gouaillard
    name: Alexandre Gouaillard
    organization: CoSMo Software
    email: alex.gouaillard@cosmosoftware.io


normative:
  RFC2119:
  RFC7675:
  RFC8838:
  RFC8840:
  RFC8853:
  RFC8863:

--- abstract

While WebRTC has been very sucessful in a wide range of scenarios, its adoption in the broadcasting/streaming industry is lagging behind.
Currently there is no standard protocol (like SIP or RTSP) designed for ingesting media into a streaming service using WebRTC and so content providers still rely heavily on protocols like RTMP for it.

These protocols are much older than WebRTC and by default lack some important security and resilience features provided by WebRTC with minimal overhead and additional latency.

The media codecs used for ingestion in older protocols tend to be limited and not negotiated. WebRTC includes support for negotiation of codecs, potentially alleviating transcoding on the ingest node (wich can introduce delay and degrade media quality). Server side transcoding that has traditionally been done to present multiple renditions in Adaptive Bit Rate Streaming (ABR) implementations can be replaced with simulcasting and SVC codecs that are well supported by WebRTC clients. In addition, WebRTC clients can adjust client-side encoding parameters based on RTCP feedback to maximize encoding quality.

Encryption is mandatory in WebRTC, therefore secure end-to-end transport of media is implicit.

This document proposes a simple HTTP based protocol that will allow WebRTC based ingest of content into streaming servics and/or CDNs.

--- middle

# Introduction

WebRTC intentionally does not specify a signaling transport protocol at application level, while RTCWEB standardized the signalling protocol itself (JSEP, SDP O/A) and everything that was going over the wire (media, codec, encryption, ...). This flexibility has allowed for implementing a wide range of services. However, those services are typically standalone silos which don't require interoperability with other services or leverage the existence of tools that can communicate with them. 

In the broadcasting/streaming world, the usage of hardware encoders that would make it very simple to plug in (SDI) cables carrying raw media, encoding it in place, and pushing it to any streaming service or CDN ingest is ubiquitous. Having to implement a custom signalling transport protocol for each different webrtc services has hindered adoption.

While some standard signalling protocols are available that can be integrated with WebRTC, like SIP or XMPP, they are not designed to be used in broadcasting/streaming services, and there also is no sign of adoption in that industry. RTSP, which is based on RTP and maybe the closest in terms of features to webrtc, is not compatible with WebRTC SDP offer/answer model.

In the specific case of ingest into a platform, some assumption can be made about the server-side which simplifies the webrtc compliance burden, as detailed in webrtc-gateway document {{!I-D.draft-alvestrand-rtcweb-gateways}}.

This document proposes a simple protocol for supporting WebRTC as ingest method which is:

- Easy to implement,
- As easy to use as current RTMP URIs.
- Fully compliant with Webrtc and RTCWEB specs.
- Allow for both ingest in traditional media platforms for extension and ingest in webrtc end-to-end platform for lowest possible latency.
- Lowers the requirements on both hardware encoders and broadcasting services to support webrtc.
- Usable both in web browsers and in native encoders.

# Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in {{!RFC2119}}.

- WHIP client: WebRTC Media encoder or producer that acts as client on the WHIP protocol and encodes and delivers the media to a remote media server.
- WHIP endpoint: Ingest server receiving the initial WHIP request.
- WHIP endpoint URL: URL of the WHIP endpoint that will create the WHIP resource
- Media Server: WebRTC media server that establishes the media session with the WHIP client and receives the media produced by it.
- WHIP resource: Allocated resource by the WHIP endpoint for an ongoing ingest session that the WHIP client can send request for altering the session (ICE operations or termination, for example).
- WHIP resource URL: URL allocated to a specific media session by the WHIP endpoint which can be used to perform operations such terminating the session or ICE restarts.


# Overview

The WebRTC-HTTP ingest protocol (WHIP) uses an HTTP POST request to perform a single shot SDP offer/answer so an ICE/DTLS session can be established between the encoder/media producer and the broadcasting ingestion endpoint.

Once the ICE/DTLS session is set up, the media will flow unidirectionally from the encoder/media producer to the broadcasting ingestion endpoint. In order to reduce complexity, no SDP renegotiation is supported, so no tracks or streams can be added or removed once the initial SDP O/A over HTTP is completed.

~~~~~
                                                                                 
 +-----------------+         +---------------+ +--------------+ +----------------+
 | WebRTC Producer |         | WHIP endpoint | | Media Server | | WHIP Resource  |
 +---------+-------+         +-------+- -----+ +------+-------+ +--------|-------+
           |                         |                |                  |        
           |                         |                |                  |        
           |HTTP POST (SDP Offer)    |                |                  |        
           +------------------------>+                |                  |        
           |201 Created (SDP answer) |                |                  |        
           +<------------------------+                |                  |        
           |          ICE REQUEST                     |                  |        
           +----------------------------------------->+                  |        
           |          ICE RESPONSE                    |                  |        
           <------------------------------------------+                  |        
           |          DTLS SETUP                      |                  |        
           <==========================================>                  |        
           |          RTP/RTCP FLOW                   |                  |        
           +------------------------------------------>                  |        
           | HTTP DELETE                                                 |        
           +------------------------------------------------------------>+         
           | 200 OK                                                      |        
           <-------------------------------------------------------------x        
                                                                    
~~~~~
{: title="WHIP session setup and teardown"}

# Protocol Operation

In order to setup an ingestion session, the WHIP client will generate an SDP offer according to the JSEP rules and do an HTTP POST request to the WHIP endpoint configured URL.

The HTTP POST request will have a content type of application/sdp and contain the SDP offer as body. The WHIP endpoint will generate an SDP answer and return it on a 201 Accepted response with content type of application/sdp and the SDP answer as body and a Location header pointing to the newly created resource.

SDP offer SHOULD use the sendonly attribute and the SDP answer MUST use the recvonly attribute.

Once a session is setup ICE consent freshness {{!RFC7675}} will be used to detect abrupt disconnection and DTLS teardown for session termination by either side.

To explicitly terminate the session, the WHIP client MUST perform an HTTP DELETE request to the resource url returned on the Location header of the initial HTTP POST. Upon receiving the HTTP DELETE request, the WHIP resource will be removed and the resources freed on the media server, terminating the ICE and DTLS sessions.

A media server terminating a session MUST follow the procedures in {{!RFC7675}} section 5.2 for immediate revocation of consent.

The WHIP endpoints MUST return an HTTP 405 response for any HTTP GET, HEAD or PUT requests on the resource URL in order to reserve its usage for future versions of this protocol specification.

The WHIP resources MUST return an HTTP 405 response for any HTTP GET, HEAD, POST or PUT requests on the resource URL in order to reserve its usage for future versions of this protocol specification.

## ICE and NAT support

In order to simplify the protocol, there is no support for exchanging gathered trickle candidates from media server ICE candidates once the SDP answer is sent.  So in order to support the WHIP client behind NAT, the WHIP media server SHOULD be publicly accessible.

The initial offer by the WHIP client MAY be sent after the full ICE gathering is complete containing the full list of ICE candidates, or only contain local candidates or even an empty list of candidates.

The WHIP endpoint SDP answer SHALL contain the full list of ICE candidates publicly accessible of the media server. The media server MAY use ICE lite, while the WHIP client MUST implement full ICE.

The WHIP client MAY perform trickle ICE or an ICE restarts {{!RFC8863}} by sending a HTTP PATCH request to the WHIP resource URL with a body containing a SDP fragment with mime type "application/trickle-ice-sdpfrag" as specified in {{!RFC8840}} with the new ice candidate or ice ufrag/pwd for ice restarts. A WHIP resource MAY not support either trickle ICE (i.e. ICE lite media servers) or ICE restart, and it MUST return a 405 Method Not Allowed for any HTTP PATCH request.

A WHIP client receiving a 405 response for an HTTP PATCH request SHALL not send further request for ICE trickle or restart. If the WHIP client gathers additional candidates (via STUN/TURN) after the SDP offer is sent, it MUST send STUN request to the ICE candidates received from the media server as per {{!RFC8838}} regardless if the HTTP PATCH is supported by either the WHIP client or the WHIP resource.

## Webrtc constraints

In order to reduce the complexity of implementing WHIP in both clients and media servers, some restrictions regarding WebRTC usage are made.

SDP bundle SHALL be used by both the WHIP client and the media server. The SDP offer created by the WHIP client MUST include the bundle-only attribute in all m-lines as per {{!RFC8843}}. Also, RTCP muxing SHALL be supported by both the WHIP client and the media server.

Unlike {{!RFC5763}} a WHIP client MAY use a setup attribute value of setup:active in the SDP offer, in which case the WHIP endpoint MUST use a setup attribute value of setup:passive in the SDP answer. 

## Load balancing and redirections

WHIP endpoints and media servers MAY not be colocated on the same server so it is possible to load balance incoming requests to different media servers. WHIP clients SHALL support HTTP redirection via 307 Temporary Redirect response code.

In case of high load, the WHIP endpoints may return a 503 (Service Unavailable) status code indicating that the server is currently unable to handle the request due to a temporary overload or scheduled maintenance, which will likely be alleviated after some delay. 

The WHIP endpoint MAY send a Retry-After header field indicating the minimum time that the user agent is asked to wait before issuing the redirected request.

## STUN/TURN server configuration

Configuration of the TURN or STUN servers used by the WHIP client is out of the scope of this document.

It is RECOMMENDED that broadcasting server provides an HTTP interface for provisioning the TUNR/STUN servers url and short term credentiasl as in {{!I-D.draft-uberti-behave-turn-rest-00}}. Note that the authentication information or the url of this API are not related to the WHIP enpoint URLs or authentication.

It could also be possilble to configure the STUN/TURN server URLS and long term credentials provided by the either broadcasting service or an external TURN provider.

## Authentication and authorization

Authentication and authorization is supported by the Authorization HTTP header with a bearer token as per {{!RFC6750}}.

## Simulcast and scalable video coding

Both simulcast and scalable video coding (including K-SVC modes) MAY be supported by both media servers and WHIP clients and negotiated in the SDP O/A.

If the client supports simulcast and wants to enable it for publishing, it MUST negotiate the support in the SDP offer according to the procedures in {{!RFC8853}} section 5.3. A server accepting a simulcast offer MUST create an answer accoding to the procedures {{!RFC8853}} section 5.3.2.

## Protocol extensions

In order to support future extensions to be defined for the WHIP protocol, a common procedure for registering and announcing the new extensions is defined.

Protocol extensions supported by the WHIP server MUST be advertised to the WHIP client on the 201 created response to initial HTTP POST request to the WHIP enpoint by inserting one Link header for each extension with the extension "rel" type attribute and the uri for the HTTP resource that will be available for receiving request related to that extension.

Protocol extensions are optionasl for bot WHIP clients and servers. WHIP clients MUST ignore any Link attribute with an unknown "rel" attribute value and WHIP servers MUST not require the usage of any of the extensions.

Each protocol extension MUST register an unique "rel" attribute values at IANA starting with the prefix: "urn:ietf:params:whip:".

For example, taking a potential extension of server to client communication using server sent events as specified in https://html.spec.whatwg.org/multipage/server-sent-events.html#server-sent-events, the url for connecting to the server side event resource for the published stream will be returned in the initial HTTP "201 Created" response with a "Link" header an a "rel" attribute of  "urn:ietf:params:whip:server-sent-events".

The HTTP 201 response to the HTTP POST request would look like:

~~~~~
HTTP/1.1 201 Created
Content-Type: application/sdp
Location: https://whip.ietf.org/publications/213786HF
Link: <https://whip.ietf.org/publications/213786HF/sse>;rel="urn:ietf:params:whip:server-side-events "
~~~~~


# Security Considerations

HTTPS SHALL be used in order to preserve the WebRTC security model.


# IANA Considerations

# Acknowledgements

--- back

