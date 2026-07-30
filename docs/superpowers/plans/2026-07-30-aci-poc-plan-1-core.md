# ACI Quality POC — Plan 1: Core (types, sentinel, store)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build and test the POC's Twilio-independent core — the shared type contract, the client-side quality sentinel (rolling window, verdict state machine, beacon evaluator), and the SQLite store — plus an architecture test that enforces the Twilio/POC dependency rule from day one.

**Architecture:** Everything in this plan is a pure library with no network, no browser, and no Twilio account. `shared/types.ts` is the contract that both sides of the boundary consume. `web/sentinel/*` turns a stream of normalized samples into a verdict and a beacon decision. `server/poc/store.ts` persists calls, events, and feedback. The architecture test greps the source tree to prove no POC code imports a Twilio SDK.

**Tech Stack:** Node 24 (native TypeScript type-stripping), pnpm, `node:test`, `node:assert/strict`, `node:sqlite`. Two devDependencies: `typescript` (for `tsc --noEmit`) and `@types/node`.

**Spec:** `~/code/pages/docs/superpowers/specs/2026-07-30-aci-quality-poc-design.md` @ `09be198`

**Verification status:** every code block in this plan was assembled into a scratch project and executed before the plan was committed, then re-verified after the shared-fixtures refactor. Result: **46/46 tests pass**, `tsc --noEmit` exits 0 under `strict` + `noUncheckedIndexedAccess` + `erasableSyntaxOnly` + `verbatimModuleSyntax` (TypeScript 5.9.3, `@types/node` 24.13.2), the architecture test was confirmed to fail and name the offending file when a forbidden import is introduced, and `tests/fixtures.ts` was confirmed not to be collected as a test file. The code here is known-good — if you hit a compile error or a red test, suspect a transcription slip rather than the plan.

## Global Constraints

Every task's requirements implicitly include this section.

- **Repo:** `~/code/aci-quality-poc`. All relative paths in this plan are relative to it. Task 8 is the one exception and edits a file in `~/code/pages` — it states an absolute path.
- **Branch:** Task 1 runs `git init`; all eight commits land on the new repo's `main`. There is no pre-existing history to protect.
- **Node 24, pnpm.** Verified present: Node v24.18.0, pnpm 11.8.0.
- **Test fixtures are shared via `tests/fixtures.ts`.** Test files import `sample()`, `windowAtMos()`, `call()`, and `event()` from there rather than redefining them. The module is built up incrementally — each task adds only the fixture its own tests need.
- **The test directory MUST be named `tests/` (plural).** Node's test runner treats every file inside a directory named `test/` (singular) as a test file, which would make it try to execute `fixtures.ts`. With `tests/`, only `*.test.ts` is collected. Verified: `tests/fixtures.ts` is correctly ignored.
- **Relative imports MUST carry the `.ts` extension** (`from './window.ts'`). Node's type-stripping requires it; omitting it fails at runtime.
- **`package.json` must set `"type": "module"`.**
- **Zero runtime dependencies in this plan.** `node:sqlite` and `node:test` are built in. The only devDependencies are `typescript` and `@types/node` (pinned to major 24 to match the runtime).
- **The dependency rule:** code under `server/poc/` or `web/sentinel/` must NOT import `twilio` or `@twilio/voice-sdk`. Task 3 enforces this with a test.
- **`lossPct` is 0–100, a percentage, never 0–1.**
- **`mos` is nullable.** Every consumer must handle `null`; unguarded arithmetic yields `NaN`.
- **`audioInLevel` / `audioOutLevel` are 0–32767.**
- **MOS bands are contiguous.** Do NOT reuse `PreflightTest.CallQuality`, whose bands have gaps at `(4.0, 4.1)` and `(3.6, 3.7)` — MOS 4.05 falls through every branch there and reports `degraded`. Task 5 has a regression test for exactly this value.
- **`CallScore` is 1–5 where 5 is Excellent** (inverted from a severity scale).
- **Sustain is tracked independently of the sample window.** The spec's §6 default of a 15 s MOS sustain exceeds the 10 × 1 s window; the beacon keeps its own `since` timestamps so a sustain longer than the window is representable. Do not widen the window to compensate.

---

### Task 1: Repo scaffold and toolchain

**Files:**
- Create: `package.json`
- Create: `tsconfig.json`
- Create: `.gitignore`
- Create: `README.md`

**Interfaces:**
- Consumes: nothing.
- Produces: `pnpm test` runs `node --test` over `**/*.test.ts`; `pnpm typecheck` runs `tsc --noEmit`. Both are used by every later task.

- [ ] **Step 1: Create the directory and initialise git**

```bash
mkdir -p ~/code/aci-quality-poc
cd ~/code/aci-quality-poc
git init
```

- [ ] **Step 2: Write `package.json`**

```json
{
  "name": "aci-quality-poc",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "engines": {
    "node": ">=24"
  },
  "scripts": {
    "test": "node --test",
    "typecheck": "tsc --noEmit"
  },
  "devDependencies": {
    "@types/node": "^24.0.0",
    "typescript": "^5.8.0"
  }
}
```

`@types/node` is required because `tsconfig.json` sets `"types": ["node"]`, and the code uses `node:sqlite`, `node:test`, `node:fs`, and `node:path`. Pin the major to **24** to match the Node 24 runtime — a newer major would type APIs this runtime does not have. These two are the only devDependencies; there are no runtime dependencies.

- [ ] **Step 3: Write `tsconfig.json`**

`erasableSyntaxOnly` is the important setting — it makes `tsc` reject any TypeScript syntax that Node's type-stripping cannot erase, so the typechecker and the runtime agree.

```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "lib": ["esnext"],
    "types": ["node"],
    "allowImportingTsExtensions": true,
    "erasableSyntaxOnly": true,
    "verbatimModuleSyntax": true,
    "noEmit": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "skipLibCheck": true
  },
  "include": ["shared", "server", "web", "tests"]
}
```

- [ ] **Step 4: Write `.gitignore`**

```
node_modules/
*.sqlite
*.sqlite-journal
.env
.DS_Store
```

- [ ] **Step 5: Write `README.md`**

```markdown
# aci-quality-poc

Proof of concept: client-side Twilio Voice JS SDK quality signals plus agent
subjective feedback, used to enable Voice Insights Advanced Features (ACI) on
demand and consume the resulting data.

Not production. See the design spec in `~/code/pages/docs/superpowers/specs/`.

## Layout and the one rule

    server/twilio/    may import 'twilio'; knows nothing about the POC
    server/poc/       may NOT import 'twilio'
    web/twilio/       may import '@twilio/voice-sdk'
    web/sentinel/     may NOT import '@twilio/voice-sdk'
    shared/types.ts   the contract both sides consume

`tests/architecture.test.ts` enforces this.

## Commands

    pnpm install
    pnpm test        # node --test, native TypeScript, no build step
    pnpm typecheck   # tsc --noEmit
```

- [ ] **Step 6: Install and verify the scripts are wired**

```bash
cd ~/code/aci-quality-poc
pnpm install
```

Expected: completes, installing only `typescript` and `@types/node`.

```bash
pnpm test
```

Expected: exits 0, reporting `tests 0` / `pass 0` / `fail 0`.

```bash
pnpm typecheck
```

**Expected: this FAILS at this point, with exactly:**

```
error TS18003: No inputs were found in config file 'tsconfig.json'.
Specified 'include' paths were '["shared","server","web","tests"]' and 'exclude' paths were '[]'.
```

That is correct and expected — the `include` directories do not exist until Task 2 creates `shared/types.ts`. **Do NOT "fix" this** by creating a placeholder file, adding `"files": []`, or narrowing `include`; Task 2 resolves it by adding the first real source file. Confirm you got TS18003 and move on.

From Task 2 onward, `pnpm typecheck` must exit 0.

- [ ] **Step 7: Commit**

```bash
cd ~/code/aci-quality-poc
git add package.json tsconfig.json .gitignore README.md pnpm-lock.yaml
git commit -m "chore: scaffold repo with Node 24 native TS and node:test"
```

---

### Task 2: The shared type contract

**Files:**
- Create: `shared/types.ts`
- Test: `tests/types.test.ts`

**Interfaces:**
- Consumes: nothing.
- Produces: all types below. Every later task and both future plans import from `shared/types.ts`. Exact names matter — later tasks reference them verbatim.

