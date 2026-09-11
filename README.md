<div align="center">

# Neuroneus — Public Site

**The argument for the company brain, and the simulations that make it legible.**

`www.limitless-stack.com`

<sub>React 19 · TypeScript 6 · Vite 7 · d3-force · Vercel Functions · Neon Postgres</sub>

</div>

---

## What this repository is

Neuroneus is a *company brain*: durable institutional memory — projects, tasks,
decisions, documents and the relationships between them — held as structured
state so that both people and language models can retrieve from it rather than
reconstruct it. The thesis is set out in full in [`MANIFESTO.pdf`](./MANIFESTO.pdf).

This repository is the **public surface**: the argument, the pricing model, the
lead pipeline, and — the substantive part — a set of **live product simulations**
that render the system's behaviour in the browser rather than describing it in a
screenshot.

The application runtime is a separate deployment.

**Why the simulations matter more than they look.** A company brain is a claim
about *emergent structure* — that a corpus of documents, people and goals has a
shape, and that surfacing the shape is the product. A static image cannot make
that argument, because the claim is about dynamics. So the site runs the thing:
a force-directed layout over a generated corpus, a concurrent agent-execution
trace, a settling knowledge graph. Each is a small deterministic model with
stated parameters, and each is reproducible bit-for-bit across machines.

---

## Table of contents

