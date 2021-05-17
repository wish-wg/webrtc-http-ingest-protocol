---
docname: draft-ietf-whish-whip-00
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
  RFC2119
  RFC7675
  RFC8838
  RFC8840
  RFC8863

--- abstract

While WebRTC has been very successful  in a wide range of scenarios, its adoption in the broadcasting/streaming industry is lagging behind.
Currently there is no standard protocol (like SIP or RTSP) designed for ingesting media in a streaming service, and content providers still rely heavily on protocols like RTMP for it.

These protocols are much older than webrtc and lack by default some important security and resilience features provided by webrtc with minimal delay.

The media codecs used in older protocols do not always match those being used in WebRTC, mandating transcoding on the ingest node, introducing delay and degrading media quality. This transcoding step is always present in traditional streaming to support e.g. ABR, and comes at no cost. However webrtc implements 
client-side ABR, also called Network-Aware Encoding by e.g. Huavision, by means of simulcast and SVC codecs, which otherwise alleviate the need for server-side transcoding. Content protection and Privacy Enhancement can be achieved with End-to-End Encryption, which preclude any server-side media processing.

This document proposes a simple HTTP based protocol that will allow WebRTC endpoints to ingest content into streaming services and/or CDNs to fill this gap and facilitate deployment.

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
- Media Server: WebRTC media server that establishes the media session with the WHIP client and receives the media produced by it.
- WHIP Resource: Allocated resource by the WHIP endpoint for an ongoing ingest session that the WHIP client can send request for altering the session (ICE operations or termination, for example).


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

The media server may terminate the session by using the Immediate Revocation of Consent as defined in {{!RFC7675}} section 5.2.

## ICE and NAT support

In order to simplify the protocol, there is no support for exchanging gathered trickle candidates from media server ICE candidates once the SDP answer is sent.  So in order to support the WHIP client behind NAT, the WHIP media server SHOULD be publicly accessible.

The initial offer by the WHIP client MAY be sent after the full ICE gathering is complete containing the full list of ICE candidates, or only contain local candidates or even an empty list of candidates.

The WHIP endpoint SDP answer SHALL contain the full list of ICE candidates publicly accessible of the media server. The media server MAY use ICE lite, while the WHIP client MUST implement full ICE.

The WHIP client MAY perform trickle ICE or an ICE restarts {{!RFC8863} by sending a HTTP PATCH request to the WHIP resource URL with a body containing a SDP fragment with mime type "application/trickle-ice-sdpfrag" as specified in {{!RFC8840}} with the new ice candidate or ice ufrag/pwd for ice restarts. A WHIP resource MAY not support either trickle ICE (i.e. ICE lite media servers) or ICE restart, and it MUST return a 405 Method Not Allowed for any HTTP PATCH request.

A WHIP client receiving a 405 response for an HTTP PATCH request SHALL not send further request for ICE trickle or restart. If the WHIP client gathers additional candidates (via STUN/TURN) after the SDP offer is sent, it MUST send STUN request to the ICE candidates received from the media server as per {{!RFC8838}} regardless if the HTTP PATCH is supported by either the WHIP client or the WHIP resource.

## Webrtc constraints

In order to reduce the complexity of implementing WHIP in both clients and media servers, some restrictions regarding WebRTC usage are made.

SDP bundle SHALL be used by both the WHIP client and the media server. The SDP offer created by the WHIP client MUST include the bundle-only attribute in all m-lines as per {{!RFC8843}}. Also, RTCP muxing SHALL be supported by both the WHIP client and the media server.

Unlike {{!RFC5763}} a WHIP client MAY use a setup attribute value of setup:active in the SDP offer, in which case the WHIP endpoint MUST use a MUST use the setup attribute value of setup:passive in the SDP answer. 

## Load balancing and redirections

WHIP endpoints and media servers MAY not be colocated on the same server so it is possible to load balance incoming requests to different media servers. WHIP clients SHALL support HTTP redirection via 307 Temporary Redirect response code.

In case of high load, the WHIP endpoints may return a 503 (Service Unavailable) status code indicating that the server is currently unable to handle the request due to a temporary overload or scheduled maintenance, which will likely be alleviated after some delay. 

The WHIP endpoint MAY send a Retry-After header field indicating the minimum time that the user agent is asked to wait before issuing the redirected request.

## Authentication and authorization

Authentication and authorization is supported by the Authorization HTTP header with a bearer token as per {{!RFC6750}}.

## Simulcast and scalable video coding

Both simulcast and scalable video coding (including K-SVC modes) MAY be supported by both media servers and WHIP clients.

# Security Considerations

HTTPS SHALL be used in order to preserve the WebRTC security model.


# IANA Considerations

# Acknowledgements

--- back