- [ ] **Step 1: Write the failing test**

The only behaviour worth asserting on a types module is that the exported band and default constants hold the values the spec fixes, since those are what later logic keys on.

Create `tests/types.test.ts`:

```typescript
import { test } from 'node:test'
import assert from 'node:assert/strict'
import { DEFAULT_BEACON_CONFIG, CATEGORICAL_FAULTS } from '../shared/types.ts'

test('default beacon config matches the spec defaults', () => {
  const byMetric = Object.fromEntries(
    DEFAULT_BEACON_CONFIG.rows.map((r) => [r.metric, r]),
  )

  assert.deepEqual(byMetric.mos, {
    metric: 'mos', op: '<', value: 3.1, sustainMs: 15_000, enabled: true,
  })
  assert.deepEqual(byMetric.jitterMs, {
    metric: 'jitterMs', op: '>', value: 30, sustainMs: 10_000, enabled: true,
  })
  assert.deepEqual(byMetric.lossPct, {
    metric: 'lossPct', op: '>', value: 5, sustainMs: 10_000, enabled: true,
  })
  assert.deepEqual(byMetric.rttMs, {
    metric: 'rttMs', op: '>', value: 400, sustainMs: 10_000, enabled: false,
  })

  assert.deepEqual(DEFAULT_BEACON_CONFIG.combinator, { kind: 'any' })
})

test('mos sustain exceeds the sample window, so sustain cannot be window-bound', () => {
  const mos = DEFAULT_BEACON_CONFIG.rows.find((r) => r.metric === 'mos')!
  assert.ok(mos.sustainMs > 10_000, 'guards the design note in Global Constraints')
})

test('categorical faults are the three non-tunable ones', () => {
  assert.deepEqual([...CATEGORICAL_FAULTS], [
    'low-bytes-sent',
    'low-bytes-received',
    'ice-connectivity-lost',
  ])
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd ~/code/aci-quality-poc && pnpm test`
Expected: FAIL — `Cannot find module '../shared/types.ts'`.

- [ ] **Step 3: Write `shared/types.ts`**

```typescript
/**
 * The contract across the Twilio/POC boundary.
 *
 * Field hazards are documented here once so they cannot be reintroduced
 * downstream. web/twilio/adapt.ts is the only place that touches raw SDK
 * payloads; everything else consumes these types.
 */

/** One 1 Hz quality sample, normalized from the SDK's RTCSample. */
export type QualitySample = {
  ts: number
  /** NULLABLE: null on the first sample and before ICE completes. */
  mos: number | null
  rttMs: number
  jitterMs: number
  /** 0–100, a PERCENTAGE. Not 0–1. */
  lossPct: number
  /** 0–32767, representing −100 dB to −30 dB. */
  audioInLevel: number
  audioOutLevel: number
  bytesSent: number
  bytesReceived: number
  /** Browser-derived casing: 'opus' | 'PCMU'. Normalize before comparing. */
  codec: string
}

export type WarningName =
  | 'high-rtt'
  | 'high-jitter'
  | 'low-mos'
  | 'high-packet-loss'
  | 'high-packets-lost-fraction'
  | 'low-bytes-sent'
  | 'low-bytes-received'
  | 'constant-audio-input-level'
  | 'ice-connectivity-lost'

export type QualityWarning = {
  ts: number
  name: WarningName
  /** false means the warning cleared. */
  raised: boolean
  /** Absent for 'ice-connectivity-lost', which emits with no second argument. */
  threshold?: { name: string; value: number }
}

/** 53405 = media, 53001 = signaling. */
export type ReconnectCause = 'media' | 'signaling'

export type Verdict = 'good' | 'fair' | 'degraded' | 'bad' | 'reconnecting'

/** Categorical faults: binary, so not threshold-tunable. */
export const CATEGORICAL_FAULTS = [
  'low-bytes-sent',
  'low-bytes-received',
  'ice-connectivity-lost',
] as const satisfies readonly WarningName[]

// ── beacon configuration ────────────────────────────────────────────────

export type MetricKey = 'mos' | 'jitterMs' | 'lossPct' | 'rttMs'

export type ThresholdRow = {
  metric: MetricKey
  op: '<' | '>'
  value: number
  sustainMs: number
  enabled: boolean
}

export type Combinator =
  | { kind: 'any' }
  | { kind: 'all' }
  /** n of the enabled rows must be met. m is the count of enabled rows. */
  | { kind: 'n-of-m'; n: number }

export type BeaconConfig = {
  rows: ThresholdRow[]
  combinator: Combinator
}

/** Deliberately loose — the demo triggers gratuitously. */
export const DEFAULT_BEACON_CONFIG: BeaconConfig = {
  rows: [
    { metric: 'mos', op: '<', value: 3.1, sustainMs: 15_000, enabled: true },
    { metric: 'jitterMs', op: '>', value: 30, sustainMs: 10_000, enabled: true },
    { metric: 'lossPct', op: '>', value: 5, sustainMs: 10_000, enabled: true },
    { metric: 'rttMs', op: '>', value: 400, sustainMs: 10_000, enabled: false },
  ],
  combinator: { kind: 'any' },
}

// ── events ──────────────────────────────────────────────────────────────

export type EventSource = 'sdk' | 'sentinel' | 'server' | 'aci'
export type EventLevel = 'info' | 'warning' | 'error'
export type Edge = 'sdk_edge' | 'client_edge' | 'carrier_edge'

export type PaneEvent = {
  /** For 'aci' rows this is the CloudEvents id, which makes inserts idempotent. */
  id: string
  callSid: string
  source: EventSource
  name: string
  level: EventLevel
  /** When it actually happened. */
  ts: number
  /** When we learned about it. ACI events lag ~90 s. */
  receivedTs: number
  edge?: Edge
  payload: unknown
}

// ── subjective feedback ─────────────────────────────────────────────────

export type FeedbackKind = 'in_call_button' | 'post_call_survey'

/** Exact Annotations API values. Note: 'no_audio' is NOT valid; 'owa' is one-way audio. */
export type QualityIssue =
  | 'low_volume'
  | 'choppy_robotic'
  | 'echo'
  | 'dtmf'
  | 'latency'
  | 'owa'
  | 'static_noise'

export type Feedback = {
  callSid: string
  kind: FeedbackKind
  /** 1–5 where 5 is Excellent, matching CallScore. */
  score?: number
  issues: QualityIssue[]
  /** Full text. Annotations truncates to 100 chars; the store keeps both. */
  comment?: string
}

/** Annotation outcome. The gating failure arrives as 17007 or HTTP 401 — docs disagree. */
export type AnnotateResult =
  | { ok: true; status: number }
  | { ok: false; status: number; errorCode?: number; message: string }

export type CallLeg = 'cca' | 'caller'
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd ~/code/aci-quality-poc && pnpm test && pnpm typecheck`
Expected: 3 tests pass; typecheck exits 0.

- [ ] **Step 5: Commit**

```bash
cd ~/code/aci-quality-poc
git add shared/types.ts tests/types.test.ts
git commit -m "feat: add shared type contract with documented field hazards"
```

---

### Task 3: Architecture test enforcing the dependency rule

**Files:**
- Create: `tests/architecture.test.ts`
- Note: the banned-import regex is built by a shared helper so both rules stay in sync, and it catches `from`, side-effect, `require()`, and dynamic-`import()` shapes plus subpaths. The walk scans `.ts` and `.tsx`.

**Interfaces:**
- Consumes: nothing.
- Produces: nothing importable. This test guards every later task, including both future plans.

Written now, before `server/twilio/` and `web/twilio/` exist, so the rule is enforced from the first line of Twilio code rather than retrofitted. It must therefore pass when the guarded directories are absent.

- [ ] **Step 1: Write the failing test**

Create `tests/architecture.test.ts`:

