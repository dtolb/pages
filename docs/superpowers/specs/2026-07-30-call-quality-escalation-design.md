# Call-quality detection and ACI escalation for Conference calls

**Date:** 2026-07-30  
**Status:** Design approved, pending implementation plan  
**Topology:** ISV / multi-tenant. One Twilio subaccount per tenant.  
**Call shape:** Agent on Twilio Voice JS SDK (browser) + PSTN caller, both in a Twilio Conference.

---

## 1. Purpose

Detect degraded call quality from free client-side SDK events. Use those events to decide when to enable Voice Insights Advanced Features (ACI) for a tenant subaccount, diagnose the fault, and disable it again.

Deployments are low-margin, so ACI cannot run permanently.

### Overview

```
                AT LOGIN          DURING CALL         ON TRIP         INSTRUMENTED      CLOSE
                ────────          ───────────         ───────         ────────────      ─────
 CLIENT         runPreflight      sample 1 Hz         beacon on            ·              ·
 browser        isTurnRequired    warning →           state change
                                  indicator

 SERVER              ·            ingest, rollups     POST Settings   pull /Metrics    export,
                                                      AdvFeatures     /Events          then both
                                                      = true                           flags false

 TWILIO              ·            Conference bridges       ·          writes to disk   retains 30 d
                                  both legs; metrics                  sdk_edge +       trace 10 d
                                  computed in-flight,                 carrier_edge
                                  then discarded

 PSTN                ·            caller leg;              ·          carrier_edge          ·
                                  no client telemetry                 only
```

The client detects, the server decides, and Twilio persists only while the flag is on. Twilio's row is empty under "on trip" because enabling ACI has no effect on the call that triggered it (§2.1).

## 2. Constraints

### 2.1 ACI is forward-only

Eligibility is fixed when a call is created. There is no backfill.

