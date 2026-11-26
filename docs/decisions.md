# Key decisions

[← Back to README](../README.md)

The project keeps a decision log: 69 numbered product and architecture decisions, each dated and recorded with its context, the options considered and what it supersedes. Decisions are amended or reversed in place, never silently rewritten. Below are seven that shaped the system, in a short ADR format.

---

## 1. One database as the only integration seam

**Context.** The web tier (TypeScript) and the data pipeline (Python) need to exchange catalog data, jobs, verdicts and metrics. One person maintains both.

**Options.** An internal HTTP API between the tiers · a message broker · a shared database with a single schema owner.

**Decision.** A shared PostgreSQL database. The web tier owns every migration; the pipeline reflects the schema at runtime and never changes it.

**Consequences.**
- ✅ Transactional consistency, one backup, one place to debug, no API versioning between the tiers.
- ⚠️ Contracts no compiler checks: schema drift and normalisation functions that must behave the same in both languages. Mitigated by a schema-verification command that CI runs against a freshly migrated database.

---

## 2. Deterministic matching first, LLM only for the gray zone

**Context.** Matching listings across stores is the core problem. A wrong merge shows a false price, which breaks the only promise the site makes.

**Options.** Similarity scoring (embeddings or fuzzy text) with a threshold · fully LLM-driven matching · deterministic rules with an LLM for the ambiguous remainder.

**Decision.** Block by category and canonical brand, group by an exact clean title, and send only a narrow "gray zone" to an LLM judge that is biased towards *not* merging. Requirement: **zero wrong merges**.

**Consequences.**
- ✅ Every merge is explainable and auditable. Judge verdicts are persisted, so a pair is never judged twice.
- ⚠️ It under-merges by design: some true duplicates stay separate until an operator joins them. That is the right failure mode for a comparator.

---

## 3. The job queue lives in PostgreSQL

**Context.** Scrapes are started by a scheduler, by the operator from the admin panel and by hand from a CLI. The web tier must not run scrapers.

**Options.** A dedicated broker (Redis/RabbitMQ) · a cloud queue · a table with `FOR UPDATE SKIP LOCKED`.

**Decision.** A jobs table. The panel only *enqueues*; the pipeline worker claims and runs jobs. All three sources write the same rows.

**Consequences.**
- ✅ No extra infrastructure. The job history, its result summaries and the run monitor all live in one table, and a crashed job is swept to "failed" automatically.
- ⚠️ Throughput is modest, which is fine for 19 stores and one worker.

---

## 4. Neutrality enforced in code, not in policy

**Context.** The business model is two-sided: free and neutral for riders, with paid listings and aggregate insights for workshops and brands. Comparators abroad have been fined for selling positions presented as objective rankings.

**Decision.** The ranking code never reads the sponsorship flag. Paid content is shown only as separate, clearly labelled blocks, never as a position in a list. Aggregate insights never include personal data.

**Consequences.**
- ✅ The trust promise is a property you can verify in the code.
- ⚠️ It closes off the easiest revenue (paid ranking) for good.

---

## 5. Price freshness must be honest

**Context.** The price-history feature was first specified on the assumption that a flat line meant "checked, and unchanged". Review proved that wrong: delta refreshes skip product pages the store itself didn't flag as changed, so "unchanged" often meant "not looked at".

**Decision.** Add a *genuine observation* timestamp that moves only when a price was actually read from the product page. Show the real age of every price, and suppress the price-history verdict when the last real observation is older than the store's refresh cadence.

**Consequences.**
- ✅ The site never states more confidence than its data supports.
- ⚠️ More pipeline work, and some products show visibly older prices. That is the honest outcome.

---

## 6. Cancel a finished admin rebuild

**Context.** A complete parallel rebuild of the operator panel (7 epics, 105 commits) had been planned as the first item of the post-launch wave, because every later feature would add screens to it.

**Decision.** After using it hands-on, I cancelled it: it didn't meet expectations. The code was archived on a branch, and a formal change proposal audited every remaining feature for dependencies on the new panel and approved a workaround for each.

**Consequences.**
- ✅ No sunk-cost spiral, and the old panel kept working throughout because the rebuild had been built alongside it.
- ⚠️ Weeks of work discarded. The lesson is in [lessons-learned.md](lessons-learned.md#1-a-finished-feature-can-still-be-the-wrong-feature).

---

## 7. A PWA instead of a native app

**Context.** A native mobile app had been fully planned: 6 epics, 41 stories, a UX specification and an architecture document. None had been started.

**Options.** A PWA with no store listing · a PWA listed on Google Play as a Trusted Web Activity · a Capacitor shell on both stores.

**Decision.** PWA now, with a Google Play listing (TWA) as the next step. The site was designed mobile-first, so the installed app *is* motopedia.cl, unchanged. iOS users install it from the browser. All 41 native stories were deprecated.

**Consequences.**
- ✅ One codebase and no second release pipeline. Installability shipped as a small, self-contained spec, with a phone-only install banner.
- ⚠️ No App Store presence, and the Play listing will depend on Play Store policy for TWAs.

---

### En español

El proyecto mantiene un registro de 69 decisiones fechadas, con contexto, opciones y lo que reemplazan. Aquí hay siete: **(1)** una sola base de datos como única costura entre las capas; **(2)** fusión determinística primero y LLM solo para la zona gris, con el requisito de cero fusiones erróneas; **(3)** la cola de trabajos vive en PostgreSQL; **(4)** la neutralidad se garantiza en el código, porque el ordenamiento nunca lee el indicador de patrocinio; **(5)** la frescura de precios debe ser honesta, con una marca de "observación real"; **(6)** se canceló un panel de administración ya construido (105 commits) porque no cumplió las expectativas; **(7)** se reemplazó la app nativa por una PWA instalable, con Google Play como siguiente paso.
