# Orsay transport layer — SSL 3.0 clients, PKI, host inventory

Reference for everything *below* the openapi HTTP: the TV's TLS
stacks, their certificate validation, Samsung's 2012 PKI, and the
complete host inventory as measured 2026-10-21…23 on a UE40EH5300
(T-MST10PDEUC-1007.0). Companion: [openapi.md](openapi.md).

## 1. Two SSL 3.0 clients

The firmware contains (at least) two independent SSL 3.0 client
stacks, distinguished by cipher list and failure behavior
[confirmed]:

| | hub/infolink client | fkp client |
|---|---|---|
| used for | `infolink.pavv.co.kr`, `sso.internetat.tv` | `fkp.samsungcloudsolution.com` |
| cipher suites | 11 (below) | 5 |
| preferred (server honours client order) | `0x002f` RSA-AES128-SHA | `0x0005` RSA-RC4-128-SHA |
| session id | none | 0x0B bytes offered |
| on bad cert | closes silently ~440 ms after handshake | proper SSL alert |
| protocol | bare SSL 3.0 (RFC 6101), no SNI, **no extensions at all** | same |

infolink ClientHello cipher list, TV's order [confirmed]: `0033`
DHE-RSA-AES128-SHA, `0039` DHE-RSA-AES256-SHA, `0016`
DHE-RSA-3DES-SHA, `0032` DHE-DSS-AES128-SHA, `0038`
DHE-DSS-AES256-SHA, `0013` DHE-DSS-3DES-SHA, `002f` RSA-AES128-SHA,
`0035` RSA-AES256-SHA, `000a` RSA-3DES-SHA, `0005` RSA-RC4-128-SHA,
`0004` RSA-RC4-MD5.

`16 03 00 00 41 01 00 00 3d 03 00 …` — 0x41 = 65-byte handshake, i.e.
a 70-byte TLS record. Note `0033` first: the TV would prefer DHE, but
a static-RSA server that honours the client's list and picks `002f`
is accepted [confirmed].

## 2. Certificate validation — what the TV actually checks

Measured by live experiment 2026-10-22 [confirmed]:

1. **Self-signed leaf (even CN-matched): rejected.** fkp client sends
   alert `022a` (bad_certificate); infolink client closes silently
   ~440 ms after handshake completes.
2. **Validation is against the served chain only.** Serve
   `leaf + intermediate/root` and the TV builds the chain from what
   it got — no external root store is consulted for these hosts.
3. **DN string types matter.** Certificates generated with modern
   defaults encode subjects as UTF8String; the TV rejects them. The
   real Samsung DNs are PrintableString. A chain whose subjects
   byte-mirror the real ones (as PrintableString) is accepted.
4. **Subject-mirrored chain: accepted** (infolink + sso). This is how
   the captures for [openapi.md](openapi.md) were taken: issue a
   private CA whose subject mirrors the real issuer DN, sign a leaf
   whose subject mirrors the real leaf DN, serve leaf+chain.
5. **fkp is stricter and unresolved**: a mirrored *leaf-only* chain
   draws alert `0228` (illegal_parameter). fkp evidently wants
   something more (real root served? key usage? SHA-256 chain?).

The security reading: a 2012 E-series trusts any server that can
present *a* plausible chain for these hosts — it has no pinned root
for them. On a LAN where the TV's DNS can be answered, its hub
traffic (and the widget-install path of
[widget-install.md](widget-install.md)) is interceptable by anyone.
Worth knowing if you own one of these sets.

## 3. Samsung's 2012 PKI (as still served in 2026)

| service | leaf | chain / root | digest |
|---|---|---|---|
| `infolink.pavv.co.kr` (osb-appsorsay) | `CN=infolink.pavv.co.kr`, valid 2019–2048 | + self-signed root `CN=Samsung Hubsite CA`, 2009–2069 | SHA-1, weak params |
| `sso.internetat.tv` (osb-internetattv) | `CN=*.internetat.tv` (leaf only served) | same `Samsung Hubsite CA` | SHA-1 |
| `fkp.samsungcloudsolution.com` | `OU=vd, CN=fkp…` | `ROOT CA SISC FKP2_PLUS` | SHA-256 |

