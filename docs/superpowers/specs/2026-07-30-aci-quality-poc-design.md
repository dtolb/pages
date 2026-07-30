# ACI quality POC — demo application design

**Date:** 2026-07-30
**Status:** Design approved, pending implementation plan
**Repo:** `~/code/aci-quality-poc` (new), Docker label `app=aci-poc` → `https://aci-poc.twilio.dtolb.com`
**Proves:** the strategy in [2026-07-30-call-quality-escalation-design.md](2026-07-30-call-quality-escalation-design.md)

---

## 1. Purpose and scope

A proof of concept that client-side Voice JS SDK quality signals, plus a call-centre agent's subjective feedback, can be used to enable Voice Insights Advanced Features (ACI) on demand and then consume the resulting data.

This will never run in production. It exists to be demonstrated. The code should be simple enough to read in one sitting, and the boundary between Twilio integration code and POC-specific logic must be obvious.

Deliberately **out of scope**, having been settled in the parent spec: budget-bounded instrumentation windows, and the cross-call correlation gate. ACI is armed gratuitously here — a single trip is enough.

Throughout this document, **CCA** means call-centre agent.

## 2. Constraints inherited from the parent spec

| Constraint | Effect on the demo |
|---|---|
| ACI eligibility is fixed at call creation, no backfill | The call that trips the beacon is never instrumented. Demo makes this visible rather than hiding it. |
| The ACI toggle is account-scoped | Single demo account, so this is a non-issue here. |
| Voice Trace is Support-only | Not exposed in the demo. |
| MOS exists only on `sdk_edge` | Caller-side MOS requires Conference Insights, not `carrier_edge`. |

## 3. Constraints discovered for this build

### 3.1 Codec cannot change mid-call

`codecPreferences` is frozen at PeerConnection construction — the SDK calls `transceiver.setCodecPreferences()` before the offer exists, and there is no public renegotiation API. `device.updateOptions({ codecPreferences })` during an active call does **not** throw; it silently has no effect on that call.

It is also not accepted by `device.connect()`. `Device.ConnectOptions` carries only `connectToken` and `params`.

Workaround used here (§8.4): disconnect, `updateOptions()`, reconnect into the same conference.

The SDK default is `['pcmu', 'opus']` — **PCMU is preferred unless overridden.**

### 3.2 The Annotations API is gated behind ACI

Writing an annotation requires Advanced Features, and the same forward-only rule applies. Error **17007**: *"Voice Insights Advanced Features not enabled on the account when the call was made."*

So CCA feedback on the very call that tripped the beacon cannot reach Annotations. The demo attempts it anyway and renders the real error (§8.3).

Other verified specifics:

- Annotation works **during** a call, not only after: *"Developers can update the Call Summary records with Annotation during or after a Call."*
- `CallScore` is **1–5 where 5 is Excellent**.
- `QualityIssues` is a comma-separated string on write, array on read. Valid: `no_quality_issue`, `low_volume`, `choppy_robotic`, `echo`, `dtmf`, `latency`, `owa`, `static_noise`. `owa` is one-way audio. **`no_audio` is not valid.**
- `ConnectivityIssue`: `unknown_connectivity_issue`, `no_connectivity_issue`, `invalid_number`, `caller_id`, `dropped_call`, `number_reachability`.
- `AnsweredBy` on this resource is only `human` / `machine` — **not** the AMD enum.
- `Comment` and `Incident` are capped at **100 characters**.
- **US1 region only.**
- The gating failure is reported inconsistently: the error catalog documents **17007**, while the Voice Insights FAQ says the same condition yields **HTTP 401**. The annotate wrapper must handle either and surface whichever arrives.

### 3.3 ACI events lag ~90 seconds

| Event type | Latency |
|---|---|
| `call-metrics.sdk` / `.gateway` | generated every 10 s, *emitted* typically within 90 s |
| `call-event.sdk` / `.gateway` | typically within 90 s |
| `call-summary.complete` | up to 30 minutes |

Client SDK events arrive at 1 Hz instantly. The two feeds can never sit adjacent in real time, so the Events pane surfaces the delay explicitly (§7.3) rather than appearing broken.

### 3.4 Conference `statusCallback` is set by the first participant

*"The `statusCallback` URL is set by the first Participant to join the conference, subsequent setting of the `statusCallback` will be ignored."* The same first-wins rule applies to `statusCallbackEvent`.