> "any calls placed before the feature is enabled will not be flagged by the Voice Insights infrastructure, and the per-interval metrics and event stream will not be stored" — [Voice Insights FAQ](https://www.twilio.com/docs/voice/voice-insights/frequently-asked-questions)

Querying an ineligible call returns **HTTP 401**, reported as Debugger error **17007**. Enabling ACI during a call yields no data for that call, including the remainder of it.

The call that triggers escalation is therefore never instrumented. It is diagnosed from client-side telemetry only. Escalation instruments subsequent calls.

### 2.2 The toggle is account-scoped

`POST insights.twilio.com/v1/Voice/Settings` accepts `AdvancedFeatures`, `VoiceTrace`, and an optional `SubaccountSid`. There is no per-call, per-number, or per-TwiML-app scoping. The existing per-tenant subaccount is the only available isolation boundary.

### 2.3 Voice Trace is not readable by the account

Voice Trace saves RTP packets for Twilio Support. There is no API and no Console download, retention is 10 days, and captures cannot be shared outside Twilio ([reference](https://www.twilio.com/docs/glossary/what-voice-trace)). Its only use here is as evidence attached to a carrier support ticket.

### 2.4 What the flag controls

> "Twilio does not write per-interval metrics or events to disk unless Voice Insights Advanced Features is enabled on the account… we analyze the stream of metrics and events in-flight, calculate the cumulative metrics for the call, and store those cumulative stats in the Call Summary." — Voice Insights FAQ

The SDK already posts sample data to `eventgw.twilio.com/v4/EndpointMetrics` in batches of 10 (~10 s) on every call, at no charge. Collecting the same data in the browser gives 1 Hz resolution and does not depend on the ACI flag.

## 3. Signal inventory

### 3.1 Available without ACI

Client-side, computed locally from `RTCPeerConnection.getStats()`. Verified against `twilio-voice.js` `master` @ `2dc8cb6`, tag 2.18.3: no entitlement check exists in the emit path. MOS is calculated in-browser in `lib/twilio/rtc/mos.ts`.

| Event | Cadence | Content |
|---|---|---|
| `sample` | 1 Hz (`SAMPLE_INTERVAL = 1000`) | `mos`, `rtt` ms, `jitter` ms, `packetsLostFraction`, packet counts, byte counts, `audioInputLevel`, `audioOutputLevel`, `codecName`, `totals{}` |
| `warning` / `warning-cleared` | on threshold breach | 9 app-visible warning names (§3.3) |
| `reconnecting` / `reconnected` | on media or signaling loss | `TwilioError`; **53405** media, **53001** signaling |
| `volume` | ~50 ms | `inputVolume`, `outputVolume`, floats 0.0–1.0 |
| `postFeedback(score, issue)` | agent-initiated | `dropped-call`, `audio-latency`, `one-way-audio`, `choppy-audio`, `noisy-call`, `echo` |

Server-side. None of these describe audio quality except error 32014.

| Surface | Content |
|---|---|
| Call status callbacks | `CallStatus`, `SipResponseCode`, `CallDuration`, `SequenceNumber` |
| Conference status callbacks | 13 `StatusCallbackEvent` values, `ReasonParticipantLeft`, `ParticipantCallStatus`, `ReasonConferenceEnded` |
| Monitor Alerts / `com.twilio.error-logs.error.logged` | Error codes, including **32014** "Call is terminated because of no audio received" |
| `Calls/{Sid}/Events.json` | Webhook request/response pairs only. Not SIP or ICE. Distinct from the gated Insights `/Events` |
| BulkExports | Call, Conference, Participant records beyond 30 days |

### 3.2 Requires ACI

All `insights.twilio.com` endpoints and all ten `com.twilio.voice.insights.*` Event Streams types. There is no basic Call Summary over REST; the free summary is Console-only.

ACI provides two things the browser cannot:

1. `carrier_edge` metrics for the PSTN leg
2. Conference mixer and region diagnostics (`detected_issues`, `mixer_region` vs `mixer_region_requested`)

### 3.3 SDK warning thresholds

From `lib/twilio/statsMonitor.ts` `DEFAULT_THRESHOLDS`:

| Warning | Metric | Threshold |
|---|---|---|
| `high-rtt` | `rtt` | > 400 ms |
| `high-jitter` | `jitter` | > 30 ms |
| `low-mos` | `mos` | < 3 |
| `high-packet-loss` | `packetsLostFraction` | > 1 % |
| `high-packets-lost-fraction` | `packetsLostFraction` | avg > 3 % over 7 samples |
| `low-bytes-sent` | `bytesSent` | 0 for 3 consecutive seconds |
| `low-bytes-received` | `bytesReceived` | 0 for 3 consecutive seconds |
| `constant-audio-input-level` | `audioInputLevel` | stdDev < 327.67 over 10 samples |
| `ice-connectivity-lost` | ICE state | immediate on disconnect |

Raise requires 3 breaches in the last 10 samples. Clear requires 0 of 10, plus a 5 s minimum hold (`WARNING_TIMEOUT`). All warnings are suppressed for the first 5000 ms of a call (`METRICS_DELAY`).

## 4. Client-side detection

A sidecar attached to each `Call`. Uses documented `Call` events only, with no access to SDK internals. Maintains a rolling 10-sample window, computes a quality state, and reports state changes to the server.

```
Call (Voice JS SDK)
  │ sample · warning · warning-cleared · reconnecting · disconnect
  ▼
QualitySentinel ── rolling 10 × 1 s window
  │
  ├─▶ local state   → indicator in agent UI, 1 Hz, stays in browser
  └─▶ beacon        → server, on state change only
                      + one rollup on disconnect via navigator.sendBeacon()
```

A 10-minute call produces 600 samples. Reporting only state changes yields roughly 0–5 beacons per call plus one rollup. `navigator.sendBeacon()` is required for the rollup so it survives page unload.

### 4.1 State derivation

State is derived from `sample`. Warning state is unsuitable as the direct driver for five reasons:

1. Warnings are suppressed for the first 5000 ms of every call. A call degraded from the start reports clean for 5 s.
2. Clearing requires 0 of 10 clean samples plus a 5 s hold, so recovery lags by at least 10 s.
3. `low-mos` fires at MOS < 3, while Insights tags `low_mos` at < 3.5. Client and ACI data will disagree unless one threshold is chosen.
4. `ice-connectivity-lost` emits with no second argument. `call.on('warning', (name, data) => data.threshold)` throws.
5. Thresholds are not configurable in v2.x. `StatsMonitor.Options.thresholds` is private and the dependency-injection hook is typed `new () => StatsMonitor`, which cannot carry configuration.

Warnings are still recorded and beaconed, because they correspond to the `sdk_edge` events ACI returns and serve as the join key between client and ACI data.

### 4.2 Field handling

| Field | Note |
|---|---|
| `mos` | Nullable. `null` on the first sample and before ICE completes. Guard before averaging. |
| `packetsLostFraction` | Range 0–100, a percentage. The SDK's `max: 1` threshold means 1 %. |
| `audioInputLevel`, `audioOutputLevel` | Range 0–32767, representing −100 dB to −30 dB. |
| `codecName` | Browser-derived casing (`'opus'`, `'PCMU'`). `Call.Codec` is lowercase. Normalise before comparing. |
| `localAddress`, `remoteAddress` | Documented but never delivered. `StatsMonitor._createSample` omits them. |

### 4.3 One-way audio

| Signal | Meaning | Condition |
|---|---|---|
| `low-bytes-sent` | Caller cannot hear the agent | 0 bytes sent, 3 consecutive seconds |
| `low-bytes-received` | Agent cannot hear the caller | 0 bytes received, 3 consecutive seconds |
| `audioOutputLevel < 300` with `bytesReceived > 0` | Far end is sending silence | sustained ≥ 10 s |

`low-bytes-*` also drives the SDK's own recovery: `call.ts` routes it to `_onMediaFailure(Call.MediaFailure.LowBytes)` and checks `hasActiveWarning('bytesSent', 'min')` before attempting an ICE restart.

The third row must be detected in the sentinel. RTP is flowing, so no SDK warning fires. Causes include a muted caller, carrier dead air, or hold music ending. It corresponds to server-side error 32014 observed from the other end of the conference.

The 300 floor is ~0.9 % of the 0–32767 range, matching the SDK's own `minStandardDeviation: 327.67` (1 % of range).

Unsuitable alternatives:

- `constant-audio-input-level` — suppressed while muted, requires ~10 s of samples, and measures the local microphone rather than the transmitted path. Corroboration only.
- `constant-audio-output-level` — excluded from the app-facing `emit` by an explicit guard (`if (warningName !== 'constant-audio-output-level')`). Never observable by application code.
- the `volume` event — local measurements. `inputVolume` reads the microphone, which remains active when the send path is dead.

### 4.4 Indicator states

`PreflightTest.CallQuality` bands are non-contiguous in source (`> 4.2`, `4.1–4.2`, `3.7–4.0`, `3.1–3.6`, `≤ 3.0`), so MOS 4.05 matches no branch and reports `degraded`. Use contiguous bands:

| State | Condition | Agent message |
|---|---|---|
| Good | p50 MOS ≥ 4.0 over last 10 s | none |
| Fair | p50 MOS 3.6 – 4.0 | none |
| Degraded | p50 MOS 3.1 – 3.6, or an active `high-*` warning | "Connection is struggling" |
| Bad | p50 MOS < 3.1, or `low-bytes-*`, or far-end silence | state the direction, per §4.3 |
| Reconnecting | `reconnecting` | "Reconnecting" |

`ice-connectivity-lost`, `low-bytes-*`, and `reconnecting` set the worst state immediately rather than waiting for MOS to decay.

The indicator changes on a single `high-*` warning; the beacon (§5) requires two. The indicator is local and free, so early feedback is cheap. Server-side escalation should not begin on one marginal metric.

### 4.5 Pre-shift preflight

`Device.runPreflight(token, options)` is static, free, and requires no ACI. Run it at agent login to identify networks that will not support a call, before any customer is routed.

| Report field | Use |
|---|---|
| `callQuality` | Overall verdict from average MOS. Absent if no sample had `mos > 0` |
| `isTurnRequired` | True when a selected candidate has `candidateType === 'relay'`, indicating UDP is blocked and media is being relayed through TURN |
| `warnings[]` | `{name, description, rtcWarning?}` |
| `stats.{jitter,mos,rtt}` | `{average, min, max}`. Absent if no sample had `mos > 0` |
| `networkTiming.{signaling,peerConnection,ice,dtls}` | Each `{start, end?, duration?}` |
| `selectedIceCandidatePairStats` | `{localCandidate, remoteCandidate}` |
| `iceCandidateStats[]` | Full candidate list |

`isTurnRequired` is the reason preflight is in scope rather than deferred: it is the only free source for "is UDP blocked on this agent's network", and the in-call sentinel cannot determine it without injecting a custom `RTCPeerConnection` (rejected, §14). Agent network topology rarely changes mid-shift, so answering it once per login is sufficient.

Requirements and behaviour:

- Needs a valid access token and a record-and-play TwiML application, or `<Echo/>` with `fakeMicInput: true`. With `fakeMicInput`, the test self-terminates after 20 s (`ECHO_TEST_DURATION`).
- Option defaults: `codecPreferences: ['pcmu','opus']`, `edge: 'roaming'`, `fakeMicInput: false`, `logLevel: 'error'`, `signalingTimeoutMs: 10000`.
- Events: `completed(report)`, `connected()`, `failed(TwilioError | DOMException)`, `sample(RTCSample)`, `warning(warning)`. `completed` and `failed` are mutually exclusive.
- `warning` emits a **single** argument, an object `{name, description, rtcWarning?}`. Published typings and documentation claim `(name, data)`.
- `stop()` raises `failed` with `CallCancelledError` (31008), not `completed`.
- `_getRTCStats()` discards all samples before the first with `mos > 0`.

Field-name corrections against Twilio's documentation examples: `TimeMeasurement` fields are `start`/`end`/`duration`, and network timing keys are `ice` and `dtls`. Examples showing `iceConnection`, `preflightTest`, `startTime`/`endTime`, or `warningsCleared` are from the mobile preflight schema and do not apply to the JS SDK. Passing custom `iceServers` appears inert at `2dc8cb6` (§13).

## 5. Trip conditions

The sentinel beacons `degraded` on any of:

- p50 MOS < 3.1 sustained ≥ 15 s
- `low-bytes-sent` or `low-bytes-received`
- far-end silence per §4.3, sustained ≥ 10 s
- `reconnecting`, either error code
- two or more concurrent `high-*` warnings

## 6. Enabling ACI

### 6.1 Basic

On a trip, enable ACI for the tenant subaccount. Add `VoiceTrace` only when a carrier fault is suspected and a support ticket is intended.

```bash
# GET first. POST is a partial update: sending only AdvancedFeatures=false
# leaves VoiceTrace unchanged.
curl -G https://insights.twilio.com/v1/Voice/Settings \
  --data-urlencode "SubaccountSid=AC..." \
  -u "$TWILIO_API_KEY:$TWILIO_API_SECRET"

curl -X POST https://insights.twilio.com/v1/Voice/Settings \
  --data-urlencode "SubaccountSid=AC..." \
  --data-urlencode "AdvancedFeatures=true" \
  --data-urlencode "VoiceTrace=true" \
  -u "$TWILIO_API_KEY:$TWILIO_API_SECRET"
```

SDK equivalent: `client.insights.v1.settings().update({ advancedFeatures, voiceTrace, subaccountSid })`. Use `.v1`; the unversioned `client.insights.settings` accessor is deprecated.

| Detail | Value |
|---|---|
| Credentials | Parent account, plus `SubaccountSid`. Standard API key is sufficient |
| Read current state | `GET` same path. No list endpoint exists; auditing is one request per subaccount |
| Inheritance | Parent enablement does not cascade to subaccounts |
| Partial update | Omitted flags are unchanged. Send both explicitly when disabling |
| Price | $0.0024/min, US PAYG list ([pricing](https://www.twilio.com/en-us/voice/pricing/us)) |
| Usage categories | `voice-insights`, `voice-insights-ptsn-insights-on-demand-minute` (`ptsn` is a typo in Twilio's API; match literally) |

The basic form spends on every trip, including agent-local faults that the sentinel has already diagnosed. §9 describes how to avoid that.

### 6.2 Closing the window

Close on either condition:

- root cause attributed, or recorded as inconclusive
- 5 consecutive clean calls for the tenant

In both cases, wait for the ingestion tail before disabling (§7.3), and export first (§8).

## 7. Diagnosis with ACI enabled

### 7.1 Edge availability

The two legs are separate Call SIDs with non-overlapping edge sets. Correlate them by `ConferenceSid`; `attributes.conference_participant` is the flag on the call side.

| Leg | Edges |
|---|---|
| Agent (Voice JS client) | `sdk_edge`, `client_edge` |
| PSTN caller | `carrier_edge` only |

There is no client-side MOS for a PSTN caller. On non-SDK edges, per-interval metrics carry only packets received, packets lost, and loss percentage — no MOS, jitter, or RTT.

Omitting the `Edge` parameter on `/Metrics` and `/Events` returns only the default edge for the call type. Iterate edges explicitly.

### 7.2 Attribution

| Evidence | Cause | Action |
|---|---|---|
| `sdk_edge` tags present, `carrier_edge` clean | Agent last mile | Checklist, §7.4 |
| `carrier_edge` `high_packet_loss` / `high_jitter` / `silence`, `sdk_edge` clean | Carrier | Support ticket with Voice Trace |
| `sdk_edge` tag `ice_failure` | UDP blocked or NAT | Firewall and egress rules |
| Flat high RTT, `settings.edge` vs `edge_location` mismatch | Edge homing | `new Device(token, { edge })` |
| `detected_issues.region_configuration` > 0, or `mixer_region` ≠ `mixer_region_requested` | Region configuration | Conference `region` |
| `packet_delay_variation` concentrated at `d200`/`d300` | Jitter | `jitter_buffer_size` |
| Loss present, `codec: PCMU` | Codec | §7.5 |
| `carrier_edge` `high_pdd`, `q850_cause`, `last_sip_response_num` | Routing or reject | Carrier escalation |

Tag thresholds differ by edge:

| Edge | Tags |
|---|---|
| `sdk_edge` | `high_latency` (RTT > 400 ms), `low_mos` (< 3.5), `high_jitter` (avg 5 ms + max 30 ms), `high_packet_loss` (> 5 %), `ice_failure` |
| Gateway edges | `silence`, `high_jitter`, `high_packet_loss` (> 5 %), `high_pdd` (> p95 for country), `high_latency` (internal RTP traversal > 150 ms), `pstn_short_duration` (< 10 s) |

`high_latency` and `high_jitter` have different definitions on SDK and gateway edges. Do not aggregate them.

### 7.3 Ingestion latency

| Artifact | Availability |
|---|---|
| `/Metrics`, `/Events` | ~90 s after call completion |
| `call-summary.partial` | ~10 min after call end |
| `call-summary.complete` | up to 30 min |
| Conference and participant summaries | at conference-end / participant-leave |

Event Streams type names: `com.twilio.voice.insights.call-metrics.sdk` and `.call-metrics.gateway` (plural), `.call-event.sdk` and `.call-event.gateway` (singular), `.call-summary.complete` / `.partial` / `.predicted-complete`, `.conference-summary.complete`, `.conference-participant-summary.complete`.

### 7.4 Agent-side checks

Available levers are limited, so this is a validation list rather than a set of fixes.

- Ethernet vs Wi-Fi; 2.4 GHz vs 5 GHz; channel congestion
- VPN or SD-WAN: split-tunnel media rather than routing it through the tunnel
- Proxy (for example Zscaler): media bypass
- UDP 10000–20000 egress to Twilio media IPs
- DSCP marking honoured downstream (`dscp: true` is the SDK default; the network must respect it)
- Bandwidth contention, CPU load, competing browser tabs

### 7.5 Codec

`Device.Options.codecPreferences` defaults to `['pcmu', 'opus']`, preferring PCMU. Opus is more loss-tolerant. A tenant showing packet loss while negotiated on PCMU can be changed in one line. `maxAverageBitrate` (valid 6000–510000; 8000–40000 for speech) is a secondary control.

## 8. Disabling ACI

1. Export `/Metrics`, `/Events`, and all summaries to local storage. Data availability after the flag is cleared is undocumented (§11).
2. Wait for the ingestion tail, up to 30 min after the last instrumented call ends.
3. `GET` settings, then `POST` both flags false explicitly.

Voice Trace is disabled separately, once Support has the Call SIDs. It captures raw RTP with 10-day retention, and Twilio documents consent obligations and HIPAA caveats. Calls using `<Pay>` or PCI redaction are never captured regardless of the flag.

## 9. Advanced: correlation before escalating

The basic flow in §6.1 enables ACI on any single trip. Most quality complaints are agent-local, and for those the sentinel already holds the answer, so ACI adds cost without adding information.

Correlating a tenant's concurrent calls before escalating avoids that. Correlation is free.

| Observation | Scope | Action |
|---|---|---|
| 1 agent degraded, ≥ 3 peers clean | Agent-local | Do not escalate. Advisory and ticket |
| ≥ 2 agents sharing an egress IP | Site network | Escalate |
| ≥ 30 % of active calls, across ≥ 2 sites | Tenant-wide or carrier | Escalate, with Voice Trace |
| Agents clean, but 32014 or far-end silence on ≥ 2 calls | PSTN side | Escalate, with Voice Trace |

Site is derived at ingest from the observed source IP of the beacon request. A browser cannot reliably determine its own public egress address, and a self-reported value would be incorrect behind NAT, VPN, or a proxy. Fully remote tenants will show one site per agent, which reduces the site rule to the agent-local rule.

Correlation requires at least 3 concurrent calls. Tenants below that cannot be disambiguated; fall back to escalating after 2 consecutive degraded calls, with attribution unknown until ACI data arrives.

## 10. Later considerations

Deliberately excluded from the initial scope.

### 10.1 Budget-bounded windows

ACI bills per minute across the whole subaccount, so a fixed wall-clock window costs a different amount per tenant depending on concurrency: 40 concurrent calls consume the same minutes in 25 minutes that 5 concurrent calls consume in 200. Expressing the window in billable minutes rather than elapsed time gives every tenant the same ceiling.

Indicative values at $0.0024/min: 1,000 billable minutes ≈ $2.40 per incident; 10,000 billable minutes ≈ $24 per tenant per month as a hard stop. Burn rate approximates the number of concurrent instrumented calls.

Requires per-tenant usage tracking, which the initial scope does not include.

### 10.2 Repeat-offender reporting

A short window will miss faults that recur on a longer cycle. Rather than enabling ACI permanently, count trips per tenant and report any tenant tripping 3 or more times in 7 days for manual review.

### 10.3 Threshold calibration

The thresholds in §4.4 and §5 are starting values. MOS distributions vary by codec, geography, and headset. Client-side history should be collected before the values are fixed.

## 11. Implementation phasing

| Phase | Component | Standalone value | Depends on |
|---|---|---|---|
| 1 | Sentinel, indicator, preflight (§4) | Agents see quality live, can correct one-way audio, and bad networks are caught at login. No server work | — |
| 2 | Ingest and rollups (§4) | Post-call history, per-agent and per-tenant trends, ticket attachments | 1 |
| 3 | ACI toggle and diagnosis (§6, §7, §8) | Instrumented windows, edge attribution, export | 2 |
| 4 | Correlation gate (§9) | Agent-local vs site vs carrier triage before spending | 3 |

Phases 1 and 2 have no per-minute cost. Building them first provides the data needed to calibrate thresholds against this deployment's baseline before phase 3 introduces cost.

## 12. Gotchas

- **32014 is `WARNING` level, not `ERROR`.** So are 32011, 32009, and conference-callback failure
  16011. A `LogLevel=error` filter on Monitor Alerts excludes them.
- **Event Streams mirrors configured callbacks.** `com.twilio.voice.status-callback.*` fires only if `statusCallback` and `statusCallbackEvent` are already set on the call.
- **`com.twilio.voice.webhook.status-callback.*` is deprecated.** Use the form without `webhook.`.
- **The conference `statusCallback` URL is set by the first participant to join** and cannot be overridden by later participants.
- **Monitor Alerts list and instance responses differ.** `request_variables`, `request_headers`, `response_body`, and `response_headers` are returned only on the instance resource, requiring list-then-fetch per SID.
- **Conference recordings are single-channel.** Per-leg dual-channel analysis requires separate `<Start><Recording>` per leg.
- **Alerts and Insights use different level enums** — `error|warning|notice|debug` versus `UNKNOWN|DEBUG|INFO|WARNING|ERROR`.
- **Retention:** Voice Insights 30 days, Voice Trace 10 days, Event Streams application logs 7 days.

## 13. Unverified

1. Whether captured per-interval data remains queryable after the flag is cleared. Determines whether the export step in §8 is mandatory. Highest impact.
2. Voice Trace pricing. No published price and no usage category.
3. Setting-propagation latency.
4. Whether the gate is evaluated at call creation (FAQ) or call end (error 17007). Documentation conflicts. Safe rule: enable before the call is created.
5. Retention conflict: the product page states 7 days free versus 30 days with ACI; the FAQ states 30 days.
6. Billing treatment of calls in progress when the flag is toggled.
7. Rate limits on the Settings endpoint. None published beyond platform-wide 429 / error 20429.
8. Preflight `iceServers` appears to be inert at `2dc8cb6`; `_initDevice` omits it and `rtc/peerconnection.ts` contains no occurrences. Default-run `isTurnRequired` is unaffected.
9. Event Streams participant-summary naming: the event-type page states `conference-participant-summary.complete`, a guide page states `participant-summary.complete`. Prefer the former.

Separately: `_checkVolume` in `call.ts` contains a defect (`newStreak = currentStreak` never increments, so its emit branch is unreachable; unchanged since 2019). It is redundant code. `constant-audio-input-level` still fires through the `minStandardDeviation` path, so one-way-audio detection is not broken.

## 14. Rejected alternatives

| Alternative | Reason |
|---|---|
| Per-call ACI enablement | Does not exist in the API |
| Enabling ACI mid-call to diagnose the current call | Forward-only; see §2.1 |
| Dedicated diagnostic subaccount with ACI always on, routing suspect traffic there | Subaccount and phone-number topology is impractical to change in a deployed platform |
| ACI always on for flagged tenants | Per-minute cost is not viable at current margins. Replaced by §10.2 |
| Custom `RTCPeerConnection` injection to read ICE candidate pairs | Adds SDK coupling. `isTurnRequired` from preflight (§4.5) covers most cases |
| Driving the indicator from SDK warning state | See §4.1 |
| `PreflightTest.CallQuality` bands | Non-contiguous in source; see §4.4 |

## 15. References

**Voice Insights**
- [Voice Insights FAQ](https://www.twilio.com/docs/voice/voice-insights/frequently-asked-questions)
- [Settings resource](https://www.twilio.com/docs/voice/voice-insights/api/call/voice-insights-settings-resource)
- [Call Metrics resource](https://www.twilio.com/docs/voice/voice-insights/api/call/call-metrics-resource)
- [Call Events resource](https://www.twilio.com/docs/voice/voice-insights/api/call/call-events-resource)
- [Call tags](https://www.twilio.com/docs/voice/voice-insights/api/call/details-call-tags)
- [Conference Summary resource](https://www.twilio.com/docs/voice/voice-insights/api/conference/conference-summary-resource)
- [Call Insights Event Streams](https://www.twilio.com/docs/voice/voice-insights/event-streams/call-insights-events)
- [What is Voice Trace?](https://www.twilio.com/docs/glossary/what-voice-trace)
- [Error 17007](https://www.twilio.com/docs/api/errors/17007)

**Voice JS SDK**
- [twilio-voice.js](https://github.com/twilio/twilio-voice.js) — `master`, tag 2.18.3
- [Call object](https://www.twilio.com/docs/voice/sdks/javascript/twiliocall)
- [Device object](https://www.twilio.com/docs/voice/sdks/javascript/twiliodevice)
- [PreflightTest](https://www.twilio.com/docs/voice/sdks/javascript/twiliopreflighttest)
- [SDK call-quality events](https://www.twilio.com/docs/voice/voice-insights/api/call/details-sdk-call-quality-events)
- [SDK error codes](https://www.twilio.com/docs/voice/sdks/error-codes)

**Free server-side surfaces**
- [Call resource](https://www.twilio.com/docs/voice/api/call-resource)
- [`<Conference>` TwiML](https://www.twilio.com/docs/voice/twiml/conference)
- [Monitor Alerts](https://www.twilio.com/docs/usage/monitor-alert)
- [Error logs Event Stream](https://www.twilio.com/docs/events/event-types/errors/error-logs)
- [BulkExports](https://www.twilio.com/docs/usage/bulkexport)
