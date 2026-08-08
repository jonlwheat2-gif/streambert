# AllManga (mkissa.to) Provider — Issue Log & Status

> **App:** Streambert (Electron + React)
> **Module:** `src/ipc/allmanga.js` (+ `src/ipc/allmangaBundleParser.js`)
> **Investigated:** Aug 7, 2026 — **Status: RESOLVED & VERIFIED (16/16)**
> **Companion doc:** `AA-CRYPTO-HANDOFF.md` (deep crypto dive)

---

## TL;DR

### The Issue

The AllManga anime provider **stopped resolving episodes**. The service migrated
from `allanime.to` → **`mkissa.to`** and added a client-computed **"aaReq"
crypto extension** to every episode GraphQL query. The old endpoint started
returning `AA_CRYPTO_MISSING` — no anime episode could fetch sources.

### The Root Cause

A **missing AES-256-GCM auth tag** in the ported aaReq builder. The site's
frontend uses WebCrypto, whose `encrypt()` output is `ciphertext || authTag`
combined — but Node keeps the GCM tag separate in `cipher.getAuthTag()`. The
blob was built as `[0x01][iv][ciphertext]` — **16 bytes short** — so the
server's tag check failed *before it even looked at the payload*. Every request
returned `AA_CRYPTO_STALE`, masking the real bug for hours.

### The Fix

Append the 16-byte auth tag: `[0x01][iv][ciphertext][authTag(16)]`. Once the
tag was appended, the server accepted every request and returned real episode
sources — **16/16 titles resolve** through the production module today.

---

## Issues Found (chronological)

### 1. Service migrated domains & added client crypto — `AA_CRYPTO_MISSING`
- **Symptom:** every episode query returned `AA_CRYPTO_MISSING` / `AA_CRYPTO_STALE`.
- **Root cause:** service moved `allanime.to` → `mkissa.to`, API moved
  `api.allanime.day` → `api.mkissa.net`, and the GraphQL endpoint now requires a
  per-request `aaReq` extension token derived from a per-build client crypto scheme.
- **Fix:** full reverse-engineering of the protocol (see §Protocol) + rewrite of
  the episode resolution layer.
- **Status:** ✅ Fixed.

### 2. THE bug — GCM auth tag never appended (the real blocker)
- **Symptom:** *every* experiment returned `AA_CRYPTO_STALE`, even with the
  correct mask, key, epoch, query hash, and headers.
- **Root cause:** WebCrypto's `encrypt()` returns `ciphertext || authTag`
  combined, but Node keeps the GCM tag separate in `cipher.getAuthTag()`. The
  ported `buildAaReq` appended only `[1][iv][ciphertext]` — the blob was
  **16 bytes short**, so the server's tag check failed *before* it ever looked
  at the payload. This masked every other hypothesis for hours.
- **Fix:** append `cipher.getAuthTag()` to the blob: `[1][iv][ct][tag]`.
- **Verification:** blob grew 164 bytes; server returned 200 with real sources.
- **Status:** ✅ Fixed (this was THE fix).

### 3. Wrong epoch window (3-day vs 7-day)
- **Symptom:** bootstrap occasionally rejected; aaReq consistently stale.
- **Root cause:** the Kotlin reference implementation used 3-day epochs; the
  live site uses **7-day windows** (`epochMs: 604800000`) with a 1-day grace.
- **Fix:** `EPOCH_WINDOW_MS = 7 days`, grace window candidate logic.
- **Status:** ✅ Fixed.

### 4. `ts` window semantics
- **Symptom:** aaReq rejected as stale despite correct epoch.
- **Root cause:** `ts` is a **5-minute-bucketed millisecond** value
  (`floor(now/300000)*300000`), not raw seconds.
- **Fix:** match the frontend's bucketing.
- **Status:** ✅ Fixed.

### 5. Query hash must match the registered GraphQL query exactly
- **Symptom:** `PersistedQueryNotFound` when the `sha256Hash` didn't match.
- **Root cause:** the episode query is a big template with fragments; a single
  whitespace difference changes the SHA-256.
- **Fix:** reconstructed the exact live query; hash
  `63e55b592086c3b006d789824bd5878638f84796a8840750f41ccb9ca9a6c06e`
  verified against the server's persisted-query registry.
- **Status:** ✅ Fixed.

### 6. Build rotation breaks everything (93 → 94, and future builds)
- **Symptom:** after the site ships a new crypto chunk, all requests 403/STALE.
- **Root cause:** each build ships new seed material (`buildId` + 4 seeds) that
  must be re-derived from the live site's obfuscated chunk.
- **Fix:** `allmangaBundleParser.js` re-derives `buildId` + seeds from the live
  chunk; on `AA_CRYPTO_STALE` the app re-derives and retries once.
- **Bug found during porting:** build 94's decoder functions are named `$n`/`br`,
  which the original `\w+` regex missed — parser returned null on build 94.
  Fixed to allow `$` in identifiers.
- **Status:** ✅ Fixed + self-healing (live-verified: build 94 re-derived,
  bootstrap epoch 2953 OK).

### 7. Cloudflare WAF challenges
- **Symptom:** `/api` and site HTML intermittently return a CF challenge page.
- **Root cause:** Node's HTTP fingerprint / rapid retry patterns trigger the WAF.
- **Fix:** correct `User-Agent`, `Sec-Fetch-*`, `Origin`, `Referer` headers +
  graceful fallback (when HTML is blocked, keep the last-known build).
- **Caveat:** WAF can still rate-limit an IP after heavy testing; a cooldown
  between test runs is recommended.
- **Status:** ✅ Mitigated (residual: intermittent rate-limit).

