# Networking & Protocols

Understanding networking is foundational for system design. This topic covers DNS, TCP/UDP, HTTP versions, TLS, and real-time transport — everything that happens below your application.

## Sub-topics

- **DNS & Service Discovery**: DNS resolution mechanics, geo-routing, service discovery patterns
- **TCP/UDP & Transport**: TCP handshake, congestion control, when to use UDP/QUIC
- **HTTP Versions & Semantics**: HTTP/1.1 vs 2 vs 3, multiplexing, head-of-line blocking
- **TLS & PKI**: TLS handshake, certificate chains, mTLS
- **WebSockets & Real-time Transport**: WebSockets vs long-polling vs SSE — trade-offs for real-time

## Why This Matters

- **Latency**: Understanding TCP/TLS handshakes explains why cold connections are slow (~100ms vs 5ms for keep-alive).
- **Scale**: DNS and load balancing at the transport layer (L4) are critical for distributing traffic.
- **Real-time systems**: Choosing between WebSockets, SSE, and polling affects architecture and scale.
- **Security**: TLS, certificate management, and mTLS are essential for production systems.

## Interview Prep

- **[Question Bank](../../interview-prep/question-banks/networking-and-protocols.md)**: 25+ questions on networking fundamentals
- **[Flashcards](../../interview-prep/flashcards/networking-and-protocols.md)**: Key concepts for quick recall

## Related Fundamentals

- [APIs & Communication](../apis-and-communication/) – Application layer above HTTP
- [Microservices Architecture](../microservices-architecture/) – Inter-service communication
- [Security](../security/) – TLS and encryption details

## Related Case Studies

- [Chat & Messaging System](../../case-studies/chat-and-messaging-system.md) – WebSockets for real-time
- [Video Streaming Platform](../../case-studies/video-streaming-platform.md) – Adaptive bitrate over HTTP

---

**Status**: ⭕ Pending detailed content. Framework ready.
