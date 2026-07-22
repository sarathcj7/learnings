# Networking and Protocols — Mock Interview Question Bank

## How to Use This

Attempt each question before reading the fundamentals. Write your answer aloud or on paper. This forces active recall. Then check your answer against the fundamentals sub-files. Mark questions you struggle with and revisit them.

---

## HTTP Versions & Performance

1. Compare HTTP/1.1 vs HTTP/2. What problems does HTTP/2 solve?
2. What's head-of-line blocking in HTTP/1.1? How does HTTP/2 eliminate it?
3. Explain server push in HTTP/2. When would you use it?
4. Your HTTP/1.1 API makes 100 requests to load a page. How many TCP connections are needed? How does HTTP/2 change this?
5. What's multiplexing in HTTP/2? Why does it matter for web performance?
6. HTTP/3 uses QUIC instead of TCP. What's the benefit?
7. Your website loads slowly on mobile. Would you migrate from HTTP/1.1 to HTTP/2? What's the first metric you'd check?
8. Explain HPACK compression in HTTP/2. Why do headers get compressed but bodies don't?
9. A single connection in HTTP/1.1 has 1 Mbps throughput. With HTTP/2 on the same connection, can you get more throughput? Why or why not?
10. What's the connection establishment overhead difference between HTTP/1.1 (TCP) and HTTP/3 (QUIC)?

---

## WebSockets & Real-Time Communication

11. When would you use WebSockets vs HTTP polling vs long polling?
12. What's the difference between WebSocket and HTTP/2 server push?
13. Your chat app needs real-time messages. Design the communication layer.
14. WebSocket connections consume server resources. How would you scale WebSocket servers?
15. You have 100k concurrent WebSocket connections. What infrastructure do you need?
16. Explain the WebSocket handshake. What HTTP headers are involved?
17. A WebSocket connection breaks. How should the client reconnect? Immediately or with backoff?
18. Compare WebSockets vs server-sent events (SSE). When would you choose each?
19. Your WebSocket server crashes. How do you notify 100k connected clients?
20. Design a multi-region WebSocket system. How do users connect to the nearest region?

---

## TLS & SSL Security

21. Explain the TLS 1.3 handshake. How many round trips does it require?
22. What's the difference between TLS 1.2 and TLS 1.3?
23. In TLS handshake, when is the shared secret established? At what step can the client start sending encrypted data?
24. What's a certificate chain? Why do you need a root CA?
25. Your server has a certificate. How does a client verify it's legitimate?
26. What's certificate pinning? When would you use it?
27. A certificate expires tomorrow. How do you update it without downtime?
28. Explain the difference between self-signed certificates and CA-signed certificates.
29. What's OCSP stapling? Why is it better than OCSP checking?
30. Your API uses mTLS (mutual TLS). Both client and server have certs. Explain the handshake.

---

## DNS & Resolution

31. Explain DNS resolution. How many queries are needed to resolve www.example.com?
32. What's DNS TTL? If you set TTL=1 hour, what happens?
33. Your load balancer has IP 1.2.3.4. You set DNS TTL=300 (5 mins). A client caches for 5 mins. You replace the load balancer with IP 5.6.7.8. How long until clients see the new IP?
34. What's DNS caching at multiple levels (OS, resolver, ISP)? How does it affect failover?
35. Explain CNAME, A, and AAAA records. When would you use each?
36. Your application migrates to a new provider. You update DNS. Some users still hit the old provider. Why?
37. What's DNS prefetching and preconnect? How would you use these in a web app?
38. Design a DNS failover strategy for a critical service.
39. Explain GeoDNS. How would you route users to the nearest datacenter?
40. What's the difference between authoritative DNS and recursive DNS?

---

## Load Balancing & Connection Management

41. Explain connection pooling. Why is it important?
42. Your database has a 100-connection limit. Your API has 1000 QPS. How do you avoid hitting the limit?
43. What's keep-alive? How does it reduce latency?
44. Explain the difference between connection pooling and multiplexing.
45. You have 10 backend servers. Load balancer receives 1000 QPS. How do you distribute traffic? (Round-robin? Least connections? Latency-aware?)
46. Your sticky session (cookie-based routing) causes load imbalance. Why? How would you fix it?
47. What's a hot shard in load balancing? How would you detect and mitigate it?
48. Design a geographically distributed load balancing strategy for a global service.

---

## Network Protocols & Layers

49. Explain the OSI model. Which layers are most relevant to system design?
50. What's the difference between TCP and UDP? When would you use UDP?
51. Your gaming app needs low latency. Would you use TCP or UDP? Justify.
52. Explain TCP flow control and congestion control.
53. What's the TCP three-way handshake? Why is SYN-ACK necessary?
54. What's a SYN flood attack? How would you defend against it?
55. Explain Nagle's algorithm. When would you disable it?
56. What's TCP fast open (TFO)? How does it reduce handshake latency?

---

## Synthesis & Cross-Cutting

57. Design a real-time stock price update system for 100k concurrent users. What protocols would you use?
58. Your payment API has P99 latency of 100ms. Clients complain about slowness. You measure 50ms network RTT. What else contributes?
59. Migrate a synchronous API (HTTP/1.1) to HTTP/2. What changes are needed?
60. Design a protocol for IoT devices that can't maintain persistent connections and have limited bandwidth.
61. Your video streaming service has users on 4G (variable latency) and fiber (stable). Design adaptive streaming.
62. Compare WebSocket vs gRPC streaming for a real-time data pipeline. Which would you choose?

---

## Advanced & Tricky Questions

63. What's BGP (Border Gateway Protocol)? How does it affect global routing?
64. Explain anycast routing. How would you use it for a global service?
65. What's the difference between forward proxy and reverse proxy? When would you use each?
66. Design a system that detects and reroutes around network partitions in real-time.
67. Your API serves both HTTP/1.1 (legacy) and HTTP/2 (new). How would you gradually migrate clients?
68. What's ALPN (Application Layer Protocol Negotiation)? Why do you need it?
69. Compare RSA and ECDSA for TLS certificates. Which is better and why?
70. Design a protocol that works over intermittent connectivity (like satellite internet).

---

## Self-Scoring Guide

- **0-25 answered well**: Not ready for interviews on networking. Study fundamentals first.
- **26-50 answered well**: You know the basics. Study weak areas and focus on trade-offs.
- **51-70 answered well**: You're interview-ready on networking.
