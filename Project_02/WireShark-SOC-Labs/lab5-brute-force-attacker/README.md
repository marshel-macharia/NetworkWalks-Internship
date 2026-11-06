# Wireshark SOC Lab: Catch the Brute Force Attacker

**Programme:** NetworkWalks Internship, Week 1, SOC Project Module: Wireshark (Extra)
**Lab:** Brute Force Catch (lab page label: Lab 7, 10 tasks)
**Tool:** Wireshark (Windows)
**Result:** 100 / 100

## Objective

A server owner reported many failed sign-ins. The goal was to analyse the packet capture, identify the protocol under attack, the target, the attacker and the account being guessed, then work out how many attempts failed and which password finally worked.

## Method

| Step | Wireshark action | Purpose |
|---|---|---|
| 1 | Statistics > Protocol Hierarchy | Find which application protocol dominates the capture |
| 2 | Display filter `ftp` | Isolate FTP commands and replies, and see who sends each |
| 3 | Display filter `ftp.response.code == 530` | List every server reply meaning "Login incorrect" and read the count in the status bar |
| 4 | Right-click a packet > Follow > TCP Stream | Read the full login conversation in order, ending with the successful login |

## Findings

| Question | Answer | Evidence |
|---|---|---|
| Protocol under attack | FTP | 117 of 369 packets (31.7%), the largest application-layer share |
| Target (FTP server) | 192.168.10.20 | Sends the `220 FTP server ready` banner and all numbered replies |
| Attacker | 192.168.10.66 | Sends every `USER` and `PASS` request |
| Username targeted | `admin` | Same `USER admin` repeated before every password guess |
| Failure code | 530 (`Login incorrect`) | Server reply after each wrong password |
| Failed logins | 24 | 24 packets displayed for `ftp.response.code == 530` |
| Success code | 230 (`Login successful`) | Final login reply in the stream |
| Password that worked | `Sunshine99` | The `PASS` line immediately before the 230 |
| Attack type | Brute force (password guessing) | One source, one account, many different passwords |

## Attack Timeline

1. The attacker connects to the FTP service on 192.168.10.20 and receives the `220` banner.
2. For each attempt the attacker sends `USER admin`, gets `331 Please specify the password`, then sends a `PASS` guess.
3. The server answers `530 Login incorrect` to each wrong guess. Guesses seen include `123456`, `password`, `Winter2025`, `Company1` and `Admin@123`, a mix of common passwords and predictable patterns.
4. After 24 failures, `PASS Sunshine99` returns `230 Login successful`.
5. The attacker runs `PWD`, which returns `"/" is the current directory`, then sends `QUIT`. This looks like a quick check of the access that was gained.

## Indicators of Compromise

- Source IP: `192.168.10.66`
- Targeted service and host: FTP on `192.168.10.20`
- Targeted account: `admin`, which is compromised (password `Sunshine99` is valid)
- Pattern: 24 consecutive `530` replies followed by a `230`

## Conclusion

The server was hit by an FTP brute force attack from 192.168.10.66 against the `admin` account. The attack succeeded on the 25th attempt, so the account should be treated as compromised.

## Recommendations

1. **Respond:** reset the `admin` password, review the account's activity and files, and block 192.168.10.66 at the firewall.
2. **Limit guessing:** enable account lockout or delay after repeated failures, and add rate limiting or a tool such as Fail2ban.
3. **Harden authentication:** enforce strong, non-dictionary passwords and prefer key-based or multi-factor access.
4. **Replace plain FTP:** move to SFTP or FTPS, because FTP sends usernames and passwords unencrypted.
5. **Detect:** alert in the SIEM on a burst of FTP `530` responses from a single source, and especially on a `230` that follows them.

## Skills Demonstrated

Protocol hierarchy analysis, Wireshark display filters, FTP response code interpretation, TCP stream reconstruction, brute force detection and incident write-up.