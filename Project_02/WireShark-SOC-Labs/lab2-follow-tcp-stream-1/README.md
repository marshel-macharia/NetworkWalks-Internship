# Wireshark SOC Lab: Follow TCP Stream 1

**NetworkWalks Internship | Week 1 | SOC Project Module: Wireshark (Extra) | Lab 2 of 6**
**Result: 100/100 (all 10 tasks correct)**

## Summary

A SOC analyst scenario in which an office chat tool was running on the network next to normal web traffic. Using Wireshark, I isolated the chat connection, rebuilt the conversation with **Follow TCP Stream**, and read what was sent. The chat ran in **plain text**, so the full conversation, including a temporary account password and a guest Wi-Fi password, was readable by anyone who could capture the traffic.

## Tools and skills

- Wireshark (display filters, Follow TCP Stream, packet list and status bar)
- TCP basics: three-way handshake, client vs server, ephemeral vs service ports
- Reading reassembled conversations and identifying exposed sensitive data

## Method

### 1. Remove the web noise

Most of the capture was web traffic on ports 80 and 443. I filtered it out to leave only the other TCP traffic:

```
tcp && !(tcp.port == 80 || tcp.port == 443)
```

| Part | Meaning |
|---|---|
| `tcp` | Show TCP packets only |
| `&&` | Logical AND |
| `!( ... )` | Logical NOT, so the packets matching the brackets are hidden |
| `tcp.port == 80 \|\| tcp.port == 443` | Either side of the packet uses port 80 (HTTP) or 443 (HTTPS) |

`tcp.port` matches both source and destination, so requests and replies are caught together. Wrapping the whole condition in `!( )` is more reliable than `tcp.port != 80`. The filter reduced the view from **246 packets to 24** (9.8%).

### 2. Identify the chat and who started it

Packets 81 to 83 are a TCP three-way handshake:

| Packet | Info | Meaning |
|---|---|---|
| 81 | `49731 → 5000 [SYN]` | `192.168.10.45` starts the connection |
| 82 | `5000 → 49731 [SYN, ACK]` | `192.168.10.20` accepts |
| 83 | `49731 → 5000 [ACK]` | Connection established |

- The computer that sends the first SYN is the **client**: `192.168.10.45`.
- The chat uses TCP **port 5000** on the server (`192.168.10.20`).
- **49731** is the client's ephemeral port, chosen by the operating system for this single connection. It is not the service port.

### 3. Follow the stream

Right-click a data packet (for example packet 84, which carries payload), then **Follow → TCP Stream**. Wireshark reassembles the connection and applies the filter `tcp.stream eq 5`, so this chat is **stream 5**. Wireshark numbers each TCP connection from 0 in the order it first appears.

The window shows red text for the client and blue text for the server. The footer reports the entire conversation as **413 bytes** (4 client packets, 5 server packets, 9 turns).

## Findings

| Task | Answer |
|---|---|
| Chat port | 5000 |
| IP that started the chat | 192.168.10.45 (client, sends the first SYN) |
| Stream number | 5 |
| Size of the conversation | 413 bytes |
| Staff member asking for help | Ali (Accounts) |
| Drive that could not be opened | Finance |
| Temporary password given by IT | `Tiger#4821` |
| Guest Wi-Fi password | `BlueHarbor77` |
| Colour of client text | Red |

The conversation: Ali from Accounts cannot open the Finance shared drive. Sara from IT resets his access and sends a temporary password in the chat. Ali then asks for the guest Wi-Fi password, and Sara sends that too, followed by a ticket number (`HD-20931`).

Note: the server sent the first data packet (the "Welcome to Office Chat v1.2" banner), so the stream begins with a blue line even though the client opened the connection.

## Security takeaways

- **Cleartext chat:** the application sends messages unencrypted on port 5000, so anyone on the network path can read them with a packet capture.
- **Credentials shared in chat:** a temporary account password and the guest Wi-Fi password were sent in the open. Both should be treated as compromised.
- **Recommendations:**
  - Use a chat tool with TLS encryption.
  - Share credentials through a password manager or a self-service reset link, not in chat.
  - Force a password change at first login for temporary passwords.
  - Rotate the guest Wi-Fi password regularly.
  - Monitor for unexpected plain-text protocols on non-standard ports.

## Result

All 10 tasks were answered correctly for a score of **100/100**.