### 8. `x-aa-boot` header is validated server-side
- **Symptom:** garbage/missing boot tokens → `403 invalid_boot_token`.
- **Root cause:** the bootstrap handshake requires an HMAC token derived from
  the mask, buildId, keyGroup, referer host, epoch, and lane.
- **Fix:** implemented `bootToken()` exactly as the frontend does.
- **Status:** ✅ Fixed (server-verified: correct token → 200, garbage → 403).

### 9. Response encryption — `tobeparsed` field
- **Symptom:** source URLs sometimes arrived inside an encrypted `tobeparsed`
  field instead of plain `sourceUrls`.
- **Root cause:** two schemes coexist — new AES-256-GCM (derived key) and legacy
  AES-256-CTR (`Xot36i3lK3:v1` static key).
- **Fix:** decrypt with the derived GCM key first, fall back to legacy key, then
  to the old CTR scheme.
- **Status:** ✅ Fixed.

### 10. New embed sources vs legacy clock paths
- **Symptom:** new-style `sourceUrl`s are **full embed URLs** (streamlare.com,
  streamsb.net, ok.ru…) rather than `--`-encoded direct-mp4 clock paths.
- **Root cause:** the service changed its source format; direct mp4s are rarer.
- **Fix:** full https embed URLs now play directly in the player webview (same
  path as vidsrc/2embed — no `isDirectMp4` → webview). Legacy `--` clock paths
  are still decoded and fetched for direct mp4 as a fallback.
- **Status:** ✅ Fixed.

### 11. Renderer integration (no changes needed)
- Verified `src/utils/useTVPlayer.js` / `useMoviePlayer.js` already handle
  `res.isDirectMp4 === undefined` by loading the URL in the webview — the same
  path as vidsrc/2embed — so no renderer changes were required.
- **Status:** ✅ No-op (verified).

### 12. Intermittent "Too many requests, try again in 5 seconds"
- **Cause:** aggressive parallel testing against the API.
- **Mitigation:** request-queue + cooldowns in the test harness; in-app caching
  of build/boot key material (10–30 min TTL).
- **Status:** ✅ Mitigated.

---

## The Protocol (as implemented)

```
1. Scrape https://mkissa.to/ HTML → entry chunk → crypto chunk (contains "aaReq")
2. Parse buildId + 4 base64 seeds  (allmangaBundleParser.js)
3. deriveMask(buildId, seeds)      → 32-byte per-build mask
4. GET /client-crypto/v1/bootstrap?buildId=<id>&k=k7
     headers: x-build-id, x-aa-boot=HMAC(mask,"aa-boot:"+buildId → "…:epoch:k7")
   → { epoch (7-day window), partB }     (partB is random per bootstrap)
5. deriveKey(mask, partB)          → AES-256-GCM key
6. Build aaReq:
     ts    = floor(now/300000)*300000
     iv    = sha256(epoch:buildId:queryHash:ts:k7)[0:12]
     payload = {"v":1,"ts","epoch","buildId","qh":queryHash,"k":"k7"}
     ct+tag = AES-256-GCM(key, iv, payload)
     blob  = [0x01][iv][ct][tag]   ← tag is REQUIRED
     aaReq = base64(blob)
7. GET /api?variables=<json>&extensions={"persistedQuery":{"sha256Hash":…},"k":"k7","aaReq":…}
     header: x-build-id: <id>
8. Parse: data.episode.sourceUrls (plain) OR decrypt data.tobeparsed (GCM/CTR)
9. On AA_CRYPTO_STALE → re-derive build from live site → retry once
```

- Search queries are crypto-free (plain POST to `api.mkissa.net/api`).
- Legacy static key `sha256("Xot36i3lK3:v1")` used only as a decrypt fallback.

---

## Remaining Known Risks (not bugs — context for future maintainers)

| Risk | Notes |
|---|---|
| **Build rotation cadence** | The site rotates builds periodically. The self-healing path needs the live site HTML to be fetchable; if Cloudflare blocks it, the app falls back to the last-known build (currently 94) and surfaces a clear error if that's stale. |
| **Third-party embeds show their own ads** | Unlike the old direct-mp4 clock paths, the new streamlare/streamsb/ok.ru players are third-party pages and may serve their own ads. The app's session-level ad-blocking still applies. |
| **Rate limiting** | Heavy parallel usage can trigger "Too many requests"; in-app caching mitigates. |
| **Cloudflare escalation** | The WAF may challenge Node-fingerprinted traffic more aggressively over time; headers are already tuned. |
| **`HARDCODED_SHOW_IDS` / `SPLIT_SEASONS`** | JoJo seasons and Spy x Family split-cours rely on hardcoded show IDs; these may drift if the service's catalog IDs change. |

---

## Verification Summary (Aug 7, 2026)

| Check | Result |
|---|---|
| Syntax check all main-process files | ✅ 11/11 OK |
| Reliability harness through the **production module** | ✅ **16/16 titles resolved** |
| Live pipeline (bootstrap → aaReq → sources) | ✅ 4/4 passes |
| Build-rotation self-healing (HTML → entry → chunk → parse → bootstrap) | ✅ build 94 re-derived, epoch 2953 OK |
| Renderer production build (`vite build`) | ✅ clean |
| Embed URLs | ✅ ok.ru = real player page; streamlare = JS redirect that resolves in the webview |

---

## Related Files

- `src/ipc/allmanga.js` — production provider (aaReq pipeline, search, player server)
- `src/ipc/allmangaBundleParser.js` — build/seed self-healing parser
- `test-allmanga-reliability.js` — 16-title regression harness (runs the real handler)
- `debug-allmanga.js` — manual debug harness
- `AA-CRYPTO-HANDOFF.md` — full reverse-engineering notes
