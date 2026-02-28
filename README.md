# Mitigate an economic denial-of-service (EDoS) bot attack targeting Azure JA4 TLS fingerprints

This repository documents a real-world case study from JURASSIC PARK TECHNOLOGY LTD on how to detect and mitigate an economic denial-of-service (EDoS) bot attack targeting Azure AppService using JA4 TLS fingerprints, while cascading protection through Azure Front Door (AFD) and Azure Application Gateway WAF.
Unlike traditional DDoS attacks that try to take your site down, this attack was designed to inflate CDN and data transfer costs without overloading the origin, exploiting cached static assets and globally distributed IPs.



## Diagram
<img width="2189" height="1144" alt="image" src="https://github.com/user-attachments/assets/4da2271b-0e95-41b3-bc68-9ab2b67d0659" />





## Scenario: JURASSIC PARK TECHNOLOGY LTD under EDoS attack
JURASSIC PARK TECHNOLOGY Ltd. hosts its client‑facing website on Azure using Azure Front Door (AFD) and App Service. The site came under an EDoS attack: the service remained available, but Azure outbound data charges increased by almost 10x. A detailed investigation showed the attack originated from a highly distributed set of IP addresses, each sending only a few requests per minute, making IP‑based blocking or simple rate limiting ineffective.

Further analysis revealed that most of the malicious traffic shared a single JA4 TLS fingerprint and consistently targeted the path /media/promote.mp4. Although the root cause was identified, neither Azure Front Door nor Application Gateway could natively block JA4 fingerprints. JURASSIC PARK therefore implemented a cascaded design: AFD would surface JA4 information, and Application Gateway would then use it to either block the traffic or present a challenge to the offending clients.

<img width="2214" height="1159" alt="image" src="https://github.com/user-attachments/assets/522ad414-9629-4841-b8c6-be5a45a25610" />

## Blocking rules (also attached as JSON)

Rule 1 – Block media requests:
If a request has the JA4 fingerprint t12d8007h2_ee0b5a6c69b8_0a9c83bf8b96 and the URL contains the word media, the request is blocked. This stops that fingerprint from downloading media files.

Rule 2 – Challenge homepage requests:
If a request has the same JA4 fingerprint and the URL contains homepage.html, the system sends a JavaScript challenge instead of blocking immediately. Real browsers can usually pass this challenge, but simple bots often fail, so this helps separate humans from bots for the homepage.


# Appendix: What is JA4 Fingerprinting?
JA4 is the latest TLS client fingerprinting standard that creates a compact, stable identifier from the TLS Client Hello packet. It uniquely identifies client software (browsers, libraries, automation tools) even when attackers rotate IP addresses, User Agents, or other headers.

Key advantages over older JA3:

More stable across TLS library updates
Harder for attackers to spoof
Shorter format (e.g. t12d8007h2_ee0b5a6c69b8_0a9c83bf8b96)

Format breakdown (for t12d8007h2_ee0b5a6c69b8_0a9c83bf8b96):
t12d8007h2 – TLS version, ciphers, extensions, ALPN
ee0b5a6c69b8 – SNI hash
0a9c83bf8b96 – full TLS handshake hash

Official reference: 
JA4+ Fingerprinting Standard
