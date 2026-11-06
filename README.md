# orsay-openapi — the 2012 Samsung Smart Hub backend, documented

Factual documentation of the **Orsay openapi protocol** — the backend
that 2011–2014 Samsung Smart TVs (E/F/H series, pre-Tizen "Smart Hub")
speak to `infolink.pavv.co.kr` and `sso.internetat.tv`.

Measured on a UE40EH5300 (T-MST10PDEUC-1007.0, SmartHub 4.6261,
Orsay/Maple2012) in October 2026, by terminating the TV's SSL 3.0
session locally (the TV accepts a subject-mirrored PKI chain) and
teeing both directions. Samsung still operates the entire backend
**today** — every documented endpoint answered live, with current
dates and 2020–2026 catalog updates.

**Nothing here redistributes Samsung firmware, captured blobs, or the
author's device identifiers** (duid/MAC/token are redacted as
placeholders). Published are protocol facts and analysis only (see
[Legal](#legal)).

## The headline findings

1. **The 2012 hub backend never died.** `infolink.pavv.co.kr` (CNAME
   `osb-appsorsay.samsungqbe.com`) still serves the full openapi:
   device bind, app catalog, widget downloads, browser config, the
   2012 German terms. Behind nginx 1.30.2 on AWS, a five-service farm
   (see [openapi.md](analysis/openapi.md) §6).
2. **E-series hubs were killed by TLS alone.** The TV speaks bare
   SSL 3.0 (RFC 6101, no extensions, no SNI); the backend now requires
   TLS 1.2+. Nothing else in the chain rejects the TV — model bind
   answers `stat="ok"` to this day. That is why ERR_MODEL_BIND is
   permanent on SSL 3.0-only clients while some later Orsay firmware
   (newer TLS stack) still shows a store. A simple TLS-upgrading relay
   (or any server that still accepts SSL 3.0) revives hub
   initialization — no protocol emulation needed for the boot ladder.
3. **The protocol is small and plain**: HTTP/1.1 GETs inside the TLS
   session, XML envelopes, a two-header auth scheme (AppKey + Token),
   per-class cache metadata. 13 endpoints observed; all documented
   here with byte-exact excerpts.
4. **Certificate validation is served-chain-only** — the TV builds the
   chain from what the server presents and accepts a mirroring fake
   chain (PrintableString DNs required). Measured rules in
   [transport.md](analysis/transport.md) §2.

## Documentation

| document | content |
|----------|---------|
| [analysis/openapi.md](analysis/openapi.md) | Protocol reference: boot ladder, AppKey/Token auth, response envelope, all 13 endpoints with excerpts, backend farm, known unknowns |
| [analysis/transport.md](analysis/transport.md) | The two SSL 3.0 clients (cipher fingerprints), certificate-validation behavior, Samsung's 2012 PKI + SPKI pins, host inventory, connectivity checks |
| [analysis/widget-install.md](analysis/widget-install.md) | The develop-mode install path: openapi gallery (not widgetlist.xml), widget id rules, install locations, hub re-download behavior |
| [analysis/endpoints.tsv](analysis/endpoints.tsv) | Machine-readable endpoint census (params, auth, headers, serving host) |

Evidence numbering (`conn<N>`) refers to the author's capture set
(both directions per TCP connection); raw captures are not published
because they embed device identifiers.

## Why

Millions of Orsay sets are in circulation; their Smart Hubs show
ERR_MODEL_BIND forever, and the official answer is "buy a newer TV."
The backend's continued operation means these hubs are revivable, and
the protocol's simplicity means a community hub server is a weekend
project, not a reverse-engineering marathon. This repo is the map for
that. SamyGO (forum.samygo.tv) covers rooting the same sets; this
covers what they talk to.

## Legal

- All facts here derive from the author's own device and own network
  traffic (interoperability research on lawfully owned hardware).
  No Samsung firmware, widget packages, or raw capture data is
  distributed.
- Samsung, Smart Hub, Orsay, Infólink are trademarks of Samsung
  Electronics; this independent documentation is not affiliated with
  or endorsed by Samsung.
- License for this repository's own material: **CC-BY-4.0** (see
  [LICENSE](LICENSE)).

Liveness claims are dated: **verified 2026-10-22/23**. If you re-check
later, pull requests with updated findings are welcome.
