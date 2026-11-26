# Lessons learned

[← Back to README](../README.md)

What went wrong, what it cost, and what changed because of it.

---

## 1. A finished feature can still be the wrong feature

**What happened.** The first item of the post-launch wave was a full parallel rebuild of the operator panel: 7 epics and 105 commits, planned first because every later feature would add screens to it. When I used it hands-on, it didn't meet expectations, and I cancelled it.

**What it cost.** Weeks of work, plus an impact audit: several planned features had been designed to live inside the new panel.

**What changed.** A formal change proposal checked every remaining feature for dependencies on the cancelled panel and approved a workaround for each, so nothing downstream silently broke. I now validate a clickable or hands-on version of an operator workflow early, before building the whole thing. And building it *alongside* the old panel, rather than replacing it in place, is what made cancelling it painless.

## 2. Long jobs were silently starving the scheduler

**What happened.** The nightly data-quality snapshot was skipped on **7 of 45 nights**. Multi-hour scrapes were running on the same scheduler executor, so the snapshot's trigger time passed while the thread was busy, and the scheduler dropped it as "misfired". No error was raised.

**What changed.** Scheduled triggers now only *enqueue* jobs, on their own executor, and never run work on the worker's thread. A long scrape can no longer make a cron disappear. The broader lesson: **a job that silently doesn't run is worse than one that fails loudly**, which is why the scraper alarm came next.

## 3. "Unchanged" is not the same as "observed"

**What happened.** The price-history feature was specified on the claim that a flat price line meant "checked, and unchanged". That claim came from the planning agent, and I had accepted it. An adversarial review traced the refresh code and proved it false: delta refreshes only fetch product pages the store itself flags as changed, so "unchanged" often meant "not looked at". Some offers kept an old price for up to 14–28 days.

**What changed.** A *genuine observation* timestamp that moves only when a price was actually read. Visible price age on every offer. The history verdict is suppressed when the data is too old to support it. For a site whose main promise is honest, dated prices, this was the most important bug the reviews caught.

## 4. LLM output has a size limit you only discover in production

**What happened.** A large batch of product titles sent for decomposition came back as truncated JSON when the response hit the token limit.

**What changed.** Batch sizes are clamped (default 25, never above 50). Every model call goes through one function that detects truncation and API errors explicitly. Results are cached by title hash so retries are cheap, and prompt caching cuts repeated cost.

## 5. Cutting scope is a skill, and cheap code makes it harder

**What happened.** A native mobile app was fully planned (6 epics, 41 stories, a UX specification, an architecture document) before I asked whether it was needed. The site was already mobile-first.

**What changed.** I dropped it for an installable PWA, with a Google Play listing to follow. When AI makes building cheap, the temptation is to build everything; the real constraint becomes maintenance and attention. **Planning is cheap, and the plan is allowed to be thrown away.**

## 6. Know exactly what your monitoring can't see

**What happened.** The scraper alarm reliably flags stores that stop being read and offers that go stale. It does *not* catch a store whose markup changed in a way that still yields a price, but the wrong one.

**What changed.** Nothing yet, and it's written down as an open gap rather than implied to be covered. A documented blind spot is a backlog item; an undocumented one is an incident waiting to happen.

---

### En español

1. **Una funcionalidad terminada puede ser la equivocada:** cancelé un panel de administración reconstruido (105 commits) después de usarlo. Ahora valido temprano con una versión usable.
2. **Los trabajos largos dejaban sin turno al planificador:** el snapshot nocturno se perdió 7 de 45 noches sin ningún error visible. Ahora los cron solo encolan, en su propio ejecutor.
3. **"Sin cambios" no es lo mismo que "revisado":** una afirmación del agente de planificación sobre el historial de precios resultó falsa. Se creó una marca de "observación real" y la antigüedad visible de cada precio.
4. **Las respuestas del LLM tienen un límite de tamaño:** JSON truncado en lotes grandes. Ahora los lotes tienen tope, hay detección explícita y caché por título.
5. **Recortar alcance es una habilidad:** reemplacé una app nativa planificada (41 historias) por una PWA.
6. **Hay que saber qué no ve el monitoreo:** la alarma no detecta precios mal leídos, y eso está documentado como un punto ciego abierto.
