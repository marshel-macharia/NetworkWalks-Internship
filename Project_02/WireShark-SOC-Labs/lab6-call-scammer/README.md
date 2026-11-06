# Wireshark SOC Lab 8: VoIP Call Replay

**Intern:** Marshal Mburu Macharia | **Programme:** NetworkWalks Internship | **Tool:** Wireshark (Windows) | **Score:** 100/100

## Objective

Analyse a packet capture containing a VoIP call, reconstruct the call from its SIP signalling and RTP media streams, replay the audio, and identify the security risk of unencrypted voice traffic.

## Scenario

A phone call between two internal extensions was captured on the network. During the call, the caller read out payment card details to settle an invoice. The task was to establish who called whom, how the call was set up, how the voice was carried, and what an eavesdropper could recover.

## Methodology and Findings

| # | Task | Method | Finding |
|---|------|--------|---------|
| 1 | Capture the traffic | Opened the capture and confirmed SIP and RTP traffic | 1051 packets, 804 of them RTP |
| 2 | Call setup protocol | Filter `sip` and inspected INVITE, 180 Ringing, 200 OK | **SIP** |
| 3 | Voice protocol | Filter `rtp`; small packets at roughly 60 ms intervals | **RTP** |
| 4 | Caller extension | Filter `sip.Method == "INVITE"`, then the From header | **101** (192.168.10.45) |
| 5 | Caller name | From header display name | **Ali Raza** |
| 6 | Codec | RTP payload type | **G.711 PCMU** (payload type 0) |
| 7 | Who answered | Telephony > VoIP Calls > Play Streams, first words | **Sara**, extension 202 (192.168.10.52) |
| 8 | Card number | Replayed the call audio | Last four digits **2394** (full number masked) |
| 9 | Expiry date | Replayed the call audio | **09/28** |
| 10 | The fix | Assessment of the exposure | Encrypt the call (SRTP for media, SIP over TLS for signalling) |

## Key Filters

```
sip
sip.Method == "INVITE"
rtp
```

Calls were replayed through **Telephony > VoIP Calls > Play Streams**.

## Call Summary

- **Caller:** Ali Raza, extension 101, 192.168.10.45
- **Receiver:** Sara, extension 202, 192.168.10.52
- **Signalling:** SIP over UDP port 5060
- **Media:** RTP, G.711 PCMU, 64 kbps, 534-byte packets, one stream in each direction (ports 16384 and 18000)

## Security Analysis

SIP signalling and RTP media were sent in clear text, so anyone able to capture traffic on the network path could rebuild the call and play it back. In this case that exposed sensitive payment card details (card number and expiry date) spoken aloud.

## Recommendations

1. **Encrypt the media** with SRTP so captured RTP cannot be decoded into audio.
2. **Encrypt the signalling** with SIP over TLS (SIPS) to protect call metadata such as extensions, names and Call-IDs.
3. **Segment voice traffic** onto a dedicated VLAN and restrict who can capture or mirror it.
4. **Never read card data aloud over the phone.** Use a PCI DSS compliant payment method, such as a secure payment link or an IVR that pauses recording and masks the input.
5. **Monitor** for unexpected packet capture tools or port mirroring on voice segments.

## Conclusion

The call was fully reconstructed from the capture, and sensitive financial data was recovered with standard Wireshark features. This shows why voice traffic that carries confidential information must be encrypted in transit.