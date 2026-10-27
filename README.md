<h1 align="center">BREAKVEIL</h1>
<p align="center"><b>Transparent MITM interception framework — HTTP · HTTPS · FTP · multi-protocol</b></p>
<p align="center">by <b>Syed Sadam Hussain Shah</b></p>

<p align="center">
  <img src="https://img.shields.io/badge/version-1.0-0a84ff" alt="version">
  <img src="https://img.shields.io/badge/platform-Linux-111" alt="platform">
  <img src="https://img.shields.io/badge/license-Proprietary-e63946" alt="license">
  <img src="https://img.shields.io/badge/status-active-51cf66" alt="status">
</p>

---

## What it is

**BreakVeil** is a transparent man-in-the-middle interception framework built for
**authorized** network penetration testing. It positions itself between a target device
and its gateway — via ARP spoofing, rogue gateway, or explicit proxy configuration —
and fully decrypts, inspects, logs, and replays all cleartext and TLS-protected
application-layer traffic.

Unlike single-protocol tools, BreakVeil handles the full stack of protocols commonly
encountered in enterprise networks in a single session: HTTPS, HTTP, FTP, SMTP, POP3,
IMAP, and generic TCP streams — with a unified credential and token harvesting pipeline
feeding into one structured report.

> **Authorized use only.** BreakVeil is licensed for use on networks and devices you own
> or have written, explicit permission to assess. MITM interception of unauthorized
> traffic is illegal in most jurisdictions.

---

## Capabilities

### Transport interception

| Protocol | Method | What is extracted |
|---|---|---|
| **HTTP** | Transparent proxy | Full request/response, POST bodies, form fields, cookies, auth headers |
| **HTTPS** | Dynamic TLS MITM (custom CA) | Everything above — full plaintext after decryption |
| **FTP** | Control-channel parse | `USER`, `PASS` commands; file transfer metadata |
| **SMTP** | Command parse | `AUTH LOGIN/PLAIN`, `RCPT TO`, subject line, attachments |
| **POP3 / IMAP** | Command parse | `USER`, `PASS`, `LOGIN`, `AUTHENTICATE PLAIN` |
| **Generic TCP** | Raw stream capture | Banner, plaintext credentials in any custom protocol |

### HTTPS interception (TLS MITM)

BreakVeil intercepts HTTPS by acting as a **transparent TLS terminator**:

1. ARP-MITM or rogue gateway positions the attacker inline.
2. When the target opens a TLS connection, BreakVeil terminates it and presents a
   **on-the-fly signed certificate** for the exact hostname — signed by BreakVeil's
   built-in CA.
3. A second TLS session is opened upstream to the real server; responses are relayed
   back transparently.
4. Both request and response streams are fully decrypted and logged before forwarding.

This technique is effective on any HTTPS target **unless** the browser has a valid
HSTS preload entry (or a previously pinned HSTS header) for that domain. HSTS causes
the browser to reject the substituted certificate outright — this is expected browser
security behaviour and cannot be bypassed without a trusted CA installed on the victim
device (which is out of scope for most assessments).

> **HSTS note:** BreakVeil does not attempt to strip HSTS headers on the initial
> connection — that window has already closed if the victim visited the site before.
> For assessments targeting HSTS-protected sites, install the BreakVeil CA on the
> target device through your authorized test methodology (MDM, GPO, manual install).

### Credential and token harvesting

BreakVeil recognises and extracts the following from intercepted traffic:

- **Form credentials** — `username`, `password`, `email`, and 20+ common field names
  in `application/x-www-form-urlencoded` and `multipart/form-data` POST bodies
- **JSON credentials** — credential keys in `application/json` POST bodies (login APIs,
  mobile apps)
- **HTTP Basic auth** — decoded directly from the `Authorization: Basic …` header
- **Bearer tokens** — extracted from `Authorization: Bearer …` and `X-Auth-Token` headers
- **Session cookies** — all `Set-Cookie` values, flagged by `HttpOnly` / `Secure` presence
- **OAuth tokens** — access tokens and refresh tokens in redirect URLs and JSON responses
- **API keys** — common header names (`X-API-Key`, `api_key`, `apikey`, `key`) and
  high-entropy strings in URL query parameters
- **FTP / SMTP / POP3 / IMAP credentials** — extracted from protocol `USER` / `PASS` /
  `AUTH` command sequences
