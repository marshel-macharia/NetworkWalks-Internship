# Wireshark SOC Lab: Follow TCP Stream, Part 2

**Platform:** NetworkWalks Internship, Week 1, SOC Project Module: Wireshark (Extra)
**Tool:** Wireshark on Windows
**Result:** 100 / 100 (10 of 10 tasks correct, first attempt)

## Objective

A staff member browsed five pages on a company intranet. An administrator reported that one page contained a hidden note. The task was to analyse the packet capture, identify the page and the stream that carried it, recover the hidden information, and trace a plain text file the same user downloaded.

## Summary of Findings

| Item | Finding |
|---|---|
| Client (staff member) | 192.168.10.45 |
| Intranet server | 192.168.10.20 (`intranet.company.local`, Apache/2.4.57 on Ubuntu) |
| Page containing the hidden note | `/staff.html` |
| Stream carrying that page | TCP stream 7 |
| Where the note is hidden | Inside an HTML comment (`<!-- ... -->`) |
| Access code | Falcon-9921 |
| Backup server named in the note | 192.168.10.77 |
| Direction of the note | Server to client (blue text in Follow TCP Stream) |
| Status code of the response | 200 OK |
| Downloaded plain text file | `/download/report.txt` (TCP stream 11) |
| Report prepared by | Nadia |

## Methodology

### 1. Identify the pages the staff member requested

```
http.request.method == "GET"
```

This lists every HTTP request in the capture. The requests from 192.168.10.45 were `/help.html`, `/index.html`, `/about.html`, `/staff.html` and `/contact.html`. Requests from other hosts (`/`, `/events.html`, `/news.html`) were unrelated traffic and were excluded.

### 2. Locate the hidden note

```
ip.addr == 192.168.10.45 && http.file_data contains "<!--"
```

- `ip.addr == 192.168.10.45` restricts the view to the staff member's traffic.
- `http.file_data` is the HTTP body (the HTML itself), so header text cannot cause false matches.
- `<!--` opens an HTML comment. Browsers do not display comments, but they are still delivered to the client, so they appear in a capture.

Exactly one packet matched: frame 112, a `200 OK (text/html)` response from the server to the client.

> A note on syntax: newer Wireshark versions reject `http contains "..."` because `http` is a protocol rather than a field. Using `frame contains "..."` or `http.file_data contains "..."` avoids that error.

### 3. Read the full conversation

Right-click the matching packet, then **Follow, TCP Stream**. The stream (`tcp.stream eq 7`) showed:

- The request (red, client to server): `GET /staff.html HTTP/1.1`
- The response (blue, server to client): `HTTP/1.1 200 OK`, `Content-Type: text/html`, followed by the page body

The body contained the visible text "Staff directory" and "Contact IT for changes", followed by the hidden comment:

> IT note: backup server is 192.168.10.77, access code Falcon-9921. Remove before publishing.

In Follow TCP Stream, red text is client to server and blue text is server to client. The note therefore travelled from the server to the client.

### 4. Find the downloaded plain text file

```
ip.addr == 192.168.10.45 && http.request.uri contains "/download/"
```

The matching request was `GET /download/report.txt HTTP/1.1`. Following that stream (`tcp.stream eq 11`) showed a `Content-Type: text/plain` response containing:

```
Quarterly report
Total sales: 48200
Prepared by: Nadia
Status: Final
```

## Security Observations

1. **Plain HTTP exposes everything.** Both the staff page and the report were served over unencrypted HTTP. Anyone with access to the network path can read the full request and response, including page content and file contents.
2. **Sensitive data left in page source.** An internal server address and an access code were stored in an HTML comment. Comments are invisible in a browser but fully visible on the wire and in "view source". The note itself said it should have been removed before publishing.
3. **Internal business data in transit.** The quarterly report (sales figures, author, status) was transferred in clear text.

## Recommendations

- Serve all intranet content over HTTPS (TLS) to prevent passive interception.
- Never store credentials, access codes or internal infrastructure details in page source, comments or any client-delivered content.
- Add a pre-publication review or automated check that strips developer comments and scans for secrets.
- Rotate the exposed access code and review access to the backup server at 192.168.10.77.
- Restrict access to the `/download/` folder and protect sensitive documents with authentication.

## Skills Demonstrated

- Applying display filters (`http.request.method`, `ip.addr`, `http.file_data`, `http.request.uri`, `frame contains`)
- Using Follow TCP Stream and interpreting its colour coding for traffic direction
- Correlating requests with responses across packets and streams
- Recognising sensitive data exposure in unencrypted traffic
- Writing up findings with evidence and remediation steps

## Lab Task Results

| # | Task | Result |
|---|---|---|
| 1 | Capture the traffic | Correct |
| 2 | Find the pages | Correct |
| 3 | Where is it hidden | Correct |
| 4 | Find the stream | Correct |
| 5 | The code word | Correct |
| 6 | The backup server | Correct |
| 7 | Which direction | Correct |
| 8 | Status code | Correct |
| 9 | The downloaded file | Correct |
| 10 | Who made the report | Correct |