Because the caller dials in first, the inbound leg sets it. Both legs therefore send an **identical superset** event list, so either join order produces the same configuration. Getting this wrong means `participant-join` never fires and the `ConferenceSid` never arrives.

### 3.5 Reference implementation exists

[`twilio/twilio-voice-js-reference-components`](https://github.com/twilio/twilio-voice-js-reference-components) contains a `twilio-voice-monitoring` component: browser agent plus REST-dialled PSTN participant in a 1:1 conference, with conference status callbacks, participant labels, `sample`/`warning` logging, and `userDefinedMessages` used to push the `ConferenceSid` to the browser. Mine it for patterns before writing our own.

## 4. Architecture

### 4.1 The dependency rule

> Code in a `twilio/` directory may import the Twilio SDK and knows nothing about the POC. Code in `poc/` or `sentinel/` may **not** import the Twilio SDK, and consumes only normalized types from `shared/types.ts`.

The POC logic depends on an interface; the Twilio layer implements it. Consequence: `sentinel/` is a pure function of sample data and is unit-testable from fixtures with no browser, no SDK, and no Twilio account.

### 4.2 Layout

```
aci-quality-poc/
├── server/
│   ├── twilio/                  ← may import 'twilio'
│   │   ├── token.ts               AccessToken + VoiceGrant
│   │   ├── twiml.ts               <Dial><Conference> for both legs
│   │   ├── webhooks.ts            /voice-inbound, /voice-agent, /conference-status
│   │   ├── aci.ts                 POST /v1/Voice/Settings
│   │   ├── annotations.ts         POST /v1/Voice/{sid}/Annotation
│   │   ├── insights.ts            Conference participant summaries
│   │   ├── event-streams.ts       Sink + Subscription bootstrap
│   │   ├── sink-handler.ts        CloudEvents parse + signature validation
│   │   └── port.ts                ← the interface poc/ depends on
│   ├── poc/                     ← may NOT import 'twilio'
│   │   ├── policy.ts              arm/disarm decision
│   │   ├── thresholds.ts          beacon matrix config
│   │   ├── store.ts               node:sqlite
│   │   └── stream.ts              SSE fan-out
│   └── routes.ts                ← the only file that wires both
├── web/
│   ├── twilio/                  ← may import '@twilio/voice-sdk'
│   │   ├── device.ts              Device lifecycle, codec swap-and-rejoin
│   │   └── adapt.ts               SDK events → normalized types
│   ├── sentinel/                ← may NOT import the SDK
│   │   ├── window.ts              rolling 10 × 1 s
│   │   ├── verdict.ts             state machine
│   │   └── beacon.ts              transition detection
│   └── ui/                        softphone, events pane, config, survey
├── shared/types.ts              ← the contract
└── docs/CONSOLE-SETUP.md        ← manual setup walkthrough (§11)
```

### 4.3 The contract

`shared/types.ts` encodes the field hazards once, at the seam, so they cannot be reintroduced downstream:

```ts
type QualitySample = {
  ts: number
  mos: number | null    // nullable — null on the first sample and pre-ICE
  rttMs: number
  jitterMs: number
  lossPct: number       // 0–100, a percentage, NOT 0–1
  audioInLevel: number  // 0–32767
  audioOutLevel: number
  bytesSent: number
  bytesReceived: number
  codec: string         // browser casing: 'opus' | 'PCMU'
}

type QualityWarning = {
  ts: number
  name: string
  raised: boolean
  threshold?: { name: string; value: number }  // absent for ice-connectivity-lost
}
```

`web/twilio/adapt.ts` is the only place that touches raw SDK payloads.

Server-side, `poc/` reaches Twilio solely through:

```ts
interface TwilioPort {
  getAciState(): Promise<{ advancedFeatures: boolean }>
  setAci(on: boolean): Promise<{ advancedFeatures: boolean }>
  annotate(callSid: string, a: Annotation): Promise<AnnotateResult>  // captures 17007
  participantSummary(cfSid: string, cpSid: string): Promise<ParticipantMetrics>
  setJitterBuffer(callSid: string, size: JitterBufferSize): Promise<void>
}
```

### 4.4 Stack

Node 24 + pnpm + Hono on the server; React + Vite on the web side; `node:sqlite` for storage (built in, zero dependencies). Dockerized with an `app=aci-poc` label for the Traefik dev box, which supplies the stable public HTTPS URL that the TwiML App, conference callbacks, and Event Streams sink all require.

The browser app is served from the same origin. Note that `getUserMedia` needs a secure context; `https://aci-poc.twilio.dtolb.com` satisfies it, as would `http://localhost` — but **not** `http://<LAN-IP>`.

## 5. Call topology

Inbound: a phone dials the Twilio number, waits, and the CCA answers into the conference.

```
 CALLER            TWILIO                 SERVER                    BROWSER
   │                 │                      │                         │
 dials ─────────────▶│                      │                         │
   │            /voice-inbound ────────────▶│                         │
   │            <Conference> ◀──────────────│  superset statusCallback
   │              waitUrl, waiting           │  set here — caller is first
   │                 │   participant-join ──▶│ ──── SSE ─────────────▶│ "caller waiting"
   │                 │                      │                         │
   │                 │                      │◀──── POST /token ───────│ CCA clicks Answer
   │                 │◀── Device.connect() ─────────────────────────── │
   │            /voice-agent ───────────────▶│                         │
   │            <Conference> same room ◀────│                         │
   │◀══ bridged ════▶│                      │                         │
   │                 │                      │        sample @1Hz ─────│ sentinel
   │                 │                      │◀─ POST /beacon ─────────│ transitions only
   │                 │                      │   policy → setAci(true)  │
   │                 │  ~90s later          │                          │
   │            Event Streams ─────────────▶│ /aci-sink                │
   │              (next call only)          │ ──── SSE ───────────────▶│ ACI events
```

Conference attributes: `startConferenceOnEnter=true`, `beep=false`, `waitUrl` set for the caller so they hear something while waiting, `maxParticipants=2`, `participantLabel` (`caller` / `cca`), and the identical `statusCallback` + superset `statusCallbackEvent` on both legs.

**`endConferenceOnExit=false` on the CCA leg** — required for the codec swap in §8.4 to work, since the conference must survive the agent briefly leaving.

A conference does not start until two participants are present, and is in `init` status until then. `ConferenceSid` is obtained from the `participant-join` callback, which is the only push carrying `ConferenceSid` and `CallSid` together. `ParticipantSid` (`CP…`) is **not** a conference webhook parameter; it exists only in the Voice Insights Conference API, which matters for §8.6.

## 6. Beacon configuration

Server owns the config so it persists and records what tripped; the client evaluates it so it fires instantly. Pushed to the browser on connect and on every edit.

| Metric | Op | Default | Sustain | Enabled |
|---|---|---|---|---|
| `mos` | `<` | 3.1 | 15 s | yes |
| `jitterMs` | `>` | 30 | 10 s | yes |
| `lossPct` | `>` | 5 | 10 s | yes |
| `rttMs` | `>` | 400 | 10 s | no |

Combinator: `ANY` (default) / `ALL` / `N-of-M`, where the user sets **N** and **M is the count of enabled rows**. Defaults are deliberately loose per the gratuitous-triggering decision.

Verdict states are inherited from the parent spec §4.4 — `good` / `fair` / `degraded` / `bad` / `reconnecting` — with the same contiguous MOS bands, so the POC and the strategy document cannot drift apart.

**Always on, not configurable** because they are categorical rather than threshold-based: `low-bytes-sent`, `low-bytes-received`, `ice-connectivity-lost`, `reconnecting` (branch on **53405** media vs **53001** signaling).

**Subjective row:** CCA pressing "quality is poor" arms ACI immediately. This is the POC's central claim — a human judgement drives the same instrumentation a metric threshold does.

Note the client sentinel derives its verdict from `sample`, not from SDK warning state, for the reasons in the parent spec §4.1 — warnings are suppressed for the first 5000 ms of a call and take at least 10 s to clear.

## 7. Event model and the Events pane

### 7.1 Envelope

```ts
type PaneEvent = {
  id: string
  callSid: string
  source: 'sdk' | 'sentinel' | 'server' | 'aci'
  name: string
  level: 'info' | 'warning' | 'error'
  ts: number          // when it happened
  receivedTs: number  // when we learned about it — carries the lag
  edge?: 'sdk_edge' | 'client_edge' | 'carrier_edge'
  payload: unknown
}
```

| Source | Origin | Latency | Exists with ACI off? |
|---|---|---|---|
| `sdk` | Raw SDK: warnings, reconnecting, preflight report | instant | yes |
| `sentinel` | POC-derived: verdict changes, beacon fired | instant | yes |
| `server` | Server actions: ACI armed/disarmed, annotation attempted | instant | yes |
| `aci` | Event Streams sink | ~90 s | **no** |

### 7.2 Samples are not rows

A 10-minute call emits 600 samples, and `call-metrics.*` fire every 10 s per call per edge. Rendering those as log lines makes the pane useless.

> Samples drive the **meters and sparklines**. Only discrete events become **rows**: warnings raised/cleared, verdict transitions, beacon fires, reconnects, ACI arm/disarm, annotation attempts, preflight completion, summaries.

### 7.3 Two-lane timeline

```
┌─ CALL #2  CAxxxx…8f2   codec opus   [INSTRUMENTED] ──────────────────────┐
│  MOS ▁▂▃▅▆▆▅▃▂▁▁▂  2.9    jitter 44ms    loss 12%    rtt 88ms           │
│                                                                          │
│  t       CLIENT · live                    │  ACI · via Event Streams     │
│ ─────────────────────────────────────────┼──────────────────────────────│
│  00:00   ● call.accepted                  │                              │
│  00:05   ⚠ high-jitter          raised    │                              │
│  00:08   ◆ verdict  fair → degraded       │                              │
│  00:11   ▲ BEACON  mos<3.1 · loss>5%      │                              │
│  00:11   ⬤ server  aci.armed              │                              │
│  00:14                                    │  ○ call-event.sdk  low-mos   │
│                                           │    sdk_edge      ⟲ +92s      │
│  00:20                                    │  ○ call-metrics  carrier_edge│
│                                           │    jitter 38ms   ⟲ +94s      │
│  00:31   ✓ high-jitter          cleared   │                              │
│  01:02   ● call.disconnected              │                              │
│  01:02   ⬤ server  annotation → 201 OK    │                              │
│  ─── call ended ───                       │  ○ call-summary.complete     │
│                                           │    ⟲ +18m  MOS avg 3.1       │
└──────────────────────────────────────────────────────────────────────────┘
```

Rationale for two lanes over a merged list or tabs:

- Source is **positional**, not merely colour-coded.
- An un-instrumented call renders with a **visibly empty ACI lane** and a `[NOT INSTRUMENTED]` badge, so the forward-only constraint needs no explanation.
- `⟲ +92s` makes the lag a visible property. The row sits at its true `ts` but is annotated with arrival delay.
- Edge is labelled on ACI rows, because a `carrier_edge` row showing only jitter and loss is complete — that edge has no MOS or RTT.

Rejected: a single merged list (retroactively inserting 90-second-old rows into a scrolled list is disorienting) and Client/ACI tabs (destroys the correlation that motivates the build).

### 7.4 Call grouping and late arrivals

Events group per call, newest first, each with an instrumentation badge. ACI events landing for a call the user is not viewing update a quiet count pill rather than moving the view:

```
┌─ CALL #1  CAxxxx…3a1   codec PCMU   [NOT INSTRUMENTED]      ⊕ 0 ACI ──┐
┌─ CALL #2  CAxxxx…8f2   codec opus   [INSTRUMENTED]          ⊕ 7 ACI ──┐
```

`⊕ 0 ACI` on call #1 is the point, not an omission.

Transport is SSE, server → browser, one-way. Beacons go the other way as ordinary POSTs.

## 8. Features

### 8.1 CCA softphone

Voice JS SDK Device. Outbound-only from the browser's perspective (the CCA joins a conference), so `register()` is not required and `incomingAllow` is unnecessary. A TwiML App **is** required — `outgoingApplicationSid` is mandatory for the Voice SDK, and there is no way to put a raw URL in the grant.

Token: `new AccessToken(accountSid, apiKeySid, apiKeySecret, { identity, ttl })`. `identity` is required by the constructor and may contain only alphanumerics and underscores. Default TTL 3600 s; `tokenWillExpire` fires 10 s before expiry by default, so raise `tokenRefreshMs` for demo comfort.

### 8.2 ACI arm/disarm

| | |
|---|---|
| Arm on | any beacon trip · CCA poor-quality button · manual toggle |
| Disarm on | manual toggle · optional idle timer, default **30 minutes** with no active call (off by default; a demo left armed overnight bills for nothing) |
| Guards | `GET` state first, skip the write if already armed; always send **both** `AdvancedFeatures` and `VoiceTrace` explicitly, since POST is a partial update |
| Recorded | every transition to the store and the pane as `server` / `aci.armed` |

State is displayed persistently as `ACI ● ARMED` / `● DISARMED`.

Optional corroboration, cheap because it is one read: Twilio's audit log records every toggle as event type `voice-insights-account-flags.updated`, carrying the timestamp, actor, source, and previous/updated values — `GET https://monitor.twilio.com/v1/Events?EventType=voice-insights-account-flags.updated`. Useful for proving on Twilio's own record that the arm happened when the demo says it did.

### 8.3 Subjective feedback

| Channel | When | Destination |
|---|---|---|
| "Quality is poor" button | mid-call | beacon trip → arm ACI · Annotation attempt · store |
| Post-call survey | on disconnect | Annotation attempt · `call.postFeedback()` · store |
| `postFeedback()` | post-call | SDK → Insights `feedback` group, visible in `sdk_edge`. **Works with ACI off** |

`postFeedback` is included specifically because it survives ACI being off, giving a free subjective channel beside the gated one. Its issue enum: `dropped-call`, `audio-latency`, `one-way-audio`, `choppy-audio`, `noisy-call`, `echo`.

Survey → Annotation mapping:

```
score 1–5        → CallScore          5 = Excellent; survey worded to match
issue checkboxes → QualityIssues      comma-separated from low_volume,
                                      choppy_robotic, echo, dtmf, latency,
                                      owa, static_noise
comment          → Comment            truncated to 100 chars;
                                      full text kept in the POC store
dropped?         → ConnectivityIssue  dropped_call | no_connectivity_issue
POC record id    → Incident           Twilio's record points back at ours
```

Every annotation is **attempted regardless** of instrumentation state, and the real response is rendered — `201` on an instrumented call, **`17007`** otherwise. The failure is part of the demo. The UI shows a character counter on the comment field and states that it truncates.

### 8.4 Codec switch

Control offers `opus` / `pcmu`. On change during a call: `disconnect()` → `device.updateOptions({ codecPreferences })` → `connect()` into the same conference room. The conference survives because `endConferenceOnExit=false` on the CCA leg and the caller is still present. The UI warns that this briefly drops and rejoins; the caller hears a 1–2 s gap.

Result: one conference, two CCA Call SIDs, **two Call Summaries on different codecs**, rendered as two segments under the same conference for comparison.

`updateOptions()` destroys and recreates the Insights event publisher, so prefer calling it between calls — which this flow does by construction.

`call.codec` is populated from the first sample, roughly 1 s after answer, not at connect time.

### 8.5 Preflight

A "Test my connection" button runs `Device.runPreflight(token)` and writes the report into the Events pane as `sdk` / `preflight.completed`: `callQuality`, `isTurnRequired`, `warnings[]`, `stats.{mos,jitter,rtt}`, `networkTiming`.

Free, requires no ACI. `isTurnRequired` (a selected candidate with `candidateType === 'relay'`, meaning UDP is blocked) is the one signal the in-call sentinel cannot obtain without injecting a custom `RTCPeerConnection`.

Requires a record-and-play TwiML app or `<Echo/>` with `fakeMicInput: true`. Note `report.stats` and `report.callQuality` are both absent if no sample had `mos > 0`, and the `warning` event emits a **single object**, not `(name, data)` as the typings claim.

### 8.6 Conference Insights for caller-side MOS

After the call, fetch `GET /v1/Conferences/{ConferenceSid}/Participants/{ParticipantSid}` for the caller's participant. This returns `metrics.inbound/outbound` including **MOS for a `call_type: carrier` participant** — which `carrier_edge` on the call-level Summary cannot provide.

`ParticipantSid` is not in any conference webhook, so it must be discovered by listing participants via the Insights Conference API.

### 8.7 Quality-degradation panel

The demo is not demonstrable unless a call can be made bad on cue.

**Server-side lever:** `jitterBufferSize` toggle (`off` / `small` / `medium` / `large`, default `large`) applied to the caller's participant via REST. Setting it `off` or `small` exposes real network jitter instead of smoothing it, and shows up on `carrier_edge` metrics.

**Client-side lever, documented in the panel:** a Chrome DevTools custom throttling profile. Its `packetLoss`, `packetQueueLength`, and `packetReordering` fields are explicitly WebRTC-targeted, unlike `latency`/throughput which only affect HTTP requests.

Verified effect on the SDK's MOS, which is computed purely from RTT, jitter, and loss:

| Packet loss | MOS | Warnings fired |
|---|---|---|
| 1 % | 4.36 | `high-packet-loss` |
| 5 % | 4.05 | both loss warnings |
| 10 % | 3.51 | both loss warnings |
| **15 %** | **2.88** | both + **`low-mos`** |
| 20 % | 2.23 | both + `low-mos`, clearly audible |

So ~5 % for "degrading", ~15 % to break MOS below 3. Small `packetQueueLength` induces jitter past the 30 ms threshold. `high-rtt` needs >400 ms, which DevTools packet fields cannot supply — that needs Network Link Conditioner.

The panel explicitly documents what does **not** work, to prevent wasted demo time: `maxAverageBitrate` (perceptual only, Opus only, MOS unmoved), codec choice (perceptual only, though it does change `codec_name`), and `iceTransportPolicy: 'relay'` (the SDK supplies no ICE servers, so this yields a failed call rather than a degraded one).

## 9. Storage

`node:sqlite`, three tables:

```
calls     call_sid PK, conference_sid, leg, codec, instrumented,
          started_at, ended_at, verdict_worst
            leg           'cca' | 'caller'
            codec         'opus' | 'PCMU' | null   (null until first sample)
            instrumented  0 | 1
            verdict_worst 'good'|'fair'|'degraded'|'bad'|'reconnecting'

events    id PK, call_sid, source, name, level, ts, received_ts,
          edge, payload_json
            source        'sdk' | 'sentinel' | 'server' | 'aci'
            id            CloudEvents id for aci rows — the dedupe key

feedback  id PK, call_sid, kind, score, issues, comment_full,
          comment_sent, annotation_status, annotation_error
            kind          'in_call_button' | 'post_call_survey'
            score         1..5  (5 = Excellent, matching CallScore)
```

`instrumented` is recorded at call creation from the then-current ACI state — that single column drives the pane's badge and explains every annotation outcome. `annotation_status` / `annotation_error` preserve the 17007 (or 401) outcomes for post-hoc inspection.

Two `calls` rows share a `conference_sid` after a codec switch (§8.4), which is exactly what makes the side-by-side codec comparison a plain SQL query. Using the CloudEvents `id` as the `events` primary key makes at-least-once delivery idempotent for free — a duplicate insert conflicts and is dropped.

## 10. Setup script

`pnpm setup:twilio` — idempotent, safe to re-run, and the primary provisioning path.

### 10.1 Interaction model

Four phases, in order. Nothing is written to the Twilio account before the confirmation prompt.

```
READ  →  PLAN  →  CONFIRM  →  APPLY  →  REPORT
```

**READ.** Fetch the current state of every resource the app needs. Read-only; no side effects.

**PLAN.** Diff desired against actual and print every item with its disposition — `✓` already correct, `+` will create, `~` will update, `!` will overwrite a value someone else set. Nothing is hidden: items needing no action are still listed, so the output is a complete picture rather than only a delta.

**CONFIRM.** Print the change count and prompt, **defaulting to No**. Skipped entirely when the plan is empty — there is nothing to confirm, so it reports "already configured, nothing to do" and exits 0. `--yes` skips the prompt for re-runs; `--dry-run` stops after PLAN.

**APPLY.** Execute only the planned changes, printing each result as it completes. Then re-verify and print the resolved `.env` values.

```
$ pnpm setup:twilio

Reading current Twilio state…

PLAN — 2 changes, 4 already correct

  ✓ Credentials          ACxxxx…  us1  (Standard key — can mint AccessTokens)
  ✓ ACI state            advancedFeatures=false      ← left as-is by design
  ! TwiML App            APxxxx…  'aci-poc-agent'
      Request URL        https://old.example.com/voice
                      →  https://aci-poc.twilio.dtolb.com/voice-agent
  ✓ Phone number         +1512xxxxxxx  →  …/voice-inbound
  + Event Streams sink   webhook  →  https://aci-poc.twilio.dtolb.com/aci-sink
  · Subscription         pending — resolves after the sink exists

  ! 1 change overwrites an existing value.

Apply 2 changes? [y/N] y

APPLYING

  ~ TwiML App            APxxxx…  Request URL updated
  + Event Streams sink   DGxxxx…  status active
  + Subscription         DFxxxx…  5 types
      call-event.sdk         schema 3
      call-event.gateway     schema 3
      call-metrics.sdk       schema 2
      call-metrics.gateway   schema 2
      call-summary.complete  schema 8
  ✓ Sink smoke test      POST /Sinks/DGxxxx…/Test → received in 240ms

RESULT — 2 applied, 0 failed

  TWILIO_TWIML_APP_SID=APxxxx…
  ACI_SINK_SID=DGxxxx…
  ACI_SUBSCRIPTION_SID=DFxxxx…
```

Three details that matter for the implementation. **The plan is partially unknowable on a first run** — subscription state cannot be determined until the sink exists, so it is honestly marked `pending` rather than guessed at. **The `!` marker is reserved for overwriting a value the script did not set**, which is the only genuinely surprising action here (repointing a TwiML App or phone number that was aimed elsewhere) and is called out separately above the prompt. And confirmation uses `node:readline/promises`, keeping the zero-dependency posture already chosen for storage.

### 10.2 What it reconciles

| Item | Desired state | Idempotency check |
|---|---|---|
| Credentials | valid, Standard key type | `GET /v1/Voice/Settings`; fail fast and readably |
| ACI state | **reported only, never changed** | read-only |
| TwiML App | exists, Request URL → `/voice-agent` | match friendly name `aci-poc-agent` |
| Phone number | inbound Voice URL → `/voice-inbound` | match the configured number |
| Event Streams sink | webhook sink → `/aci-sink` | match on `Description` |
| Subscription | the five pinned types at pinned versions | reconcile via `SubscribedEvents`, don't recreate |
| Sink health | test event received | `POST /Sinks/{Sid}/Test` |

**The ACI row is deliberately read-only.** The script must never enable Advanced Features, because "call 1 is un-instrumented" is the demo's opening move. If setup armed ACI, the first and most important lesson would be impossible to show.

### 10.3 What it deliberately cannot do

| Not automated | Why |
|---|---|
| Create the API key it authenticates with | Chicken-and-egg. Standard keys are creatable via REST, but only using the Auth Token or a Main key — and Main keys cannot be created via API at all. Cleaner to require `TWILIO_API_KEY` / `_SECRET` in `.env`. |
| Buy a phone number | Spends money. The script verifies and configures a number you already own. |
| Enable ACI | See above — that is the demo's job. |
| Create the Chrome DevTools throttling profile | Browser-local, not an API surface. |

Note also that **Standard** is the required key type: *"You can create Access Tokens using Main and Standard API Keys. Creating Access Tokens is not yet supported with Restricted API Keys."* A Restricted key will fail at token minting, which is a confusing failure mode worth asserting against at step 1.

### 10.4 Pinned schema versions

Versions are pinned to values verified against `/v1/Schemas/{Id}/Versions`. The Webhook Quickstart still documents `schema_version: 1` for call-summary — stale by seven versions, and a stale-but-valid version **silently delivers a degraded payload rather than erroring**, so this is asserted rather than defaulted.

Sink creation: `SinkType=webhook`, `SinkConfiguration={"destination":…,"method":"POST","batch_events":false}`. **No validation step** — sinks are `active` on creation and the Validate resource is deprecated (*"There is no longer a need for validating a Sink"*). `Types` on the subscription is a **repeated** parameter whose values are JSON *strings* with snake_case inner keys `type` and `schema_version`; writing `schemaVersion` there is silently wrong because the object is serialized verbatim onto the wire.

The Console also has a working UI for sinks and subscriptions that does support webhook sinks, so the script is a convenience rather than the only path — which is what makes the deferred walkthrough in §11 viable.

### 10.5 Sink handler requirements

Three non-obvious behaviours, each of which silently breaks the integration if missed:

1. **Parse the raw body.** Signature validation is the JSON variant: HMAC-SHA1 over the URL *including* the `bodySHA256` query parameter, with no POST-parameter concatenation. Re-serializing the JSON breaks the hash. Use `validateRequestWithBody(authToken, signature, url, rawBody)` — note signature precedes url.
2. **Accept `data` as string or object.** Twilio's own docs contradict each other on this. `typeof e.data === 'string' ? JSON.parse(e.data) : e.data`.
3. **Never return a non-429 4xx.** A bare `400` or `404` causes Twilio to **permanently discard** the event. Return `429` for backpressure, `5xx` for real faults, `200` even for duplicates. Ack within 5 seconds and process asynchronously.

Also: delivery is at-least-once and may be out of order. Dedupe on the CloudEvents `id`; sort by embedded timestamp.

## 11. Manual console walkthrough — deferred to the final step

`docs/CONSOLE-SETUP.md`, a hand-provisioning companion so the demo can be set up and understood without trusting the script. **Written as the last implementation task**, once the app's real URLs, env var names, and failure modes are settled — writing it earlier means rewriting it.

Scope when written: Standard API key creation (Standard is required — Restricted cannot mint AccessTokens), phone number purchase and inbound webhook wiring, TwiML App creation and its Request URL field, the Advanced Features toggle location, Event Streams sink and subscription via the Console UI, where to read error 17007 and sink delivery failures, and the Chrome DevTools throttling profile.

Console navigation paths have already been verified and are recorded in this session's research; three cautions carry into that document. Twilio's canonical inbound-voice tutorial has a **documentation bug** — its current-Console tab describes the *messaging* configuration in a voice tutorial, so the legacy breadcrumb is the verified one. The Voice Request URL field has **three different documented names** across pages. And direct Console URLs embed a region segment plus a literal `__account__` placeholder token that is pasted as-is rather than substituted.

## 12. Demo script

```
1. Preflight            clean baseline, isTurnRequired=false        → events pane
2. Phone → Twilio #     caller waits, CCA answers                   ACI: DISARMED
3. CALL 1               DevTools 15 % loss → MOS ~2.9
                        warnings fire · verdict → bad · BEACON      → ACI ARMED
                        ACI lane stays EMPTY                        ← lesson 1
                        "quality poor" → Annotation → 17007         ← lesson 2
4. Post-call survey     Annotation → 17007 again (call predates ACI)
5. CALL 2               [INSTRUMENTED] · same degradation
                        client events instant
                        ~90 s later → carrier_edge lands in ACI lane
6. Codec switch         opus ⇄ PCMU, rejoin, two summaries compared
7. Post-call survey     Annotation → 201 OK
8. Conference Insights  caller-side MOS (carrier_edge cannot give it)
9. Disarm
```

Steps 3 and 4 are the moments that justify the strategy, and both are failures rendered honestly.

## 13. Non-goals

- Production readiness, authentication, multi-agent support, or multi-tenancy.
- Budget-bounded windows and the correlation gate (parent spec §10.1 and §9 respectively).
- Voice Trace.
- Tests beyond unit tests for `sentinel/`, which is the only component with logic worth asserting and is deliberately built to be testable in isolation.

## 14. Deliverables

| Path | Contents | When |
|---|---|---|
| `docs/superpowers/specs/2026-07-30-aci-quality-poc-design.md` | this document (pages repo) | now |
| `~/code/aci-quality-poc/` | the application | implementation |
| `~/code/aci-quality-poc/docs/CONSOLE-SETUP.md` | manual walkthrough (§11) | **final implementation task** |

## 15. References

**Parent spec**
- [Call-quality detection and ACI escalation](2026-07-30-call-quality-escalation-design.md)

**Twilio APIs used**
- [Voice Insights Settings resource](https://www.twilio.com/docs/voice/voice-insights/api/call/voice-insights-settings-resource)
- [Call Annotation resource](https://www.twilio.com/docs/voice/voice-insights/api/call/call-annotation-resource)
- [Conference Participant Summary](https://www.twilio.com/docs/voice/voice-insights/api/conference/conference-participant-resource)
- [Event Streams Sink resource](https://www.twilio.com/docs/events/event-streams/sink-resource)
- [Event Streams Subscription resource](https://www.twilio.com/docs/events/event-streams/subscription)
- [Call Insights Event Streams types](https://www.twilio.com/docs/voice/voice-insights/event-streams/call-insights-events)
- [`<Conference>` TwiML](https://www.twilio.com/docs/voice/twiml/conference)
- [Conference Participant REST resource](https://www.twilio.com/docs/voice/api/conference-participant-resource)
- [Access Tokens](https://www.twilio.com/docs/iam/access-tokens)
- [Error 17007](https://www.twilio.com/docs/api/errors/17007)
- [Event delivery and duplication](https://www.twilio.com/docs/events/event-delivery-and-duplication)

**SDK**
- [twilio-voice.js](https://github.com/twilio/twilio-voice.js) — `master`, tag 2.18.3
- [twilio-voice-js-reference-components](https://github.com/twilio/twilio-voice-js-reference-components) — `twilio-voice-monitoring`
- [Device object](https://www.twilio.com/docs/voice/sdks/javascript/twiliodevice)
- [PreflightTest](https://www.twilio.com/docs/voice/sdks/javascript/twiliopreflighttest)
