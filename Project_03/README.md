# PENETRATION TESTING REPORT

**Footprinting & Network Scanning Phases**

W2-PM-FINAL | CYBERSECURITY | NETWORKWALKS

| Field | Detail |
|---|---|
| Pentester Name (Cybersecurity Professional) | Marshel Mburu Macharia |
| Program/Batch | Networkwalks Cybersecurity Internship, batch: [YOUR BATCH] |
| Date | 12 November 2026 |
| Modules completed | W2-PM1 (Footprinting with Multiple Kali Tools)<br>W2-PM5 (Zenmap Network Scanning) |
| Client/Target | 1. networkwalks.com (public website and DNS records, in scope of the Week 2 authorization)<br>2. My own local LAN (192.168.100.0/24) |
| Permission secured from client? | Yes, for networkwalks.com under the Week 2 Letter of Authorization. The LAN scanned is my own network. |
| Phases covered | Phase 1: Reconnaissance & Footprinting<br>Phase 2: Scanning & Network Discovery<br>Phase 3-5: In Progress |

---

## 1. Liability Disclaimer

I have performed these activities only on the systems and devices where I had secured written permission, or on devices and networks that I own myself. All these materials are for education and research purposes only. Do not use anything from here to break the law. The instructor, the authors and Networkwalks are not responsible for what you do with this knowledge. Every action you take is your own responsibility. Misuse can lead to criminal charges, heavy fines, loss of your job and a permanent record. In most countries unauthorised access is a crime even when nothing is damaged.

## 2. Introduction

This report covers footprinting the **networkwalks.com** domain using six Kali Linux tools (W2-PM1) and scanning my own local network with **Zenmap** (W2-PM5). One module covers the footprinting phase and the other covers the scanning phase, so together they show how an attacker moves from gathering public information to mapping live hosts on a network. It is the Week 2 part of my internship program at Networkwalks.

The footprinting commands were run in Kali Linux. The network scan was run from my Windows PC with Zenmap installed, because the Kali virtual machine was attached to virtual networks and was not connected to my Wi-Fi network (see section 4.2, Task 2). Every step below includes the exact command used, the result I observed, a screenshot as evidence, and a short note on why the finding matters from an attacker's point of view.

Scope was respected throughout: only networkwalks.com and my own local network were tested. Only passive or light information-gathering and discovery activities were performed. No exploitation, brute-forcing, denial-of-service or password attacks were carried out.

MAC addresses visible in the Zenmap screenshots have been blacked out for privacy.

## 3. Tools Used

| Tool | Purpose |
|---|---|
| Kali Linux & Windows | Operating systems used for reconnaissance and scanning activities |
| whois | Find domain registration details (registrar, dates, name servers) |
| whatweb | Fingerprint web technologies (server, CMS, plugins, IP) |
| nslookup | Resolve the domain name to its IP address using DNS |
| curl -I | Read the HTTP response headers of the website |
| wafw00f | Detect whether a Web Application Firewall protects the site |
| dnsrecon | Enumerate DNS records (SOA, NS, MX, A, TXT, SRV) |
| Zenmap / Nmap | Scan the local subnet to find live hosts, open ports and services |
| Windows CMD (`ipconfig`) and Kali `ip a` / `ip route` | Identify local IP address, subnet and gateway |

## 4. Activities Performed

### 4.1 Footprinting & Reconnaissance (W2-PM1)

I performed reconnaissance against **networkwalks.com** using six Kali Linux tools. Each command was run with `2>&1 | tee <file>.txt` so the full output was saved as evidence while still being shown on screen.

#### Task 1: Domain registration details (whois)

**Command:** `whois networkwalks.com`

![Task 1 - whois](screenshots/task1-whois.png)

| Field | Result |
|---|---|
| Registrar | GoDaddy.com, LLC (IANA ID 146) |
| Created / Updated / Expires | 6 Nov 2019 / 12 Nov 2025 / 6 Nov 2027 |
| Name servers | NS6135.HOSTGATOR.COM, NS6136.HOSTGATOR.COM |
| DNSSEC | Unsigned |
| Domain status | clientDeleteProhibited, clientRenewProhibited, clientTransferProhibited, clientUpdateProhibited |

**Observation:** The domain is registered with GoDaddy but its DNS is handled by HostGator name servers, which suggests the site is hosted with HostGator. The four client lock statuses protect the domain against accidental or malicious deletion, transfer and update. DNSSEC is not enabled, so DNS answers for the domain are not cryptographically validated. No registrant personal details were visible in the captured output.

#### Task 2: Web technology fingerprinting (whatweb)

**Command:** `whatweb networkwalks.com`

![Task 2 - whatweb](screenshots/task2-whatweb.png)