| §   | Section                                                     |
| --- | ----------------------------------------------------------- |
| 1   | [Layout and scale](#1-layout-and-scale)                     |
| 2   | [Deterministic graph generation](#2-deterministic-graph-generation) |
| 3   | [Force-directed layout](#3-force-directed-layout)           |
| 4   | [The agent execution trace](#4-the-agent-execution-trace)   |
| 5   | [Bilingual copy as a type](#5-bilingual-copy-as-a-type)     |
| 6   | [Resolution independence](#6-resolution-independence)       |
| 7   | [Lead pipeline](#7-lead-pipeline)                           |
| 8   | [Discoverability](#8-discoverability)                       |
| 9   | [Running it](#9-running-it)                                 |

---

## 1. Layout and scale

12,879 lines across 81 source files.

| Area       |  Lines | Files | Role                                            |
| ---------- | -----: | ----: | ----------------------------------------------- |
| `src/`     | 12,516 |    74 | Site, simulations, bilingual copy, pricing model |
| `api/`     |    251 |     5 | Lead capture, abuse guards                       |
| `scripts/` |     53 |     1 | Domain synchronisation across static artefacts   |
| `db/`      |     59 |     1 | Throttle and submission schema                   |

```
src/
├── components/home/
│   ├── overview/          ConnectsGraph · HarborCanvas · seeded generator
│   ├── demos/
│   │   ├── knowledge/     force-directed knowledge graph
│   │   ├── team/          org graph, drag-and-pin, side panel
│   │   ├── autopilot/     goal and task boards, tab cycling
│   │   └── briefing/      the morning-briefing surface
│   └── chat/              agent execution trace (§4)
├── copy/                  site.en.ts is the schema; site.de.ts must satisfy it
├── pages/                 Home · Pricing · LetsTalk · YC · legal
└── lib/                   i18n · SEO · site origins · form guards
```

**Bundle, measured and stated plainly.**

| Artefact | Raw       | gzip      |
| -------- | --------: | --------: |
| JS       | 677.6 KB  | 197.8 KB  |
| CSS      |  58.4 KB  |  11.4 KB  |

Single chunk; **no code splitting on this deployment.** Every simulation, both
language bundles and `d3-force` are in the entry chunk. This is a known,
unaddressed cost rather than an oversight — it is recorded here because a README
that only lists wins is not a measurement, it is an advertisement. The
application runtime, where the cost actually bites, *is* split and measured.

---

## 2. Deterministic graph generation

The "Neuroneus connects the dots" graph is **not authored data.** All 56 nodes
and 85 links are generated from a seeded PRNG, so the identical graph emerges on
every run, every build and every machine.

```ts
/** Mulberry-style integer hash PRNG — deterministic across engines. */
function makeRng(seed: number) {
  let t = seed;
  return () => {
    t |= 0;
    t = (t + 1831565813) | 0;
    let n = Math.imul(t ^ (t >>> 15), 1 | t);
    n = (n + Math.imul(n ^ (n >>> 7), 61 | n)) ^ n;
    return ((n ^ (n >>> 14)) >>> 0) / 4294967296;
  };
}
```

`Math.random()` is deliberately unused. It is unseedable and its algorithm is
implementation-defined, so it yields a different graph per engine and per
reload — an unreproducible figure, and therefore one whose appearance cannot be
reasoned about or reviewed. Everything here is integer-domain: `Math.imul` for
32-bit multiplication with defined overflow, `|0` and `>>>` for coercion, a
single division by `2³²` at the end. No float accumulates, so the output is
bit-identical across engines.

| Parameter     | Value | Effect                                           |
| ------------- | ----: | ------------------------------------------------ |
| `SEED`        |     7 | Pinned. Change it for a different but valid graph |
| `NODE_COUNT`  |    56 | Nodes                                            |
| Links         |    85 | Emergent from the wiring rules, not configured   |
| Colour buckets|     8 | `id mod 8`, one ellipse anchor per bucket        |
| `child` share |   0.3 | `rng() < 0.3`, else `ref`                        |

### The reproducibility hazard, written down deliberately

> Call order matters. The generator draws from **the same stream** for node
> jitter first and link wiring second, so reordering these loops silently
> produces a different graph.

This is the seeded-simulation trap in its general form: a single PRNG stream
consumed by multiple consumers couples them, so a refactor that touches neither
consumer's logic still perturbs the result. Nothing errors. The output is still
valid, still deterministic — just *different*, and the diff is a picture, so no
test catches it. Documenting the coupling at the point of definition is the
cheapest available mitigation; separate streams per consumer would be the
expensive one, and is not warranted at this size.

Clusters hang together because link wiring **prefers an earlier node of the same
colour**, with de-duplication through a normalised `min-max` key. Community
structure is therefore a consequence of the generative rule, not a layout
parameter.

---

## 3. Force-directed layout

The knowledge-graph demo runs a real `d3-force` simulation. Parameters are
explicit, ordered, and matched to the deployed configuration:

```ts
forceSimulation(nodes)
  .force('link',    forceLink(links).id(d => d.id).distance(36))
  .force('charge',  forceManyBody().strength(-75))
  .force('collide', forceCollide(d => nodeRadius(d.val) + 3).iterations(2))
  .force('x',       forceX(d => clusterCenter(d.type).x).strength(0.14))
  .force('y',       forceY(d => clusterCenter(d.type).y).strength(0.14))
```

An n-body relaxation: `charge` supplies pairwise repulsion (Barnes–Hut
approximated), `link` a spring at rest length 36, `collide` a hard non-overlap
constraint at two iterations per tick, and the positional `x`/`y` forces a weak
attractor toward each type's cluster anchor. The `0.14` strength is the term that
decides whether type structure is *visible* or merely *present* — too low and the
clusters dissolve into the repulsion field, too high and the layout is a set of
disconnected discs with the link structure crushed out of it.

**Pre-settling.** The simulation is advanced to rest before first paint:

```ts
for (let i = 0; i < 600 && sim.alpha() >= sim.alphaMin(); i++) sim.tick();
```

600 ticks, guarded by the `alphaMin` convergence criterion so a converged layout
stops early rather than burning the remaining budget. The graph therefore appears
*already organised* — the honest depiction, since the claim is that the structure
is inherent in the corpus, not that it assembles itself while you watch. The live
simulation then only has to react to dragging.

**This settling moved from module scope into a `useMemo`.** At module load it ran
on import — before any user had navigated to the section, on a thread that owed
its time to first paint. Same 600 ticks, no longer at import time.

`d3` mutates the nodes it is handed, writing `x`/`y`/`vx`/`vy` in place, so the
simulation is given copies. Releasing a dragged node clears its pin and lets the
forces reclaim it.

---

## 4. The agent execution trace

The Neuron chat demo is the site's central claim rendered as behaviour: what it
looks like when a system that already holds your company's context is asked a
question.

**It is time-driven, not queue-driven.** One clock starts when the question is
sent, and every step, intermediate thought and character of the answer is keyed
off *milliseconds elapsed*:

```
t=0 ──────────────────────────────────────────────────────────▶
     ├── search_docs      ▓▓▓▓▓▓▓▓▓░░░
     ├── traverse_graph        ▓▓▓▓▓▓▓▓▓▓▓▓░░░
     └── collate                   ▓▓▓▓▓▓▓░░
                                        └── answer streams ───▶
```

Nothing is chained off the previous step finishing. Steps therefore **legitimately
overlap**, which is what makes the trace read as concurrent work rather than as a
progress bar — because concurrent work is what it is depicting. A sequential
queue would have been simpler to write and would have misrepresented the system.

Every timing derived from answer length is **computed per language**: the German
answer is a different length, so a hard-coded schedule would drift against it.
Demo fixture copy lives beside the timings it drives rather than in `src/copy`,
since it is content *of the model*, not of the site — and the two are only
readable together.

---

## 5. Bilingual copy as a type

English and German, with the guarantee enforced by the compiler rather than by
review.

```ts
// site.en.ts IS the schema. Copy in ./index.ts is derived from it,
// so site.de.ts cannot compile with a key missing, renamed or mistyped.
export const siteEn = { … } as const;
```

The English object is the source of truth; the `Copy` type is *derived* from it;
the German file is checked against that derived type. A missing translation is a
**build failure**, not a production `undefined` rendered to a customer in the
wrong language.

327 lines of English, 329 of German — structurally identical by construction.
The two-line delta is German requiring more line breaks, not more keys; that is
exactly what the type guarantees.

This is the same discipline as §3 of the runtime's README applied to copy: make
the invariant a compile error, because the alternative is a human remembering.

---

## 6. Resolution independence

The demos are authored once at a fixed 1440 × 840 desktop geometry and scaled to
fit, rather than reimplemented per breakpoint:

```ts
const measure = () => setScale(el.clientWidth / DEMO_WIDTH);
```

Measured in a layout effect and kept current with a `ResizeObserver` — which also
catches container-driven changes that a `window` resize listener would miss
entirely. The wrapper's height tracks the scale, so a shrunken demo does not
reserve the vertical space of a full-size one.

The initial value is `0.65`, matching the constrained desktop case, so the first
paint lands close to correct before measurement resolves.

**A caveat, kept rather than quietly dropped.** The mobile scaling pass was
verified structurally and arithmetically, **not visually on a device.** Drag and
tap in the team map and the goal/task tabs travel through a CSS transform, and
hit-testing through a transform is exactly where this class of fix fails. Stated
here because an unverified claim presented as a verified one is worse than the
gap it conceals.

---

## 7. Lead pipeline

Two endpoints — `api/waitlist.ts` and `api/demo_request.ts` — sharing one guard
module. Four independent layers, ordered cheapest-first:

**1 — Honeypot.** A `company_website` field no human ever sees. Filled means bot.

**2 — Server-side rate limiting.**

| Parameter          | Value           |
| ------------------ | --------------- |
| Window             | 10 minutes      |
| Max per window     | 5               |
| Minimum gap        | 3 seconds       |
| Key                | Salted IP hash  |

The browser-side limiter is a courtesy; **this** is the one that holds, because
it cannot be cleared from the client. Expired rows are deleted on the way past,
so the table prunes itself without a scheduled job.

**3 — Hashed callers, never stored IPs.** `SHA-256(THROTTLE_SALT ‖ ip)`. An IP is
personal data under GDPR, and a rate limiter only ever needs *"same caller as
before"*, never *who*. The salt is load-bearing: the IPv4 space is 2³² and an
unsalted digest is exhaustively reversed in seconds.

**4 — Field clamping.** `clampField` trims and truncates every input at the
server: `name` 100, `email` 255, short 500, long 1000. Never trust a length the
browser claims to have enforced. Body parsing handles both the object Vercel
supplies when `Content-Type` cooperates and the raw string it supplies otherwise.

Email validation uses a single regex shared with the client, so both sides agree
on what an email is — divergent validators produce the failure mode where the
form submits successfully and the row is rejected.

---

## 8. Discoverability

- **JSON-LD** structured data: `VideoObject` for the YC talks (so they surface as
  video results rather than a bare page link), plus `BreadcrumbList`.
- **Canonical URLs** and Open Graph tags derive from one origin constant.
- **`youtube-nocookie`** embeds, so no tracking cookie is set before anyone
  presses play.
- **Skip link** and ARIA labelling throughout the navigation.

Every absolute URL reads from `src/lib/site.ts`, so relocating the domain is one
environment variable rather than a search through the tree. `robots.txt` and
`sitemap.xml` are static files that also carry absolute URLs, so they are
reconciled by script:

```bash
node scripts/sync-domain.mjs    # after changing VITE_SITE_URL
```

`VITE_*` is inlined at build time, so a domain change requires a redeploy.

### `/yc`

The site carries a page citing the two Y Combinator talks the product thesis is
built on — *How To Build An AI-First Company* and *The New Way To Build A
Startup*. Where a thesis is borrowed, it is attributed. The manifesto argues the
part that is not.

---

## 9. Running it

```bash
npm install
npm run dev
```

```bash
npm run build
```

### Environment

| Variable                              | Purpose                    |
| ------------------------------------- | -------------------------- |
| `VITE_SITE_URL`, `VITE_APP_URL`       | Public origins, build-time |
| `DATABASE_URL`                        | Neon Postgres              |
| `THROTTLE_SALT`                       | Rate-limit hashing (§7)    |
| `RESEND_API_KEY`                      | Lead notification          |
| `LEAD_NOTIFY_FROM`, `LEAD_NOTIFY_TO`  | Notification routing       |

`VITE_*` variables are **inlined into the public bundle** — never put a secret in
one.

---

## Status

| Component                                    | State                     |
| -------------------------------------------- | ------------------------- |
| Site, pricing, legal, bilingual copy          | **Shipped**               |
| Simulations — graph, force layout, agent trace| **Shipped**               |
| Lead pipeline with four-layer guards          | **Shipped**               |
| SEO, structured data, domain sync             | **Shipped**               |
| Mobile scaling                                | Shipped; **not yet device-verified** (§6) |
| Code splitting                                | Not done; cost stated (§1)|

---

<div align="center">
<sub>Built in Germany. Data resident in Frankfurt.</sub>
</div>
