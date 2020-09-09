# WHIP - WebRTC HTTP ingest protocol draft

While WebRTC has been very sucessful in a wide range of scenarios, its adption in the broadcasting/streaming industry is lagging behind.
Currently there is no standard protocol (like SIP or RTSP) designed for ingesting media in a streaming service, and content providers still rely heavily on protocols like RTMP for it.

These protocols are much older than webrtc and lack by default some important security and resilience features provided by webrtc with minimal delay.

The media codecs used in older protocols do not always match those being used in WebRTC, mandating transcoding on the ingest node, introducing delay and degrading media quality. This transcoding step is always present in traditionnal streaming to support e.g. ABR, and comes at no cost. However webrtc implements 
client-side ABR, also called Network-Aware Encoding by e.g. Huavision, by means of simulcast and SVC codecs, which otherwise alleviate the need for server-side transcoding. Content protection and Privacy Enhancement can be achieve with End-to-End Encryption, which preclude any server-side media processing.

This document proposes a simple HTTP based protocol that will allow WebRTC endpoings to ingest content into streaming servics and/or CDNs to fill this gap and facilitate deployment.

## Current Draft
- [draft-murillo-whip-00](https://tools.ietf.org/html/draft-murillo-whip-00)
