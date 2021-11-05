# WHIP - WebRTC HTTP ingest protocol draft

While WebRTC has been very successful in a wide range of scenarios, its adoption in the broadcasting/streaming industry is lagging behind.
Currently there is no standard protocol (like SIP or RTSP) designed for ingesting media into a streaming service using WebRTC and so content providers still rely heavily on protocols like RTMP for it.

These protocols are much older than WebRTC and by default lack some important security and resilience features provided by WebRTC with minimal overhead and additional latency.

The media codecs used for ingestion in older protocols tend to be limited and not negotiated. WebRTC includes support for negotiation of codecs, potentially alleviating transcoding on the ingest node (wich can introduce delay and degrade media quality). Server side transcoding that has traditionally been done to present multiple renditions in Adaptive Bit Rate Streaming (ABR) implementations can be replaced with [simulcasting](https://webrtcglossary.com/simulcast/) and SVC codecs that are well supported by WebRTC clients. In addition, WebRTC clients can adjust client-side encoding parameters based on RTCP feedback to maximize encoding quality.

Encryption is mandatory in WebRTC, therefore secure end-to-end transport of media is implicit.

This document proposes a simple HTTP based protocol that will allow WebRTC based ingest of content into streaming services and/or CDNs.

## Current Draft
- [draft-ietf-wish-whip-00](https://datatracker.ietf.org/doc/html/draft-ietf-wish-whip-01)
