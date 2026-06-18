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

---

# HANDOVER_RESULT — Statisk duplikat-audit høyrepanel ↔ sentrum (2026-06-18)

Svar på TL;DR-konsolideringspunkt fra REVIEW-2026-06-17 §1: «En statisk audit av
høyrepanel + sentrum-flata for resterende duplisering (jeg så ennå flere i dag).»

Auditen scannet `src/mobile/livsplan_v12/desktop.jsx` (RightPanel + RightSummary)
og hoved-flatene (PageHjem, PageLivsplan, PageObs, PageStyrke, PageBrain, PageTasks,
PageKalender) for genuine data-/layout-duplikater.

## Reelle duplikasjoner funnet: 2

### 1. KPI-tallene (I dag · Forfalt · Uka · Innboks) — HØY ALVORLIGHET

**Forekomst A:** `livsplan_v12/desktop.jsx:344-356` (RightSummary KPI-rad)
**Forekomst B:** `livsplan_v12/shared.jsx:142-165` (SmartTiles, rendres i sentrum)

Begge beregner og viser de samme fire tallene:
- `inDay / c.today` (oppgaver med state='today')
- `overdue / c.overdue` (forfalt-cookie)
- `inWeek / c.week` (state='week')
- `inboxN / c.inbox` (state='inbox')

På desktop ≥1280px ser brukeren «I dag: 5» både i høyrepanelet og i SmartTiles
i sentrum samtidig. Forvirrende — særlig fordi tallene OPPDATERES asynkront
hvis sources er forskjellige (RightSummary regner direkte mens SmartTiles bruker
lpCounts-hjelper).

**Forslag:** Skjul SmartTiles fra desktop-bredder (`@media (min-width: 1280px)
{ display: none }`) når RightPanel er synlig, eller flytt KPI-radene til
sentrum og fjern fra RightSummary. Førstnevnte er minst-risiko (en CSS-endring).

### 2. Forfalt-stack — LAV ALVORLIGHET