| Item | Result |
|---|---|
| Redirect | `http://` returned 301 Moved Permanently to `https://`, which returned 200 OK |
| Web server | Apache |
| CMS | WordPress 7.1.4 |
| Plugin | WordPress Download Manager 3.3.76 |
| Libraries | jQuery 3.7.1, HTML5, Google Tag Manager |
| Server IP / country | 192.232.216.135 / United States |
| Other | Page title "Networkwalks Academy", public email info@networkwalks.com, cookie `__wpdm_client` (HttpOnly) |

**Observation:** The website is a WordPress site on Apache, and the WordPress and plugin versions are disclosed through the generator meta tag. Version information helps an attacker look up known issues for those exact versions. WhatWeb also reported Bootstrap as version 7.1.4, the same number as WordPress. Bootstrap has no such version, so this is probably a mis-detection, which shows that fingerprinting results need checking.

#### Task 3: DNS resolution (nslookup)

**Command:** `nslookup networkwalks.com`

![Task 3 - nslookup](screenshots/task3-nslookup.png)

**Result:** The domain resolved to **192.232.216.135** using Google's resolver (8.8.8.8). The answer was non-authoritative (from the resolver's cache, not from the domain's own name servers).

**Observation:** The IP matches the one WhatWeb reported, so the two tools agree on where the site is hosted. The IP is the actual server an attacker would look at next.

#### Task 4: HTTP response headers (curl)

**Command:** `curl -I https://networkwalks.com`

![Task 4 - curl](screenshots/task4-curl.png)

| Header | Value / observation |
|---|---|
| Status | HTTP/2 200 |
| `server` | Apache (no version number disclosed) |
| `set-cookie` | `__wpdm_client` with `secure; HttpOnly` (no `SameSite` attribute) |
| `referrer-policy` | no-referrer-when-downgrade |
| `permissions-policy` | Limits private-state-token features to Google, gstatic, reCAPTCHA, Cloudflare challenges and hCaptcha |
| `link` | Exposes the WordPress REST API at `/wp-json/` |
| `x-nginx-cache`, `x-endurance-cache-level` | Caching layer in front of Apache |

**Observation:** No `Strict-Transport-Security` (HSTS), `X-Frame-Options`, `X-Content-Type-Options` or `Content-Security-Policy` headers appeared in the response. The Apache version is hidden, which is good, and the cookie has the Secure and HttpOnly flags. The REST API endpoint confirms WordPress.

#### Task 5: Web Application Firewall detection (wafw00f)

**Command:** `wafw00f https://networkwalks.com`

![Task 5 - wafw00f](screenshots/task5-wafw00f.png)

**Result:** The site is behind **ModSecurity (SpiderLabs)**, detected with only 2 requests.

**Observation:** A WAF is present, which filters common attacks such as SQL injection and cross-site scripting before they reach WordPress. This is a positive finding, but it does not replace keeping the software patched.

#### Task 6: DNS enumeration (dnsrecon)

**Command:** `dnsrecon -d networkwalks.com`

![Task 6 - dnsrecon](screenshots/task6-dnsrecon.png)

| Record | Result |
|---|---|
| SOA | ns6135.hostgator.com (50.87.144.87) |
| NS | ns6135.hostgator.com (50.87.144.87), ns6136.hostgator.com (192.232.216.131) |
| A | networkwalks.com: 192.232.216.135 |
| MX | mail.networkwalks.com: 192.232.216.135 |
| TXT (SPF) | `v=spf1 +a +mx +ip4:50.87.144.87 +include:websitewelcome.com ~all` |
| TXT | A `google-site-verification` record |
| SRV | `_autodiscover._tcp` pointing to cpanelemaildiscovery.cpanel.net (port 443) |
| Software | Both name servers disclosed BIND version 9.16.23-RH |
| DNSSEC | No answer to the DNSSEC query |

**Observation:** The name servers match the whois result. The website and mail server share the same IP, and the SRV records point to cPanel email discovery, which confirms a shared cPanel hosting setup. The name servers reveal their software version, which is unnecessary information to expose. The SPF record ends in `~all` (softfail), which is weaker than `-all`. This scan did not check DKIM or DMARC records, so nothing is claimed about them. Only the standard scan was run (no brute-force option).

### 4.2 Network Scanning with Zenmap (W2-PM5)

#### Task 1: Install Zenmap

Zenmap 7.99 was already present on the Kali machine, and Zenmap with Nmap 7.993 was installed on the Windows PC from nmap.org.

![Zenmap version on Kali](screenshots/pm5-task1-install-kali.png)

![Zenmap installed on Windows](screenshots/pm5-task1-install-windows.png)

#### Task 2: Identify my IP address and subnet

**Commands:** `ipconfig` (Windows), `ip a` and `ip route` (Kali)

![ipconfig on Windows](screenshots/pm5-task2-ipconfig.png)

