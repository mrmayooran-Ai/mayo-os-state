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