SPKI SHA-256 of the real infolink leaf (pin it if you proxy upstream
to the real backend) [confirmed]:
`F5:98:FD:85:3A:D7:D7:C5:BD:AD:F2:D9:1A:18:0F:78:55:B7:FE:AE:C3:91:AF:71:1D:D1:D8:69:2E:67:35:3F`;
sso leaf:
`A4:60:9F:35:19:61:BD:48:CC:EA:E9:27:49:4D:EA:AD:83:E1:FA:90:20:20:9F:C9:3A:D5:5A:16:7F:05:93:81`.

"Samsung Hubsite CA" is not publicly trusted — it was evidently an
in-house CA for these devices [likely].

## 4. Host inventory (2026-10)

| host | resolves to | protocol | status |
|---|---|---|---|
| `infolink.pavv.co.kr` → `osb-appsorsay.samsungqbe.com` | 18.159.183.67 (AWS eu-central-1) | SSL3 client → **TLS 1.2+ only** | backend live; answers bind etc. to any TLS 1.2 client |
| `sso.internetat.tv` → `osb-internetattv.samsungqbe.com` | AWS | SSL3 → TLS 1.2+ only | live |
| `fkp.samsungcloudsolution.com` | 18.140.79.177 | **SSL 3.0 only both directions** (rejects TLS 1.2+ with "unsupported protocol") | live — the one host that still speaks 2012's TLS |
| `fkp.samsungcloudsolution.NET` | 3.1.186.239 | same | fallback the TV uses if the .com host is unreachable |
| `oempprd.samsungcloudsolution.com` | CloudFront | HTTP | EMP/widget server; contacted once during hub init |
| `empdownprd.samsungcloudsolution.com` | — | HTTP | EMP download host |
| `www.samsungrm.net` | AWS ELB "PRD-RM-EMP-2016" | HTTP | remote management/telemetry |
| `www.samsung.com` | Akamai | HTTP | connectivity check (§5) |
| `notice.samsungcloudsolution.com` | — | — | named by device/lookup; never observed contacted |
| `log.internetat.tv` | — | https | log sink from conf/log; not observed contacted |
| `apps-ors-multi-cdn.samsungcloudcdn.com`, `apps-pub.samsungcloudcdn.com` | — | HTTP | widget packages / channel config files |
| CloudFront `d1oxlq5h9kq8q5`, `d26q3s8tmkw63c` | — | HTTP | widget thumbnails |

## 5. Connectivity checks (pre-hub)

Before/around hub init the TV performs, in order [confirmed]:

1. DNS A for `ns11.whois.co.kr` ×3 — 2012-era "is there internet"
   probe (any DNS answer counts).
2. `GET http://www.samsung.com/global/products/tv/infolink/us.xml` →
   expects exactly (53 bytes, `Content-Length: 53`):
   `<?xml version="1.0" encoding="UTF-8"?>\n<rsp>ok</rsp>`
   — replicate byte-exact if you ever intercept this host.
3. Then model bind etc. per [openapi.md](openapi.md) §2.

## 6. Observation method

Captures were taken with a TLS-upgrading relay: TV ⇄ SSL 3.0
(subject-mirrored chain, §2) ⇄ relay ⇄ TLS 1.2 + SNI ⇄ real backend,
teeing both directions per connection. The hand-rolled SSL 3.0
endpoint must implement RFC 6101 exactly: RSA ClientKeyExchange as a
raw modulus-sized block (no length prefix), key_block seeded with
`ServerRandom||ClientRandom` (reversed vs the master secret seed),
pad_1/pad_2 MAC construction (not HMAC), and the explicit CBC
padding-length byte. OpenSSL ≥ 1.1 cannot serve SSL 3.0 at all, hence
hand-rolling.