**Forekomst A:** `livsplan_v12/desktop.jsx:359-378` (RightSummary «Forfalt-stack»,
opp til 5 forfalt items som klikkbare knapper).
**Forekomst B:** Ingen direkte i dag, men forfalt-counter i KPI-rad (se #1) +
forfalt-merker per ItemLine i sentrum overlapper semantisk.

I dag er det greit fordi sentrum ikke ekspanderer forfalt til en egen liste.
Risikoen er hvis PageToday eller en triage-modus legges til som viser
«5 forfalt» som egen liste — da har vi tre steder.

**Forslag:** Ingen aksjon nå, men noter som invariant: «forfalt-stack lever
kun i RightSummary; sentrum kan vise antall, ikke liste.»

## Pseudo-duplikasjoner (ikke aksjon)

- **ItemLine** brukes både i RightPanel og sentrum — dette er intentional
  delt komponent, ikke duplisering.
- **7-dagers utsikt** finnes kun i RightSummary; ingen sentrum-versjon
  enda. Hvis sentrum ekspanderes, harmoniser.

## Rekkevidde-merknad

Duplikasjonene er KUN på desktop ≥1280px (RightPanel skjules under).
Mobile flater har ikke problemet.

## Anbefalt rekkefølge

1. **Quick win (5 min):** legg `display: none` på SmartTiles i v12 når
   skjermbredden er ≥1280px (RightPanel viser samme info).
2. **Invariant-doc:** noter «forfalt-stack lever kun i RightSummary» i
   den interne arkitektur-doc'en (Notion-side `3636fb3c-...`).
3. **Senere:** når PageToday vurderes, sjekk om 7-dagers utsikt bør
   speiles eller flyttes — IKKE dupliseres.

---

# HANDOVER_RESULT — /brain (Journal) review (2026-06-18)

Memo §1: «/brain — tro at dette er kraftigste flata du har bygget…
verdt en egen review-runde.»

## Strukturoversikt (6 fane-komponenter + 9 støttekomp.)

| Komp | LOC | Ansvar |
|---|---|---|
| Graph.jsx | 1019 | 3D auto-roterende mindmap fra journal-entries (nodes = entries, edges = topic-overlap). Custom SVG, ingen tunge 3D-libs. |
| Tidslinje.jsx | 382 | Day-One-stil feed (måned → uke → entry). Voice-innspilling (MediaRecorder). FAB+. |
| Oversikt.jsx | 297 | Nøkkeltall, aktivitetsgraf, mood/tags + **«Spør Mayo» RAG** (ChromaDB→Claude). |
| Psykolog.jsx | 749 | Innsikt + Samtale. Weekly auto-refleksjoner. **Bruker marked.parse + dangerouslySetInnerHTML**. |
| Kart.jsx | 119 | Fase-2 preview (entries→koordinat). Kun konsept i dag. |
| Media.jsx | 75 | Fase-2 preview (bildegalleri). Tomt i dag. |
| components/ | 1484 | FullscreenEditor (autosave 1.4s debounce), FilterTile, StatsStrip, UnifiedBar m.fl. |

## Sterke sider

1. **Solid 3D-grafarkitektur** — custom SVG, auto-rotate/drag/pinch (etter
   denne sesjonen), 3D→2D toggle, halo-ringer + breathing mood-ring,
   edge-pulsing via tick-basert bezier-interpolasjon.
2. **Modulær komponent-trestruktur** — Tidslinje + FullscreenEditor er
   separate state-maskiner (`saved|typing|saving|error`), persistingRef
   forhindrer overlappende PATCH.
3. **«Spør Mayo» RAG** — grounded svar: ChromaDB-matches + Claude-svar
   med 60s timeout-handling, fallback til lokale deterministiske
   innsikter hvis remote feiler.

## Risiko / forbedringspotensiale

**1. 🔴 BLOCKER — XSS-vektor i Psykolog.jsx (linje 134, 454, 471)**
   `marked.parse()` brukes direkte i `dangerouslySetInnerHTML` uten
   sanitizing. Hvis vault-fil eller Claude-svar inneholder rå HTML,
   kjøres det. **Fix:** `dompurify` wrapper rundt marked.parse(). Est: 15 min.

**2. ⚠️ ARIA-mangel på 1019-linjes Graph.jsx**
   SVG-nodes/edges har ingen `role="button"` eller `aria-label`. Screen-
   readers får ingen kontekst. Tidslinje har 1 aria-label. Est: 45 min.

**3. ⚠️ Voice-opplasting timeout udokumentert (Tidslinje.jsx:164-184)**
   `handleStop` setter `voiceBusy=true` uten timeout. Hvis transcribe
   henger i 30s+ stillhet, ingen UI-tilbakemelding. Est: 30 min.

**4. ⚠️ `className="private"` er KUN visuell maskering (Oversikt.jsx:17,146)**
   Innholdet ligger fullt i DOM — debugger kan lese alt. Ikke
   sikkerhetsmekanisme. Hvis ekte data skal skjules → backend-rendering
   eller end-to-end-kryptering.

**5. ↘ Race i FullscreenEditor (FullscreenEditor.jsx:41-66)**
   `currentEntryId` + `idRef` synkroniseres ikke før persist() returnerer.
   Rask entry-bytte kan PATCH'e feil ID. Fix: useEffect-cleanup som
   avbryter pending PATCH ved unmount/prop-endring.

**6. ↘ Duplikat SOURCE_COLOR (Graph.jsx:32-38 vs lib/journal.js)**
   Begge steder definerer source-styling. Ny source = manuell sync.
   Fix: trekk til journal.js som eksport.

## Prioritert neste-steg

1. **[BLOCKER]** Sanitize marked.parse — `dompurify` (15 min)
2. **[HIGH]** ARIA på Graph.jsx nodes + keyboard-nav (45 min)
3. **[MED]** Voice timeout + countdown (30 min)
4. **[MED]** Refactor SOURCE_COLOR (20 min)
5. **[LOW]** localStorage fallback for Graph-filtre (25 min)

**Estimat: ~2.5 timer total for hele suiten.**

**Konklusjon:** /brain er kraftig og godt bygget — hovedrisikoen er
XSS via marked-parsing + ARIA-mangel. Resten er optimisering.

---

# HANDOVER_RESULT — /kalender etter gcal-pull (2026-06-18)

Memo §1: «Ny gcal-pull endrer hva som vises der, men ingen visuell
verifisering.»

## Hva gjør gcal-pull?

`modules/calendar/gcal_pull.py` poller Google Calendar (privat + Obs BYGG)
via syncToken (kun endringer siden sist), mapper events tilbake til
`item` og `meeting` ved oppslag på
`extendedProperties.private.mayo_item_id / mayo_meeting_id`. Fallback
ved manglende ext-properties: oppslag på `gcal_event_id`, re-stempel
ext, sett `gcal_synced_at`.

## Felter / tabeller den skriver til

**`item`** (007_item_gcal.sql):
  - `title` (strip ✓-prefiks), `scheduled_at`, `gcal_calendar_id`,
    `gcal_synced_at`. SQL i gcal_pull.py:212–223.

**`meeting`** (013_meeting_gcal.sql):
  - `title`, `scheduled_at`, `gcal_calendar_id`, `gcal_synced_at`.

Cancellation-håndtering: nullstiller `gcal_event_id` (linjer 140–159).

## ⚠️ KRITISK gap: SPA leser fra annen tabell

`mobile/pages/PageKalender.jsx:286-309` henter fra `/api/db/calendar`
som leser `calendar_event`-tabellen (db_api/calendar_module.py:72-85):

```sql
SELECT id, source_calendar, title, start_at, end_at, all_day, location,
       description FROM calendar_event WHERE user_id=$1 AND start_at BETWEEN $2 AND $3
```

**Men gcal_pull skriver til `item` + `meeting` — IKKE `calendar_event`.**

Pull-data til item-felt vises altså ikke direkte i kalender-UI med mindre
det finnes en synker mellom `item.scheduled_at` → `calendar_event`. Må
verifiseres at en sånn projeksjon finnes (eller at /api/db/calendar
unioner alle tre tabellene). Hvis ikke: UI-skille mellom kalender og
livsplan/obs-bygg er reelt — pull oppdaterer kun task-listen, ikke
kalender-renderingen.

## Endepunkter for smoke-test

1. `GET /api/db/calendar?days_ahead=30&days_back=30` — verifiser
   non-empty events, start_at/end_at/title.
2. `GET /api/db/items?state=scheduled` — verifiser `scheduled_at` ble
   pull-oppdatert, `gcal_synced_at IS NOT NULL`.
3. `GET /api/db/items/{item_id}` — verifiser `gcal_event_id`,
   `gcal_calendar_id`, `gcal_synced_at` finnes.

## Topp-edge-cases som kan kræsje pull-loopen

**A) Timezone / malformed dateTime (gcal_pull.py:81-96)**
   `_parse_event_time()` returnerer None silent ved malformed input →
   `scheduled_at=NULL` → item «stille tap» i kalender-UI.

