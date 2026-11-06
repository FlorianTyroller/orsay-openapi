# Samsung Orsay openapi — protocol reference

Factual reference of the 2012 Orsay Smart Hub backend protocol as
spoken by a UE40EH5300 (T-MST10PDEUC-1007.0, SmartHub 4.6261),
reconstructed from plaintext captures taken 2026-10-22/23 by
terminating the TV's SSL 3.0 session with a subject-mirrored chain
(see [transport.md](transport.md)). Samsung still operates the whole
API — every documented endpoint answered live on those dates.
Companion docs: [transport.md](transport.md) (SSL 3.0 clients, PKI,
hosts), [widget-install.md](widget-install.md) (develop-mode install
path), [endpoints.tsv](endpoints.tsv) (census).

Confidence tags: **[confirmed]** = read from a capture, byte-quoted
(identifiers redacted as `<duid>`, `<mac>`, `<session>`, `<h32>`);
**[likely]** = strong evidence, one unverified link; **[guess]** =
shape only. Evidence labels `conn<N>` refer to the author's capture
set (both directions per connection; not published).

## 1. Shape

Plain HTTP/1.1 inside an SSL 3.0 session to `infolink.pavv.co.kr:443`
(CNAME → `osb-appsorsay.samsungqbe.com`, AWS, nginx/1.30.2 fronting a
microservice farm — see §6). All captured traffic is **GET**; no POST
was ever observed on this host [confirmed]. Requests are small header
sets (`Connection: close`), responses XML with `Content-Length` and
`Connection: Close`. `Accept-Encoding: gzip` is offered but **no
response was ever gzipped** [confirmed]. One TCP/SSL connection usually
carries one exchange; a capture can contain several back-to-back
exchanges.

Common headers on every request [confirmed]:

```
Host: infolink.pavv.co.kr
Connection: close
User-Agent: Mozilla/5.0 (SmartHub; SMART-TV; U; Linux/SmartTV; Maple2012)
  AppleWebKit/534.7 (KHTML, like Gecko) SmartTV Safari/534.7
Content-Type: text/xml
cache-control: no-cache
Accept: */*
Accept-Encoding: gzip
Content-Length: 0
```

Cache-busting `?timestamp=` on everything. Two clocks are used
[confirmed]: device-class endpoints send boot-relative milliseconds
(86582…), service-class endpoints send epoch milliseconds
(1792700513937 ≈ 2026-10-22).

### 1.1 Response envelope

Success [confirmed] (`conn110`):

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<rsp cacheDate="…" cacheKey="device.model.bind_TV,LED,X10+,UE5300,EU_GERMANY"
     cacheExpireDate="…" stat="ok">…</rsp>
```

Failure [confirmed] (`conn35`, HTTP 400):

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<rsp stat="fail">
    <err code="107" msg="Invalid Service Request"/>
    <desc>Invalid widgetid.</desc>
</rsp>
```

Only one error shape was ever provoked (err 107, invalid widget id —
the backend rejects non-numeric IDs; see
[widget-install.md](widget-install.md) for why that matters).
HTTP-level: 400 on that error, 200 otherwise. `standalone="no"` on
success, `"yes"` on the error — meaningless, but byte-exact emulation
should copy it [confirmed].

Cache headers on cached endpoints [confirmed]: `ETag`
(`"<md5>"`), response-level `cacheDate`/`cacheExpireDate`, and for the
channel family `OSBCacheInfo: <http-date>; <max-age-seconds>`.

## 2. Device identity & boot ladder

Measured boot sequence (2026-10-22 evening session, `conn110`–`conn128`;
repeats identically on every hub start) [confirmed]:

| # | request | auth | yields |
|---|---------|------|--------|
| 1 | `GET /openapi/device/model/bind` | none | `<model id="12_X10PLUS_2D"/>` |
| 2 | `GET /openapi/device/lookup` | none | smarthub version, apiDomain, noticeDomain |
| 3 | `GET /openapi/device/auth/seedkey` | AppKey | session id, user id, seedkey, expiry |
| 4 | `GET /openapi/conf/date` | AppKey+Token | server time |
| 5 | `GET /openapi/conf/log?service=widget` | AppKey+Token | log-sink config |
| 6 | `GET /openapi/service/widget/list?lang=de` | AppKey+Token | catalog of this model's widgets |
| 7 | `GET /openapi/service/widget/signed/<id>` ×N | AppKey+Token | per-widget download metadata |
| 8 | `GET /openapi/service/widget/gallery?…` | AppKey+Token | store listing |
| 9 | `GET /openapi/update/notice` | none (duid) | disclaimer/notice pointer |
| 10 | `GET /openapi/service/channel/gallery?category=BrowserConfig` | AppKey+Token | channel listing |
| 11 | `GET /openapi/service/channel/5347` | AppKey+Token | browser config data |
| 12 | `GET /openapi/service/disclaimer/770` | none (duid) | the German 2012 terms, 66 KB |

