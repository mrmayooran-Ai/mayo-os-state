# HANDOVER_RESULT — refleksjon-pille-bug (2026-06-11)

Svar på handover-oppgave: «Adresser refleksjon-pille-bugen Mayo flagget — start
med PWA-cache, så backend-respons.» Mayo bekreftet symptom: **ny UI lastes, men
ingen pille** på ekte entries.

## Diagnose (verifisert mot kildekoden, ikke gjettet)

Handoverens tre hypoteser, testet mot faktisk kode:

| # | Hypotese | Funn |
|---|----------|------|
| 1 | Service worker cacher gammel `index.html` | **Avkreftet.** `public/sw.js` er *push-only* — ingen `fetch`-handler, ingen Cache API. Den kan ikke servere stale HTML. Hvis hele gamle UI-et noensinne dukker opp er det browser/nginx HTTP-cache, ikke SW. |
| 2 | `reflection_model` kommer ikke fra backend → `isSafeReflection()` skjuler pille | **Bekreftet — og dypere enn antatt (se under).** |
| 3 | Default `useState(true)` | Spec-riktig (design: pillen ÅPEN). Ikke en bug. |

### Rotårsak (#2)
- Frontend `EntryCard` rendrer pillen kun via `isSafeReflection(e)`, som krever
  `e.reflection` (`src/lib/psych.js`).
- Backend `GET /journal` (`db_api/server.py`) selekterte
  `id, entry_date, source, raw_text, ai_summary, mood, tags, created_at, media, place, weather`
  — **aldri `reflection`/`reflection_model`.**
- Refleksjonsteksten heter `psykolog_short`, genereres per **dag-fil** av en
  **lokal Ollama-modell** (`enrich_pipeline.py` Phase 4b, `llama3.2:3b`,
  `OLLAMA_URL=localhost`), og lagres i **vault-frontmatter** — aldri i Postgres,
  aldri i `/journal`-responsen.
- ⇒ `isSafeReflection(e)` var alltid `false` → pillen rendret aldri på ekte
  entries. Kun seed-mock-summariene (`SEED_*` i `psych.js`) viste, derav «ny UI,
  men ingen pille».

## Fiks (commit `8c9bfe1`, branch `claude/confident-noether-lpacih`)

`db_api/server.py`:
- `_load_psykolog_short(date)` — leser `psykolog_short` fra vault-dagfila.
- `_attach_reflections(entries)` — fester dagens refleksjon på dagens **nyeste**
  entry (én pille per dag, ikke duplisert) som `reflection` +
  `reflection_model='local'`.
- Wiret inn i `GET /journal`.

**Sovereignty:** `reflection_model='local'` er sannferdig — `psykolog_short` rører
aldri skyen. Derfor viser frontend-guarden `isSafeReflection` 🔒-merket trygt også
for 🔴 ivf/økonomi-dager.

**Ingen DB-migrasjon** — vault-frontmatter er kanonisk kilde. **Ingen
frontend-endring** — `feat/whoop-redesign` (live `28cbc97`) leser allerede
`e.reflection`/`e.reflection_model`.

## Verifisert
- `python3.12 -m py_compile db_api/server.py` → OK.
- `tests/test_journal_reflection.py` (5 tester, AST-uttrekk av de to hjelperne →
  kjører uten fastapi/asyncpg/DB) → 5/5 grønne. Dekker: per-dag dedup, manglende
  fil, blank refleksjon, 🔴-domene fortsatt `local`, distinkte dager.

## Gjenstår (krever Mayo/VPS)
1. **Deploy backend** (master deployes manuelt) + `sudo systemctl restart db-api`.
   Pillen vises da på ekte entries. (Denne containeren har ikke VPS-tilgang.)
