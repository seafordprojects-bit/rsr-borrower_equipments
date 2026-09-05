Phase 2 — Offline Provisional Identity Display
Status: Planned, not started. Do not begin implementation until explicitly instructed in a fresh session.
Depends on: `offline-punch-v2` (Phase 1) fully merged to `main`, deployed to both tablets, version-stamp verified on both devices, and run through at least one real work day without issues.
Owner decision authority: Raffy. All decisions below were made deliberately in a planning session on 2026-09-05 and should not be re-litigated without a clear reason — if something here seems wrong during implementation, flag it to Raffy rather than silently deviating.
Problem being solved
Today, when a kiosk tablet is offline, it cannot identify who is punching — `identifyByPin`'s server RPC has nothing to reach, so `curEmp` never gets set, and the screen shows no name, just "Saved offline - pending." All six punch buttons are enabled (the tablet "decides nothing" — see `kp()`'s offline branch, kiosk/index.html:2342-2351). The actual identity/duplicate-check authority remains 100% server-side at sync time via `sync_offline_punch` — this is confirmed solid and Phase 2 must not change it.
Phase 2 only adds a local, cached, explicitly-labeled-as-unverified display layer so a worker gets a name-shaped confirmation on screen instead of a blank state — cosmetic/UX only, never authoritative.
Architecture
New table: `kiosk_devices` (device_id, site, token_hash, registered_at, revoked_at).
New RPC: `kiosk_pin_list(p_device_id, p_token)` — returns salted PIN hashes + name and position only (not full employee record) for that device's registered site. Rejects unknown/revoked devices. Must NOT be callable with the anon key alone — requires the device token.
Tablet fetches this list after every successful sync, caches locally (IndexedDB or localStorage), and the cache expires after 7 days without a successful sync.
Offline PIN entry does a local hash lookup against the cached list purely for display. The real punch queuing/sync path (`offlinePin`, `offqToday`, `sync_offline_punch`) is completely unchanged — this local lookup never gates or blocks a punch, it only decides what text/styling to show.
Decisions locked in (2026-09-05 planning session)
1. Device registration
Owner-only (Raffy). No delegation to Jamaica or anyone else. Registration happens via the admin panel, showing the token once at setup time, same pattern as the existing admin-PIN-gated flows.
2. Token storage & security
Simple local storage (IndexedDB/localStorage) after one-time entry at registration — no extra PIN-gating on top, no token rotation. The only safeguard is the existing 7-day cache expiry (already free from the base design). Rationale: only Raffy ever registers a device, tablets live in controlled yard environments (not public-facing), so the realistic threat model here is much narrower than the earlier anon-key exposure issue. Extra gating (PIN-to-reveal-token, rotating short-lived tokens) was considered and explicitly rejected as disproportionate effort for this threat model — do not add it without a new decision from Raffy.
3. Provisional match found (PIN found in local cache, tablet offline)
Show BOTH a text label AND visual styling — redundant signals, since the audience is a worker glancing briefly, not reading carefully.
Exact display:
```
[Name] (unverified)
[Position]

Not yet confirmed — will update once this tablet reconnects.

Saved offline - pending
```
Styling: muted/grey text and/or dashed border around the identity card, visually distinct from the solid/bold styling used for a fully-online, server-confirmed identity card. The bottom "Saved offline - pending" line is unchanged from current behavior.
4. No local match at all (PIN not in cache — new hire, wrong-site roster, or genuine typo)
Do NOT say "wrong passcode" or imply the tablet knows the PIN is incorrect — it genuinely cannot tell a typo apart from a valid-but-uncached PIN (new hire not yet synced, or a worker from the other site's roster). Show an honest message instead:
```
Can't confirm your name right now.
Wala pay koneksyon aron ma-verify — ma-record gihapon ni imong punch,
ug ma-ayo ra sa dihang mabalik ang signal.
```
(English gloss for dev reference: "Your punch is still being recorded — you'll show up correctly once this tablet reconnects." Confirm exact Bisaya phrasing with Raffy/Jamaica before shipping — approximate translation only, not verified by a native speaker.)
Punch still queues and syncs exactly as today — this only changes the displayed text.
5. `kiosk_pin_list` return fields
Name and position only. Do not include phone, department, or other fields — minimize what's cached on-device.
6. Device state fallback — three distinct situations
Never registered (no token on this device): behave exactly as today — blank, no name, no new message. This is just "Phase 2 not turned on for this device yet," not an error state.
Registered but stale (valid token, but 7+ days since last successful sync, cache expired): show a distinct message — `"This tablet hasn't connected in a while — names won't show until it's back online."` This is a soft signal, mainly useful to Raffy/Jamaica noticing a real per-device connectivity problem, not the worker.
Explicitly revoked (Raffy revoked this device's token via admin panel — lost/stolen/decommissioned tablet): show a hard, unmistakable warning — `"This device is no longer active. Please contact Raffy or the office."` This must be visually distinct from the other two states (should not look like a normal offline screen) since the whole point of revoking a device is to make continued use of it obvious, not silent.
7. Rollout
Both tablets (Carmen and Mandaue) registered and switched to Phase 2 together, not staged one-at-a-time. (Contrast with Phase 1's deploy, which was intentionally staged — Phase 2's registration step is simpler/lower-risk than Phase 1's core sync/identity fixes, so simultaneous rollout was judged acceptable here.)
Explicitly out of scope for Phase 2
Any change to the actual authorization/duplicate-detection logic — that remains 100% server-side, unchanged.
Any change to `offlinePin`/`offqToday`/`sync_offline_punch` — this is purely a display-layer addition.
Token rotation, short-lived tokens, or PIN-gating the token display (considered, rejected — see §2).
Delegating device registration to Jamaica or anyone besides Raffy.
Before implementation begins
Confirm Phase 1 (`offline-punch-v2`) has been running cleanly in real production use for a reasonable period on both tablets.
Confirm exact Bisaya wording for §4 with a native speaker (Jamaica) before it ships to production.
This spec should be treated as the source of truth — if implementation reveals a genuine problem with a decision here, flag it back to Raffy rather than silently changing course.
