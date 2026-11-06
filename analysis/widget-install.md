# The Orsay widget install path — develop mode, gallery vs widgetlist

How widgets actually get installed on a 2012 E-series hub, as measured
live 2026-10-22/23 — including the parts the official SDK docs do not
tell you. The widget *package* format (config.xml, icons, SEF plugin
APIs) is officially documented and still available: see Samsung's
[legacy platform docs](https://developer.samsung.com/smarttv/legacy/api-references.html).
This document covers the install *protocol* and its gotchas.
Companion: [openapi.md](openapi.md).

## 1. Develop mode — the sideloading door

Smart Hub has a hidden `develop` account: log in as `develop` with
PIN `111111` (local, no SSO round-trip), then set a **dev server IP**
in its settings. Two facts the forums get wrong [confirmed]:

- **The login and the dev-server IP do not survive a TV reboot** —
  re-enter both before every sync. Error `ERROR_DEVELOPMENT_002` =
  dev server setting lost.
- The TV must be able to complete hub initialization (see
  [transport.md](transport.md)) before the develop account is even
  reachable — the account lives inside the hub UI.

## 2. What sync actually fetches

"Benutzeranwendung synchronisieren" / user-application sync reads
**both** of these from the dev server [confirmed]:

1. `GET /widgetlist.xml` — the folklore format. It **is fetched, but
   is not sufficient**: listing a widget here alone does not install
   it (adding a `<version>` element makes no difference either).
2. `GET /openapi/service/widget/gallery?category=Manager` — **this is
   the one that installs widgets**, in the same openapi XML dialect
   the real backend uses (schema in [openapi.md](openapi.md) §4.3).

A dev server therefore needs to answer the openapi gallery with
`<widget>` entries whose `<download>` points at a zip; serving a
widgetlist.xml alone is cargo cult.

### 2.1 widgetlist.xml (still expected)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<rsp stat="ok">
<list>
<widget id="mywidget">
<title>My Widget</title>
<version>1.000</version>
<compression size="22300" type="zip"/>
<description>…</description>
<download>http://<devserver-ip>/Widget/mywidget.zip</download>
</widget>
</list>
</rsp>
```

### 2.2 Widget ids are validated — by the real backend

During sync the TV asks the **real backend**
`GET /openapi/service/widget/signed/<id>` for every widget it
learned about. A non-numeric id (e.g. `SamyGO-F-SM-MST-Fix`) draws
HTTP 400, `<err code="107">`, `<desc>Invalid widgetid.</desc>` — and
the hub then rejects the *entire* list with "special characters not
allowed in widget id" [confirmed]. Use ids like `mywidget`
(lowercase alphanumeric; numeric like the store's `111299001086`
also works).

Note what this implies: **sync mixes your dev server with Samsung's
live backend** — your DNS must resolve `infolink.pavv.co.kr` for the
TV, and entries in your list get cross-checked against Samsung.

## 3. Install locations & lifecycle

- User (develop-installed) widgets → `/mtd_rwcommon/widgets/user/<id>/`;
  store widgets → `/mtd_rwcommon/widgets/normal/<id>/`;
  `/mtd_down` is a symlink to `/mtd_rwcommon` [confirmed].
- **Store widgets re-download themselves after deletion** — the hub
  re-fetches from the still-live catalog on next sync. Real debloat
  requires filtering the `widget/signed` + `gallery` answers en
  route, not deleting directories.
- The gallery answer needs the full `<widget>` schema (§4.3 of
  [openapi.md](openapi.md)); a missing `<title>` element renders the
  entry as "Abnormal Title" in the hub [confirmed].
- Deleting a widget app from the hub can also kill processes and
  delete files on writable partitions that the widget (or things
  started from Samsung's own `/mtd_rwarea/start.sh` rc hook) left
  behind [confirmed, one observed incident].

## 4. The widget runtime, briefly

WebKit/534.7 ("Maple2012") pages, XHTML 1.0; TV hardware is reachable
only through `<object classid="clsid:SAMSUNG-INFOLINK-*">` plugin
embeds whose methods are called via `.Execute()` — e.g.
`SAMSUNG-INFOLINK-PLAYER` with `Execute('InitPlayer', url)` /
`Execute('StartPlayback', pos)`. The player's HTTP stack identifies
as `Lavf52.104.0` (2012 ffmpeg), always sends `Range: bytes=0-`, and
cannot do TLS to modern CDNs (no SNI-era support) — feed it plain
HTTP. Full API surface: Samsung's
[legacy API references](https://developer.samsung.com/smarttv/legacy/api-references.html).

Sandbox observations [confirmed]: widget-spawned processes cannot
`connect()` outbound (inbound `listen()` and stdin/stdout streaming
do work); writable areas are `/mtd_rwcommon`, `/mtd_rwarea`, and
`/dtv` (tmpfs).