**B) Duplikat gcal_event_id (007_item_gcal.sql:16)**
   `idx_item_gcal` er ikke UNIQUE. To items kan peke på samme event-ID
   → fallback-lookup returnerer kun den første → sletting i Google
   etterlater zombi-items med null `gcal_event_id`.

**C) Kalender-flytt + ext-stripping (gcal_pull.py:299-308)**
   Hvis bruker flytter event mens pull-sweep kjører, Google mister ext-
   properties → fallback til gcal_event_id-oppslag. Hvis bruker
   samtidig redigerer item-tittel lokalt: implisitt konflikt
   (`gcal_synced_at` verner kun app→GCal, ikke fullstendig motsatt vei).

**D) SyncToken >30 dager (gcal_pull.py:127-129)**
   Google 410 → rekursivt initial-sync på siste 30 dager. Hvis
   `mayo-gcal-pull.sh` har ligget nede en måned, kan re-fetch skape
   duplikater hvis dedup-invariantene ikke holder. settings_kv lagrer
   token — hvis den slettes manuelt, samme risiko.

## Anbefalinger

1. **[HIGH]** Verifiser at `calendar_event`-tabellen unioneres med
   item/meeting i `/api/db/calendar`, ellers pull-data ikke vises i UI.
2. **[MED]** Vurder UNIQUE-index på `(user_id, gcal_event_id)` for å
   forhindre zombi-duplikater.
3. **[MED]** Eksplisitt logging hvis `_parse_event_time` returnerer
   None — i dag taper du items uten loud signal.
4. **[LOW]** Cron-vakthund: alert hvis sist vellykket pull >24t siden.

---