| Item | Result |
|---|---|
| Active adapter | Wireless LAN adapter Wi-Fi 2 |
| IPv4 address | 192.168.100.35 |
| Subnet mask | 255.255.255.0 (/24) |
| Default gateway | 192.168.100.1 |
| Network scanned | 192.168.100.0/24 (256 addresses) |

The Windows PC also has virtual host-only adapters (192.168.56.1 and 192.168.208.1) created for the virtual machine software. The Kali VM was attached to virtual networks as well (`eth0` 10.0.0.2/24 and `eth1` 192.168.208.30/24 host-only), so it was not on the Wi-Fi network. For this reason the live-network scan was run from the Windows host.

![ip a on Kali](screenshots/pm5-task2-kali-ip-a.png)

#### Task 3: Find live hosts (ping scan)

**Zenmap profile:** Ping scan. **Command:** `nmap -sn 192.168.100.0/24`

![Ping scan](screenshots/pm5-task3-pingscan.png)

**Result:** 256 IP addresses scanned, **6 hosts up**, in 18.93 seconds.

| IP address | MAC vendor | Note |
|---|---|---|
| 192.168.100.1 | Huawei Technologies | Default gateway (the router) |
| 192.168.100.5 | Shenzhen Bilian Electronic | |
| 192.168.100.22 | Intel Corporate | |
| 192.168.100.23 | Unknown | Likely a randomised MAC address (inference) |
| 192.168.100.29 | Hon Hai Precision Ind. | |
| 192.168.100.35 | none shown | The scanning PC itself |

**Observation:** Only 6 of 256 addresses responded. The vendor name comes from the first part of the MAC address and identifies the network chip maker, not necessarily the device type. Ping scans can miss hosts that ignore discovery probes, so 6 is the number of responding hosts.

#### Task 3b: Ports and services (quick scan)

**Zenmap profile:** Quick scan. **Command:** `nmap -T4 -F 192.168.100.0/24` (the 100 most common ports per host)

![Quick scan, part 1](screenshots/pm5-task3b-quickscan-1.png)

![Quick scan, part 2](screenshots/pm5-task3b-quickscan-2.png)

**Result:** 256 addresses scanned, 6 hosts up, in 29.39 seconds. (The Zenmap host list in the first screenshot also shows an extra entry, 192.168.100.39. It does not appear in the Nmap output and is not counted: the scan output states 6 hosts up.)

| Host | Result |
|---|---|
| 192.168.100.1 (router) | 23/tcp open telnet, 53/tcp open domain, 80/tcp open http; 21/tcp ftp and 22/tcp ssh filtered; 95 ports closed |
| 192.168.100.5 | All 100 scanned ports closed |
| 192.168.100.22 | 7070/tcp open (labelled realserver by Nmap); 99 ports filtered |
| 192.168.100.23 | All 100 scanned ports closed |
| 192.168.100.29 | All 100 scanned ports closed |
| 192.168.100.35 (my PC) | 135/tcp msrpc, 139/tcp netbios-ssn, 445/tcp microsoft-ds open; 97 ports closed |

**Observation:**
- The most notable finding is **Telnet (port 23) open on the router**. Telnet sends data, including logins, in clear text, so it should be disabled in favour of SSH or HTTPS.
- DNS (53) and HTTP (80) on the router are normal for a home router. Port 80 means the admin page is served without encryption.
- Three devices have no common ports open, which is good.
- On 192.168.100.22, "realserver" is only the name Nmap associates with port 7070. It does not confirm what is actually running, and no service-version scan was run against it.
- My own PC exposes standard Windows networking services (RPC, NetBIOS, SMB). These are normal for Windows but are a known attack surface on a local network.

#### Task 4: Network topology

![Zenmap topology](screenshots/pm5-task4-topology.png)

The topology shows all six hosts connected through the scanning machine (localhost) in a flat, single-subnet layout. The router (192.168.100.1) and my PC (192.168.100.35) are coloured yellow, which matches their three open ports each, and the other hosts are green. Padlock icons appear on 192.168.100.1 and 192.168.100.22, the two hosts that had filtered ports.

## 5. Risk Analysis / Impact

