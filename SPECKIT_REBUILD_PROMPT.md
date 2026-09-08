# EdgeConnect Transactional CLI — clean-room rebuild brief

**A complete Spec-Driven Development package (GitHub Spec Kit) for building, from
scratch, a transactional CLI abstraction layer for HPE Aruba EdgeConnect SD-WAN.**

This document contains no source code from any prior implementation. It carries
three things: the product specification (what to build and why), the governing
decisions already made (so the build does not re-litigate them), and a field
guide of verified HPE EdgeConnect Orchestrator/ECOS API behavior that took real
lab time to learn. Everything an agent needs is in this file; everything it
must not do is stated explicitly.

---

## §0 — Operator guide (read this yourself; the agent starts at §1)

### What this is

You are standing up a **new repository in a new environment** and rebuilding
this product from a specification, not from prior code. This brief is the
specification. The agent must treat it — plus the public vendor artifacts named
in §A.14 — as its only references.

**Intellectual-property note.** This brief specifies behavior, records design
decisions, and states facts about a vendor's public API. It contains no code.
If your migration constraint is contractual or legal rather than merely
hygienic, have the appropriate party review this document before use: a
specification written with knowledge of a prior implementation is a strong
practical clean-room input, but it is not a formally partitioned clean-room
process on its own.

### Names are placeholders

This brief uses **`ecx`** everywhere a project identity appears:

| Placeholder | Meaning | Replace with |
|---|---|---|
| `ecx` | CLI binary name and Python package | your chosen name |
| `ECX_*` | environment-variable prefix (`ECX_ORCH_URL`, `ECX_API_KEY`, `ECX_ENVELOPE_KEY`, `ECX_INSECURE`) | your prefix |
| `~/.ecx/` | per-user state directory | your dot-dir |
| keyring service `ecx` | OS keyring service name | your service name |

Do a global search-and-replace before feeding the stage inputs, or tell the
agent the chosen name in your first message and let it substitute.

### Bootstrap

1. Create the new empty repository; initialize git.
2. Install Spec Kit and initialize the SDD scaffolding for your agent harness
   (Spec Kit supports Claude Code, Copilot, Gemini CLI, Cursor, and ~30 others):

   ```bash
   uv tool install specify-cli --from git+https://github.com/github/spec-kit.git
   specify init . --integration <your-agent>     # older releases: --ai <your-agent>
   ```

   This installs the `/speckit.*` commands (`constitution`, `specify`,
   `clarify`, `plan`, `tasks`, `analyze`, `implement`, `checklist`) into the
   harness. If the harness has no slash-command support, paste each stage
   input as a plain instruction — the flow is the same.
3. Commit **this file** into the new repo as `docs/rebuild-brief.md` on day 0,
   and tell the agent to split Appendix A out as `docs/research/api-field-guide.md`
   and Appendix C as `docs/research/field-lessons.md`. They are living reference
   documents, not one-shot prompt text.

### Stage map — feed in this order

| Step | Command | Input | Gate before proceeding |
|---|---|---|---|
| 1 | (first message) | §1 mission preamble | agent restates mission + constraints correctly |
| 2 | `/speckit.constitution` | §2 | constitution committed; you ratify it |
| 3 | `/speckit.specify` | §3 (Feature 001 — grammar/IA) | — |
| 4 | `/speckit.clarify` | answer from §7 register | no unresolved blocking question |
| 5 | `/speckit.plan` → `/speckit.tasks` → `/speckit.analyze` → `/speckit.implement` | §8 for the plan | milestone M1 exit criteria (§9) |
| 6+ | repeat 3–5 for Features 002–006 (§4, §5, §6 …) | same pattern | per-milestone exit criteria (§9) |

Grammar is deliberately **Feature 001**. In the prior effort the command
taxonomy was retrofitted late, after the surface had grown four meanings of
`show` and two spellings of scope — the single largest source of rework and
operator confusion. The rebuild ratifies the grammar before the first command
ships.

### Where you (the human) are required

- **Ratify** the constitution and each feature spec (Spec Kit will pause for
  this; the §7 register pre-answers the known decision points).
- **Live-fabric evidence sessions.** The agent can reach `mock-verified` on its
  own. Every claim above that (live-read, no-op-write, change-and-rollback)
  requires a session against a real Orchestrator that you supervise, following
  §5's evidence ladder. Do not let the agent label anything live-verified
  without one.
- **Provision the shape-survey targets.** The single highest-leverage thing
  you can do for build quality: give the agent read-only reach into **2–3
  real Orchestrators on different releases** (lab boxes; cloud and on-prem if
  you have both) early — a least-privilege read-only API key per box where
  the release supports scoped keys, else a key for a read-only user role.
  The agent fans out one survey worker per Orchestrator (Appendix E) and
  writes the resources against observed response shapes instead of the
  spec's guesses. Survey campaigns are read-only by construction (the
  harness's transport refuses writes and deny-listed GETs), but review
  Appendix E's rules with the agent before the first run all the same.
- **Credentials.** Never paste an API key into the transcript. Environment
  variables or OS keyring only (§4 spec covers the rules the tool itself must
  enforce).

---

## §1 — STAGE INPUT: agent mission preamble

Paste this as the opening instruction (and/or into the repo's agent
instructions file, e.g. `AGENTS.md` / `CLAUDE.md`):

---

You are building **`ecx`**: a transactional CLI abstraction layer for HPE Aruba
EdgeConnect SD-WAN. Everything an operator does in the Orchestrator UI —
orchestrator-level (Business Intent Overlays, security policy, templates and
template groups, associations) and appliance-level (interfaces, BGP, OSPF,
DHCP, VRRP, routes) — becomes expressible from a Junos-flavored CLI, with the
transactional semantics the Orchestrator API never had:

- **Candidate configuration**: `set`/`delete` accumulate locally; nothing
  touches the Orchestrator until `commit`.
- **`show | compare`**: a canonical diff of exactly what commit will send.
- **`commit confirm <minutes>`**: auto-rollback by a detached watchdog that
  survives SSH death, unless confirmed in time.
- **`rollback <n>`**: Junos-style history from crash-safe journaled snapshots.
- **Template-ownership detection, fail-closed**: direct appliance changes that
  a template push would silently revert are refused without an explicit
  override — and so are changes whose ownership could not be established.
- **Tier-0 raw passthrough**: `ecx api get|post|put|delete <path>` reaches any
  Orchestrator or appliance-proxy endpoint from day one — journaled for audit,
  loudly outside the transaction guarantees.

### Your references — complete and closed

1. `docs/rebuild-brief.md` (this brief): specification, decisions, API field
   guide, behavioral contracts, field lessons.
2. The public vendor artifacts in its §A.14: the Orchestrator's own OpenAPI
   documents, the vendor's published Postman collections, and the open-source
   `pyedgeconnect` SDK — as *reference material for endpoint facts only*.
3. A live Orchestrator, **only** in operator-supervised evidence sessions.

You must **not** consult, retrieve, or reproduce any prior implementation of
this product. If you believe you have seen one, say so and continue from this
brief alone.

### Operating rules

1. **Follow the SDD flow**: constitution → specify → clarify → plan → tasks →
   analyze → implement, per feature, in the order §9 gives. The constitution
   check in a plan *blocks*; a non-negotiable principle left unsatisfied stops
   the plan.
2. **Never invent an endpoint, payload field, or job shape.** Every API claim
   in code or docs carries one of the evidence tags defined in §A.0. Where the
   vendored spec and this brief disagree, stop and report; where both are
   silent, the resource stays a stub that says so.
3. **Shapes before code.** The vendor's spec and any mock are *models* of the
   API; real Orchestrators diverge from both, and they diverge from each
   other across releases. Before curating a resource family, run the
   read-only shape survey (Feature 004 harness; Appendix E protocol) against
   every Orchestrator the operator provisions, and write `normalize()` and
   the views against the **observed** shapes — captured as committed,
   sanitized fixtures — with the spec as the fallback only where no capture
   exists. A resource written against spec or mock alone says so in its
   evidence record. Where you have the harness and credentials, surveying is
   your job, fanned out one worker per Orchestrator; do not ask the operator
   to paste responses by hand.
4. **Fail closed, everywhere.** Unknown, partial, stale, timed-out, or
   unverified outcomes never read as success. When in doubt between refusing
   and guessing, refuse with a reason.
5. **Evidence-gate your claims.** "Implemented", "mock-verified",
   "live-read-verified", "live-change-and-rollback-verified" are different
   statements. Documentation and coverage output state which one is true.
   Without a live session, your ceiling is mock-verified — say so.
6. **Clarifications**: consult §7 (pre-answered decision register) before
   asking the operator. If a genuinely new product question arises, batch your
   questions (up to three), state your recommended default for each, and if
   the operator is absent, proceed on the recommended default and record it in
   the feature spec as an assumption with an expiry.
7. **Definition of done** for any task: local gate green (lint + strict types +
   tests), the feature's acceptance criteria demonstrably checked, and — for
   anything user-visible — the installed-wheel smoke test passing from outside
   the source tree.
8. **Session hygiene**: keep a `docs/sitrep/` note per working session — what
   changed, what is verified at which level, what is deliberately left open.

---

## §2 — STAGE INPUT: `/speckit.constitution`

---

Create the project constitution from the following principles. Mark each as
shown. Every principle carries a review question a reviewer can answer yes/no
about a specific diff; a principle that cannot be checked is a preference and
belongs in CONTRIBUTING, not here. Non-negotiable principles have no exception
path; conventions may be departed from with a recorded reason **and an
expiry** in the feature's spec.

**I. Intent-separated interfaces — non-negotiable.**
Operational state, candidate configuration, committed running configuration,
and native vendor output are four different things. A command names which one
it means, and no token sequence may mean two of them.
*Review question:* for every command this change adds or alters, can you name
its single intent (operational / candidate / running / native), its source,
and its scope from the command tokens alone, without reading the
implementation?
*Checked by:* the grammar spec's command table; a command absent from it is
not shippable.

**II. Safety truth over convenience — non-negotiable.**
Unknown, partial, stale, timed-out, or unverified outcomes never become
success by omission. Where the truth cannot be established, the command says
so; it does not pick the reassuring interpretation. This includes the inverse
direction: a response the code does not understand is an **error**, never
"absent" or "empty" — coercing an unrecognized shape to a confident negative
is the same defect as coercing it to success.
*Review question:* for each way this change can fail to establish the truth —
empty, absent, unsupported, denied, unreachable, timed out, stale, partial,
malformed — does the operator get a distinct, visible outcome? Can any of them
be mistaken for success, or for a confident "not present"?
*Checked by:* per-state tests, not a single happy-path test. `{}`, `None`,
`""`, HTTP 204, `[]`, and an unexpected-shape 200 each have intentional,
separately-tested meanings; a renderer may never reduce a valid response to
zero visible characters.

**III. Model-first, with a labelled native escape hatch — convention.**
The normalized, model-driven view is primary. Native vendor output remains
available, is always explicitly requested, and is always labelled native.
*Review question:* can an operator reach native output without asking for it
by name? If a model cannot represent the data, does the command say so rather
than silently degrading to native?

**IV. One grammar, one resolution path — convention.**
The interactive shell, the scriptable CLI, and machine output share nouns,
scope ordering, and error semantics; and every surface that names an instance
— completion, listing, show, set, drift — resolves that name through the same
single implementation.
*Review question:* does every command added here have a one-to-one canonical
mapping between shell and scriptable forms, with the same nouns in the same
order? Is there exactly one function that turns (scope, noun, name) into a
fetchable reference, and do completion and fetch both use it? Would a name
returned by enumeration round-trip through fetch?
*Checked by:* paired tests over one fixture exercising both surfaces, plus a
registry-wide round-trip property test (every enumerated reference fetches).

**V. Evidence-gated support — non-negotiable.**
"Implemented", "green against the mock", "live-read verified", and
"live-write verified" are different claims. Documentation, coverage output,
and issue status state which one applies; the stronger claim is never implied
by silence. Claims above mock level are invalid without a recorded
Orchestrator version, ECOS version, auth mode, date, and a re-readable source.
*Review question:* what is the strongest evidence that actually exists for
this change — spec, mock, live read, live write? Does every place that
describes it to a user say exactly that, and not more?
*Checked by:* a machine-readable evidence ledger validated on load; coverage
output reads it; tests refuse a ledger entry that over-claims.

**VI. Reversible evolution — convention.**
Renames and grammar changes ship with explicit, warned, tested compatibility
behavior, removed only at a declared boundary. An old command form either
keeps working or fails loudly naming its replacement — it never silently
changes meaning.
*Review question:* is there any input for which an old form now returns a
different kind of data without saying so?

**VII. Secrets stay sealed — non-negotiable.**
Credential material — the operator's API key, and fabric-held secrets such as
BGP neighbor passwords, SNMP communities, and OSPF auth keys — never appears
on argv, in rendered output, in exported audit records, in exception text, or
unencrypted in persisted state. Masking preserves the field name and a
truncated digest (a change hint), never the value. Where no sealing key is
available, secret-bearing writes fail closed before touching disk; sealed
state that cannot be opened fails loudly and is never read as "no value here".
*Review question:* seed a sentinel secret through this change's paths — does
it appear anywhere on disk unencrypted, or in any rendered/exported surface
unmasked?
*Checked by:* sentinel-sweep tests across candidate, journal, exports, diffs,
errors, and logs.