# HANDOVER_RESULT — /tasks task-IA-konsolidering (2026-06-18)

Memo §1: «Fire steder med task-konsept — vurder task-IA-konsolidering:
én master-tabell (item antakelig), én lese-projeksjon (/tasks/unified),
én UI per kontekst.»

## TILSTAND: 5 task-flater (4 + duplikat)

| Flate | Route | Komp | Datakilde |
|---|---|---|---|
| Tasks (desktop) | `/tasks` | `routes/Tasks.jsx` (36KB) | `crm_task` |
| Livsplan v1.2 | `/livsplan` | `PageLivsplanV12` | `item` |
| Obs BYGG oppgaver | `/obs-bygg/oppgaver` | `MeetingActionItems.jsx` | `meeting_action_item` |
| Kalender tasks | `/calendar/tasks` | **samme `Tasks.jsx`** | `crm_task` |
| Tasks mobil | `/tasks` (mobil) | `PageTasks.jsx` (100KB **verbatim port**) | `crm_task` |

**Tre parallelle tabeller** med ulike skjemaer + ~100KB duplikat-kode
mellom `Tasks.jsx` og `PageTasks.jsx`.

## KRITISK FUNN: /api/db/tasks/unified eksisterer allerede, men brukes IKKE

`tasks_module.py:192-301` har endepunkt som unioner `crm_task` +
`meeting_action_item` + `reminder` med `source`-felt. Søk etter
`/tasks/unified` i SPA: **null treff**. Backend-projeksjon ligger
klar, frontend forblir fragmentert.

## Lekkasje / inkonsistens

1. **Livsplan ↔ Obs oppgaver: NULL kobling.** Kryss av i /livsplan
   oppdaterer `item.state='done'`, ikke `meeting_action_item`. Samme
   oppgave kan stå som åpen i obs-flaten i evighet.
2. **Tasks ↔ Obs: delvis koblet.** `meeting_action_item.task_id` kan
   peke til `crm_task`, men fylles IKKE konsekvent.
3. **Apple Reminders-sync** kobler kun `crm_task.reminder_id` ↔ Apple.
   Migrasjon vekk fra `crm_task` MÅ flytte sync-logikken.

## Konsolideringsforslag (5 faser, ~10–12 timer)

**Fase 1 (4–5t — anbefalt nå):**
- Erkjenn `item` som master (mest moden, 26 felt, kompleks IA).
- Legg til `item.assigned_to: str | null` (matcher `action_item.assignee`).
- Migrer `meeting_action_item` → `item` med `source='meeting'` +
  `origin_ref=meeting_id`.
- Oppdater `/obs-bygg/oppgaver` til å lese fra `/api/db/items?source=meeting`
  i stedet for `/action-items`.
- Valider at Apple Reminders-sync fortsatt virker.

**Fase 2–5 (senere, ~6–8t):**
- Migrer `crm_task` → `item` med `source='task'`. RISKY pga Apple sync.
- Aktiver unified read-projection i alle UI-er.
- Slett duplikat-komponenter (`PageTasks.jsx` ↔ `Tasks.jsx`).
- Slett legacy-tabeller.

## Risikoer

**Høy:**
- `meeting_action_item.assignee` er string (navn), `item` har ingen
  `assigned_to` — krever migrasjon.
- Apple Reminders-sync er knyttet til `crm_task.reminder_id`. Hvis du
  migrerer uten omhug → mister Apple-sync.

**Moderat:**
- `/livsplan` og `/obs-bygg` kan begge vise samme item hvis det har
  `track='jobb'` + `source='meeting'`. Strikt konvensjon:
  livsplan = privat, obs = jobb.

**Lav:**
- Undertasks: `item.parent_id` finnes, `meeting_action_item` har ikke
  undertasks → no issue.

## Anbefaling

**Ja, konsolider — start med Fase 1 (~4–5t):**
1. `assigned_to` på `item`
2. Migrer `meeting_action_item` → `item` (data-migrering 017_consolidate_tasks.sql)
3. Re-pek frontend `/obs-bygg/oppgaver`
4. Valider Apple Reminders-sync intakt

Deretter (separat sesjon) vurder Fase 2–5 med spesielt fokus på
`crm_task` → `item` (Apple sync er kritisk og bryter lett).
