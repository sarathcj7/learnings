# Security — Mock Interview Question Bank

## How to Use This

Attempt each question before reading the fundamentals. Write your answer aloud or on paper. This forces active recall. Then check your answer against the fundamentals sub-files. Mark questions you struggle with and revisit them.

---

## Authentication & Authorization

1. Compare basic auth vs bearer token vs session cookies. When would you use each?
2. Explain OAuth 2.0. What's the flow for a third-party app accessing user data?
3. What's the difference between authentication and authorization?
4. Your API uses JWT tokens. A token is compromised. How do you revoke it?
5. Design a single sign-on (SSO) system for multiple internal services.
6. What's the difference between OpenID Connect and OAuth 2.0?
7. Your mobile app needs to authenticate with your backend. Design it (token refresh, storage, etc).
8. Explain SAML. When would you use it vs OAuth?
9. What's multi-factor authentication (MFA)? Why is it important?
10. Design a passwordless authentication system (e.g., magic links or biometric).

---

## Password Management & Hashing

11. How should you store passwords? Explain hashing, salting, and iteration.
12. Compare bcrypt vs scrypt vs Argon2. Which is best and why?
13. What's the "salting" part of password hashing? Why not just hash?
14. A user's password database is leaked. What's your damage control plan?
15. What's the difference between hashing and encryption for passwords?
16. Your password hashing uses SHA-256. Is this secure? Why or why not?
17. Explain password iteration/stretching. Why do modern algorithms do this?
18. What's the relationship between password length and security?
19. Design a system for secure password resets.
20. What's the OWASP password requirement guidance?

---

## Session Security

21. What's CSRF (Cross-Site Request Forgery)? How do you prevent it?
22. Explain session fixation attacks. How would you defend against them?
23. What's the difference between session-based auth and token-based auth?
24. Your web app uses session cookies. How do you set the cookie headers securely?
25. A user logs out. Their session token is still valid. Is this a problem?
26. Explain the SameSite cookie attribute. What values should you use?
27. What's the HTTPOnly flag on cookies? Why is it important?
28. Your API uses tokens in headers vs cookies. Which is more secure for a web app?
29. How long should session tokens be valid? What's the trade-off?
30. Design a secure session management system with token refresh.

---

## Encryption

31. What's the difference between encryption at rest vs in transit?
32. Your database is encrypted at rest. A hacker gets the encryption keys. Is it secure?
33. Explain symmetric vs asymmetric encryption. When would you use each?
34. What's TLS/SSL? How does it provide encryption in transit?
35. You're encrypting user data with AES-256. What's the key management strategy?
36. What's the difference between encryption and hashing?
37. How do you protect encryption keys? Explain key rotation.
38. What's a hardware security module (HSM)? When would you use one?
39. Design an end-to-end encryption system where the server can't read user data.
40. What's forward secrecy in TLS? Why is it important?

---

## SQL Injection & Web Vulnerabilities

41. Explain SQL injection. How do you prevent it?
42. What's parameterized queries? Why do they prevent SQL injection?
43. Your ORM (e.g., Django ORM) prevents SQL injection. Why can you still be vulnerable?
44. Explain XSS (Cross-Site Scripting). How do you prevent it?
45. What's the difference between stored XSS and reflected XSS?
46. Your API returns user data as JSON. Is it vulnerable to XSS? Why or why not?
47. What's the Content Security Policy (CSP) header? How would you use it?
48. Explain command injection. How is it different from SQL injection?
49. What's OS injection? Give an example and prevention strategy.
50. Design a form that safely accepts and stores user input.

---

## API Security

51. Should your API use HTTP or HTTPS? When, if ever, is HTTP acceptable?
52. How do you rate-limit an API? Why is it security-related?
53. Your API key is leaked. How do you revoke it? How do users get a new one?
54. Explain API versioning and deprecation. Why is it security-related?
55. What's the difference between public, internal, and private APIs in terms of security?
56. Design authentication and rate-limiting for a public REST API.
57. Your API accepts file uploads. What are the security risks?
58. How do you prevent DDOS attacks on your API?
59. What's the OWASP top 10 API vulnerabilities?
60. Design an API security audit checklist.

---

## Infrastructure & Access Control

61. Explain the principle of least privilege. How would you apply it?
62. What's network segmentation? Why is it important?
63. Design network security for a multi-tier application (web, app, database).
64. What's a firewall? What traffic would you block/allow?
65. Explain VPN and when you'd use it.
66. What's SSH key management? Compare vs password-based login.
67. Design secure access for on-call engineers (remote SSH, VPN, etc).
68. What's data exfiltration? How would you detect it?
69. Design a zero-trust security model for a company.
70. Explain secrets management (API keys, credentials, certificates).

---

## Compliance & Privacy

71. What's GDPR? How does it affect your system design?
72. Explain the "right to be forgotten" under GDPR. How would you implement it?
73. What's PCI-DSS? When does it apply?
74. Design a system that's GDPR-compliant (data retention, deletion, privacy).
75. What's an audit log? Why is it important for compliance?

---

## Synthesis & Cross-Cutting

76. Design authentication and authorization for a multi-tenant SaaS platform.
77. A security audit finds your API keys stored in environment variables. What's the risk? How do you fix it?
78. Design a secure payment flow (avoiding PCI-DSS burden with tokenization).
79. Your service handles sensitive data. Design encryption, key management, and audit logging.
80. Compare security trade-offs: convenience (password) vs security (MFA + YubiKey).

---

## Advanced & Tricky Questions

81. What's the difference between confidentiality, integrity, and availability?
82. Explain cryptographic signatures. How are they different from encryption?
83. What's the birthday attack on hashing? When is it a concern?
84. Design a threat model for your application.
85. What's a replay attack? How do you prevent it?
86. Compare a WAF (Web Application Firewall) vs network firewall.
87. What's secret sharing (e.g., Shamir's Secret Sharing)? When would you use it?
88. Design secure handling of third-party OAuth tokens.
89. What's the security implication of storing user PII in logs?
90. How would you design a secure system with offline-first mobile app architecture?

---

## Self-Scoring Guide

- **0-25 answered well**: Not ready for interviews on security. Study fundamentals first.
- **26-50 answered well**: You know the basics. Study weak areas and compliance.
- **51-90 answered well**: You're interview-ready on security.