```typescript
import { test } from 'node:test'
import assert from 'node:assert/strict'
import { readdirSync, readFileSync, existsSync, statSync } from 'node:fs'
import { join } from 'node:path'

/** Recursively collect .ts/.tsx files under dir. Returns [] if dir does not exist. */
function tsFiles(dir: string): string[] {
  if (!existsSync(dir)) return []
  const out: string[] = []
  for (const entry of readdirSync(dir)) {
    const full = join(dir, entry)
    if (statSync(full).isDirectory()) out.push(...tsFiles(full))
    else if (entry.endsWith('.ts') || entry.endsWith('.tsx')) out.push(full)
  }
  return out
}

/**
 * Build a regex that catches every realistic way `pkg` could be imported:
 *   - `import x from 'pkg'` / `import { x } from 'pkg'` / `export { x } from 'pkg'`
 *   - subpath imports, e.g. `from 'pkg/lib/x'`
 *   - side-effect imports: `import 'pkg'`
 *   - CommonJS: `require('pkg')`
 *   - dynamic imports: `import('pkg')` / `await import('pkg')`
 * A single builder guarantees both banned-package rules stay in sync.
 */
function bannedImportPattern(pkg: string): RegExp {
  const escaped = pkg.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
  const opener = `(?:\\bfrom\\s+|\\bimport\\s*\\(?\\s*|\\brequire\\s*\\(\\s*)`
  return new RegExp(`${opener}['"]${escaped}(?:['"]|/)`)
}

const RULES = [
  {
    dir: 'server/poc',
    banned: bannedImportPattern('twilio'),
    label: "server/poc must not import 'twilio'",
  },
  {
    dir: 'web/sentinel',
    banned: bannedImportPattern('@twilio/voice-sdk'),
    label: "web/sentinel must not import '@twilio/voice-sdk'",
  },
]

for (const rule of RULES) {
  test(rule.label, () => {
    const offenders = tsFiles(rule.dir).filter((f) =>
      rule.banned.test(readFileSync(f, 'utf8')),
    )
    assert.deepEqual(offenders, [], `${rule.label} — offending files listed above`)
  })
}

test('the guarded directories are the ones the spec names', () => {
  assert.deepEqual(RULES.map((r) => r.dir), ['server/poc', 'web/sentinel'])
})
```

- [ ] **Step 2: Run test to verify it passes vacuously, then prove it can fail**

Run: `cd ~/code/aci-quality-poc && pnpm test`
Expected: 3 new tests pass (the guarded dirs don't exist yet, so there are no offenders).

Now prove the test actually detects violations. A guard that cannot fail is worthless, so exercise **all four import shapes against both packages** — eight cases. For each: create the file, run the test, confirm it fails and names the file, delete the file.

```bash
cd ~/code/aci-quality-poc
mkdir -p server/poc web/sentinel

# 'twilio' in server/poc — four shapes
printf "import twilio from 'twilio'\nexport const x = twilio\n" > server/poc/v.ts
node --test tests/architecture.test.ts 2>&1 | grep -E "^\u2716|'server/poc/v.ts'"
printf "import 'twilio'\n"            > server/poc/v.ts   # side-effect
node --test tests/architecture.test.ts 2>&1 | grep -E "^\u2716|'server/poc/v.ts'"
printf "const t = require('twilio')\nexport default t\n" > server/poc/v.ts   # CommonJS
node --test tests/architecture.test.ts 2>&1 | grep -E "^\u2716|'server/poc/v.ts'"
printf "await import('twilio')\n"     > server/poc/v.ts   # dynamic
node --test tests/architecture.test.ts 2>&1 | grep -E "^\u2716|'server/poc/v.ts'"
rm -f server/poc/v.ts

# '@twilio/voice-sdk' in web/sentinel — same four, one of them in a .tsx
printf "import { Device } from '@twilio/voice-sdk'\nexport const d = Device\n" > web/sentinel/v.tsx
node --test tests/architecture.test.ts 2>&1 | grep -E "^\u2716|'web/sentinel/v.tsx'"
rm -f web/sentinel/v.tsx
printf "import '@twilio/voice-sdk'\n" > web/sentinel/v.ts
node --test tests/architecture.test.ts 2>&1 | grep -E "^\u2716|'web/sentinel/v.ts'"
printf "const s = require('@twilio/voice-sdk')\nexport default s\n" > web/sentinel/v.ts
node --test tests/architecture.test.ts 2>&1 | grep -E "^\u2716|'web/sentinel/v.ts'"
printf "await import('@twilio/voice-sdk')\n" > web/sentinel/v.ts
node --test tests/architecture.test.ts 2>&1 | grep -E "^\u2716|'web/sentinel/v.ts'"
rm -f web/sentinel/v.ts
```

Expected: every one of the eight runs FAILS the matching rule and names the offending file. If any shape passes, the regex is wrong — stop and report it.

Also confirm a subpath import is caught, since that is the shape most likely to be missed:

```bash
printf "export { X } from 'twilio/lib/sub'\n" > server/poc/v.ts
node --test tests/architecture.test.ts 2>&1 | grep -E "^\u2716|'server/poc/v.ts'"
rm -f server/poc/v.ts
```

- [ ] **Step 3: Remove all probe files and directories, confirm green**

```bash
cd ~/code/aci-quality-poc
rm -f server/poc/v.ts web/sentinel/v.ts web/sentinel/v.tsx
rmdir server/poc server web/sentinel web 2>/dev/null
git status --short   # must be empty
pnpm test && pnpm typecheck
```

Expected: `git status` empty, all tests pass, typecheck exits 0. The guarded directories must NOT exist after this task — later tasks create them.

- [ ] **Step 4: Commit**

```bash
cd ~/code/aci-quality-poc
git add tests/architecture.test.ts
git commit -m "test: enforce the twilio/poc dependency rule"
```

---

### Task 4: Sentinel sample window

**Files:**
- Create: `web/sentinel/window.ts`
- Create: `tests/fixtures.ts`
- Test: `tests/window.test.ts`

**Interfaces:**
- Consumes: `QualitySample` from `shared/types.ts`.
- Produces: `WINDOW_SIZE: number`, and `class SampleWindow` with `push(s: QualitySample): void`, `get size(): number`, `p50Mos(): number | null`, `latest(): QualitySample | null`, `all(): readonly QualitySample[]`. Task 5 consumes `p50Mos()`; Task 6 consumes `latest()`. Also produces `tests/fixtures.ts` exporting `sample(over?: Partial<QualitySample>): QualitySample`, which Tasks 5, 6, and 7 extend and import.

- [ ] **Step 1a: Create the shared fixture module**

Create `tests/fixtures.ts`. Later tasks append to this file; do not duplicate these helpers into individual test files.

```typescript
import type { QualitySample } from '../shared/types.ts'

/** A healthy 1 Hz sample. Override only the fields a test cares about. */
export function sample(over: Partial<QualitySample> = {}): QualitySample {
  return {
    ts: 0,
    mos: 4.2,
    rttMs: 40,
    jitterMs: 5,
    lossPct: 0,
    audioInLevel: 8000,
    audioOutLevel: 8000,
    bytesSent: 1600,
    bytesReceived: 1600,
    codec: 'opus',
    ...over,
  }
}
```

- [ ] **Step 1b: Write the failing test**

Create `tests/window.test.ts`:

```typescript
import { test } from 'node:test'
import assert from 'node:assert/strict'
import { SampleWindow, WINDOW_SIZE } from '../web/sentinel/window.ts'
import { sample } from './fixtures.ts'

test('window is 10 samples', () => {
  assert.equal(WINDOW_SIZE, 10)
})

test('push evicts the oldest beyond WINDOW_SIZE', () => {
  const w = new SampleWindow()
  for (let i = 0; i < 15; i++) w.push(sample({ ts: i, mos: 4 }))
  assert.equal(w.size, 10)
  assert.equal(w.latest()!.ts, 14)
  assert.equal(w.all()[0]!.ts, 5)
})

test('p50Mos is null on an empty window', () => {
  assert.equal(new SampleWindow().p50Mos(), null)
})

test('p50Mos ignores null mos samples', () => {
  const w = new SampleWindow()
  w.push(sample({ ts: 0, mos: null }))
  w.push(sample({ ts: 1, mos: null }))
  w.push(sample({ ts: 2, mos: 3.0 }))
  assert.equal(w.p50Mos(), 3.0)
})

test('p50Mos is null when every sample has null mos', () => {
  const w = new SampleWindow()
  for (let i = 0; i < 3; i++) w.push(sample({ ts: i, mos: null }))
  assert.equal(w.p50Mos(), null, 'must not return NaN')
})

test('p50Mos takes the median, not the mean, so one spike does not dominate', () => {
  const w = new SampleWindow()
  for (const m of [4.3, 4.2, 4.3, 1.0, 4.2]) w.push(sample({ mos: m }))
  assert.equal(w.p50Mos(), 4.2)
})

test('p50Mos averages the middle pair on an even count', () => {
  const w = new SampleWindow()
  for (const m of [3.0, 4.0]) w.push(sample({ mos: m }))
  assert.equal(w.p50Mos(), 3.5)
})

test('latest is null on an empty window', () => {
  assert.equal(new SampleWindow().latest(), null)
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd ~/code/aci-quality-poc && pnpm test`
Expected: FAIL — `Cannot find module '../web/sentinel/window.ts'`.

- [ ] **Step 3: Write `web/sentinel/window.ts`**

```typescript
import type { QualitySample } from '../../shared/types.ts'

/** 10 samples at 1 Hz — a 10-second view. */
export const WINDOW_SIZE = 10

/**
 * A fixed-size rolling window of quality samples.
 *
 * Does NOT track sustain duration: the beacon does that separately, because a
 * configured sustain may legitimately exceed the window (the default MOS
 * sustain is 15 s against a 10 s window).
 */
export class SampleWindow {
  #samples: QualitySample[] = []

  push(s: QualitySample): void {
    this.#samples.push(s)
    if (this.#samples.length > WINDOW_SIZE) this.#samples.shift()
  }

  get size(): number {
    return this.#samples.length
  }

  all(): readonly QualitySample[] {
    return this.#samples
  }

  latest(): QualitySample | null {
    return this.#samples.at(-1) ?? null
  }

  /**
   * Median MOS across the window, skipping null samples.
   * Returns null when no sample has a usable MOS — never NaN.
   */
  p50Mos(): number | null {
    const vals = this.#samples
      .map((s) => s.mos)
      .filter((m): m is number => m !== null)
      .sort((a, b) => a - b)

    if (vals.length === 0) return null

    const mid = Math.floor(vals.length / 2)
    return vals.length % 2 === 1
      ? vals[mid]!
      : (vals[mid - 1]! + vals[mid]!) / 2
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd ~/code/aci-quality-poc && pnpm test && pnpm typecheck`
Expected: 8 window tests pass; typecheck exits 0.

- [ ] **Step 5: Commit**

```bash
cd ~/code/aci-quality-poc
git add web/sentinel/window.ts tests/fixtures.ts tests/window.test.ts
git commit -m "feat: add sentinel sample window with null-safe median MOS"
```

---

### Task 5: Verdict state machine

**Files:**
- Create: `web/sentinel/verdict.ts`
- Modify: `tests/fixtures.ts` (append `windowAtMos`)
- Test: `tests/verdict.test.ts`

**Interfaces:**
- Consumes: `SampleWindow` from `web/sentinel/window.ts`; `Verdict`, `WarningName`, `CATEGORICAL_FAULTS` from `shared/types.ts`; `sample` from `tests/fixtures.ts`.
- Produces: `MOS_BANDS: { good: number; fair: number; degraded: number }`, `type VerdictInput`, and `computeVerdict(input: VerdictInput): Verdict`. Task 6 does not consume this; the UI in Plan 2 does. Also appends `windowAtMos(mos: number | null): SampleWindow` to `tests/fixtures.ts`.

- [ ] **Step 1a: Append `windowAtMos` to the fixture module**

Add to `tests/fixtures.ts`, and add the `SampleWindow` import at the top of that file:

```typescript
import { SampleWindow } from '../web/sentinel/window.ts'

// ... existing sample() ...

/** A window of 5 samples all at the given MOS. */
export function windowAtMos(mos: number | null): SampleWindow {
  const w = new SampleWindow()
  for (let i = 0; i < 5; i++) w.push(sample({ ts: i, mos }))
  return w
}
```

- [ ] **Step 1b: Write the failing test**

Create `tests/verdict.test.ts`:

```typescript
import { test } from 'node:test'
import assert from 'node:assert/strict'
import { computeVerdict, MOS_BANDS } from '../web/sentinel/verdict.ts'
import type { WarningName } from '../shared/types.ts'
import { windowAtMos } from './fixtures.ts'

function verdictFor(
  mos: number | null,
  warnings: WarningName[] = [],
  reconnecting = false,
) {
  return computeVerdict({
    window: windowAtMos(mos),
    activeWarnings: new Set(warnings),
    reconnecting,
  })
}

test('bands are contiguous with no gaps', () => {
  assert.deepEqual(MOS_BANDS, { good: 4.0, fair: 3.6, degraded: 3.1 })
})

test('MOS 4.05 is good — regression against the PreflightTest band gap', () => {
  // PreflightTest.CallQuality has gaps at (4.0,4.1) and (3.6,3.7); 4.05 falls
  // through every branch there and reports 'degraded'. Ours must not.
  assert.equal(verdictFor(4.05), 'good')
})

test('MOS 3.65 is fair — the other PreflightTest gap', () => {
  assert.equal(verdictFor(3.65), 'fair')
})

test('band boundaries are inclusive at the lower edge', () => {
  assert.equal(verdictFor(4.0), 'good')
  assert.equal(verdictFor(3.99), 'fair')
  assert.equal(verdictFor(3.6), 'fair')
  assert.equal(verdictFor(3.59), 'degraded')
  assert.equal(verdictFor(3.1), 'degraded')
  assert.equal(verdictFor(3.09), 'bad')
})

test('null MOS reports good rather than alarming on absent data', () => {
  assert.equal(verdictFor(null), 'good')
})

test('an active high-* warning floors the verdict at degraded', () => {
  assert.equal(verdictFor(4.5, ['high-jitter']), 'degraded')
  assert.equal(verdictFor(3.8, ['high-packet-loss']), 'degraded')
})

test('a high-* warning does not improve an already-bad verdict', () => {
  assert.equal(verdictFor(2.0, ['high-jitter']), 'bad')
})

test('categorical faults short-circuit to bad regardless of MOS', () => {
  assert.equal(verdictFor(4.5, ['low-bytes-sent']), 'bad')
  assert.equal(verdictFor(4.5, ['low-bytes-received']), 'bad')
  assert.equal(verdictFor(4.5, ['ice-connectivity-lost']), 'bad')
})

test('reconnecting overrides everything', () => {
  assert.equal(verdictFor(4.5, [], true), 'reconnecting')
  assert.equal(verdictFor(1.0, ['low-bytes-sent'], true), 'reconnecting')
})

test('a null MOS still passes through the flooring step', () => {
  // Regression: an early `return 'good'` on null MOS would let a live fault
  // read as good, which is the worst failure mode for an agent-facing indicator.
  assert.equal(verdictFor(null, ['high-jitter']), 'degraded')
  assert.equal(verdictFor(null, []), 'good')
})

test('low-mos floors the verdict despite not sharing the high- prefix', () => {
  assert.equal(verdictFor(4.5, ['low-mos']), 'degraded')
})

test('low-mos does not improve an already-worse verdict', () => {
  assert.equal(verdictFor(2.0, ['low-mos']), 'bad')
})

test('constant-audio-input-level is deliberately not a flooring warning', () => {
  // Suppressed while muted, and measures the local mic rather than the
  // transmitted path — corroborating evidence, not a quality verdict.
  assert.equal(verdictFor(4.5, ['constant-audio-input-level']), 'good')
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd ~/code/aci-quality-poc && pnpm test`
Expected: FAIL — `Cannot find module '../web/sentinel/verdict.ts'`.

- [ ] **Step 3: Write `web/sentinel/verdict.ts`**

```typescript
import type { Verdict, WarningName } from '../../shared/types.ts'
import { CATEGORICAL_FAULTS } from '../../shared/types.ts'
import type { SampleWindow } from './window.ts'

/**
 * Contiguous MOS bands, lower edge inclusive.
 *
 * Deliberately NOT PreflightTest.CallQuality, whose bands leave (4.0, 4.1) and
 * (3.6, 3.7) matching no branch — MOS 4.05 reports 'degraded' there.
 */
export const MOS_BANDS = {
  good: 4.0,
  fair: 3.6,
  degraded: 3.1,
} as const

/**
 * Warnings that floor the verdict at 'degraded'. This is an explicit named
 * set rather than a 'high-' prefix test, so a future WarningName can't
 * silently opt out of the floor just by not sharing that prefix.
 *
 * 'low-mos' is included: it's a threshold breach on a metric where *lower*
 * is worse, semantically identical to the high-* warnings — it just doesn't
 * share their prefix.
 *
 * 'constant-audio-input-level' is deliberately excluded: it's suppressed
 * while the agent is muted and measures the local microphone rather than
 * the transmitted path, so it's corroborating evidence, not on its own a
 * verdict on call quality.
 */
const FLOORING_WARNINGS: ReadonlySet<WarningName> = new Set<WarningName>([
  'high-rtt',
  'high-jitter',
  'low-mos',
  'high-packet-loss',
  'high-packets-lost-fraction',
])

export type VerdictInput = {
  window: SampleWindow
  activeWarnings: Set<WarningName>
  reconnecting: boolean
}

export function computeVerdict(input: VerdictInput): Verdict {
  // Reconnecting is a distinct state, not a severity — it outranks everything.
  if (input.reconnecting) return 'reconnecting'

  // Categorical faults are binary and mean the audio path is broken.
  for (const fault of CATEGORICAL_FAULTS) {
    if (input.activeWarnings.has(fault)) return 'bad'
  }

  const mos = input.window.p50Mos()

  // No usable MOS yet (first sample, or pre-ICE). Absence of data is not a
  // fault, so the banded result is 'good' — but it still must pass through
  // the flooring step below rather than returning early.
  const banded: Verdict =
    mos === null ? 'good'
    : mos >= MOS_BANDS.good ? 'good'
    : mos >= MOS_BANDS.fair ? 'fair'
    : mos >= MOS_BANDS.degraded ? 'degraded'
    : 'bad'

  // An active quality warning floors the verdict at degraded, but never
  // improves a worse one.
  const hasFlooringWarning = [...input.activeWarnings].some((w) => FLOORING_WARNINGS.has(w))
  if (hasFlooringWarning && (banded === 'good' || banded === 'fair')) return 'degraded'

  return banded
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd ~/code/aci-quality-poc && pnpm test && pnpm typecheck`
Expected: 14 verdict tests pass (9 band/precedence + 5 flooring regressions); typecheck exits 0.

- [ ] **Step 5: Commit**

```bash
cd ~/code/aci-quality-poc
git add web/sentinel/verdict.ts tests/fixtures.ts tests/verdict.test.ts
git commit -m "feat: add verdict state machine with contiguous MOS bands"
```

---

### Task 6: Beacon evaluator

**Files:**
- Create: `web/sentinel/beacon.ts`
- Test: `tests/beacon.test.ts`

**Interfaces:**
- Consumes: `BeaconConfig`, `MetricKey`, `QualitySample`, `ThresholdRow`, `WarningName`, `CATEGORICAL_FAULTS` from `shared/types.ts`.
- Produces: `type BeaconReason = { metric: MetricKey | WarningName; detail: string }`, `type BeaconState = { tripped: boolean; reasons: BeaconReason[] }`, and `class BeaconEvaluator` with `constructor(config: BeaconConfig)`, `setConfig(c: BeaconConfig): void`, `observe(sample, activeWarnings, reconnecting): BeaconState | null`. `observe` returns `null` when nothing changed — Plan 2's beacon POST fires only on a non-null return.

This task owns sustain tracking, keeping its own `since` timestamps per metric so a configured sustain may exceed `WINDOW_SIZE`.

- [ ] **Step 1: Write the failing test**

Create `tests/beacon.test.ts`:

```typescript
import { test } from 'node:test'
import assert from 'node:assert/strict'
import { BeaconEvaluator } from '../web/sentinel/beacon.ts'
import type { BeaconConfig, WarningName } from '../shared/types.ts'
import { sample } from './fixtures.ts'

const NONE = new Set<WarningName>()

function config(over: Partial<BeaconConfig> = {}): BeaconConfig {
  return {
    rows: [
      { metric: 'lossPct', op: '>', value: 5, sustainMs: 3000, enabled: true },
      { metric: 'jitterMs', op: '>', value: 30, sustainMs: 3000, enabled: true },
    ],
    combinator: { kind: 'any' },
    ...over,
  }
}

test('a breach shorter than the sustain does not trip', () => {
  const b = new BeaconEvaluator(config())
  assert.equal(b.observe(sample({ ts: 0, lossPct: 12 }), NONE, false), null)
  assert.equal(b.observe(sample({ ts: 2000, lossPct: 12 }), NONE, false), null)
})

test('a breach held past the sustain trips once, then dedupes', () => {
  const b = new BeaconEvaluator(config())
  b.observe(sample({ ts: 0, lossPct: 12 }), NONE, false)
  const trip = b.observe(sample({ ts: 3000, lossPct: 12 }), NONE, false)
  assert.equal(trip?.tripped, true)
  assert.deepEqual(trip?.reasons.map((r) => r.metric), ['lossPct'])

  // Still breaching — no state change, so no repeat fire.
  assert.equal(b.observe(sample({ ts: 4000, lossPct: 12 }), NONE, false), null)
})

test('sustain resets when the condition clears', () => {
  const b = new BeaconEvaluator(config())
  b.observe(sample({ ts: 0, lossPct: 12 }), NONE, false)
  b.observe(sample({ ts: 1000, lossPct: 0 }), NONE, false)  // clears
  // Restarts the clock, so 3000 is only 2000ms into the new breach.
  assert.equal(b.observe(sample({ ts: 2000, lossPct: 12 }), NONE, false), null)
  assert.equal(b.observe(sample({ ts: 4000, lossPct: 12 }), NONE, false), null)
  assert.equal(b.observe(sample({ ts: 5000, lossPct: 12 }), NONE, false)?.tripped, true)
})

test('a sustain longer than the sample window is representable', () => {
  const b = new BeaconEvaluator(
    config({ rows: [{ metric: 'mos', op: '<', value: 3.1, sustainMs: 15_000, enabled: true }] }),
  )
  for (let t = 0; t <= 14_000; t += 1000) {
    assert.equal(b.observe(sample({ ts: t, mos: 2.5 }), NONE, false), null, `t=${t}`)
  }
  assert.equal(b.observe(sample({ ts: 15_000, mos: 2.5 }), NONE, false)?.tripped, true)
})

test('untripping reports tripped:false exactly once', () => {
  const b = new BeaconEvaluator(config())
  b.observe(sample({ ts: 0, lossPct: 12 }), NONE, false)
  b.observe(sample({ ts: 3000, lossPct: 12 }), NONE, false)
  const cleared = b.observe(sample({ ts: 4000, lossPct: 0 }), NONE, false)
  assert.equal(cleared?.tripped, false)
  assert.equal(b.observe(sample({ ts: 5000, lossPct: 0 }), NONE, false), null)
})

test('disabled rows are ignored', () => {
  const b = new BeaconEvaluator(
    config({ rows: [{ metric: 'lossPct', op: '>', value: 5, sustainMs: 0, enabled: false }] }),
  )
  assert.equal(b.observe(sample({ ts: 0, lossPct: 99 }), NONE, false), null)
})

test('null mos never satisfies a threshold', () => {
  const b = new BeaconEvaluator(
    config({ rows: [{ metric: 'mos', op: '<', value: 3.1, sustainMs: 0, enabled: true }] }),
  )
  assert.equal(b.observe(sample({ ts: 0, mos: null }), NONE, false), null)
})

test('ALL requires every enabled row', () => {
  const b = new BeaconEvaluator(config({ combinator: { kind: 'all' } }))
  // Only loss breaching.
  assert.equal(b.observe(sample({ ts: 0, lossPct: 12 }), NONE, false), null)
  assert.equal(b.observe(sample({ ts: 3000, lossPct: 12 }), NONE, false), null)
  // Now both.
  const trip = b.observe(sample({ ts: 6000, lossPct: 12, jitterMs: 44 }), NONE, false)
  assert.equal(trip, null, 'jitter sustain has not elapsed yet')
  assert.equal(
    b.observe(sample({ ts: 9000, lossPct: 12, jitterMs: 44 }), NONE, false)?.tripped,
    true,
  )
})

test('n-of-m trips at n met rows', () => {
  const b = new BeaconEvaluator(config({ combinator: { kind: 'n-of-m', n: 2 } }))
  b.observe(sample({ ts: 0, lossPct: 12, jitterMs: 44 }), NONE, false)
  assert.equal(
    b.observe(sample({ ts: 3000, lossPct: 12, jitterMs: 44 }), NONE, false)?.tripped,
    true,
  )
})

test('categorical faults trip immediately regardless of combinator', () => {
  const b = new BeaconEvaluator(config({ combinator: { kind: 'all' } }))
  const trip = b.observe(sample({ ts: 0 }), new Set<WarningName>(['low-bytes-sent']), false)
  assert.equal(trip?.tripped, true)
  assert.deepEqual(trip?.reasons.map((r) => r.metric), ['low-bytes-sent'])
})

test('reconnecting trips immediately', () => {
  const b = new BeaconEvaluator(config())
  assert.equal(b.observe(sample({ ts: 0 }), NONE, true)?.tripped, true)
})

test('setConfig clears sustain state', () => {
  const b = new BeaconEvaluator(config())
  b.observe(sample({ ts: 0, lossPct: 12 }), NONE, false)
  b.setConfig(config())
  assert.equal(b.observe(sample({ ts: 3000, lossPct: 12 }), NONE, false), null)
})

test('reconfiguring while tripped does not emit a spurious recovery', () => {
  // The threshold matrix is a user-facing control, and a non-null return drives
  // a paid-feature toggle. Editing config mid-incident must not read as recovery.
  const b = new BeaconEvaluator(config())
  b.observe(sample({ ts: 0, lossPct: 12 }), NONE, false)
  assert.equal(b.observe(sample({ ts: 3000, lossPct: 12 }), NONE, false)?.tripped, true)

  b.setConfig(config())
  // Still breaching at 12% — nothing recovered, so nothing may be reported.
  assert.equal(b.observe(sample({ ts: 4000, lossPct: 12 }), NONE, false), null)
  // Once the restarted sustain elapses it re-trips.
  assert.equal(b.observe(sample({ ts: 7000, lossPct: 12 }), NONE, false)?.tripped, true)
})

test('reconfiguring while untripped does not emit a spurious trip', () => {
  const b = new BeaconEvaluator(config())
  b.setConfig(config())
  assert.equal(b.observe(sample({ ts: 0, lossPct: 0 }), NONE, false), null)
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd ~/code/aci-quality-poc && pnpm test`
Expected: FAIL — `Cannot find module '../web/sentinel/beacon.ts'`.

- [ ] **Step 3: Write `web/sentinel/beacon.ts`**

```typescript
import type {
  BeaconConfig,
  MetricKey,
  QualitySample,
  ThresholdRow,
  WarningName,
} from '../../shared/types.ts'
import { CATEGORICAL_FAULTS } from '../../shared/types.ts'

export type BeaconReason = {
  metric: MetricKey | WarningName
  detail: string
}

export type BeaconState = {
  tripped: boolean
  reasons: BeaconReason[]
}

/**
 * Decides whether the beacon should fire, and reports only state CHANGES.
 *
 * Sustain is tracked here rather than in SampleWindow, so a configured sustain
 * may exceed the window length. The default MOS sustain is 15 s against a 10 s
 * window, which would otherwise be unreachable.
 */
export class BeaconEvaluator {
  #config: BeaconConfig
  /** metric → timestamp when its condition most recently became true. */
  #since = new Map<MetricKey, number>()
  #tripped = false
  /** When true, the next observe() re-derives state honestly but reports no transition. */
  #suppressNextEmission = false

  constructor(config: BeaconConfig) {
    this.#config = config
  }

  setConfig(config: BeaconConfig): void {
    this.#config = config
    this.#since.clear()
    this.#tripped = false
    this.#suppressNextEmission = true
  }

  /** Returns the new state on a transition, or null when nothing changed. */
  observe(
    sample: QualitySample,
    activeWarnings: Set<WarningName>,
    reconnecting: boolean,
  ): BeaconState | null {
    const reasons: BeaconReason[] = []

    // Categorical faults and reconnection bypass thresholds and combinators.
    for (const fault of CATEGORICAL_FAULTS) {
      if (activeWarnings.has(fault)) {
        reasons.push({ metric: fault, detail: 'categorical fault' })
      }
    }
    if (reconnecting) {
      reasons.push({ metric: 'ice-connectivity-lost', detail: 'reconnecting' })
    }

    // Threshold rows, gated by sustain.
    const met: ThresholdRow[] = []
    for (const row of this.#config.rows) {
      if (!row.enabled) continue

      const value = metricValue(sample, row.metric)
      const breaching =
        value !== null && (row.op === '<' ? value < row.value : value > row.value)

      if (!breaching) {
        this.#since.delete(row.metric)
        continue
      }

      const first = this.#since.get(row.metric) ?? sample.ts
      this.#since.set(row.metric, first)
      if (sample.ts - first >= row.sustainMs) met.push(row)
    }

    if (combinatorSatisfied(this.#config, met.length)) {
      for (const row of met) {
        reasons.push({
          metric: row.metric,
          detail: `${row.metric} ${row.op} ${row.value}`,
        })
      }
    }

    const tripped = reasons.length > 0

    // A reconfigure restarts every sustain clock, so the very next observe()
    // would otherwise report a spurious transition (e.g. a false "recovered"
    // while still breaching). Re-derive #tripped honestly but swallow that
    // one emission; a genuine transition will surface on the observe() after.
    if (this.#suppressNextEmission) {
      this.#suppressNextEmission = false
      this.#tripped = tripped
      return null
    }

    if (tripped === this.#tripped) return null

    this.#tripped = tripped
    return { tripped, reasons }
  }
}

function metricValue(s: QualitySample, m: MetricKey): number | null {
  switch (m) {
    case 'mos': return s.mos
    case 'jitterMs': return s.jitterMs
    case 'lossPct': return s.lossPct
    case 'rttMs': return s.rttMs
  }
}

function combinatorSatisfied(config: BeaconConfig, metCount: number): boolean {
  if (metCount === 0) return false
  const enabled = config.rows.filter((r) => r.enabled).length

  switch (config.combinator.kind) {
    case 'any': return metCount >= 1
    case 'all': return enabled > 0 && metCount === enabled
    case 'n-of-m': return metCount >= config.combinator.n
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd ~/code/aci-quality-poc && pnpm test && pnpm typecheck`
Expected: 14 beacon tests pass (12 core + 2 reconfigure regressions); typecheck exits 0.

- [ ] **Step 5: Commit**

```bash
cd ~/code/aci-quality-poc
git add web/sentinel/beacon.ts tests/beacon.test.ts
git commit -m "feat: add beacon evaluator with window-independent sustain tracking"
```

---

### Task 7: SQLite store

**Files:**
- Create: `server/poc/store.ts`
- Modify: `tests/fixtures.ts` (append `call` and `event`)
- Test: `tests/store.test.ts`

**Interfaces:**
- Consumes: `CallLeg`, `Feedback`, `PaneEvent`, `Verdict` from `shared/types.ts`.
- Produces: `type CallRow`, `type FeedbackRow`, and `class Store` with `constructor(path?: string)`, `close(): void`, `upsertCall(row: CallRow): void`, `endCall(callSid: string, endedAt: number, verdictWorst: Verdict): void`, `insertEvent(e: PaneEvent): boolean`, `eventsForCall(callSid: string): PaneEvent[]`, `insertFeedback(f: Feedback, annotation: { status: number; errorCode?: number; message?: string }): void`, `feedbackForCall(callSid: string): FeedbackRow[]`, `callsInConference(conferenceSid: string): CallRow[]`. Plan 3's routes consume all of these.

- [ ] **Step 1a: Append `call` and `event` to the fixture module**

Add to `tests/fixtures.ts`, along with the two imports at the top of that file:

```typescript
import type { CallRow } from '../server/poc/store.ts'
import type { PaneEvent } from '../shared/types.ts'

// ... existing sample() and windowAtMos() ...

export function call(over: Partial<CallRow> = {}): CallRow {
  return {
    callSid: 'CA1',
    conferenceSid: 'CF1',
    leg: 'cca',
    codec: 'opus',
    instrumented: false,
    startedAt: 1000,
    endedAt: null,
    verdictWorst: null,
    ...over,
  }
}

export function event(over: Partial<PaneEvent> = {}): PaneEvent {
  return {
    id: 'e1',
    callSid: 'CA1',
    source: 'sentinel',
    name: 'verdict.degraded',
    level: 'warning',
    ts: 1000,
    receivedTs: 1000,
    payload: { from: 'fair' },
    ...over,
  }
}
```

- [ ] **Step 1b: Write the failing test**

Create `tests/store.test.ts`:

```typescript
import { test } from 'node:test'
import assert from 'node:assert/strict'
import { Store } from '../server/poc/store.ts'
import type { Feedback } from '../shared/types.ts'
import { call, event } from './fixtures.ts'

test('a call round-trips, including the instrumented flag', () => {
  const s = new Store()
  s.upsertCall(call({ instrumented: true }))
  const [row] = s.callsInConference('CF1')
  assert.equal(row!.callSid, 'CA1')
  assert.equal(row!.instrumented, true, 'boolean must survive the 0/1 round trip')
  assert.equal(row!.endedAt, null)
  s.close()
})

test('upsertCall is idempotent on call_sid', () => {
  const s = new Store()
  s.upsertCall(call({ codec: null }))
  s.upsertCall(call({ codec: 'PCMU' }))
  const rows = s.callsInConference('CF1')
  assert.equal(rows.length, 1)
  assert.equal(rows[0]!.codec, 'PCMU')
  s.close()
})

test('endCall records the end time and worst verdict', () => {
  const s = new Store()
  s.upsertCall(call())
  s.endCall('CA1', 9000, 'bad')
  const [row] = s.callsInConference('CF1')
  assert.equal(row!.endedAt, 9000)
  assert.equal(row!.verdictWorst, 'bad')
  s.close()
})

test('two calls share a conference after a codec switch', () => {
  const s = new Store()
  s.upsertCall(call({ callSid: 'CA-agent-1', codec: 'PCMU' }))
  s.upsertCall(call({ callSid: 'CA-agent-2', codec: 'opus' }))
  s.upsertCall(call({ callSid: 'CA-caller', leg: 'caller', codec: null }))

  const rows = s.callsInConference('CF1')
  assert.equal(rows.length, 3)
  const codecs = rows.filter((r) => r.leg === 'cca').map((r) => r.codec).sort()
  assert.deepEqual(codecs, ['PCMU', 'opus'], 'the codec comparison query')
  s.close()
})

test('insertEvent returns true for a new event', () => {
  const s = new Store()
  s.upsertCall(call())
  assert.equal(s.insertEvent(event()), true)
  assert.equal(s.eventsForCall('CA1').length, 1)
  s.close()
})

test('a duplicate CloudEvents id is dropped, making delivery idempotent', () => {
  const s = new Store()
  s.upsertCall(call())
  assert.equal(s.insertEvent(event({ id: 'dup' })), true)
  assert.equal(s.insertEvent(event({ id: 'dup' })), false)
  assert.equal(s.eventsForCall('CA1').length, 1)
  s.close()
})

test('events preserve source, edge, and the ts/receivedTs lag', () => {
  const s = new Store()
  s.upsertCall(call())
  s.insertEvent(event({
    id: 'aci1', source: 'aci', name: 'call-metrics',
    edge: 'carrier_edge', ts: 1000, receivedTs: 93_000,
    payload: { jitter: 38 },
  }))
  const [row] = s.eventsForCall('CA1')
  assert.equal(row!.source, 'aci')
  assert.equal(row!.edge, 'carrier_edge')
  assert.equal(row!.receivedTs - row!.ts, 92_000)
  assert.deepEqual(row!.payload, { jitter: 38 })
  s.close()
})

test('events come back oldest first', () => {
  const s = new Store()
  s.upsertCall(call())
  s.insertEvent(event({ id: 'b', ts: 2000 }))
  s.insertEvent(event({ id: 'a', ts: 1000 }))
  assert.deepEqual(s.eventsForCall('CA1').map((e) => e.id), ['a', 'b'])
  s.close()
})

test('feedback keeps the full comment and the truncated one it sent', () => {
  const s = new Store()
  s.upsertCall(call())
  const long = 'x'.repeat(150)
  const f: Feedback = {
    callSid: 'CA1', kind: 'post_call_survey', score: 2,
    issues: ['owa', 'choppy_robotic'], comment: long,
  }
  s.insertFeedback(f, { status: 201 })

  const [row] = s.feedbackForCall('CA1')
  assert.equal(row!.kind, 'post_call_survey')
  assert.equal(row!.score, 2)
  assert.deepEqual(row!.issues, ['owa', 'choppy_robotic'])
  assert.equal(row!.commentFull.length, 150)
  assert.equal(row!.commentSent.length, 100, 'Annotations caps Comment at 100')
  assert.equal(row!.annotationStatus, 201)
  assert.equal(row!.annotationError, null)
  s.close()
})

test('a failed annotation preserves the 17007 outcome', () => {
  const s = new Store()
  s.upsertCall(call())
  s.insertFeedback(
    { callSid: 'CA1', kind: 'in_call_button', issues: [] },
    { status: 401, errorCode: 17007, message: 'Advanced Features not enabled' },
  )
  const [row] = s.feedbackForCall('CA1')
  assert.equal(row!.annotationStatus, 401)
  assert.equal(row!.annotationError, '17007 Advanced Features not enabled')
  s.close()
})

test('feedback with no comment stores empty strings, not null', () => {
  const s = new Store()
  s.upsertCall(call())
  s.insertFeedback({ callSid: 'CA1', kind: 'in_call_button', issues: [] }, { status: 201 })
  const [row] = s.feedbackForCall('CA1')
  assert.equal(row!.commentFull, '')
  assert.equal(row!.commentSent, '')
  assert.equal(row!.score, null)
  s.close()
})

test('a redelivered upsert cannot resurrect a completed call', () => {
  // Guards the deliberate omission of started_at/ended_at/verdict_worst from
  // the ON CONFLICT SET clause. Adding them back makes this test fail — which
  // was confirmed by simulating exactly that refactor.
  const s = new Store()
  s.upsertCall(call({ codec: 'PCMU', instrumented: false, startedAt: 1000 }))
  s.endCall('CA1', 9000, 'bad')
  s.upsertCall(call({ codec: 'opus', instrumented: true, startedAt: 1000 }))

  const [row] = s.callsInConference('CF1')
  assert.equal(row!.endedAt, 9000, 'endedAt must not be clobbered')
  assert.equal(row!.verdictWorst, 'bad', 'verdictWorst must not be clobbered')
  assert.equal(row!.startedAt, 1000, 'startedAt must not be clobbered')
  assert.equal(row!.codec, 'opus', 'codec should be updated')
  assert.equal(row!.instrumented, true, 'instrumented should be updated')
  s.close()
})

test('upsertCall updates conference_sid, leg, and instrumented on conflict', () => {
  const s = new Store()
  s.upsertCall(call({ conferenceSid: 'CF1', leg: 'cca', instrumented: false }))
  s.upsertCall(call({ conferenceSid: 'CF2', leg: 'caller', instrumented: true }))

  const [row] = s.callsInConference('CF2')
  assert.equal(row!.conferenceSid, 'CF2', 'conferenceSid should be updated')
  assert.equal(row!.leg, 'caller', 'leg should be updated')
  assert.equal(row!.instrumented, true, 'instrumented should be updated')
  s.close()
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd ~/code/aci-quality-poc && pnpm test`
Expected: FAIL — `Cannot find module '../server/poc/store.ts'`.

- [ ] **Step 3: Write `server/poc/store.ts`**

```typescript
import { DatabaseSync } from 'node:sqlite'
import type {
  CallLeg,
  Feedback,
  FeedbackKind,
  PaneEvent,
  QualityIssue,
  Verdict,
} from '../../shared/types.ts'

/** Annotations caps both Comment and Incident at 100 characters. */
export const COMMENT_MAX = 100

export type CallRow = {
  callSid: string
  conferenceSid: string | null
  leg: CallLeg
  codec: string | null
  /** Recorded at call creation from the then-current ACI state. */
  instrumented: boolean
  startedAt: number
  endedAt: number | null
  verdictWorst: Verdict | null
}

export type FeedbackRow = {
  callSid: string
  kind: FeedbackKind
  score: number | null
  issues: QualityIssue[]
  commentFull: string
  commentSent: string
  annotationStatus: number
  annotationError: string | null
}

const SCHEMA = `
CREATE TABLE IF NOT EXISTS calls (
  call_sid       TEXT PRIMARY KEY,
  conference_sid TEXT,
  leg            TEXT NOT NULL,
  codec          TEXT,
  instrumented   INTEGER NOT NULL,
  started_at     INTEGER NOT NULL,
  ended_at       INTEGER,
  verdict_worst  TEXT
);

-- id is the CloudEvents id for 'aci' rows, so INSERT OR IGNORE makes
-- at-least-once delivery idempotent for free.
CREATE TABLE IF NOT EXISTS events (
  id          TEXT PRIMARY KEY,
  call_sid    TEXT NOT NULL,
  source      TEXT NOT NULL,
  name        TEXT NOT NULL,
  level       TEXT NOT NULL,
  ts          INTEGER NOT NULL,
  received_ts INTEGER NOT NULL,
  edge        TEXT,
  payload     TEXT NOT NULL
);

CREATE INDEX IF NOT EXISTS events_by_call ON events (call_sid, ts);

CREATE TABLE IF NOT EXISTS feedback (
  id                INTEGER PRIMARY KEY AUTOINCREMENT,
  call_sid          TEXT NOT NULL,
  kind              TEXT NOT NULL,
  score             INTEGER,
  issues            TEXT NOT NULL,
  comment_full      TEXT NOT NULL,
  comment_sent      TEXT NOT NULL,
  annotation_status INTEGER NOT NULL,
  annotation_error  TEXT
);
`

export class Store {
  #db: DatabaseSync

  constructor(path = ':memory:') {
    this.#db = new DatabaseSync(path)
    this.#db.exec(SCHEMA)
  }

  close(): void {
    this.#db.close()
  }

  upsertCall(row: CallRow): void {
    this.#db
      .prepare(
        `INSERT INTO calls
           (call_sid, conference_sid, leg, codec, instrumented,
            started_at, ended_at, verdict_worst)
         VALUES (?, ?, ?, ?, ?, ?, ?, ?)
         ON CONFLICT(call_sid) DO UPDATE SET
           conference_sid = excluded.conference_sid,
           leg            = excluded.leg,
           codec          = excluded.codec,
           instrumented   = excluded.instrumented
         -- Do NOT update started_at, ended_at, or verdict_worst: a redelivered
         -- upsert must not resurrect or clobber a completed call.`,
      )
      .run(
        row.callSid,
        row.conferenceSid,
        row.leg,
        row.codec,
        row.instrumented ? 1 : 0,
        row.startedAt,
        row.endedAt,
        row.verdictWorst,
      )
  }

  endCall(callSid: string, endedAt: number, verdictWorst: Verdict): void {
    this.#db
      .prepare(`UPDATE calls SET ended_at = ?, verdict_worst = ? WHERE call_sid = ?`)
      .run(endedAt, verdictWorst, callSid)
  }

  /** Returns false when the id was already present (a duplicate delivery). */
  insertEvent(e: PaneEvent): boolean {
    const result = this.#db
      .prepare(
        `INSERT OR IGNORE INTO events
           (id, call_sid, source, name, level, ts, received_ts, edge, payload)
         VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)`,
      )
      .run(
        e.id, e.callSid, e.source, e.name, e.level,
        e.ts, e.receivedTs, e.edge ?? null, JSON.stringify(e.payload),
      )
    return result.changes > 0
  }

  eventsForCall(callSid: string): PaneEvent[] {
    const rows = this.#db
      .prepare(`SELECT * FROM events WHERE call_sid = ? ORDER BY ts, id`)
      .all(callSid) as Record<string, unknown>[]

    return rows.map((r) => ({
      id: r.id as string,
      callSid: r.call_sid as string,
      source: r.source as PaneEvent['source'],
      name: r.name as string,
      level: r.level as PaneEvent['level'],
      ts: r.ts as number,
      receivedTs: r.received_ts as number,
      ...(r.edge === null ? {} : { edge: r.edge as PaneEvent['edge'] }),
      payload: JSON.parse(r.payload as string),
    }))
  }

  insertFeedback(
    f: Feedback,
    annotation: { status: number; errorCode?: number; message?: string },
  ): void {
    const full = f.comment ?? ''
    const error =
      annotation.errorCode !== undefined || annotation.message !== undefined
        ? `${annotation.errorCode ?? ''} ${annotation.message ?? ''}`.trim()
        : null

    this.#db
      .prepare(
        `INSERT INTO feedback
           (call_sid, kind, score, issues, comment_full, comment_sent,
            annotation_status, annotation_error)
         VALUES (?, ?, ?, ?, ?, ?, ?, ?)`,
      )
      .run(
        f.callSid,
        f.kind,
        f.score ?? null,
        JSON.stringify(f.issues),
        full,
        full.slice(0, COMMENT_MAX),
        annotation.status,
        error,
      )
  }

  feedbackForCall(callSid: string): FeedbackRow[] {
    const rows = this.#db
      .prepare(`SELECT * FROM feedback WHERE call_sid = ? ORDER BY id`)
      .all(callSid) as Record<string, unknown>[]

    return rows.map((r) => ({
      callSid: r.call_sid as string,
      kind: r.kind as FeedbackKind,
      score: r.score as number | null,
      issues: JSON.parse(r.issues as string) as QualityIssue[],
      commentFull: r.comment_full as string,
      commentSent: r.comment_sent as string,
      annotationStatus: r.annotation_status as number,
      annotationError: r.annotation_error as string | null,
    }))
  }

  callsInConference(conferenceSid: string): CallRow[] {
    const rows = this.#db
      .prepare(`SELECT * FROM calls WHERE conference_sid = ? ORDER BY started_at, call_sid`)
      .all(conferenceSid) as Record<string, unknown>[]

    return rows.map((r) => ({
      callSid: r.call_sid as string,
      conferenceSid: r.conference_sid as string | null,
      leg: r.leg as CallLeg,
      codec: r.codec as string | null,
      instrumented: r.instrumented === 1,
      startedAt: r.started_at as number,
      endedAt: r.ended_at as number | null,
      verdictWorst: r.verdict_worst as Verdict | null,
    }))
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd ~/code/aci-quality-poc && pnpm test && pnpm typecheck`
Expected: 13 store tests pass; full suite green (**55 tests** — 3 types, 3 architecture, 8 window, 14 verdict, 14 beacon, 13 store); typecheck exits 0.

- [ ] **Step 5: Commit**

```bash
cd ~/code/aci-quality-poc
git add server/poc/store.ts tests/fixtures.ts tests/store.test.ts
git commit -m "feat: add sqlite store with idempotent event inserts"
```

---

### Task 8: Record the resolved spec contradiction

**Files:**
- Modify: `~/code/pages/docs/superpowers/specs/2026-07-30-aci-quality-poc-design.md` (§6)

The spec sets a 15 s MOS sustain against a 10 × 1 s window, which is unrepresentable if sustain is window-bound. Task 6 resolved it by tracking sustain independently. The spec should say so, or the next reader re-derives the contradiction.

- [ ] **Step 1: Add the resolution note to §6**

In `~/code/pages/docs/superpowers/specs/2026-07-30-aci-quality-poc-design.md`, immediately after the threshold matrix table in §6, insert:

```markdown
**Sustain is tracked independently of the sample window.** The default MOS sustain of 15 s exceeds the 10 × 1 s window, so `BeaconEvaluator` keeps its own per-metric "condition true since" timestamps rather than scanning the window. The window remains 10 samples for `p50Mos` and the verdict; widening it to accommodate sustain would change the verdict's smoothing characteristics for no benefit.
```

- [ ] **Step 2: Verify the spec still has no placeholders and commit**

```bash
cd ~/code/pages
grep -nEi "TBD|TODO|FIXME" docs/superpowers/specs/2026-07-30-aci-quality-poc-design.md || echo "clean"
git add docs/superpowers/specs/2026-07-30-aci-quality-poc-design.md
git commit -m "docs: record that beacon sustain is window-independent"
```

Expected: `clean`, then a successful commit.

---

## Definition of done

- [ ] `pnpm test` — 55 tests pass, 0 fail
- [ ] `pnpm typecheck` — exits 0
- [ ] `tests/architecture.test.ts` demonstrably fails when a forbidden import is introduced (proven in Task 3, Step 2)
- [ ] No runtime dependencies in `package.json`; `typescript` and `@types/node` are the only devDependencies
- [ ] `git log` shows one commit per task

## What Plan 2 picks up

`server/twilio/` (token, twiml, webhooks), `web/twilio/` (device, adapt), the softphone UI, and Docker/Traefik deployment — producing an inbound call into a conference with live quality display driven by the sentinel built here.

Plan 2 consumes from this plan: `QualitySample` and `QualityWarning` as the adapter's output types, `SampleWindow`, `computeVerdict`, `BeaconEvaluator`, and `Store`.