- **JWT tokens** — captured and decoded in-session; optionally fed directly into
  **jwtforge** for attack chain continuation

All extracted items are tagged with timestamp, source IP, destination IP, protocol,
and confidence level, and written to a structured JSON harvest log alongside the session
PCAP.

### Traffic capture

- Full raw PCAP export (`.pcap`) — compatible with Wireshark, tshark, Zeek
- Per-session plaintext HTTP log (request line + headers + body, one file per host)
- TLS secrets log (NSS / SSLKEYLOGFILE format) for third-party decryption in Wireshark
- Streaming live view of requests / responses in the terminal UI

### Injection and replay

- **Request replay** — resend any captured request with a modified token or body
- **Response tamper** — rewrite any response field or inject a JavaScript payload before
  forwarding to the client
- **Cookie injection** — set an arbitrary `Set-Cookie` in a response to the victim
- **Redirect injection** — 302-redirect the victim to an attacker-controlled URL

### Reporting

At session end, BreakVeil produces:

| Output | Description |
|---|---|
| `harvest.json` | Structured credential + token log (JSON) |
| `session.pcap` | Raw packet capture |
| `http_log/` | Per-host plaintext request/response logs |
| `tls_keys.log` | NSS key log for Wireshark decryption |
| `report.html` | Prioritized HTML report — credentials, tokens, session cookies, high-value findings |

---

## Architecture overview

```
  ┌─────────────────────────────────────────────────────┐
  │                    BREAKVEIL CORE                    │
  │                                                     │
  │  ┌──────────┐   ┌───────────────┐   ┌───────────┐  │
  │  │  ARP /   │──▶│  Transparent  │──▶│ Protocol  │  │
  │  │ Redirect │   │  TCP Proxy    │   │ Dispatcher│  │
  │  └──────────┘   └───────────────┘   └─────┬─────┘  │
  │                                           │        │
  │              ┌────────────────────────────┤        │
  │              │            │               │        │
  │        ┌─────▼──┐  ┌──────▼──┐  ┌────────▼──┐     │
  │        │ HTTP   │  │ TLS     │  │ FTP/SMTP  │     │
  │        │ Parser │  │ MITM    │  │ POP3/IMAP │     │
  │        └─────┬──┘  └──┬─────┘  └─────┬─────┘     │
  │              └─────────┴──────────────┘           │
  │                        │                          │
  │               ┌─────────▼─────────┐               │
  │               │  Harvest Pipeline  │               │
  │               │  credentials       │               │
  │               │  tokens / cookies  │               │
  │               │  API keys / JWTs   │               │
  │               └─────────┬─────────┘               │
  │                         │                         │
  │               ┌─────────▼─────────┐               │
  │               │   PCAP + Logs +   │               │
  │               │   HTML Report     │               │
  │               └───────────────────┘               │
  └─────────────────────────────────────────────────────┘
```

---

## Limitations

| Constraint | Reason |
|---|---|
| **HSTS preloaded domains** | Browser rejects the substituted certificate; cannot be bypassed without a trusted CA on the device |
| **Certificate pinning** | Mobile apps / thick clients that pin their leaf or CA cert will abort the connection |
| **TLS 1.3 0-RTT** | Early data may bypass the proxy before the intercept socket is established |
| **QUIC / HTTP/3** | UDP-based; BreakVeil currently intercepts TCP only |
| **Encrypted DNS (DoH/DoT)** | DNS queries tunnelled over TLS/HTTPS to resolvers bypass the DNS interception layer |

---

## Intended use

BreakVeil is designed for:

- **Internal network penetration tests** — validate whether credentials and session tokens
  are exposed to a network-level attacker
- **Wireless assessments** — combined with a rogue AP, assess what a café/hotel attacker
  can extract from connected devices
- **Mobile app assessments** — intercept app traffic after installing the BreakVeil CA via
  MDM or manual configuration
- **Thick-client assessments** — capture non-browser TLS traffic from desktop applications

---

## Source availability

Source code is not publicly released. For access, licensing inquiries, or enterprise
deployment, contact:

**Syed Sadam Hussain Shah** — [LinkedIn](https://qa.linkedin.com/in/syed-sadam-h-950a62336)

---

## License

Proprietary — all rights reserved. Unauthorized copying, distribution, or use is prohibited.

**Author:** Syed Sadam Hussain Shah — [LinkedIn](https://qa.linkedin.com/in/syed-sadam-h-950a62336)
