# Wireshark SOC Lab: Identify the Attacker's IP

**Program:** NetworkWalks Internship | **Project:** 2 (Lab 3 on the platform) | **Tool:** Wireshark | **Result:** 100 / 100

## 1. Objective

A staff member reported unusual traffic to the company web server. As the SOC analyst, I captured and analysed the packets in Wireshark to determine **what type of attack occurred** and **who the attacker is**.

## 2. Summary of Findings

| Item | Finding |
|---|---|
| Attack type | TCP SYN port scan (reconnaissance) |
| Attacker IP | **192.168.10.66** (MAC `00:0c:29:66:66:66`) |
| Target IP | **192.168.10.20** (web server, MAC `00:0c:29:a1:20:20`) |
| Attacker source port | **52311** (the same on every probe) |
| Ports probed | **40** distinct destination ports |
| Open ports found | **22 (SSH), 80 (HTTP), 443 (HTTPS)** |
| Closed ports | 37 (answered with `RST, ACK`) |
| Scan duration | about 0.49 s (3.400 s to 3.8875 s in the capture) |
| MITRE ATT&CK | T1046, Network Service Discovery (Discovery) |
| Capture size | 309 packets |

## 3. Method and Evidence

### Task 2: Show only the packets that start a TCP connection

Filter: `tcp.flags.syn == 1` -> **Displayed: 73** of 309 packets.

This count covers every packet with the SYN flag set, which is two kinds of packet:

- **55 initial SYNs** that start a connection (40 from the attacker, 15 from normal hosts).
- **18 SYN-ACK replies** from the server (3 to the attacker's probes on open ports, 15 to the normal hosts).

Adding `&& tcp.flags.ack == 0` isolates only the 55 connection-initiating packets, which is the form used in the later tasks to count the attacker's probes.

### Tasks 3-4: Identify the attacker and the target

Statistics -> Conversations -> TCP shows one host, **192.168.10.66**, opening many short conversations with **192.168.10.20**, while the other hosts (192.168.10.31, .32, .45, .52, .58) each hold one normal conversation on port 80 or 443.

### Task 5: Constant source port

In the Info column every attacker packet starts with `52311 ->`, while the destination port changes in each packet (21, 25, 80, 110, 139, 3389, 5900 ...). One fixed source port with many destination ports is typical of an automated scanner.

### Task 6: How many ports were probed?

The TCP Conversations list has 55 conversations in total, 15 of them normal traffic.

Filter: `ip.src == 192.168.10.66 && tcp.flags.syn == 1 && tcp.flags.ack == 0` -> **Displayed: 40**. Each SYN targets a different port, so **40 ports** were probed.

Supporting check: adding `&& tcp.dstport == 1` returns a single packet (the last probe, at 3.8875 s), confirming that port 1 is already one of the 40.

In the Conversations list the last attacker row is port 1, followed only by normal traffic, so the scan ends there.

### Task 7: Open ports

Filter: `ip.src == 192.168.10.20 && tcp.flags.syn == 1 && tcp.flags.ack == 1 && ip.dst == 192.168.10.66` -> **Displayed: 3**. An open port answers a SYN with a SYN-ACK. The open ports are **22, 80 and 443**.

### Tasks 8-9: Closed ports

Filter: `ip.src == 192.168.10.20 && ip.dst == 192.168.10.66 && tcp.flags.reset == 1` -> **Displayed: 37**. A closed port replies with **`RST, ACK`**.

Cross-check: 3 open + 37 closed = 40 probes, which matches Task 6.

| Port state | Server reply | Attacker follow-up |
|---|---|---|
| Open | `SYN, ACK` | `RST` (handshake is never completed) |
| Closed | `RST, ACK` | none |

### Task 10: Attack type

**Port scan.** The other options do not fit. A SYN flood would send thousands of SYNs to exhaust the server, a ping sweep uses ICMP across many hosts, and a brute force attack repeats logins against one service. Here a single host sent 40 SYNs to 40 different ports on one target.

## 4. Analysis

- **Technique:** the attacker sent a SYN to each port and, on a SYN-ACK, answered with a RST without finishing the handshake. This half-open pattern is the behaviour of a **SYN (stealth) scan**, the default scan type of tools such as Nmap (`-sS`).
- **Speed:** 40 probes in about 0.49 s (one every 12.5 ms) means an automated tool, not manual probing.
- **Impact:** no exploitation or data transfer was seen in this capture. The attacker did learn which services are exposed, and SSH (22) is the most likely next target for brute-force attempts.

## 5. Recommendations

1. Block **192.168.10.66** at the firewall and investigate the machine it came from.
2. Alert on many SYNs from one source to many ports in a short time.
3. Restrict SSH (22) to trusted IP ranges or a VPN, and use key-based authentication.
4. Review web server and authentication logs for activity from the same source after the scan.
5. Close or filter ports that do not need to be reachable.

## 6. Wireshark Filters Used

| Purpose | Filter |
|---|---|
| All SYN-flagged packets | `tcp.flags.syn == 1` |
| Connection-initiating SYNs only | `tcp.flags.syn == 1 && tcp.flags.ack == 0` |
| Attacker's SYN probes | `ip.src == 192.168.10.66 && tcp.flags.syn == 1 && tcp.flags.ack == 0` |
| Open-port replies (SYN-ACK) | `ip.src == 192.168.10.20 && tcp.flags.syn == 1 && tcp.flags.ack == 1 && ip.dst == 192.168.10.66` |
| Closed-port replies (RST) | `ip.src == 192.168.10.20 && ip.dst == 192.168.10.66 && tcp.flags.reset == 1` |

## 7. Result

Scored **100 / 100**. All 10 tasks were answered correctly: capture the traffic, SYN filter (73), attacker IP (192.168.10.66), target IP (192.168.10.20), source port (52311), ports probed (40), open ports (22, 80, 443), closed-port reply (`RST, ACK`), closed ports (37), and attack type (port scan).

## 8. What I Learned

- How a TCP handshake looks in a capture, and how SYN, SYN-ACK and RST-ACK separate open from closed ports.
- How to use display filters and Statistics -> Conversations to find an attacker's pattern in a busy capture.
- Match the filter to the question's exact wording. `tcp.flags.syn == 1` counts SYN and SYN-ACK packets (73), while adding `tcp.flags.ack == 0` counts only connection starts (55).