Include the standard sections: exceptions (conventions only, reason + expiry),
amendment process (owner ratifies; semantic versioning; amendment history
table), and a statement that the constitution check in plans blocks rather
than records.

---

## §3 — STAGE INPUT: `/speckit.specify` — Feature 001: command grammar & information architecture

---

**Feature: the complete CLI command taxonomy, ratified before any network
command ships.** This is first deliberately: in field use, an organically
grown surface ended up with one `show` root meaning four different things,
shell and scriptable forms spelling scope differently, instance names that
completion offered but fetch could not find, and reports that dumped raw
data-structure text into human tables. The grammar is the contract everything
else implements against.

### The four intents

Every read command resolves to exactly one; no token sequence may resolve to
two (Principle I):

| Intent | Means | Source | Freshness |
|---|---|---|---|
| operational | running state of the network — sessions, routes, flows, versions | live device/Orchestrator read | live |
| running configuration | configuration as it exists on the target now, normalized | live read (Orchestrator or appliance proxy) | live |
| candidate | staged intent, not yet committed | local state dir | local |
| native | the vendor's own configuration text | live read via proxy | live |

`native` is a **format of running configuration** (a `--format native` value),
not a fifth intent — but operators think of it as a thing, so the grammar must
answer them where they are.

### Canonical shape

```
show [<scope-noun> <scope-key>] <domain> [<collection>] [<instance>] [flags]
```

- **Scope is outermost-first and mandatory** — scope → kind → instance,
  identical across shell and scriptable surfaces. EdgeConnect has no implicit
  "this device"; every command has a subject.
- Scope nouns: `appliance <name>` (one appliance, via the Orchestrator proxy;
  cost class *single*), `fabric` (every appliance, bounded fan-out; cost class
  *fanout*), and *none* (the Orchestrator itself; *single*).
- Operational state: `show appliance <name> <domain> [<collection>] [<instance>]`,
  `show fabric <domain> ...` (e.g. `show appliance BR1 bgp summary`,
  `show fabric flows summary`).
- Configuration: `show configuration [running|candidate] [appliance <name>|fabric] [<kind> [<instance>]]`
  and `show configuration appliance <name> --format native`.
- **The datastore token is optional and defaults to `running`.** `candidate`
  is never implicit — the only unnamed datastore is the live one, so an
  operator can never be shown staged intent while believing they see the
  device.
- Configuration mode (the interactive `configure` sub-mode): mode carries the
  intent, so bare `show` is the candidate at the current level; `show compare`
  (and the alias `show | compare`) is candidate-vs-running.
- CLI-state reads live under bare `show` and are a deliberate fifth category
  (subject = the tool, not the fabric): `show journal`, `show pending`,
  `show locks`, `show coverage`, `show commands`.
- A **nonterminal lists its valid continuations and exits 0**; it never picks
  one and never makes an API call. Static help never costs a request.

### Reserved words

Because the datastore token is optional, these are reserved and rejected as
kind aliases at startup: `running`, `candidate`, `appliance`, `fabric`,
`configuration`, `orchestrator`, `orchestrators` (the last two reserved ahead
of a future multi-Orchestrator selector — see §7 D-9).

### Nouns and aliasing

Resource kinds are addressed by **user-facing nouns scoped by the command**,
never by internal registry keys. Internal keys may carry prefixes (scope,
generator provenance); no user surface — completion, help, errors, docs,
flags (including any `--kind` flag on drift/apply) — ever shows or accepts
them. The alias namespace is **per scope**, not flat: appliance firewall
zones and the Orchestrator's zone definitions are both naturally `zones`, and
scope disambiguates (`show appliance BR1 zones` vs `show configuration zones`).
Uniqueness within scope and reserved-word rejection are checked at
registration time so a collision fails for a developer, never for an operator.

### One resolution path (hard requirement, from a real defect)

There is exactly one implementation that turns (scope, noun, instance-name)
into a fetchable reference, and every surface uses it: shell completion, bare
nonterminal listings, `show`, `set`/`delete`, drift enumeration. Enumerating
and fetching must agree structurally, and this is a registry-wide invariant
test: **for every kind, every reference its enumeration returns must fetch
successfully** against the same backend. The observed failure this rule
exists to prevent: completion listed four overlay names from one endpoint;
fetching any of those names returned "(not present)" because the fetch path
resolved and interpreted the same endpoint differently. An instance the tool
itself just listed must never be reported absent; if fetch cannot interpret
what the server returned, that is an `error` naming the unexpected shape
(Principle II), never `not_found`.

### Outcomes and exit codes

Named outcomes shared by human and JSON output; the exit code says which
terminal state was reached:

| Outcome | Meaning | Exit |
|---|---|---|
| `ok` | result, non-empty | 0 |
| `empty` | target answered; genuinely empty | 0 (explicit "no configuration" line — never zero output) |
| `not_found` | path valid, object positively does not exist | 4 |
| `unsupported` | valid ask; this software/API does not implement it | 5 |
| `invalid` | syntactically wrong command | 2 |
| `denied` | permission refused | 6 |
| `unreachable` | target not reachable | 7 |
| `timeout` | bounded wait elapsed | 7 |
| `partial` | fan-out: some targets failed (per-target rows, failures marked) | 8 |
| `stale` | served from cache, only ever under explicit `--stale-ok` | 0 (with age + source annotation) |
| `error` | malformed/unexpected response (names target, kind, operation, cause) | 1 |

Comparison-family commands (`diff`, `drift`, `apply --dry-run`) use the
git-style convention as a **declared exception**: 0 = no differences,
1 = differences found, 8 = incomplete — and **8 outranks 1**: a run that could
not evaluate part of its scope has not earned "clean" *or* a plain drift
answer.

Renderer rule (Principle II): no valid response may render as zero visible
characters, and `empty`/`not_found`/`unsupported`/`error` are visibly
different sentences.

### Fan-out cost declaration

A `fabric` command makes one call per appliance. Interactive (TTY): prompt
first, naming the appliance count and a coarse duration estimate; `--yes`/`-y`
skips. Non-interactive: never prompt (a prompt hangs pipelines) — warn on
stderr with the same two figures and proceed. The count comes from the
resolver cache so the warning itself costs no API call. **Test the wiring on
every fan-out command**, both TTY and piped: a cost guard that exists but is
not on a command's path is the observed failure mode (a fabric-wide
configuration report was seen issuing 18 proxied calls with no declaration).

### Human rendering discipline

- Human output never contains raw serialized data structures (no
  `{'key': ...}` dict/list reprs in table cells). Nested structures are
  projected into designed columns or summarized fields; full structure belongs
  to `--format json|yaml`.
- Tables have a width budget; wide value sets summarize (counts, first-N +
  "…") rather than wrap into unreadable walls.
- Diagnostics (API call logs, timing) go to **stderr** via the logging
  subsystem, never interleaved into stdout program output, and default to
  quiet in the interactive shell; a verbosity flag turns them on.
- Fabric/summary reports are composed of per-section views each with its own
  designed schema, and each section is also reachable alone
  (`show configuration fabric <section>`).

### Flags

`--format {yaml,json,native}` on reads (`native` valid only for running
configuration); `--appliance <name>` as the scriptable spelling of the
appliance scope (same nouns, same order); `--max-concurrency`, `--timeout`
(bounded, always); `--stale-ok` (**opt-in**; without it a read is live or it
fails); `--yes`.

### New-verb admission

The write/lifecycle verbs are: `set`, `delete`, `load`, `commit`, `confirm`,
`discard`, `rollback`, `apply`, plus `api` (Tier-0). A new top-level verb is
admissible only if all three hold: (V1) it acts rather than reports (reads
belong under `show`); (V2) no existing verb can carry it without changing that
verb's meaning; (V3) its intent is neither one of the four read intents nor a
transaction-lifecycle transition. A flag on an existing verb is the default;
a flag repeated across three commands to express one operation counts against
V2, not for it.

### Acceptance criteria (turn into tests)

1. Every command maps to one intent, source, scope, cost class, and output
   schema; the grammar table is parsed by a test so code and spec cannot
   drift.
2. No token sequence resolves to both operational state and configuration.
3. Nonterminals list continuations, exit 0, and make zero API calls
   (request-count asserted).
4. Shell ↔ scriptable one-to-one mapping over a shared fixture.
5. Registry-wide enumeration→fetch round-trip property holds.
6. Reserved words rejected as aliases at startup; per-scope uniqueness
   enforced; the `zones` collision resolved by scope alone.
7. Every outcome distinguishable in human and JSON modes; the
   zero-visible-characters rule tested over `{}`, `None`, `""`, 204, `[]`,
   and an unexpected-shape 200.
8. Fan-out declaration fires on every fanout command — TTY (prompt) and piped
   (stderr warning) both tested per command, not per helper.
9. No user surface emits an internal registry key (asserted over completion,
   help, usage, error, and doc output).
10. `show commands` renders the whole surface offline — intent, scope,
    mutability, support status — generated from the parser's own tables.

---

## §4 — STAGE INPUT: `/speckit.specify` — Feature 002: the transactional core

---

**Feature: candidate → compare → commit(-confirm) → rollback over a crash-safe
journal, with a detached watchdog, host-scoped locking, and one canonical
identity per Orchestrator.** This is the product's reason to exist; every
guarantee here is load-bearing and every mechanism below exists because its
absence was a real failure mode.

### Candidate configuration

- `set <noun> <instance> <path...> <value>` / `delete ...` stage typed intent
  locally under `~/.ecx/`, keyed by target identity (see below). Nothing
  touches the Orchestrator until commit.
- The candidate is **client-side staged intent, not a materialized
  configuration tree** (the Junos candidate lives on the device; this one
  cannot). It is materialized against fresh server state at compare/commit
  time.