Identity headers, grouped by which requests carry them [confirmed]:

| header | value (this TV) | on |
|--------|-----------------|----|
| `duid` | `<duid>` (13 chars) | bind, lookup, notice, disclaimer |
| `macaddr` | `<mac>` (12 hex, lowercase) | bind, seedkey |
| `localset` | `EU_GERMANY` | bind |
| `lineup` | `UE5300` | bind |
| `chipset` | `X10+` | bind |
| `displaytype` | `LED` | bind |
| `producttype` | `TV` | bind |
| `cis_smarthub_block` | `OFF` | bind |
| `CountryCode` | `DE` | lookup, seedkey, notice, disclaimer |
| `Firmcode` | `T-INFOLINK2012-1003` | lookup, seedkey |
| `Client` | `T-INFOLINK2012-1003+SmartHub/4.6261` | lookup |
| `ModelId` | `12_X10PLUS_2D` | lookup, seedkey (value from bind step 1) |

`ModelId 12_X10PLUS_2D` decomposes as year-class 12 + SoC family X10+
+ 2D [likely].

## 3. Auth: AppKey / Token / seedkey

Two secrets ride in headers [confirmed]:

- **AppKey** — per-service constant, byte-exact [confirmed, `conn113`]:
  `f53e13dd-ca52-1172-9e58-100000230005-8d3b3ef9-1e60-4e70-a026-c741be2569f1`.
  The middle token `100000230005` looks like a numeric service id
  embedded in the UUID chain [guess]. The SSO host uses a different,
  shorter key (§5). These are firmware constants, not per-device
  secrets — they ride in every request of every device of the class.
- **Token** — `<session>+<user>+<h32>`

  Structure: `<8-hex session id>+<user id>+<32 hex>` [confirmed].
  Session id from the seedkey response; user id is
  `_12_X10PLUS_2D_<mac>` = `_` + ModelId + `_` + MAC — i.e. the device
  itself is the "user" [confirmed]. The 32-hex component is presumably
  an MD5 over (seedkey · something); derivation **unknown** [guess].
  Emulation does not need it: the backend accepted a replayed Token
  verbatim, and the TV kept one Token for the entire captured period
  (3 days, single seedkey call, expiry 21 days:
  `<expired>2026-11-12T20:44:25+0000</expired>`) [confirmed].

seedkey response (`conn112`, trimmed, values redacted) [confirmed]:

```xml
<rsp stat="ok"><session id="<session>">
  <user protected="no" id="_12_X10PLUS_2D_<mac>" confirmed="yes"/>
  <seedkey><uuid>-<uuid></seedkey>
  <expired>2026-11-12T20:44:25+0000</expired>
  <country code="DE" supported="yes"/>
  <isp code="…" name="…" supported="yes"/>
</session></rsp>
```

The seedkey pairs with the `<encryption type=… seedkey=…>` fields in
widget/signed answers (§4.3): packages with non-empty `encryption`
are presumably decrypted with it [likely — always empty in our
captures, all URLs plain HTTP zips].

## 4. Endpoint reference

### 4.1 `device/model/bind` — `conn110`

```
GET /openapi/device/model/bind?timestamp=86582
```
All §2 identity headers. Response [confirmed]:

```xml
<rsp cacheDate="…" cacheKey="device.model.bind_TV,LED,X10+,UE5300,EU_GERMANY"
     cacheExpireDate="…" stat="ok">
  <model id="12_X10PLUS_2D"/>
  <cacheFlushTime>2026-06-29T06:42:43+00:00</cacheFlushTime>
  <flushAPI>/device/model/bind</flushAPI>
</rsp>
```

This is the gate that killed the hub on the real internet: the
endpoint exists and answers, but only over ≥TLS 1.2 (see
[transport.md](transport.md)).

### 4.2 `device/lookup` — `conn111`

