# Wireshark SOC Lab: HTTP Plain Text Credentials

**NetworkWalks Internship | Week 1 | SOC Project Module: Wireshark (Extra)**
**Result: 100 / 100**

## Objective

Analyse a packet capture of a staff member signing in to an internal portal and show that credentials sent over HTTP can be read by anyone who can see the traffic.

## Scenario

A staff member signed in to the old staff portal (`portal.company.local`). The task was to identify the protocol, the request details, the user and the credentials, and recommend a fix.

## Tools

- Wireshark (Windows), live capture on the WiFi interface
- Display filters and Follow TCP/HTTP Stream

## Method

| Step | Filter / action | Purpose |
|---|---|---|
| 1 | Capture traffic while signing in to the portal | Generate the evidence |
| 2 | `http.request.method == "POST"` | Isolate form submissions, where sign-ins are sent |
| 3 | Select a POST packet and expand **HTML Form URL Encoded** | Read the submitted form fields as a list |
| 4 | Right-click a packet, Follow, TCP Stream | Read the full request and response exchange |

## Findings

| Question | Answer |
|---|---|
| Protocol carrying the sign-in | HTTP (port 80, unencrypted) |
| Method used | POST |
| Page the form submitted to | `/login` |
| Host | `portal.company.local` |
| Sending PC | 192.168.10.45 |
| Server | 192.168.10.20 (Apache/2.4.57, Ubuntu) |
| Sign-in attempts | 2 (packets 76 and 80) |
| Username | `farhan.k` |
| First attempt | Password `Mango2025`, server replied "Invalid username or password" |
| Second attempt | Password `Mango#2026`, server replied "Welcome back, farhan.k" |
| Packet details line listing the credentials | HTML Form URL Encoded |

## Analysis

The login form submits with `method="post"` to `/login`, and the data travels as `application/x-www-form-urlencoded` over plain HTTP. Nothing is encrypted, so the username and password appear in readable text in the packet details and in the followed stream.

The two attempts show a typical pattern: a failed sign-in with an old-looking password followed by a successful one with a changed password. Because both passwords were exposed, an attacker on the network path would learn the current valid credential and the previous one, which attackers often use to guess related passwords on other systems.

## Impact

- Anyone able to capture traffic between the client and the portal (shared WiFi, compromised switch, rogue device) can recover valid staff credentials.
- Reused passwords put other company and personal accounts at risk.

## Recommendations

1. **Serve the portal over HTTPS (TLS)** and redirect all HTTP requests to HTTPS.
2. Enable HSTS so browsers refuse to fall back to HTTP.
3. Force a password reset for accounts whose credentials were sent in clear text.
4. Add multi-factor authentication to the portal.
5. Monitor for plain-text credential submissions with a detection rule on HTTP POST requests to login paths.

## Skills Demonstrated

Packet capture, display filtering, TCP stream reconstruction, credential exposure analysis, incident findings write-up and remediation recommendations.