- Candidate mutation is a **locked read-modify-write cycle**, not merely an
  atomic file replace: atomic replacement prevents a torn candidate but not a
  lost one (two shells that both loaded at T0 and saved at T1/T2 keep only the
  second's work). Pin a test to that distinction.
- One shell must not silently commit or clear staged intent created by another
  session; unacknowledged foreign staging is surfaced and requires
  acknowledgement.
- `discard` drops staged intent. `load <noun> <instance> <file>` stages a full
  desired document through the same canonicalization path as `set`.

### Compare and plan

- `show | compare` / `diff` renders the canonical structural diff of exactly
  what commit will send: desired intent is canonicalized through the same
  normalization path as server state, so both sides meet in one canonical
  shape. Diffing raw server JSON against raw intent is forbidden — it
  manufactures phantom drift.
- Planning walks: dependency ordering (a declared `dependencies` relation,
  topologically sorted, e.g. group before association) → per-item ownership
  check → shared-write-target collision check → reversibility gate. A plan
  refusal names the item, the guard, and the remedy.

### Commit and commit-confirm

- `commit` materializes the plan, snapshots each touched object (fresh
  re-fetch immediately before write — snapshot-before-write, at commit time,
  never reusing compare-time state), applies in dependency order, verifies
  each item post-apply against freshly fetched state (not against what was
  staged), and journals every step.
- **Commit-time drift fails closed.** If server state moved since compare, the
  commit refuses; `--rebase` explicitly opts in to re-merging intent over
  current state and re-displaying the plan. Silently recomputing the diff and
  carrying on folds another operator's change into this changeset.
- **Partial failure auto-reverts**: on a mid-changeset failure, already-applied
  steps are reverted from the journal snapshots and the report states exactly
  what state the fabric is in. A failed revert is its own loud terminal state
  (`REVERT_FAILED`) naming exactly what remains un-reverted.
- `commit confirm <minutes>` applies, then arms a **detached watchdog** that
  reverts at the deadline unless a bare `commit` (confirm) lands inside the
  window. Confirm-vs-revert must be atomic (a confirm marker and a revert
  cannot both win).
- `commit confirm` **requires API-key auth**: a background watchdog cannot
  replay an interactive session login, and pretending otherwise is fake
  safety.

### The watchdog (contract, not implementation detail)

- Default backend: a detached daemon (double-fork + setsid; cwd `/`, umask
  077, fds to a per-transaction log). Immune to SIGHUP/SSH teardown; requires
  nothing beyond POSIX.
- **Arming is verified**: the committing process waits for the watchdog to
  write its own pid file; if it does not appear within a bound, the commit
  auto-reverts immediately rather than leaving an unprotected unconfirmed
  change.
- "Is this transaction being driven?" is answered by **a running process**
  (pid + process start-time token liveness probe), never by a scheduled
  intention. A timer that runs nothing until it fires would make every armed
  transaction look orphaned for the whole window — the watchdog's job is to be
  *present* so its absence is evidence. (A systemd transient *service* is an
  acceptable documented alternative backend, with the linger caveat for
  SSH-only accounts; a systemd *timer* is structurally wrong.)
- On every CLI start: an **orphan scan** (non-terminal transaction + no live
  watchdog) reports and offers `rollback --pending`. Covers host reboot inside
  the window and a manually killed watchdog.

### Journal (crash-safe, and it doubles as the audit log)

Per-transaction directory under `~/.ecx/journal/`:

- a small state index, atomically rewritten (tmp + rename + directory fsync):
  transaction id, target identity (canonical origin + display host), state
  machine, confirm deadline, item references. States:
  `PENDING → APPLYING → APPLIED_UNCONFIRMED | CONFIRMED → REVERTING →
  REVERTED | REVERT_FAILED`, plus `AUDIT_ONLY` for Tier-0 calls.
- an append-only event log, fsync per record — **the source of truth**; on
  meta/event disagreement, events win. A torn final line after a crash is
  tolerated on read; any deeper corruption is **named, not skipped** — an
  audit trail that quietly describes a smaller history than the one on disk
  is worse than none.
- snapshot **bodies live in a rollback-private file, not the exportable event
  log**; the event log carries each body's SHA-256 digest and size. Rollback
  cross-checks body against recorded digest and refuses on mismatch, so a
  lost or tampered snapshot can never be silently restored; `rollback <n>`
  verifies its restore and refuses a lost snapshot.
- a confirm marker file (fsynced) and the watchdog pid file.
- History: last N terminal transactions kept (default 10); **non-terminal
  journals are never pruned**. `rollback <n>` restores the nth prior
  CONFIRMED transaction's snapshots as a **new journaled transaction** —
  rollbacks are themselves revertible — and authorizes its source (target
  identity match) before restoring anything.
- Audit surfaces: `show journal` (human, newest first), `--json`, and
  `--events` (one self-contained NDJSON record per line, stamped with
  transaction and Orchestrator identity, oldest first, SIEM-shippable).
  Snapshot bodies are **redacted by default** in exports (digest + size
  survive, so an auditor can prove two exports describe the same state
  without seeing config); `--include-snapshots` opts in with secret-named
  fields still masked. Selecting a transaction id not in the journal is an
  error, never an empty export.

### Locking

- Host-scoped advisory locks: `flock` where available (the kernel drops it on
  SIGKILL, so a stale lock is structurally impossible), with an
  `O_EXCL`-plus-start-token fallback elsewhere.
- One **commit lock** shared by commit, confirm, revert, and rollback; the
  watchdog's revert takes the same lock but waits far longer — a watchdog that
  gave up would leave an unconfirmed change applied. The lock record names the
  transaction holding it; `show locks` reports holders.

### One identity per Orchestrator

- Every piece of persisted or compared state — candidate store, locks,
  journal and rollback history, resolver cache, keyring entry — is keyed by a
  **canonical origin**: `scheme://host[:port][/path]`, derived from the
  *effective API base* through the same function the HTTP client uses to build
  it (so `https://x` and `https://x/gms/rest` are one identity, while
  `http://x`, `https://x:8443`, and `https://x/tenant-b` are different ones).
  File names embed a digest of the unsanitized origin.
- The display host exists for rendering only. **Nothing keys state by it** —
  enforce with a source-scan test (with a reasoned allowlist) covering the
  test tree too, because the two are both plain strings and the type system
  cannot tell them apart.
- A snapshot or candidate staged against one origin is refused against any
  other. (A clean-slate build has no legacy hostname-keyed state, so no
  adoption mechanism is needed — but the refusal itself is day-one behavior.)

### Tier-0 raw passthrough

`ecx api get|post|put|delete <path> [--body file] [--appliance NAME]` reaches
any Orchestrator endpoint, or any ECOS endpoint via the appliance proxy, from
day one. Journaled as `AUDIT_ONLY` with parameters recorded through redaction;
prints a no-rollback-guarantees banner; never enters the candidate or a
confirm window; **never retried** (Tier-0 cannot prove an arbitrary GET is
safe to replay — §A.13).

### Credentials and transport

- Configuration: `ECX_ORCH_URL` (+ optional `ECX_API_KEY`); OS keyring
  fallback (service `ecx`, username = canonical origin) consulted only when
  the env var is unset. A keyring that is installed but will not open (locked
  session, no D-Bus) is reported as exactly that — it is a different answer
  from "no key stored", and conflating them sends the operator to re-store a
  key that was already there.
- Credentials never on argv, never journaled, never printed; settings redact
  their own key in repr; API error text is scrubbed at construction (an
  exception's text is the one string guaranteed to travel).
- TLS verification on by default; the insecure switch exists and nags every
  run.
- Timeouts on every network call, clamped to sane bounds; retries bounded
  with backoff and only where §A.13's policy allows.

### Acceptance criteria

Encode §B.1–§B.4 (behavioral contracts appendix) as tests. Headline items:
watchdog survives SSH-parent death and reverts on deadline (real detached
process in a slow-marked e2e test); arming failure auto-reverts; orphan scan
detects reboot/kill; two-shell lost-update prevented; commit-time drift
refuses without `--rebase`; partial failure auto-revert; REVERT_FAILED loud;
rollback digest verification; rollback-of-rollback; origin separation
(`http://` vs `https://` are two stores); journal torn-line tolerance and
corrupt-directory naming; Tier-0 audit record with masked params.

---

## §5 — STAGE INPUT: `/speckit.specify` — Feature 003: safety systems

---

**Feature: the guards that make writes trustworthy — template ownership,
async-job truth, reversibility, shared write targets, secrets at rest, retry
policy, and the evidence ladder.** Build these *with* the first write path,
not after it: in field use every one of these was a retrofit forced by an
incident or a near-miss.

### Template ownership, fail-closed and change-aware

The Orchestrator pushes template groups; a direct appliance write to a
template-governed piece of config is silently reverted by the next push. The
API exposes no "managed-by" field; ownership is **derived** (§A.7 has the
verified model and endpoints). Requirements:

1. Ownership is a tri-state: `owned` / `unowned` / `unknown`. Only a positive
   `unowned` permits a free write; `owned` and `unknown` both refuse without
   `--override-template`. Unreadable association, unreadable selection,
   unreadable template body, or a section name the fabric's vocabulary does
   not contain ⇒ `unknown`, never a confident `unowned`.
2. Ownership answers a question about **the change**, not the kind — the
   check receives the diff. Live evidence (§A.7): templates govern *fields*
   for some sections (BGP templates carry timers/defaults, never peer
   existence — adding a peer is locally significant and must be permitted)
   and govern *entries by priority band* for map-type sections
   (Orchestrator-pushed entries occupy a priority band; out-of-band entries
   are local). Three layers, in order:
   - **L1 — vocabulary from the fabric**: section names are read from the
     live template-group listing, never guessed; an unknown mapping answers
     `unknown`.
   - **L2 — priority band, structural**: for changes addressed to
     priority-keyed entries, governed iff the priority falls in the band —
     and the band is **derived from the selected template's own entries**
     (self-describing, survives a release that moves the range), falling back
     to `unknown` when the body cannot be read.
   - **L3 — declared governed subtrees**: for sections the template
     partitions by field, the resource declares which subtrees the template
     governs (e.g. BGP: system-level settings only); a change under a
     governed subtree is owned, outside is not. Needed by BGP/OSPF/VRRP/
     routes; everything else keeps whole-resource behavior.
   - A changeset spanning governed and local parts is refused whole ("commit
     them separately" is the remedy).
3. Never build ownership decisions on the API's `gms_marked`,
   `options.merge`, or `templateApply` flags — live data contradicts all
   three (§A.7).
4. The ownership check is **re-read immediately before the write phase, under
   the commit lock** — associating a template group between compare and
   commit must not slip past the guard.
5. The refusal names the owning group and the exact override.

### Async jobs: success is an allowlist

Most Orchestrator writes are asynchronous (§A.5). Requirements:

1. Every action key is polled to a terminal state; `apply()` returns only
   after terminal or timeout; **timeout is failure**.
2. Terminal-record classification: failure tokens (in status or result) ⇒
   FAILED; a status matching no known success token ⇒ UNKNOWN; empty result
   on a success status ⇒ SUCCESS; result matching the **success-shape
   allowlist** ⇒ SUCCESS; anything else ⇒ **UNKNOWN, and UNKNOWN fails the
   transaction**. Never infer success from the absence of failure words — no
   failure-word list is ever complete (a localized Orchestrator defeats it
   instantly).
3. An unknown shape is quoted verbatim back to the operator in the failure
   detail, which is how the allowlist grows; every allowlist entry records
   the Orchestrator/ECOS version it was observed on (§A.5 carries the two
   entries already earned).
4. Keyless writes are confirmed by evidence, not by the absence of an error:
   the save-changes flag re-polled until clear (§A.5), and fire-and-204
   pushes correlated via the action log by target + time window, answering
   UNKNOWN when more than one record shares the window.
5. Do not use the job record's boolean completion field as a tiebreaker — it
   is observed false on success for some operations (§A.5).

### Reversibility classes

Declared per resource, enforced by the engine: **REVERSIBLE** (exact
snapshot/restore) and **COMPENSABLE** (compensating action exists; the
compensator tolerates an already-absent object) both support commit-confirm;
**IRREVERSIBLE** (deletes, upgrades, licenses — no undo) refuses confirm
windows and requires `--force` — a confirm window that cannot actually revert
is fake safety. A generated stub may claim at most COMPENSABLE, and only when
a GET exists on the write's path (no GET ⇒ nothing to snapshot ⇒
IRREVERSIBLE); nothing in a spec proves a GET's response is accepted verbatim
by the write, so a stub never claims REVERSIBLE. For COMPENSABLE rollbacks,
verify what the compensator leaves behind, not just that it ran.

### Shared write targets

Two individually-correct changes can replace the same server object (the
worked case: appliance deployment and DHCP both write the deployment object —
DHCP is a subtree of it). Ordering is not a fix. Requirements: each curated
kind either **declares the server object its apply replaces** or records why
a target would be wrong for it (per-entry editors, delta reconcilers,
read-modify-write splicers) — the partition is total and enforced by a
registry-wide test that refuses an undecided kind. Planning refuses a
changeset containing two items that replace one target, before the first
write, with a structured collision object naming the target and every
conflicting reference. **Never derive targets mechanically** — the property is
semantic; a mechanical derivation was tried and produced garbage
(longest-common-prefix of two template paths), and a wrongly asserted target
causes false refusals. Pin a test to the exactly-known collision set so new
declarations cannot silently introduce false refusals.

### Secrets at rest and in flight

Four layers, in order, each covering the previous one's hole:

1. **Detection**: one shared name-token detector (`password`, `community`,
   `authKey`, `token`, … any spelling); biased toward false positives — the
   cost of over-matching is a hidden value, never a leaked one.
2. **Separation**: snapshot bodies live in the rollback-private store, not
   the exportable event log, regardless of detection — covering secrets under
   unrecognized names.
3. **Encryption**: detected secret values in the candidate, and whole
   snapshot bodies in the private store, are envelope-encrypted (AES-256-GCM)
   under a key from `ECX_ENVELOPE_KEY` (exclusive when set) or an
   auto-created OS-keyring entry. No key ⇒ secret-bearing **writes fail
   closed before disk**; sealed state with no/wrong key ⇒ loud error, never
   "no value here" (a revert that reads an unopenable snapshot as absent
   would delete the resource). Blobs are never modified by a failed open.
4. **Redaction at every exit**: rendered diffs/plans, candidate display,
   audit exports, Tier-0 journaled parameters, exception text. Masked value =
   field name + truncated digest (change hint).

`ecx rotate-key` re-seals everything under a fresh key, crash-safe by
ordering (outgoing key parks in a previous-key slot before the new key
replaces it; deleted only after every blob is rewritten; interrupted rotation
is simply re-run). Refuses under an environment-supplied key (the exporter
owns that lifecycle). Document plainly: once anything is sealed, the state
directory alone is not a backup.

### Retry policy: the transport holds a veto

GET is **not** idempotent on this API — vendored specs describe GETs that
clear counters, generate dumps, log out sessions, and delete BGP state
(§A.13). Policy: per-call classes (bounded-with-backoff for reviewed curated
reads; never for Tier-0 and for writes), plus a spec-derived deny list of
mutating GETs that overrides every caller. Derive the deny list from the
vendored spec's own operation summaries and gate it with a test: any GET
whose summary opens with an action verb must be classified (mutating or
reviewed-safe) with a reason, so a future vendor release introducing a new
read-shaped action fails the gate instead of quietly joining the retryable
set. Ambiguity resolves to never-retry. Writes are never blindly retried —
the transaction engine re-fetches and re-diffs instead.

### Evidence ladder

Six levels, recorded per resource in a machine-readable ledger shipped with
the package, validated on load, readable offline via `show coverage
--evidence`:

1 `implemented` → 2 `mock-verified` (the repo's own ceiling) →
3 `live-read-verified` → 4 `live-no-op-write-verified` (read current state,
commit it back unchanged, re-plan must be empty — the cheapest test with real
teeth, and **a non-empty first diff is a normalization bug, not a surprise to
work around**) → 5 `live-change-and-rollback-verified` (real change, verified
live, rolled back, persistence confirmed) → 6 `production-supported` (plus
failure-path evidence: reboot persistence, template-owned refusal observed
live, injected job failure, permission-denied cleanliness).

Level 5 is the floor for calling a write path supported. Entries at level ≥3
are invalid without Orchestrator version, ECOS version, auth mode, date, and
a re-readable source — an observation without a version is not evidence about
a fabric anyone else has. Mock evidence cannot reach the top; the ledger
validator refuses over-claims.

### Acceptance criteria

Encode §B.5–§B.8 as tests. Headline: ownership tri-state fail-closed cases
(unknown section name, unreadable selection, band derivation fallback); BGP
peer-add permitted while template-set field refused; ownership re-check under
commit lock catches a between-compare-and-commit association; UNKNOWN job
shape fails the transaction and quotes the shape; save-changes verified via
the unsaved-changes flag; IRREVERSIBLE refuses confirm; collision pair
refused with structured output from both entry points; secret sentinel sweep
finds nothing unmasked/unencrypted; mutating-GET never retried even when a
caller asks; ledger over-claim refused.

---

## §6 — STAGE INPUT: `/speckit.specify` — Feature 004 (coverage & pipeline), Feature 005 (declarative), Feature 006 (operational views)

---

### Feature 004: resource coverage model and spec pipeline

**Tier model** (how coverage grows safely):

| Tier | What | In transactions? |
|---|---|---|
| 0 raw | `ecx api ...` — any endpoint, day one | never; audit journal only |
| 1 generated | spec-ingested models + a registered plugin stub | never — a stub cannot even be planned until curated |
| 2 curated | real normalization, true reversibility, ownership | full commit-confirm |

Tier 1 is **developer scaffolding, not operator coverage**: the stub's
normalization raises a distinct not-curated error, so planning stops it
before any guard question arises; the wiring below exists so finishing it is
a small edit against working code. The registry-wide gate: every kind below
curated must raise exactly that error from normalize; every curated kind must
prove, against a non-trivial sample fetched through its own fetch path (leaf
values, not container truthiness): normalize runs on real state, normalize is
idempotent (`normalize(normalize(x)) == normalize(x)`), and canonical state
re-planned yields an empty diff. Prove the gate itself bites with constructed
failures (a stub that returns, a non-fixed-point normalize, an
empty-shell canonicalizer, a dirty re-plan).

**Resource contract** (design it with these lessons; details in §8): fetch /
normalize / canonicalize-desired / diff / apply / verify / rollback /
list-refs / ownership hook (receives the diff) / write-target declaration /
dependencies / declarative capability. Contract decisions that were learned
late and should be day-one:

- fetch returns a **tri-state**: present(state) / absent(positive evidence:
  404 or a documented empty shape) / uninterpretable(raise with the shape) —
  never coerce "I don't understand this 200" to absent (Principle II; §C.1).
- normalization gets access to the resolver (name↔ID resolution belongs in
  canonical form); canonicalize-desired receives current state (enables
  plan-time constraint checks without a redundant fetch).
- an explicit per-resource marker for "this list is an ordered configuration,
  do not sort it" (priority lists are order-as-config; sorting is the bug).
- a per-commit options channel (force / override-template / resource-specific
  flags like delete-dependencies) that reaches apply, so resources do not
  smuggle operator intent through desired state.
- unknown server fields **pass through** normalization untouched (a field the
  spec does not carry rides passthrough; dropping one manufactures permanent
  phantom drift).
- full-object-replacement is an explicit declared property, not prose.

**Spec pipeline**: vendor the public OpenAPI baselines (§A.14) as package
data; one read-only spec module over them (operation index, bounded
cycle-safe `$ref` resolution, path normalization); a sync tool that diffs a
live/published spec against the baseline (when fetching with a key, send the
key **only** to the configured Orchestrator's origin); a payload-example
distiller from the vendor's published Postman collections (the specs leave
most write bodies untyped; the collections supply shape for most of them —
shape only: the vendor fills scalars with placeholders, so they prove field
names and nesting, never values); generators that emit typed models and a
registered Tier-1 stub per operation. `show coverage` reports every kind
(scope/reversibility/tier/evidence) and `--endpoints` the whole endpoint
universe by tier, offline, from the shipped baselines. Packaging rule: package
data is an explicit reviewed declaration (no automatic include-by-git); an
installed-wheel test proves the baselines and ledger actually ship — the
observed failure was a wheel whose coverage command confidently reported an
empty endpoint universe.

**Shape survey harness** (a first-class deliverable of this feature; protocol
in Appendix E): a tool that probes a set of read-only endpoints against one or
more live Orchestrators and captures **sanitized shape fixtures** — structure
skeletons, field names, observed enum values and discriminators, never
customer values — stamped with origin digest, Orchestrator version, ECOS
versions, auth mode, and date. It is structurally incapable of writing: GET
only, through the product's own client so the mutating-GET veto (§5, §A.13)
binds at the transport, bounded concurrency, every probe journaled
`AUDIT_ONLY`. Outputs land in a committed fixtures tree keyed by endpoint and
Orchestrator version, and a **shape-diff** mode compares two capture sets —
across Orchestrators, or across time on one — so vendor drift is a report,
not a surprise. These fixtures are load-bearing three ways: the mock's data
model is generated or checked against them; every `normalize()` gets golden
tests per captured version; and each resource-family spec's *source
validation* section cites them. Resources are written **tolerant-reader
style** against the union of captured shapes: branch on observed structural
discriminators, not on version-string guessing, and let unknown fields ride
passthrough.

**Coverage targets** (the UI-parity catalog to grow through, with the API
realities of §A.10–A.11): orchestrator scope — interface-labels,
template-group, template-association, template-group-priority, bio,
bio-association, overlay-priority, security-policy, zones (+ segment↔zone
map), regions, region-association, regional-overlay, internal-subnets,
ip-address-group, ip-service-group, app-express-group,
app-express-association, loopback-orch, snat-maps, schedule-timezone,
appliance-info; appliance scope — deployment (interfaces/IP/VLANs), dhcp,
vrrp, routes, bgp, ospf, loopback, zones, security-maps, acl, nat-maps,
nat-pools, qos-map, optimization-map, route-map, shaper, inbound-shaper,
snmp, logging, mgmt-services, banners. Most "orchestrator UI" surfaces are
GET-only at the Orchestrator; the real write path is the ECOS API via the
appliance proxy (§A.4) — plan scope accordingly.

**Mock Orchestrator**: a bundled fake implementing every endpoint the curated
kinds use, for offline development and CI (`ecx --mock <port>`). Lessons: the
mock is *a model of the API, not the API* — record divergence when live
contradicts it and fix the mock toward live; give at least one
appliance-scope kind genuinely per-appliance-distinct instance names (an
all-singletons mock lets instance-scoping bugs pass); seed state persistently
(a handler that regenerates fresh state on every unwritten read surprises
test authors); implement documented-but-optional endpoints only alongside a
live capture.

### Feature 005: declarative desired-state (GitOps)

A reviewed directory in git as the intent source:
`fabric/<noun>/<instance>.yaml` and
`appliances/<name>/<noun>/<instance>.yaml` — **user-facing nouns in paths,
never internal keys** (an internal key can contain a path separator and would
silently split into two directory levels).

- `drift [--from <dir>]` compares every instance of every kind against
  intent (default: the local candidate). Rows: in-sync / drift / undeclared
  (an instance nobody declared is *undeclared*, never clean — reporting it
  clean is how an unmanaged fabric passes a drift check) / unreadable /
  unsupported (never fetched — no canonical form to compare, and the round
  trip is pure control-plane load). Exit 0/1/8, 8 outranks 1. "Differs from
  declared intent" and "has unsaved running-config changes" are different
  axes — the latter is a note, never folded into drift rows.
- `apply --from <dir> [--dry-run]`: **one** transaction engine and one
  materialization path shared with the candidate flow (an intent-source
  abstraction both implement), so drift can never report something apply
  would not do, and dry-run runs the same non-mutating preflight as apply
  (race-sensitive guards revalidated under the commit lock at apply).
- Governing contract: the directory is **partial/additive authority** — it
  governs only declared references; undeclared live objects are
  `out_of_scope`, never clean, never implied deletions. **Deleting a file
  never deletes its object**; an explicit `state: absent` declaration is the
  only deletion mechanism, valid only for kinds with evidence-backed deletion
  + rollback. No prune in v1. An **empty declaration set is invalid** (it
  looks exactly like a wrong path or failed checkout; guessing "change
  nothing" turns a mistake into a success report).
- Envelope, versioned and explicit: `apiVersion` + `state` required, neither
  inferred; an unknown version fails closed **without rewriting the file**.
  A document may restate kind/name/appliance; where document and path
  disagree, the load fails rather than picking a winner.
- Loading is all-or-nothing and offline: one unreadable file, unknown noun,
  duplicate reference, schema failure, or symlink escaping the tree
  invalidates the whole set, every problem reported in one run, before any
  client is even constructed.
- A non-empty candidate refuses directory apply (never merge two intents into
  one transaction).
- **Materialization capability gates writes per resource, default
  unsupported**: a declaration is *typed partial intent*; a resource earns
  `present`-writability only by proving it can build a complete write target
  from current state + declaration without erasing unknown, unmodeled,
  redacted, or write-only fields — and by holding live write evidence
  (Feature 003's ladder). Blindly POSTing a partial document to a
  full-replacement endpoint replaces fields nobody declared. Blocked kinds
  show as blocked in the plan; the command ships before the write paths are
  proven, refusing honestly.
- Idempotency: re-applying unchanged declarations performs zero writes.

### Feature 006: operational views and fabric reports

Read-only reports are a separate species from resources (no
normalize/diff/apply contract, no reversibility class — modeling them as
resources would imply guarantees they cannot honor). Shared bounded fan-out:
concurrency-capped, failure-isolating; one unreachable appliance is a marked
row, never a lost report; label API-error rows as errors, not "unreachable"
(an appliance can be perfectly reachable while the query is at fault).
Initial set: fabric config breakdown by section (each section its own
designed view, §3 rendering rules); native running-config read
(deny-by-default **exact-match** verb allowlist — `show`/`display` heads
only; the ECOS `debug` namespace is not read-only, §A.13); fabric software
versions with skew detection (orchestrator + per-appliance
active/backup/next-boot partitions; never assume the first listed partition
is current); flow matrix and address search (§A.12 has the verified
parameter semantics); per-appliance BGP operational views
(summary/neighbors from one state call; `routes` honestly `unsupported` —
no RIB endpoint exists, §A.12 — and still listed by the nonterminal, marked
unsupported, pointing at the counts that do exist and at Tier-0). A
configured-but-not-observed BGP peer is shown as exactly that, never
inferred established; the state object's own neighbor count is authoritative
and a row-count mismatch reports `partial`.

---

## §7 — Pre-answered clarification register

When `/speckit.clarify` (or the agent) raises a product question, answer from
this register first. These are settled; do not re-litigate. New questions
follow §1 rule 5.

| # | Question | Decision |
|---|---|---|
| D-1 | Is the datastore token mandatory under `show configuration`? | Optional, defaults to `running`; `candidate` never implicit. Consequence: reserved words (§3). |
| D-2 | `show compare` vs `show \| compare` | Both; the pipe form is an alias kept for operators' fingers. |
| D-3 | Scope ordering | Outermost-first (scope → kind → instance), uniform across shell and scriptable surfaces. |
| D-4 | What is "candidate"? | Client-side staged intent, materialized against server state at compare/commit. Not a device-side tree; the grammar must not imply one. |
| D-5 | Is native a source or a format? | A format of running configuration (`--format native`). |
| D-6 | Kind naming | Per-scope user-facing nouns; internal registry keys never surface anywhere, including flags. |
| D-7 | Fan-out cost & staleness | Prompt when a TTY can answer, warn on stderr and proceed when it cannot; `--stale-ok` strictly opt-in. |
| D-8 | Outcome/exit model | §3's table; comparison commands use 0/1/8 with 8 outranking 1, as a declared exception. |
| D-9 | Multi-Orchestrator selector | Deferred feature; the noun is `orchestrator` (the word for the no-scope subject; `fabric` is taken as a scope noun meaning "every appliance"). Reserve `orchestrator`/`orchestrators` as aliases now. No ambient sticky selection ever: a shell session may carry a visible selection; scriptable writes with no explicit target refuse. Registry = a reviewable file; `show orchestrators` lists; no mutation verbs in v1. No cross-target atomicity, ever — per-target transactions, per-target confirm windows, per-target watchdogs. |
| D-10 | New top-level verbs | Admissible only under §3's V1–V3 test. |
| D-11 | Ownership granularity | Change-aware, three layers (L1 vocabulary / L2 derived priority band / L3 declared subtrees); mixed changesets refused whole; the kind→section map is a corrected hand-maintained list (loud when wrong) rather than inferred (quiet when wrong); never read `gms_marked`/`merge`/`templateApply` for ownership. |
| D-12 | Watchdog backend | Detached double-fork daemon by default; systemd transient *service* as documented opt-in; a timer is structurally wrong (§4). |
| D-13 | Commit-confirm auth | API key required; interactive session auth refuses confirm windows. |
| D-14 | Declarative authority | Partial/additive; explicit-absent-only deletion; empty set invalid; no prune in v1; materialization capability per resource, default unsupported (§6). |
| D-15 | Tier-1 stubs | Cannot be planned or committed at all until curated; best-effort writes, if ever wanted, get their own explicit surface rather than a weakened normalization contract. |
| D-16 | Direct-to-appliance access | Out of scope for v1. Appliances are reached only via the Orchestrator proxy; the ECOS API has no RBAC, so direct mode waits for a separate credential-brokering gateway. |
| D-17 | Service orchestration (3rd-party integrations) | Deferred entirely — an epic-sized surface (~100 endpoints), not a resource. |
| D-18 | Secrets model | §5's four layers; envelope key from env (exclusive) or keyring; rotation crash-safe; no key ⇒ secret writes fail closed. |
| D-19 | Job classification | Allowlist success, tokened failure, everything else UNKNOWN = transaction failure; allowlist entries require version-stamped live evidence. |
| D-20 | Comparison exit codes | 0 none / 1 differences / 8 incomplete; 8 outranks 1. |
| D-21 | Formatter | Adopt an auto-formatter from day 0 (a clean tree has no legacy-diff cost). |
| D-22 | MCP / agent front-end | Not in v1. If added later, it is a thin front end over the transaction engine — a few tools mapping to plan/compare/commit/rollback/show — never a reflective one-tool-per-endpoint surface, and never a second product surface without the same guarantees. |

---

## §8 — STAGE INPUT: `/speckit.plan` (shared technical plan)

---

Constraints and choices for every feature's plan. The constitution check runs
against §2; anything below marked *mechanic* is a correctness requirement,
not a style preference.

### Stack

- **Python ≥ 3.12** (single language; the tool targets a Linux server the
  operator SSHes into; no sudo required to install or run).
- HTTP: `httpx`. Models/validation: `pydantic` v2. CLI: `typer` (scriptable
  surface) + `prompt_toolkit` (interactive shell) + `rich` (rendering).
  Logging: `structlog` **to stderr**. YAML: `PyYAML`. Secrets: `keyring` +
  `cryptography` (AES-256-GCM envelope). Mock server: `fastapi`/`uvicorn`
  behind an optional extra. Dev gate: `ruff` (lint **and** format) +
  `mypy --strict` + `pytest` (+ `respx` for HTTP fixtures; property-based
  tests where canonicalization idempotency is the claim).
- Packaging: `pyproject.toml`; console script `ecx`; explicit package-data
  declaration (spec baselines, payload examples, evidence ledger, `py.typed`)
  with automatic include **off**; a `make check` (lint+types+tests) and
  `make smoke` (build wheel, install into a clean venv, exercise the CLI from
  outside the source tree). CI runs the gate on every push/PR across
  supported Python versions and always runs the installed-wheel job.

### Component map (one module ≈ one responsibility)

| Component | Responsibility | Key mechanics |
|---|---|---|
| settings/config | env + keyring resolution; **canonical origin** derivation shared with the client; self-redacting repr | §4 identity rules |
| client | httpx wrapper; API-key header auth; session-auth alternative; appliance proxy helper (with query-param passthrough — a proxied endpoint's flags are not body fields); retry delegation; error scrubbing at construction | §A.1, §A.4, §A.13 |
| retry policy | per-call classes + spec-derived mutating-GET veto | §5 |
| resolver | name↔ID for appliances (hostname↔nePk), overlays, template groups; per-target cache; **namespaced cache keys** (two callers caching different shapes under one key was a real defect); a "did not resolve" error that suggests near-misses | §C.4 |
| resource contract | the plugin protocol (§6 Feature 004 contract decisions) | tri-state fetch; diff-aware ownership hook; options channel |
| registry | kind registration; per-scope alias map; reserved-word rejection; dependency-ordered planning; the single scoped-instance enumeration every surface uses | §3 |
| candidate store | locked read-modify-write staging; intent-source abstraction (candidate + desired-directory implement it); acknowledgement of foreign staging; sealed secret values | §4 |
| differ | structural diff over canonical states; stable ordering; honors the list-is-order marker | |
| transaction engine | plan (deps → ownership → collisions → reversibility) → snapshot → apply → verify → confirm-window lifecycle; drift-fail-closed with `--rebase`; partial-failure auto-revert; entry-point parity (candidate commit and declarative apply reach every guard through one plan builder) | §4, §5 |
| journal | per-txn dir; atomic meta; fsync'd event log; rollback-private snapshot store with digest cross-check; retention; orphan scan; corrupt-history named | §4 |
| watchdog | detached daemon; arm-verified; pid-liveness contract; same journal events in any backend | §4 |
| locking | flock + fallback; one commit lock; holder metadata | §4 |
| jobs | action-key polling; allowlist classifier; save-changes verifier; keyless correlation; the separate numeric preconfig channel | §A.5 |
| ownership | L1/L2/L3 change-aware model; per-plan cache (the check costs 2–3 API calls; a multi-resource changeset on one appliance re-asks) | §5, §A.7 |
| secrets (redaction + vault) | detector; masking; envelope encryption; rotation | §5 |
| audit export | journal → human/JSON/NDJSON; redaction-by-default | §4 |
| spec module + tools | vendored baselines; sync/diff; payload examples; model/stub generators | §6 |
| evidence ledger | validated-on-load records; coverage integration | §5 |
| reports | fan-out engine + operational views | §6 |
| mock | bundled fake Orchestrator | §6 |
| cli (main + shell + render) | typer app; prompt_toolkit shell; shared renderers; outcome classification at both dispatch boundaries | §3 |

### Cross-cutting mechanics (non-negotiable in review)

1. **Durability ordering**: temp-file + rename + directory fsync for any
   atomically-replaced index; fsync per appended event record; tolerate a
   torn final line on read; everything else about a corrupt journal is loud.
2. **Every network call has a timeout**; fan-out is concurrency-capped;
   the Orchestrator is a low-QPS control plane — never poll stats through it,
   cache resolver data per target with explicit refresh.
3. **Test the wiring, not just the guard.** For every safety guard, at least
   one test proves the *call path* invokes it (deleting the guard must fail a
   test that exercises the public entry point, not only a unit test of the
   guard). Recurring field defect: a guard that exists but is not on the
   path. Periodically verify by mutation: remove a guard, confirm a test
   fails, restore.
4. **Two entry points, one engine**: any check reachable from candidate
   commit must be reachable identically from declarative apply — enforced by
   a shared plan-builder and a parity test.
5. **Logging is diagnostics**: stdout is program output only; structlog to
   stderr; interactive shell defaults quiet.
6. **No reflective API exposure**: nothing enumerates an SDK or spec and
   exposes operations wholesale as user surface (that pattern produced a
   641-tool agent server with credentials in arguments in the field — §C.7).
7. **`repr` hygiene**: any object that can hold a credential redacts it.

### Milestone-mapping note for plans

Each feature's plan must state which §9 milestone(s) it lands in, and its
constitution-check table. Anything the plan cannot verify without live gear
is listed under "not obtainable without gear" and labeled per Principle V.

---

## §9 — Delivery roadmap (`/speckit.tasks` per milestone)

Build vertically: each milestone ships something an operator can run, gated
by the previous one's exit criteria. Do not start a milestone early.

**M1 — Grammar and skeleton (Feature 001).**
Parser + registry + outcome model + renderer discipline + `show commands`
offline reference + shell/scriptable parity harness. Mock server skeleton.
Exit: §3 acceptance 1–10 green; `make smoke` proves the wheel.

**M2 — Transactional core with one trivial resource (Feature 002).**
Candidate, diff, commit, commit-confirm + watchdog, rollback history, locks,
journal + audit export, origin identity, Tier-0 `api` passthrough, orphan
recovery. One curated resource proves the loop end-to-end: **interface-labels**
(orchestrator-scope singleton, full-replace, no async job — the simplest
write on the API). Exit: §B.1–§B.4 green including the slow detached-watchdog
e2e; interface-labels reaches mock-verified for
set→compare→commit-confirm→auto-revert→commit.

**M3 — Safety systems (Feature 003).**
Ownership (L1/L2/L3), jobs classifier + save-changes verifier, reversibility
enforcement, write-target partition, secrets vault + redaction + rotate-key,
retry veto, evidence ledger + `show coverage`. Exit: §B.5–§B.8 green;
sentinel sweep clean; coverage/evidence read from the wheel.

**M4 — Shape survey: harness + first multi-Orchestrator campaign
(Feature 004; Appendix E).**
Build the survey harness on M2's client (Tier-0 GET + retry veto + redaction
+ audit journaling), then fan out one worker per provisioned Orchestrator and
capture the read-only endpoint set for every planned resource family, plus
Appendix E's standing shape questions. Sanitize, commit fixtures, run the
first cross-Orchestrator shape diff, and feed the mock from the captures.
Exit: fixtures on ≥2 real Orchestrator releases for every M5/M6 endpoint the
operator's fabrics can answer; every standing question answered or explicitly
recorded as still-open; zero non-GET and zero deny-listed calls in the
campaign journal. Repeat this milestone's campaign whenever a new
Orchestrator release appears — it is a loop, not a phase.

**M5 — Phase-1 verticals (orchestrator scope).**
template-group, template-association (the push trigger — async, keyless,
action-log correlated), bio, bio-association, security-policy, zones +
segment↔zone map, overlay/template priorities. Exit: mock-verified all; the
enumeration→fetch round-trip property holds for every kind **including
against the M4 live-shape fixtures** (§C.1 is the cautionary case).

**M6 — Appliance scope via the proxy.**
deployment (validate-then-apply; reboot-required surfaced), dhcp (a subtree
of deployment — the collision pair with it), vrrp, routes (delta-based,
COMPENSABLE), bgp + ospf (full-object POSTs; L3 ownership subtrees), loopback
(read + diff; write only if §A.10's open question resolves), appliance zones
+ security-maps, acl/nat/qos/optimization/route-map/shapers, snmp/logging/
mgmt-services/banners/timezone, appliance-info. Save-changes after every
proxied write, batched per transaction, verified. Exit: mock-verified all;
normalize golden tests green on the M4 fixtures; ownership verdicts correct
on the recorded live vocabulary fixtures.

**M7 — Fleet visibility (Features 005 read-half + 006).**
drift (candidate + `--from`), operational views (bgp state, versions, flows,
fabric config sections, native read), fan-out engine. Exit: drift's
undeclared/unreadable/unsupported rows and 8-outranks-1 proven; §6 rendering
rules hold on every report.

**M8 — Declarative apply (Feature 005 write-half).**
Envelope + loader + preflight parity + capability gate + single-transaction
apply. Exit: dry-run/apply parity test; capability gate blocks unproven
kinds; the whole set of governing decisions (D-14) demonstrably enforced.

**M9 — Live evidence campaign (operator-supervised; repeatable).**
Read sweep across every curated kind (expect normalization surprises — the
mock is a model); no-op round trips; then real change-and-rollback on the
safest kinds first (banners-class before bgp-class). Record every result in
the ledger with versions. Exit: an honest coverage report; divergence between
mock and live fed back as fixtures and mock fixes.

Post-v1 backlog (do not build early): multi-Orchestrator targeting (D-9),
fleet lifecycle (discovery/upgrade/backup/preconfig — IRREVERSIBLE class,
§A.11 upgrade safety rules), an agent/MCP front end over the engine (D-22),
response caching with TTL, direct-to-appliance broker (D-16).

---

## Appendix A — EdgeConnect Orchestrator / ECOS API field guide

### A.0 Evidence tags (used throughout; carry them into code comments and docs)

| Tag | Means |
|---|---|
| `[SPEC]` | stated by the vendor's OpenAPI 3.0 documents (Orchestrator REST 7.2.0, ~870 paths; ECOS/appliance REST 7.2.0, ~437 paths) |
| `[SDK]` | stated by the open-source `pyedgeconnect` SDK's code/docstrings |
| `[LIVE-9.7]` | observed against a live Orchestrator 9.7.0.43282 / ECOS 9.7.0.0_109184 lab fabric |
| `[FIELD]` | practitioner-reported, version unrecorded — treat as a lead, not evidence |
| `[INFERRED]` | plausible reading, unverified — never build a guarantee on it |

Path style throughout is the Orchestrator ≥ 9.3 **query-parameter** form
(pre-9.3 used path segments; do not support pre-9.3 in v1).

### A.1 Auth and transport

- Base URL `https://{host}/gms/rest`; append it when the operator's URL lacks
  it, and derive target identity from the same normalized base. `[SDK]`
- **API-key auth**: header `X-Auth-Token: <key>` on every request,
  sessionless. Works on cloud-hosted Orchestrators (`*.silverpeak.cloud`,
  `*.silverpeaksystems.net`) and on-prem 9.x. `[SDK]` `[LIVE-9.7]` Never put
  the key in a query string.
- **Session auth**: `POST /authentication/login` (user/password), cookie
  based; the CSRF token must be echoed back as `X-XSRF-TOKEN` on subsequent
  requests; logout is a **GET**; a `loginType` field participates. `[SDK]`
  `[INFERRED]` — exercise against a live login before trusting; API-key mode
  is the primary path and the only one commit-confirm accepts.
- Expected statuses: GET 200; POST/PUT 200/201/204; DELETE 200/204. `[SDK]`
- The Orchestrator is a **low-QPS control plane**: no stats polling through
  it; listing endpoints paginate by time windows (`startTime`/`endTime`
  in epoch **milliseconds**) plus `limit`. `[SDK]` `[FIELD]`
- A `cached=true|false` query flag exists on ~88 read endpoints: `true`
  returns the Orchestrator's last-known copy, `false` forces a live pull from
  the appliance — this flag *is* the freshness switch `--stale-ok` maps to.
  `[SDK]` Some endpoints **require** `cached` and 422 without it (observed on
  the orchestrator-scope deployment read and software-versions read).
  `[SPEC]` `[LIVE-9.7]`

### A.2 Identity primitives

- `nePk` — appliance primary key, e.g. `3.NE`; regex `^\d{1,10}\.\w{1,10}$`.
  Everything API-side speaks nePk; everything user-facing speaks hostname —
  the resolver owns the mapping. `[SDK]` `[LIVE-9.7]`
- `GET /appliance` — full inventory, `list[dict]`, ~33 keys per appliance.
  Load-bearing: `id`/`nePk` (same value), `hostName`, `site`, `networkRole`
  (0 spoke / 1 hub / 2 mesh), `model`, `softwareVersion`,
  `hasUnsavedChanges` (bool — the save-changes verifier polls this),
  `rebootRequired`, `state` (0 unknown / 1 normal / 2 unreachable /
  3 unsupported-version / 4 out-of-sync / 5 sync-in-progress),
  `zoneList.zones`, `interfaceList.interfaceLabels`. `[SDK]` `[LIVE-9.7]`
  Appliances in state 2 (maintenance/unreachable) fail proxied reads with
  400s — enumerate them and mark rows unreadable; do not filter them out
  (that turns "could not check" into "clean"). `[LIVE-9.7]`

### A.3 Interface labels (the trivial first resource)

`GET/POST /gms/interfaceLabels` — orchestrator-scope singleton, full-replace
write, no async job. `[SDK]` `[FIELD: reliable]` The ideal end-to-end proving
ground for the transaction loop.

### A.4 The appliance proxy and save-changes

- **Almost no appliance-config write endpoints exist at the Orchestrator.**
  Appliance config writes go to the appliance's own REST API through the
  proxy: `GET/POST/DELETE /appliance/rest?nePk={nePk}&url={ecosPath}`, body
  passed verbatim. The ECOS base is `/rest/json`; the `url=` parameter takes
  the path after it. `[SDK]` `[LIVE-9.7]`
- Proxied writes mutate **running config only**. Persist with save-changes or
  the change is lost on reboot (`hasUnsavedChanges` flips true). The platform
  does not do this for you; it is the CLI's responsibility, batched once per
  transaction over every appliance written, and a non-success save fails the
  transaction. `[SDK]` `[LIVE-9.7]`
- Save-changes: `POST /appliance/saveChanges` body `{"nePks": [..]}` (≥9.3),
  or single `POST /appliance/saveChanges?nePk=X` body `{}` → `{"clientKey"}`
  polled via the action log. `[SDK]` A keyless variant exists off-spec:
  verify by polling `hasUnsavedChanges` until clear on every written
  appliance; unreadable flag ⇒ UNKNOWN; still set at deadline ⇒ FAILED.
  `[LIVE-9.7]`
- The proxy helper needs **query-param passthrough** (some ECOS endpoints
  take required query flags, e.g. zones' `deleteDependencies`), and it
  validates `nePk` before building the URL.

### A.5 Async jobs

- Canonical pattern: write → `clientKey`/guid → poll
  `GET /action/status?key=` → **the response is an array** (group pushes fan
  out one record per appliance under one `guid`) → poll until terminal.
  `[SDK]` `[FIELD]`
- Record fields: `id`, `user`, `ipAddress`, `nepk` (**lowercase**), `name`,
  `description`, `taskStatus` (string), `startTime`/`endTime`/`queuedTime`
  (epoch **ms**; endTime 0 while running), `percentComplete`,
  `completionStatus` (bool), `result` (string), `guid`. `[SDK]`
- **`completionStatus` is unreliable** — observed false on success for ECOS
  upgrades, with logLevel stuck at ERROR. Never a tiebreaker. `[FIELD]`
- Terminal detection is tolerant (endTime set, percentComplete done, or a
  done-shaped status) — being finished is a weaker claim than having worked.
  Classification of a terminal record (Feature 003 rules): failure tokens
  (`Failed`, `Cancelled`, `Error`, `Aborted`, `Rejected`) in status or
  result ⇒ FAILED; status not matching a success token
  (`Completed`/`Done`/`Finished`, case-insensitive substrings) ⇒ UNKNOWN;
  empty result on success status ⇒ SUCCESS; result matching the allowlist ⇒
  SUCCESS; else UNKNOWN ⇒ transaction fails, shape quoted.
- Success-shape allowlist earned so far: result prefix `Success` `[FIELD]`;
  the exact string `Saved change on appliance successfully` `[LIVE-9.7]` —
  the *whole* string, deliberately: the success word is at the end, where a
  prefix match would also admit a future "…partially". This shape cost a
  real transaction to learn: the write landed, the poller could not confirm
  it, the engine auto-reverted, and the revert hit the same unknown shape —
  fail-closed working as designed, and why every entry is version-stamped.
- Keyless template-association pushes: per-appliance results exist only as
  action-log records under a guid; correlate via
  `GET /action?startTime=&endTime=&appliance={nePk}` (ms epochs — the SDK
  docstring wrongly says seconds) by appliance + window; more than one
  matching guid ⇒ UNKNOWN naming them. `[SDK]` `[LIVE-9.7]`
- Preconfig apply is a **different protocol**:
  `GET /gms/appliance/preconfiguration/apply?preconfigId=` returns numeric
  `taskStatus` 0/1/2, and `completionStatus` is meaningful only at 2. `[SDK]`
- `POST /action/cancel?key=` exists `[SPEC]`; `GET /action/inProgress`
  returns 400 in practice — do not use `[FIELD]`. `POST /broadcastCli`
  returns its guid as **plain text**, not JSON `[FIELD]`.

### A.6 Templates and template groups

| Operation | Endpoint | Notes |
|---|---|---|
| List groups (+ selection flags) | `GET /template/templateGroups` | also the **section-name vocabulary**: every template name the Orchestrator knows (46 on 9.7) `[LIVE-9.7]` |
| Get one group | `GET /template/templateGroups?templateGroup=NAME` | `{name, templates: [{name, valObject}]}` — template names are config *section* names; `valObject` is the section payload `[SDK]` |
| Update group content | `POST /template/templateGroups?templateGroup=NAME` | |
| Create / delete group | `POST /template/templateCreate` / `DELETE /template/templateGroups?templateGroup=` | create returns 204 without templates, 200 with `[SDK]` |
| Selected sections of a group | `GET/POST /template/templateSelection?templateGroup=` | list of section names; POST replaces `[SDK]` `[LIVE-9.7]` |
| Associations (all / one) | `GET /template/applianceAssociation[?nePk=]` | `{nePk: [group,...]}` / `{"templateIds": [...]}` `[SDK]` |
| Associate | `POST /template/applianceAssociation?nePk=X` `{"templateIds": [...]}` | **complete replacement** — to add, include existing groups. 204. **This is the push trigger** (async → action log). `[SDK]` |
| Applied history | `GET /template/history?nePk=&latestOnly=` (204 when none), `GET /template/history/groupList?nePk=` | `[SDK]` |
| Priorities | `GET/POST /template/templateGroupsPriorities` | note the exact spelling — a nearby singular variant appears in prose but the plural is the path `[SDK]` |

### A.7 Template-ownership signals (the verified model)

- **No `templateApplied`/`appliedTemplates` field exists anywhere.** Ownership
  is derived: associated groups (A.6) × each group's selected sections ×
  a kind→section mapping. `[SDK]` `[LIVE-9.7]`
- **The vocabulary is knowable — read it, never guess it.** Live checking
  found five kinds mapped to section names that do not exist (fail-open: the
  guard could never see an owner). Real names observed `[LIVE-9.7]`: `bgp`,
  `ospf`, `vrrp`, `routes`, `banners`, `dns`, `snmp`, `mgmtServices`,
  `logging`, `shaper`, `acls`, `securityMaps`, `natMaps`, `qosMaps`,
  `routeMaps`, `optmap` (optimization maps — **not** `optimizationMaps`),
  `routesRedistributeMaps`, `adminDistance`, `cli`, `datetime`,
  `secureWebServicesConfig`, `webconfig`, `inboundShapers`→ real sibling is
  `shaper`. No section exists for deployment/interfaces, DHCP, NAT pools, or
  appliance zones on that build.
- **Authority is per *field* for protocol sections** `[LIVE-9.7]`: the `bgp`
  template body carries keepalive/hold/next-hop-self/route-target defaults
  and **no peer key at any depth** — peer existence is locally significant.
  Same shape for `ospf` (timers/auth, not interfaces/areas), `vrrp`
  (advTimer/priority/preempt, not vrid/VIP), `routes` (auto-subnet/ecmp
  flags, not subnet entries). By contrast `banners`/`dns`/`snmp` template
  bodies *are* the config. Hence ownership layer L3 (declared governed
  subtrees) for the first four.
- **Authority is per *entry* by priority band for map sections**
  `[LIVE-9.7]`: Orchestrator-pushed entries occupied 1000–9999 with no
  exception (natMaps 1000; qosMaps 1000–1040 with local 20000/20001/65535
  alongside; optmap 1000–1510 with local 10020/10070/65535;
  routesRedistributeMaps 1000–1100); optmap's pushed entries matched the
  template byte-for-byte. Derive the governed band from the selected
  template's own entries (D-11) rather than hard-coding the range.
- **Do not use these as provenance**: `gms_marked` is `False` on exactly the
  entries the template pushed and `True` on local ones `[LIVE-9.7]`;
  `options.merge`/`templateApply` values contradict observed behavior and
  their semantics are undocumented `[LIVE-9.7]`.
- ECOS `routeMaps` is route *policy* (traffic steering); route
  *redistribution* lives at ECOS `redistributionMaps` (template section
  `routesRedistributeMaps`) and also holds the per-peer BGP route-maps that
  BGP neighbors reference by name — a BGP resource's config points at objects
  outside it. `[LIVE-9.7]`
- Cache ownership lookups per plan (association + selection + bodies ≈ 2–3
  calls per item otherwise), and namespace the cache keys.

### A.8 Business Intent Overlays

| Operation | Endpoint | Notes |
|---|---|---|
| List / create | `GET/POST /gms/overlays/config` | list of overlay objects; id server-assigned int, name user-facing `[SDK]` |
| Get one / modify / delete | `GET/PUT/DELETE /gms/overlays/config?overlayId=N` | **modify auto-pushes to the fabric** (async, no key surfaces — verify orchestrator-side config); delete removes appliances from the overlay `[SDK]` |
| ⚠ shape trap | `GET ...?overlayId=N` observed answering 200 with a shape a strict single-object reader mis-took for "absent" on one live fabric | treat any unexpected 200 body as `error`, never absent (§C.1) `[LIVE]` |
| Regional | `GET/POST/PUT /gms/overlays/config/regions...` (`regionId` 0 = global) | PUT merge-vs-replace **unverified** — do a defensive read-modify-write `[INFERRED]`; no endpoint removes a single overlay×region entry, so that entry class is non-deletable — say so `[SPEC]` |
| Priorities | `GET/POST /gms/overlays/priority` | priority→overlayId map, **full overwrite**, unique priorities `[SDK]` |
| Association | `GET /gms/overlays/association` (`{overlayId: [nePk,...]}`); `POST` same shape **adds** (union); `POST /gms/overlays/association/remove` removes; `DELETE ?overlayId=&nePk=` single (204) | note the asymmetry with template association (which replaces) `[SDK]` |

Decommission cascade `[FIELD]`: deleting an overlay association triggers the
Orchestrator to delete tunnels/IPSLA/QoS and then **re-apply templates**
(30–90 s). Never delete tunnels individually to "clean up".

### A.9 Security policy and zones

- Orchestrator-side read: `GET /securityMaps?nePk=&cached=` (per-appliance
  policy). **No orchestrator-side write for the per-appliance object** — the
  UI's orchestrated write is
  `POST /vrf/config/securityPolicies?map={srcSeg}_{dstSeg}` with
  `{data, options: {merge, templateApply:false}, settings}` `[SDK]` `[FIELD]`;
  the per-appliance path is the proxy with `url=securityMaps` `[SDK]`.
- `GET /vrf/config/securityPolicies` **requires** the `map` parameter — there
  is no "list all policies" form of it; `GET /vrf/config/securityPoliciesSegments`
  returns the configured `src_dst` pair names `[SPEC]` (adopt with a live
  capture; not universally present).
- Rule structure: map → zone-pair key `"<fromZoneId>_<toZoneId>"` → priority
  → `{match, set/action, misc, comment}`; priority 65535 is the implicit
  terminal deny; match fields include src/dst ip/port, protocol, application,
  app group, dscp, dns wildcards, geo, service, vrf, overlay, internet, acl.
  `[SDK]` `[SPEC]`
- Zones: orchestrator zone definitions + `nextId` allocator + segment↔zone
  map (`vrfZonesMap`); ECOS-side `POST /zones` **requires** the
  `deleteDependencies` query parameter. `[SPEC]` `[SDK]`
- In the UI, security policy normally arrives via the `securityMaps` template
  section — a direct proxy write on a template-managed appliance is exactly
  the footgun ownership detection exists to catch.

### A.10 Appliance-scope configuration endpoints

| Area | Read | Write | Notes |
|---|---|---|---|
| Deployment (interfaces/IP/VLAN/DHCP) | `GET /deployment?nePk=` (live call through to the appliance) `[SDK]` | proxy POST `url=deployment` — **full-object replace**: GET, modify, POST whole `[SDK]` | `POST url=deployment/validate` first → `{err, rebootRequired}` `[SDK]`. Object: `scalars` (~60 read-only platform limits), `sysConfig` (mode, maxBW, ifLabels, license, zones, vrfs), `mgmtIfData`, `modeIfs[].applianceIPs[]` (ip/mask/vlan/label/lan-wan/dhcp/harden/NAT/maxBW/dhcpd/zone/vrf), `dpRoutes`, `vifs`, `dhcpFailover` `[SDK]` |
| DHCP | — | — | **No `/dhcp*` endpoint exists.** DHCP server/relay is the `dhcpd` subtree of deployment interfaces + top-level `dhcpFailover`; changing DHCP = POST the whole deployment ⇒ the canonical shared-write-target collision `[SDK]` |
| VRRP | `GET /vrrp?nePk=&cached=` (config+state merged per entry) | proxy POST `url=vrrp` (entry list) `[SDK]` | |
| Static routes | `GET /subnets?nePk=&cached=` (large; per-entry state incl. advertise flags, learned, admin distance) | proxy `url=subnets3/configured` (destructive full replace) and `.../addMultiple`, `.../deleteMultiple` (delta — preferred; bodies inferred from SDK convention `[INFERRED]`) | nested `{prefix: {cidr: {...}}}`; `zone_id` 65534 = none; per-entry `gms_marked` `[SDK]` |
| BGP | `GET /bgp/config/system?nePk=`, `GET /bgp/config/neighbor?nePk=` (+ `allVrfs` forms) `[SPEC]` | proxy POST `url=bgp/config/system` and `url=bgp/config/neighbor` — **two distinct objects**; do not double-carry neighbors in both `[SPEC]` `[LIVE-9.7: change-and-rollback verified]` | |
| OSPF | `GET /ospf/config/{system,interfaces}?nePk=` `[SPEC]` | proxy POST, same pattern | spec-confirmed, not live-write-verified |
| Loopback | `GET /virtualif/loopback?nePk=&cached=` | ECOS spec lists POST `/virtualif/loopback` (+ per-name POST/DELETE) `[SPEC]` — treat write as **open question**, verify live before enabling | orchestration pool: `GET/POST /loopbackOrch` (full overwrite — GET first), `/loopbackOrch/pool`; a `mgmtIp` vs `mgmtIP` casing inconsistency exists between write and read `[SDK]`; reclaim takes `id` as a query parameter and "reclaim all" has no confirmed route `[SPEC]` |
| ACL / NAT / QoS / optimization / route-map / shapers | orchestrator GETs exist but are **read-only**; e.g. `/acls`, `/dnatMaps` are GET-only | all writes are appliance-proxy | **real ECOS paths matter**: NAT pools live at `nat/natPools` (bare `natPools` 404s), SNAT maps at `vrf/config/snatMaps` `[LIVE-9.7]`; `natMaps` has a merge option, `natPools` does not `[SPEC]`; inter-segment D-NAT has no write anywhere — read-only view `[SPEC]` |
| Common settings | proxy reads/writes for snmp, logging, mgmt-services, banners, timezone, dns (`resolver`) | | ⚠ `logging/remote`'s `self` key is a **nested settings object**, not an id echo — stripping it as metadata destroys the receiver config `[SPEC]` |
| Subnet-sharing options | none | `POST /subnets/setSubnetSharingOptions?nePk=` | **write-only, no read path** ⇒ cannot be a transactional resource (no fetch ⇒ no snapshot ⇒ no rollback); model as explicit fire-and-forget or not at all `[SPEC]` |
| CLI fallback | — | proxy `url=cli` `{"command"}`, `url=cliMultiple` `{"commands"}`; orch-wide `POST /broadcastCli` `{"nePks", "cmdList"}` | text response; **no per-appliance output retrieval from broadcast** — per-appliance reads go through the proxy `cli` path `[SDK]` `[SPEC]` |
| Appliance info | orchestrator-scope, one object per appliance (location/contact/overlay settings) | orchestrator-scope write | `[LIVE-9.7: change-and-rollback verified]` |

### A.11 Fleet lifecycle (post-v1, recorded now)

Preconfig: `/gms/appliance/preconfiguration` CRUD; `configData` is
base64-encoded YAML; `.../validate` returns per-line YAML errors; apply is
async on the numeric channel (A.5); `autoApply` provisions on discovery;
matching by serial then tag. `[SDK]` Upgrade safety `[FIELD]`: validate
first (`POST /validateApplianceUpgrade`), refuse `upgradable:false`,
hub-first ordering, ≤5 appliances per batch. Hostname update:
`POST /hostname?nePk=` → 204, **no action key** — verify by re-reading
inventory. `[SDK]`

### A.12 Operational-state endpoints and schema traps

- BGP state: `GET /bgp/state?nePk=` (orchestrator form; `cached`, `vrfId`
  params) — one response carries both the `summary` block and
  `neighbor.neighborState`. `[SPEC]` Traps: `neighborState` is an **object
  keyed "0","1",…**, not an array, and the schema documenting two keys is an
  artifact, not a limit — iterate every numeric key; `bgp_state` is an
  integer enum where 0 = "Not Enabled" is a *successful* answer, not an
  error; `bgp_state_str` is typed integer while described as string —
  unverified, render defensively; `neighborCount` is authoritative for how
  many peers exist. **No BGP route-table (RIB) endpoint exists in either
  spec** — only counts (`num_bgp_rtes_rcvd` etc., `rcvd_pfxs`/`sent_pfxs`
  per peer); report `routes` as `unsupported`. **Never call**
  `GET /bgp/vrfs/{vrfId}/state`: its own spec summary says it *deletes*
  BGP state. `[SPEC]`
- Flows: `GET /flow` with address filters; **`ipEitherFlag=true` is
  directional** (ip1 = source only) despite the name — send `false` to match
  either end `[LIVE: verified after the name-based reading silently missed
  every inbound-heavy host]`. Two appliances on one fabric answered `/flow`
  with fast deterministic 500s while siblings answered 200 — treat per-target
  API errors as `error` rows, not "unreachable". `[LIVE]`
- Versions: orchestrator + per-appliance active/backup/next-boot partitions;
  never assume the first listed partition is the running one — pick by the
  flag that says so, and build the fixture so a wrong implementation fails.
  `[SPEC]` `[FIELD]`

### A.13 Read-shaped verbs that mutate (retry/allowlist input)

`[SPEC]` — the baselines' own summaries: GETs that clear idle time, generate
a sys-dump on the appliance, log out the session, create from a blueprint,
close a gRPC connection (`/oro/debug/closeGrpcConnection`), and delete
segment BGP state. Consequences: (1) the retry veto list (§5) is derived
from these summaries and gated by a test; (2) the native-config read
allowlist admits exact `show`/`display` command heads only — the ECOS
`debug` namespace is not read-only (`DELETE /debug/generic/{}` deletes
module data; the `debug` CLI verb arms logging on a busy box) and is denied
wholesale until someone enumerates safe subcommands live, added as exact
two-token heads, never as a bare verb.

### A.14 Public artifacts to obtain fresh (do not copy from any prior repo)

1. **OpenAPI documents**: every Orchestrator serves its own Swagger/OpenAPI
   JSON from its built-in API docs UI (log into your Orchestrator → the REST
   API documentation page). Capture the Orchestrator document and the
   appliance (ECOS) document; vendor them as package data and record the
   version captured. The 7.2.0 pair (≈870 + ≈437 paths) is a known-good
   baseline shape.
2. **pyedgeconnect** — HPE Aruba's open-source SDK
   (`github.com/aruba/pyedgeconnect`, MIT): use as an endpoint reference.
   Known defects — do **not** replicate: latitude/longitude transposed in
   discovered-appliance payloads; a cancel-task function that GETs status
   instead of cancelling; assorted docstring paths that are SDK module names
   rather than REST paths (`/route_policy`, `/nat_policy`,
   `/optimization_policy`, `/acls/{name}`, `/third_party_services` do not
   exist as REST paths); boolean query params interpolated as Python
   `True`/`False`; a preconfig branch assigning a tuple to a path. Trust
   the *code's* path strings over docstring tables when they disagree.
3. **Vendor Postman collections** for Orchestrator 9.3–9.6 (published by HPE
   Aruba Networking on Postman): payload-shape source for the write bodies
   the OpenAPI documents leave untyped (shape only — scalars are
   placeholders).
4. HPE Aruba Networking's Orchestrator REST API documentation portal for
   anything newer than the captured baseline.

---

## Appendix B — Behavioral contracts (encode each as a test)

### B.1 Watchdog and confirm window

1. Arm a commit-confirm, SIGKILL the parent shell/SSH session: the watchdog
   survives, and at deadline reverts and journals the revert. (Real detached
   process; mark slow.)
2. Watchdog fails to start (pid file never appears): the commit auto-reverts
   immediately; the operator is told the confirm window never armed.
3. Confirm and deadline race: exactly one of {confirmed, reverted} wins;
   never both, never neither.
4. Host "reboots" mid-window (kill watchdog, clear pid): next CLI invocation
   reports the orphaned transaction and `rollback --pending` restores it.
5. Session-auth (no API key): `commit confirm` refuses up front.

### B.2 Candidate, locking, identity

1. Two concurrent stagers: both edits survive (lost-update test — not merely
   torn-file).
2. Shell B cannot silently commit or clear shell A's staging without
   acknowledgement.
3. `http://orch` and `https://orch` produce two disjoint candidate stores,
   journals, and locks; `https://orch` and `https://orch/gms/rest` produce
   one.
4. A snapshot journaled against origin A refuses to restore against origin B.
5. Source-scan test: nothing keys persisted state by the display host.

### B.3 Commit engine

1. Server state moved between compare and commit ⇒ refusal naming the drift;
   `--rebase` rebuilds and redisplays the plan.
2. Mid-changeset failure ⇒ applied steps auto-revert from snapshots; report
   names fabric state exactly; a failing revert ⇒ `REVERT_FAILED`, loud,
   naming what remains.
3. Post-apply verification compares freshly fetched state, not staged state.
4. Rollback verifies snapshot digest; a tampered/lost body refuses restore.
5. Rollback of a rollback works (rollbacks are journaled transactions).
6. Empty diff ⇒ zero API writes (idempotent re-commit).

### B.4 Journal and audit

1. Torn final event line ⇒ journal opens, prior events intact; any deeper
   corruption ⇒ named error on every surface (stderr for the NDJSON stream).
2. Export with unknown transaction id ⇒ error, not an empty export.
3. Snapshot bodies absent from default export; digest+size present;
   `--include-snapshots` masks secret-named fields.
4. Tier-0 call ⇒ `AUDIT_ONLY` record with masked parameters.
5. Meta/events disagreement resolves to events.

### B.5 Ownership

1. Section name absent from the fabric's vocabulary ⇒ `unknown` ⇒ refusal
   without override.
2. Unreadable association / selection / template body ⇒ `unknown`.
3. BGP: peer add permitted on a template-managed appliance; a change to a
   template-carried system field refused; one changeset containing both ⇒
   refused whole.
4. Map entry outside the derived priority band permitted; inside refused;
   band underivable ⇒ `unknown`.
5. Template group associated between compare and commit ⇒ the pre-write
   re-check under the commit lock refuses.
6. Source test: no ownership decision reads `gms_marked` / `merge` /
   `templateApply`.

### B.6 Jobs and persistence

1. Terminal record with unrecognized shape ⇒ UNKNOWN ⇒ transaction fails,
   shape quoted verbatim in the failure detail.
2. "Completed + Invalid configuration"-style record ⇒ FAILED (failure token
   wins over completed status).
3. Poll timeout ⇒ failure, then quiesce/verify before revert.
4. Save-changes verified per appliance; still-unsaved at deadline ⇒ FAILED;
   unreadable flag ⇒ UNKNOWN.
5. Keyless push with two candidate guids in the window ⇒ UNKNOWN naming both.

### B.7 Secrets

1. Sentinel secret staged, committed, snapshotted, exported, and rendered:
   appears nowhere unmasked; on disk only sealed.
2. No envelope key available ⇒ staging a detected secret fails before disk;
   snapshotting one fails before the first fabric write.
3. Sealed blob + wrong key ⇒ loud error; a revert never treats it as "no
   value"; blob unmodified.
4. Interrupted rotation re-runs to completion (both key slots consulted
   mid-flight).
5. API error text with a secret query param ⇒ masked at construction.

### B.8 Guards on the path

1. Registry-wide: every kind's enumerated references fetch successfully
   (mock + recorded live-shaped fixtures).
2. Fetch receiving an unexpected-shape 200 ⇒ `error` naming the shape; never
   `not_found`/absent. (§C.1)
3. Collision pair (deployment + dhcp) refused from *both* entry points
   (candidate commit, declarative apply) with structured output.
4. IRREVERSIBLE kind refuses `commit confirm`; proceeds only with `--force`.
5. Mutating GET never retried even when the caller requests retry.
6. Fan-out cost declaration observed on every fanout command, TTY and piped.
7. Drift: unreachable appliances ⇒ unreadable rows + exit 8 even when no
   drift found ("no drift" is not a claim a partial run can make).
8. The shape-survey harness refuses any non-GET method and any deny-listed
   GET **at the transport**, even when the probe list asks for one; the
   refusal is journaled.
9. Survey sanitizer: a capture seeded with sentinel secrets, real hostnames,
   and address values commits only shape skeletons — sentinels appear
   nowhere in the fixtures tree.

---

## Appendix C — Field lessons (defects already paid for; encode, don't repeat)

**C.1 Unknown shape became "not present".** A live fetch-by-id returned 200
with a body shape the reader didn't expect; the code coerced any
non-matching body to "absent" and the CLI confidently printed `(not
present)` for four overlays its own completion had just listed. The lesson
is structural: `None`/absent is a *positive* claim requiring positive
evidence (404, documented empty shape); everything else unrecognized raises.
This is Principle II's inverse direction and B.8.2's test.

**C.2 Success by omission — four independent instances.** (a) commit noticed
server drift, recomputed, and carried on — folding another operator's change
into the changeset; (b) an installed build reported "0 of 0 endpoints
covered" because data files never shipped in the wheel — a confident wrong
answer, not an error; (c) a fan-out keyed success off "value is not None",
silently dropping endpoints that legitimately answer null; (d) job success
inferred from the absence of English failure words, so "Completed + Invalid
configuration" read as success. One rule covers all four: success is an
allowlist; absence of failure is not presence of success.

**C.3 The guard that exists but is not on the path.** Repeatedly, a
well-tested guard was simply not invoked by the real entry point (a clear
path that bypassed the staging guard; a collision check reachable for only
the one pair someone had thought about; a completeness test satisfiable by a
base-class no-op). Hence §8's "test the wiring" mechanic and the
entry-point-parity rule. Verify guards by deleting them and watching a test
fail.

**C.4 One cache key, two shapes.** Two components cached different values
(group names vs whole group bodies) under one unnamespaced cache key;
whichever ran first won, and reference names became entire objects.
Namespace cache keys; test the constant and the behavior in the order that
broke.

**C.5 Mechanical derivation of semantic facts.** Deriving shared write
targets from URL prefixes produced nonsense (`/template/template`); deriving
template-governed subtrees from path overlap marks a new BGP peer as
governed (the template carries the same field names as a peer); deriving
"the first listed partition is current" from fixture coincidence passed
every assertion. Semantic properties are declared per kind and verified
against live data; a list that is wrong loudly beats inference that is
wrong quietly.

**C.6 Claims that drift.** A hand-maintained per-row "✅ shipped" table said
more than the evidence; the fix was a machine-readable ledger validated on
load, plus a stated rule that ✅ means exactly "mock-verified" unless the
ledger says more. Never let prose carry evidence claims that a machine
holds.

**C.7 The reflective API server.** A prior side effort exposed every public
SDK method as an agent tool: 641 tools, ~250 of them writes, TLS off,
credentials in tool arguments, no transactions. If an agent surface is ever
wanted, it wraps the transaction engine with a handful of verbs (D-22).
Classification of read vs write is by the verbs a method actually issues —
dozens of `get_*` helpers POST.

**C.8 The retrofit tax.** Origin-keyed identity, locking, fail-closed jobs,
fail-closed ownership, secrets sealing, write-target declarations, and the
evidence ledger were all retrofits in field history, each forced by an
incident. The rebuild's §9 puts all of them in M2–M3, before resource
breadth. Resist the temptation to "add resources first, harden later" —
breadth is the cheap part.

**C.9 Rendering debt reads as product debt.** Raw dict reprs in table cells,
debug logs interleaved with reports, and 400-column tables made a working
report feel broken. Rendering discipline (§3) is part of the grammar
feature, not polish.

**C.10 The mock is a model.** Every normalizer written against the mock met
surprises on a real fabric (fields the spec doesn't carry, per-appliance
distinct names, endpoints requiring parameters the mock didn't). Treat live
sweeps as the mock's test suite: every divergence becomes a recorded fixture
and a mock fix.

---

## Appendix D — Glossary

| Term | Meaning |
|---|---|
| Orchestrator | HPE Aruba EdgeConnect SD-WAN central manager (formerly Silver Peak GMS); REST base `/gms/rest` |
| ECOS | EdgeConnect appliance OS; its own REST API (`/rest/json`), reached in v1 only via the Orchestrator's appliance proxy |
| nePk | appliance primary key (`"3.NE"`); the id every API call speaks |
| BIO | Business Intent Overlay — policy construct mapping traffic classes to topologies/paths |
| Template group | named bundle of config *sections* pushed by the Orchestrator to associated appliances; the push silently reverts out-of-band changes to governed config |
| Section | a template's unit of config (e.g. `bgp`, `securityMaps`, `optmap`); the vocabulary is fabric-readable |
| Appliance proxy | `POST /appliance/rest?nePk=&url=` — Orchestrator relays to the appliance's ECOS API |
| Save-changes | persisting an appliance's running config to flash after proxied writes; unsaved changes vanish on reboot |
| Action key / guid | handle for polling the Orchestrator's async job log |
| Candidate | locally staged, typed intent — not a device-side config tree |
| Canonical state | post-normalization comparable form; both diff sides pass through it |
| Origin | canonical `scheme://host[:port][/path]` identity of one Orchestrator; the key for all persisted state |
| Confirm window | the commit-confirm interval during which the detached watchdog will revert unless confirmed |
| Tier 0/1/2 | raw passthrough / generated stub / curated resource |
| Evidence ladder | implemented → mock-verified → live-read → live-no-op-write → live-change-and-rollback → production-supported |
| Shape fixture | sanitized, version-stamped capture of a real endpoint's response structure; golden input for normalize and the mock |

---

## Appendix E — Read-only shape survey protocol (the agentic fan-out)

The single cheapest way to prevent the §C.1 class of defect: observe the real
API's response shapes on several real Orchestrators **before** writing the
code that interprets them, and keep observing on every new release. This
appendix is the protocol; Feature 004 builds the harness.

### E.1 Safety rules (non-negotiable; the harness enforces them structurally)

1. **GET only, through the product client.** The harness has no code path
   that issues POST/PUT/DELETE, and the transport-level mutating-GET veto
   (§A.13) binds — so a probe list containing `GET /bgp/vrfs/{id}/state`
   (a GET that *deletes* BGP state) is refused by construction, not by
   reviewer vigilance. This is what makes it safe to hand to parallel
   agents.
2. **Least-privilege credentials**: one read-only API key (or read-only-role
   key) per Orchestrator, provisioned by the operator, stored per-origin in
   env/keyring like any other credential. Never a read-write key "because it
   was handy".
3. **Control-plane manners**: bounded concurrency (default low single
   digits), bounded total call budget per campaign, backoff on 5xx, and no
   stats endpoints in tight loops — the Orchestrator is a low-QPS control
   plane.
4. **Every probe journaled** `AUDIT_ONLY`, so a campaign is auditable after
   the fact: which origin, which paths, when, by what budget.
5. **Version capture first**: a campaign opens by recording the
   Orchestrator version, every appliance's ECOS version/partition, and the
   auth mode. A capture without versions is not evidence (Principle V).

### E.2 Sanitization (what a fixture may contain)

A committed fixture carries **structure, not data**: field names, nesting,
types, array-vs-object-keyed collections, observed enum values and
discriminator fields, key-shape classes for map-like objects (numeric string
/ nePk-shaped / IP-shaped / free-form), and cardinality notes ("object keyed
by numeric strings, 5 keys observed"). It must not carry: secrets (the §5
detector runs over every capture, biased to over-match), production
hostnames/FQDNs, public IPs, serial numbers, license fields, usernames, or
site names — replace with stable placeholder tokens so equality across
captures still compares. The origin is recorded as its digest plus a
human label the operator chooses (`lab-97`, `cloud-96`). Review the first
campaign's fixtures by hand before commit; after that the sanitizer tests
(§B.8.9) carry it.

### E.3 Campaign shape (how to fan out agentically)

- **One worker per Orchestrator**, in parallel; each worker owns one origin,
  one credential, one journal, one output tree. Workers never share state —
  merging happens after, in the shape-diff step. A worker that cannot reach
  its target reports `unreachable` rows; it never retries into another
  worker's target.
- **Probe list is data, versioned in the repo**: endpoint + params template +
  which resource family and standing question it feeds. Start from the union
  of: every read endpoint in §A.6–A.12, every curated kind's fetch/list
  endpoints, and §E.5's standing questions. Parameterized probes (nePk,
  overlayId, group names) enumerate from the inventory/listing endpoints
  first, then probe a bounded sample (e.g. 2 spokes + 1 hub, one overlay,
  one group), not the whole fabric.
- **Merge and diff**: after workers return, run the shape-diff across
  origins. Three outcome classes per endpoint: *stable* (same skeleton
  everywhere — safe to model tightly), *variant* (skeleton differs by
  release/deployment — model tolerant-reader with the observed
  discriminators, add a golden test per variant), *unanswerable* (nothing
  provisioned could answer — the resource records it and stays conservative).
- **Write the CLI accordingly**: each resource-family spec's source-validation
  section cites the fixtures by path; `normalize()` golden tests run against
  every captured variant; the mock serves the captured shapes (not invented
  ones); divergence discovered later (M9 or the field) becomes a new fixture
  plus a mock fix, never a silent code accommodation.

### E.4 Cadence

Run a full campaign: before M5 (first resource breadth), on every new
Orchestrator release the operator provisions, and before promoting any
resource past `mock-verified`. Re-runs are cheap (the harness exists); the
diff report is the deliverable — "9.8 changed these four skeletons" is
exactly the early warning a CLI vendor-tracking a control plane needs.

### E.5 Standing shape questions (the first campaign answers these)

Carried from field history; each is a known ambiguity where spec, SDK, and
observation disagree or are silent:

1. `GET /gms/overlays/config?overlayId=N` — single object or array? Does the
   shape differ between releases? (The §C.1 defect lived here.)
2. `GET /bgp/state` — `bgp_state_str` real type (spec says integer, description
   says string); `peer_state` enum values; `time_established` /
   `time_last_update` units (epoch s vs ms); maximum observed
   `neighborState` key count.
3. `GET /vrf/config/securityPoliciesSegments` — present on which releases,
   and exact response shape.
4. Orchestrator-scope `GET /deployment?nePk=&cached=` — shape vs the
   appliance-proxy deployment object; is `cached` genuinely required (422)?
5. `natMaps`/`natPools`/policy-map `options` block — full observed key set
   per release; which carry `merge`.
6. `GET /flow` — do the two fast-500 appliances reproduce on other fabrics
   (parameter rejection vs appliance fault)? Which filter params exist per
   release?
7. Template section vocabulary per release (`GET /template/templateGroups`
   name list) and per-section body skeletons — feeds ownership L1/L2/L3 and
   the derived priority band.
8. `subnets3/configured/addMultiple` / `deleteMultiple` request/response
   shapes (write bodies are `[INFERRED]` — capture the *read* side and any
   documented examples only; never probe a write).
9. `GET /virtualif/loopback` and `loopbackOrch` on a **populated** fabric —
   the `mgmtIp`/`mgmtIP` casing fold, and pool/history shapes.
10. Action-log records across releases: verbatim `taskStatus`/`result`
    strings for successful operations (grows the §A.5 allowlist with
    version-stamped entries — capture from reads of `GET /action`, not by
    performing writes).