```xml
<rsp … stat="ok">
  <smarthub id="10120000000"><title><![CDATA[Smart Hub for Webkit]]></title>
    <version>4.6261</version><updated>2025-01-22T11:54:25+09:00</updated></smarthub>
  <country>DE</country><isp>no</isp>
  <noticeDomain>notice.samsungcloudsolution.com</noticeDomain>
  <apiDomain>infolink.pavv.co.kr</apiDomain>
  <date>2026-10-22T07:41:15+00:00</date>
</rsp>
```

Interesting: `smarthub id="10120000000"` — the hub itself is a widget
(the same id the browser-config channel 5347 points at) [confirmed].
`<updated>2025-01-22</updated>` — the hub metadata was still being
touched in 2025.

### 4.3 `service/widget/*` — the app catalog

**`list?lang=de`** (`conn115`, 27 KB): every widget for this model
(18 ids). Per-entry schema [confirmed]:

```xml
<widget id="10120000099" size="12697600" version="1.125"
        published="2026-04-29T05:35:44+0000" updated="2014-03-26T10:52:39+0000"
        expired="2100-01-01T00:00:00+0000" removable="true">
  <title lang="de"><![CDATA[Samsung Apps]]></title>
  <category>Default</category>
  <author>Samsung Networks</author>
  <vendor><![CDATA[]]></vendor>
  <compression size="7664122" type="zip"/>
  <description lang="de"><![CDATA[Samsung Apps]]></description>
  <featured system="signature" priority="1"/>
  <emp stat="no"/>
  <groupname key="SamsungApps"/>
  <featured system="store"/><featured system="forced"/><featured system="preinstall"/>
  <media:thumbnail url="http://d26q3s8tmkw63c.cloudfront.net/…png" width="" height=""/>
  <pubDate>Wednesday, 29 04 2026 05:35:44 +0000</pubDate>
</widget>
```

`featured system=` values seen: default, store, forced, preinstall,
signature [confirmed]. `size` = uncompressed, `compression/@size` =
zip bytes. Thumbnails live on CloudFront (`d1oxlq5h9kq8q5`,
`d26q3s8tmkw63c`). `pubDate` uses the nonstandard
`Wednesday, 29 04 2026 05:35:44 +0000` format [confirmed].

Observed catalog (18 ids, all requested via `signed` during sync)
[confirmed]: 10110001001 MBRWidget, 10120000099 Samsung Apps,
11101307904 Tagesschau, 111299000607 feratel, 111299001086 Deezer,
111299001432 Zattoo, 111399000990/111399000102, 111299001605,
111477001106, 121299000101, 3201412000679, 3201508004904,
3201608010221/222/224/225, 3201609010551. Ids are 10–13 digits;
leading `1` = 2011–2012 era, leading `3` = 2015+ store additions
[guess from the dates].

