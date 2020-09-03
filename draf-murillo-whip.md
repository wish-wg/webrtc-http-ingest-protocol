---
docname: draft-murillo-whip-00
title: WebRTC-HTTP ingestion protocol (WHIP)
abbrev: Whip
category: info

ipr:
area: Applications and Real-Time Area (art)
keyword: Internet-Draft

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
  RFC3711:
  RFC8285:

informative:
  RFC6464:
  RFC6465:
  RFC6904:

--- abstract

While WebRTC has been very sucessfull in a wide range of scenarios, its adption in the broadcasting/streaming industry is lagging behind.
Currently there is no standard protocol (like SIP or RTSP) designed for ingesting media in a streaming service, and content providers still rely heavily on protocols like RTMP for it.

These protocols are much older than webrtc and lack by default some important security and resilience features provided by webrtc with minimal delay.

The media codecs used in older protocols do not always match those being used in WebRTC, mandating transcoding on the ingest node, introducing delay and degrading media quality. This transcoding step is always present in traditionnal streaming to support e.g. ABR, and comes at no cost. However webrtc implements 
client-side ABR, also called Network-Aware Encoding by e.g. Huavision, by means of simulcast and SVC codecs, which otherwise alleviate the need for server-side transcoding. Content protection and Privacy Enhancement can be achieve with End-to-End Encryption, which preclude any server-side media processing.

This document proposes a simple HTTP based protocol that will allow WebRTC endpoings to ingest content into streaming servics and/or CDNs to fill this gap and facilitate deployment.

--- middle

# Introduction

WebRTC intentionaly does not specify a signaling transport protocol at application level, while RTCWEB standardized the signalling protocol itself (JSEP, SDP O/A) and everything that was going over the wire (media, codec, encryption, ...). This flexibility has allowed for implementing a wide range of services. However, those services are typically standalone silos which don't require interoperability with other services or leverage the existence of tools that can communicate with them. 

In the broadcasting/streaming world, the usage of hardware encoders that would make it very simple to plug in (SDI) cables carrying raw media, encoding it in place, and pushing it to any streaming service or CDN ingest is ubiquitous. Having to implement a custom signalling transport protocol for each different webrtc services has hindered adoption.

While some standard signalling protocols are available that can be integrated with WebRTC, like SIP or XMPP, they are not designed to be used in broadcasting/streaming services, and there also is no sign of adoption in that industry. RTSP, which is based on RTP and maybe the closest in terms of features to webrtc, is not compatible with WebRTC SDP offer/answer model.

In the specific case of ingest into a platform, some assumption can be made about the server-side which simplifies the webrtc compliance burden, as detailled in webrtc-gateway document. https://tools.ietf.org/html/draft-ietf-rtcweb-gateways-02

This document proposse a simple protocol for supporting WebRTC as ingest method which is:
- Easy to implement,
- As easy to use as current RTMP URI.
- Fully compliant with Webrtc and RTCWEB specs.
- Allow for both ingest in traditionnal media platforms for extention and ingest in webrtc end-to-end platform for lowest possible latency.
- Lowers the requirements on both hardware encoders and broadcasting services to support webrtc.
- Usable both in web browsers and in native encoders.

# Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in {{RFC2119}}.

# Overview

The WebRTC-HTTP ingest protocol (WHIP) uses an HTTP POST request to perform a single shot SDP offer/answer so an ICE/DTLS session can be established between the encoder/media producer and the broadcasting ingestion endpoint.

Once the ICE/DTLS session is set up, the media will flow unidirectionally from the encoder/media producer broadcasting ingestion endpoint. In order to reduce complexity, no SDP renegotiation is supported, so no tracks or streams can be added or removed once the initial SDP O/A over HTTP is completed.

~~~~~
+-----------------+         +---------------+ +--------------+
| WebRTC Producer |         | WHIP endpoint | | Media Server |
+---------+-------+         +-------+- -----+ +------+-------+
          |                         |                |
          |                         |                |
          |HTTP POST (SDP Offer)    |                |
          +-------------------------+                |
          |202 Accepted (SDP answer)|                |
          +<------------------------+                |
          |          ICE REQUEST                     |
          +----------------------------------------->+
          |          ICE RESPONSE                    |
          <------------------------------------------+
          |          DTLS SETUP                      |
          <==========================================>
          |          RTP FLOW                        |
          +------------------------------------------>
~~~~~
{: title="WHIP session setup"}

# Protocol Operation

In order to setup an ingestion session, the WebRTC encoder/media producer will generate an SDP offer according the the JSEP rules and do an HTTP POST request to the WHIP endpoint configured URL.

The HTTP POST request will have a content type of application/sdp and contain the SDP offer as body. The WHIP ingestion endpoint will generate an SDP answer and return it on a 202 Accepted response with content type of application/sdp and the SDP answer as body.

Once session is setup ICE consent freshness [rfc7675] will be used to detect abrupt disconnection and DTLS teardown for session termination by either side.

## ICE and NAT support

In order to simplify the protocol, there is no support of exchanging gathered tickle ICE candidates one the SDP offer or answer is sent.
So in order to support encoders/media producers behind NAT, the WHIP media server MUST be publicly accessible.

The initial offer by the encoder/media producer MAY be sent after the full ICE gathering is complete containing the full list of ICE candidates, or only contain local candidates or even an empty list of candidates.
The WHIP endpoint SDP answer SHALL contain the full list of ICE candidates publicly accessible of the media server. The media server MAY use ICE lite, while the encoder/media producer MUST implement full ICE.

If the Encoder/Media producer gathers additional candidates (via STUN/TURN) after the SDP offer is sent, it will send directly a STUN request to the ICE candidates received from the media server as per [draft-ietf-ice-trickle-21].

## Load balancing and redirections

Encoders/media MAY not be colocated on the same server so it is possible to load balance incoming request to different media server. Encoders/media producers SHALL support HTTP redirection via 307 Temporary Redirect response code.

In case of high load, the WHIP endpoints may return a 503 (Service Unavailable) status code indicating that the server is currently unable to handle the request due to a temporary overload or scheduled maintenance, which will likely be alleviated after some delay. 
The server MAY send a Retry-After header field indicating the minimum time that the user agent is asked to wait before issuing the redirected request.

## Authentication and authorization

Authtentication and authorization is supported by the Authorization HTTP header with a bearear token as per [rfc6750].

## Simulcast and scalable video coding

Both simulcast and scalable video coding (including K-SVC modes) MAY be supported by both media servers and encoders/media producers.

# Security Considerations

HTTPS SHALL be used in order to preserve WebRTC security model.


# IANA Considerations

# Acknowledgements

--- back