| # | Risk / Finding | Evidence / Observation | Potential Impact | Risk Level |
|---|---|---|---|---|
| 1 | CMS and plugin versions exposed | WhatWeb identified WordPress 7.1.4 and WordPress Download Manager 3.3.76 | Attackers can match exact versions to known issues and focus their effort | Medium |
| 2 | Missing HTTP security headers | curl response had no HSTS, X-Frame-Options, X-Content-Type-Options or CSP; cookie has no SameSite | Weaker browser-side protection against downgrade, clickjacking and content-type attacks | Medium |
| 3 | DNS infrastructure details exposed | dnsrecon showed name servers, mail server, SPF/SRV records and the BIND version 9.16.23-RH | Helps build an infrastructure profile; version banners aid targeted research | Medium |
| 4 | DNSSEC not enabled | whois showed DNSSEC unsigned; dnsrecon DNSSEC query got no answer | DNS responses cannot be cryptographically validated | Low |
| 5 | SPF policy is a softfail | SPF record ends in `~all` | Email spoofing of the domain is less strictly blocked | Low |
| 6 | Server IP and REST API endpoint identifiable | nslookup resolved 192.232.216.135; `/wp-json/` advertised in headers | Shows where the service runs and confirms WordPress | Low |
| 7 | WAF technology identifiable | wafw00f identified ModSecurity (SpiderLabs) | Reveals part of the security architecture, but the WAF itself is a positive control | Low |
| 8 | Telnet open on the home router | Quick scan: 23/tcp open on 192.168.100.1 | Clear-text administration protocol could expose router logins to anyone on the network | Medium |
| 9 | Windows file-sharing services reachable on the LAN | Quick scan: 135, 139 and 445 open on 192.168.100.35 | Normal for Windows, but a common attack surface if the network is untrusted | Low |
| 10 | Unidentified open port on a LAN device | 7070/tcp open on 192.168.100.22 | Unknown service should be identified by its owner | Low |

Risk level key: Critical, Medium, Low

The risks above are observations from the footprinting and scanning exercises, not confirmed vulnerabilities. The practical exercises involved only information gathering and host discovery. No exploitation or vulnerability validation was performed. The presence of a software version, IP address, DNS record or open port does not by itself mean the system is vulnerable. Further authorized security testing would be required to confirm any actual vulnerability.

## 6. Recommendations

**Review publicly exposed technology information**
Remove the WordPress generator tag and review what versions and endpoints (such as `/wp-json/`) are publicly visible.

**Keep software updated**
Keep WordPress core, themes and plugins (including WordPress Download Manager) updated and reviewed against current security advisories.

**Add HTTP security headers**
Add `Strict-Transport-Security`, `X-Frame-Options`, `X-Content-Type-Options` and a `Content-Security-Policy`, use a stricter `Referrer-Policy`, and add `SameSite` to cookies.

**Harden DNS and email records**
Enable DNSSEC, hide the BIND version banner, and move SPF from `~all` to `-all` once all legitimate senders are listed. Review DKIM and DMARC as a follow-up check.

**Properly configure and monitor the WAF**
Keep ModSecurity enabled with an up-to-date rule set and review its logs, while still patching the underlying software.

**Disable Telnet on the router**
Use SSH or HTTPS for router administration, change default admin credentials, turn off remote management and keep router firmware updated.

**Review services on local devices**
Close services that are not needed, identify what is listening on port 7070 of 192.168.100.22, set the Windows network profile to Private and keep the Windows firewall enabled.

**Perform regular internal network discovery and investigate unknown devices**
Periodically scan your own network, keep a device inventory, and investigate any unexpected device.

**Perform security testing with authorization**
Reconnaissance and scanning should only be performed against systems and networks where appropriate authorization has been provided.

## 7. Conclusion

During Week 2 of my Cybersecurity & Ethical Hacking internship, I completed practical activities covering footprinting, reconnaissance and network scanning.

In the footprinting activity, I used six Kali Linux tools to collect information about networkwalks.com. The domain is a WordPress site on Apache, hosted on shared cPanel/HostGator infrastructure at 192.232.216.135 and protected by a ModSecurity WAF. The main observations were version disclosure, missing HTTP security headers, no DNSSEC and a softfail SPF policy.

In the network scanning activity, I used Zenmap to identify my local network configuration (192.168.100.0/24), discover 6 live hosts, check their common ports and view the network topology. The main observation was Telnet open on the router, along with standard Windows networking services on my own PC.

The exercises showed me that a security professional can learn a significant amount about an environment from public information and network responses before attempting anything intrusive. I also learned that findings should be documented clearly: what was done, what was found, what it means, what risk it may create and how to reduce it. Finally, I learned that reconnaissance and scanning must always stay within an authorized scope. These activities were completed as part of the assigned educational cybersecurity lab.

## 8. Evidences Collected

Screenshots collected as evidence during the activities (stored in the `screenshots/` folder):

- `task1-whois.png`
- `task2-whatweb.png`
- `task3-nslookup.png`
- `task4-curl.png`
- `task5-wafw00f.png`
- `task6-dnsrecon.png`
- `pm5-task1-install-kali.png`, `pm5-task1-install-windows.png`
- `pm5-task2-ipconfig.png`, `pm5-task2-kali-ip-a.png`
- `pm5-task3-pingscan.png`
- `pm5-task3b-quickscan-1.png`, `pm5-task3b-quickscan-2.png`
- `pm5-task4-topology.png`

Raw command output saved with `tee` is stored in the `outputs/` folder (`task1-whois.txt` to `task6-dnsrecon.txt`).

-End-