**`gallery`** (`conn120`/`conn128`): store listing, same `<widget>`
schema plus `<featured/>`, `<emp stat=…/>`, long CDATA descriptions.
Three query shapes observed [confirmed]: `?category=CP_YourVideo&lang=de`
(hub's video section), `?featured=signature&lang=de`,
`?featured=partners&lang=de` — i.e. filtering by category or by the
`featured system=` groups above. A fourth shape, `?category=Manager`,
is the **develop-mode** query — it goes to the developer's own server,
not the backend (see [widget-install.md](widget-install.md)).

**`signed/<id>`** (`conn10`, `conn100`–`conn108`): per-widget download
metadata [confirmed]:

```xml
<rsp … stat="ok"><list>
  <widget id="111477001106" version="1.003" updated="2016-11-10T17:57:13+0000"
          url="http://apps-ors-multi-cdn.samsungcloudcdn.com/package/21/217ecce69c044ce9b84c1a4066665cb6/66665cb6.zip"
          size="9199616">
    <compression type="zip" size="3580867"/>
    <updateforced stat="no"/>
    <encryption type="" seedkey=""/>
    <emplist stat="no"/>
  </widget>
</list></rsp>
```

Package URL pattern [confirmed]:
`apps-ors-multi-cdn.samsungcloudcdn.com/package/<2 hex>/<32 hex>/<last 7 hex of the 32>.zip`.
Plain HTTP. Invalid id → err 107 (`conn35`, `conn71`).

### 4.4 `service/channel/*` — config channels

`channel/gallery?category=BrowserConfig` (`conn10`): RSS 2.0 with
`xmlns:secapi="http://openapi.samsung.com/api/1.0"`, one `<item
secapi:chid="…">` per channel; 3 channels for DE (chid 4650/2011,
5347/2012, 5781/2013) [confirmed].

`channel/5347` (`conn12`): the 2012 browser config payload — points at
`http://apps-pub.samsungcloudcdn.com/files/channel/201507/5347/15490/Default_1.100.xml`
via `<secapi:widget id="10120000000"/>` [confirmed]. I.e. the "channel"
is a versioned config feed consumed by the hub widget itself.

### 4.5 `conf/*`, `update/notice`, `service/disclaimer/<id>`

- `conf/date` (`conn113`):
  `<rsp stat="ok"><date>2026-10-22T20:21:54+00:00</date></rsp>`.
- `conf/log?service=widget` (`conn114`): logging config — TV-side
  batching parameters and the upstream sink
  `https://log.internetat.tv/openapi/log/widget` (queue_max 300,
  expiration 600, threshold 60, loglevel 55; per-event sinks "home",
  "samsungapps", "application"). The TV's POST format to that sink was
  never captured [confirmed config / unknown usage].
- `update/notice` (`conn122`): notice/disclaimer pointer with per-item
  versions (service 2.000, log 2.000, voice_pp 2.000, nuance_pp
  2.000). Polled repeatedly while the hub runs.
- `service/disclaimer/770` (`conn23`): 66 KB of 2012 German terms HTML
  inside `<item id="service"><content><![CDATA[…]]></content>`.
  Served by `tnc-r53-prd.samsungqbe.com` (the only endpoint on that
  host).

## 5. SSO host variant — `sso.internetat.tv`

The login service speaks the same openapi dialect with three twists
(`sso/conn1`) [confirmed]:

```
GET /openapi/user/auth/seedkey HTTP/1.1
Host: sso.internetat.tv
ModelId: UE5300                    ← lineup! not the 12_X10PLUS_2D id
WidgetModelID: 12_X10PLUS_2D       ← the bind id, under a new name
AppKey: 85d225b3-ec13-418d-9f89-161add252b06   ← different, shorter key
Token: <session>+_12_X10PLUS_2D_<mac>+<h32>    ← infolink Token reused
```

Response: `<session id="<session>"><seedkey>…</seedkey><expired>…+09:00</expired></session>`
— no `<user>` element, expiry in +09:00 (Seoul) [confirmed]. Note the
TV proffers the *infolink* Token to get an SSO session — sessions are
per-service but chained through the device Token [likely].

The local `develop`/`111111` account (used for sideloading) does not
require a successful SSO login, but the hub does not reach the login
screen until `sso.internetat.tv` answers *something* protocol-shaped
[confirmed — hub stalled at "no connection to server" until relayed].

## 6. Backend farm (all still live 2026-10)

One virtual host (`infolink.pavv.co.kr` / `osb-appsorsay.samsungqbe.com`,
AWS + nginx/1.30.2) fans out to per-class services, revealed by the
`HostName` response header [confirmed]:

| HostName | serves | share of 302 captured responses |
|---|---|---|
| `appsappsyncors-vg.samsungqbe.com` | widget list/signed/gallery | 133 |
| `appschannel-vg.samsungqbe.com` | channel/*, update/notice | 80 |
| `appsdevice-vg.samsungqbe.com` | device/* | 51 |
| `appsconf.samsungqbe.com` | conf/* | 34 |
| `tnc-r53-prd.samsungqbe.com` | disclaimer | 4 |

Other infrastructure named by the protocol (not contacted in the
author's rig): `notice.samsungcloudsolution.com` (from lookup),
`log.internetat.tv` (from conf/log), `apps-ors-multi-cdn` /
`apps-pub.samsungcloudcdn.com` (packages, plain HTTP), two CloudFront
thumbnail hosts. See [transport.md](transport.md) for the full host
inventory.

## 7. Known unknowns

- **Token 32-hex derivation** — md5-shaped, never cracked, not needed
  for emulation (replay accepted).
- **fkp protocol** (`fkp.samsungcloudsolution.com`) — binary, not
  openapi; never captured (its client rejects the mirrored chain with
  TLS alert 0228 — see [transport.md](transport.md)). Known only as
  `TESTPING` strings from an early probe.
- **Log POST format** to `log.internetat.tv` — config seen, traffic
  not.
- **Encrypted widget packages** — `encryption/@type` always empty in
  captures; the seedkey-decryption path is unexercised.
- **`update/notice` polling trigger** — 19 polls over ~4 h, no pattern
  established.