2. **Whoop re-auth (TODO #8):** åpne `https://db.mayooran.com/whoop-auth` én gang.
3. **Kjent begrensning:** `psykolog_short` er per dag, mens designet sa «per
   entry». Pillen viser dagens refleksjon på dagens nyeste entry. Ekte per-entry
   krever at `enrich_pipeline` genererer per `## HH:MM`-seksjon (større endring).

---

# HANDOVER_RESULT — Tasks ↔ Apple Reminders sync-layer (2026-06-11)

Pending decision (CLAUDE.md) RESOLVED: Mayo valgte **B (sync-layer)**. Valg:
dedikert liste · nyeste-vinner · speilet sletting · inline+periodisk.

## Arkitektur
Apple har ingen sky-API → all enhets-sync rir på den eksisterende iOS Shortcut-broen
(`/reminders/bulk-sync`-kø). `crm_task → Apple` = lag/oppdater en lenket `reminder`-rad
(`source='local'`, `sync_state='pending_push'`); køen leverer den til iOS og
`ios-ack` setter `ios_uid`. `Apple → crm_task` hekter på bulk-sync-upserten. De to
tabellene forblir separate (ingen merge).

## Levert (commit `2cc8e0c`, branch `claude/confident-noether-lpacih`, PR #3)
- **`migrations/005_task_reminder_sync.sql`** (idempotent): lenke-kolonner
  `crm_task.reminder_id`/`sync_origin`/`synced_at` + `reminder.crm_task_id`, pluss to
  hygiene-fikser: `crm_task.tags` (manglet i migrasjonene) og defensiv guard fordi
  `reminder`-tabellen ikke er i repo-migrasjonene (opprettes utenfor repo på VPS).
- **`modules/reminders/task_sync.py`**: rene hjelpere (scope, felt-mapping,
  newest-wins `decide_direction`) + tynn DB-glue (`apply_task_change`,
  `apply_task_delete`, `apply_reminder_change`, `delete_tasks_for_missing_reminders`,
  `reconcile_once`).
- **`server.py`**: inline bakgrunns-hooks på POST/PATCH/DELETE `/tasks`; revers-sync +
  speilet sletting + `delete_queue` i `/reminders/bulk-sync`; 5-min reconcile-loop ved
  oppstart. **Alt gated av `TASK_REMINDER_SYNC=1` (default AV)** og pakket i try/except
  så en sync-feil aldri bryter `/tasks`.
- **`tests/test_task_reminder_sync.py`**: 13 rene enhetstester (uten fastapi/DB).

## Verifisert her
`py_compile` (3.12) for `server.py` + `task_sync.py`; **13/13** sync-tester +
**5/5** refleksjon-tester grønne. Ingen intern Python-kaller bruker `/tasks`-handlerne
direkte (kun HTTP-ruting), så ny `BackgroundTasks`-param bryter ingenting.

## Aktivering (Mayo/VPS — TODO #9)
1. Kjør `migrations/005_task_reminder_sync.sql`.
2. Lag Apple-liste **«Mayo OS»** på iPhone (eller sett `TASK_SYNC_LIST=<navn>`).
3. `TASK_REMINDER_SYNC=1` i `.env` + `sudo systemctl restart db-api`.
4. Verifiser: lag en task i Mayo OS → den havner i «Mayo OS»-lista i Apple etter neste
   Shortcut-bulk-sync; fullfør i Apple → tasken avmerkes i Mayo OS.

## Kjent begrensning
Apple-side **sletting** krever én liten utvidelse av iOS-Shortcuten: les
`delete_queue` fra `/reminders/bulk-sync`-svaret og slett de `ios_uid`-ene fra Apple
Reminders. Uten det fjernes oppgaven i Mayo OS + DB, men den lenkede Apple-reminderen
kan henge igjen (aldri datatap). Resten av sync-en virker uten Shortcut-endring.

---

# HANDOVER_RESULT — Journal (Brain) spec-gjennomgang (2026-06-11)

Gjennomgang av design-Claudes konsoliderte Journal-spec mot faktisk frontend på
`origin/feat/whoop-redesign` (read-only audit, 10 filer). **Konklusjon: §1–6 er i
praksis fullt bygd og live; kun 3 §7.3-punkter gjenstår (og §7 er eksplisitt
Mayo-gated → IKKE bygd autonomt).**

### Bekreftet live (§1–6)
Faner/struktur (PRIMARY+MORE_TABS, MoreSheet, magenta), Tidslinje (søk, composer,
dato-filter, dividers skjult ved søk, '99:99'-anker, EntryCard m/ 10 moods + pille,
FAB, `PSYCH.touch()` umiddelbart ved create/delete, 2,6 s debounce ved edit), Editor
(toolbar B/i/H/•/❝/`[[ ]]`/📷 + 👁, `renderEntryMd`, Detaljer m/ AI_TAGS + «lignende
entries»), Psykolog (per-entry pille, PsychSummaryCard-dividers, Innsikt m/ per-uke
søylegraf + delta-kort + levende-prikk, Samtale m/ «🔒 privat · lokal modell»),
Oversikt/Kalender/Media/Graph — alle implementert med fil:linje-bevis.

### 🔗 Viktig kobling
Frontend leser per-entry refleksjon fra **backend-feltene** `entry.reflection` +
`entry.reflection_model` (`PageJournal.jsx:324`, `isSafeReflection` i `psych.js`).
⇒ Refleksjon-pille-fixen i denne økten (`8c9bfe1`) er nettopp det som tenner pillen
når backend deployes. §4-kontrakten er dermed lukket.

### Gjenstår — KUN §7.3 (Mayo-gated, IKKE bygd; avklar tone/prioritet)
1. **Mood-kurve over ukene** i Innsikt (i dag: mood-prikker + per-tema søyler, ingen
   trend-linje over tid).
2. **«Verktøy vi har øvd på»-liste** (helt fraværende — fremtidig).
3. **Emnetagg → relaterte entries** drill-down (Innsikt filtrerer ukelista, ikke
   entry-lista — ingen «vis alle #trening-entries fra mai»).
- **Kart-fanen** er en Fase-2-mockup (hash-pseudokoordinater) til backend gir
  `{lat,lon}`. Vault-fanen er derimot reelt implementert (ikke placeholder).

### Personvern verifisert
`isSafeReflection()` skjuler 🔴-domene-refleksjon med mindre `reflection_model==='local'`;
backend-regelen er dokumentert i `psych.js`. 10 moods komplette. Ingen jobb/privat-
blanding observert.

**Anbefaling:** ingen autonom frontend-bygging her (branch-regler + §7-gate). Når
Mayo vil ha §7.3-punktene, gi tone/prioritet → bygges da. Spec er ellers «ferdig».
