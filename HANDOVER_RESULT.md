# HANDOVER_RESULT — scan_ingest Fase 0 (recon) (2026-06-29)

**Til:** planlegger-sesjon (claude.ai)
**Fra:** Elmars (VPS-Claude)
**Trigger:** `HANDOVER-SCAN-INGEST-V2.md` (`c3b6696`) — Fase 0+1 forhåndsgodkjent.

## Sammendrag

Fase 0 ferdig. Ett **signifikant skjema-mismatch** mellom handover og virkelighet:
`strength_session.sets` er **jsonb-felt på session-tabellen**, ikke separat
`strength_sets`-tabell som handoveren antar. Påvirker hele Fase 2-design (BE må
PATCHe jsonb, ikke INSERT i raddtabell). Alt annet i Fase 0 forløp uten overraskelser.

🛑 STOP før Fase 1 oppnådd. Starter Fase 1 i samme commit-kø hvis du sier OK.

---

## 1. LiteLLM vision-tilgang ✅

**Tilgjengelige modeller i gateway** (live `GET /v1/models`):
`aurora · aurora-8b · lokal · embed · privat · claude · claude-haiku · gemini · pt-daily · pt-weekly`

**Vision-test mot `claude-haiku`-aliaset** (Claude Haiku 4.5,
`claude-haiku-4-5-20251001`): grønn. Sendte 8×8 PNG via base64 i
`image_url`-content; modellen svarte med beskrivelse av bildet. Status:
`finish_reason="length"` (max_tokens=50 var lavt), tekst: «Jeg ser et lite grått
ikon eller symbol …».

**Anbefaling for `scan-ocr`-alias:**
- Handover §4 spec'er `claude-3-5-sonnet-20241022`. Det er fortsatt en gyldig
  vision-modell og finnes i Anthropic-API.
- Men **claude-haiku 4.5** (allerede i config, nyere) gir ~6× lavere kostnad
  (~$1/M input + $5/M output vs ~$3/M + $15/M) med god håndskrift-kvalitet.
- Forslag: start med `claude-haiku-4-5-20251001` som `scan-ocr` → bytt til
  Sonnet hvis Mayos kvalitetstest viser at Haiku mister detaljer.

**Hva må gjøres for Fase 1:** legg til `scan-ocr`-alias i
`infra/litellm/config.yaml`:
```yaml
  - model_name: "scan-ocr"
    litellm_params:
      model: "claude-haiku-4-5-20251001"   # start her; bytt om kvalitet svikter
      api_key: "os.environ/ANTHROPIC_API_KEY"
```
Restart `mayo-litellm` etter. `pt_llm.py`-mønsteret (alias-baseret routing)
gjenbrukes; ingen ny dep.

---

## 2. strength_session-skjema 🚨 (handover-antakelse FEIL)

Handover §2.1 sier «INSERT-er rader i `strength_sets` koblet til den session».
**`strength_sets`-tabellen finnes ikke.**

Faktisk skjema:
```
Table "public.strength_session"
   Column   |           Type           | Nullable |    Default
------------+--------------------------+----------+---------------
 id         | bigint                   | not null | nextval(...)
 user_id    | text                     | not null |
 ts         | timestamp with time zone | not null |
 name       | text                     | not null | ''
 note       | text                     | not null | ''
 sets       | jsonb                    | not null | '[]'
 created_at | timestamp with time zone | not null | now()
```

- `user_id` er **TEXT** (de andre tabellene bruker UUID — `item.user_id`,
  `note.user_id`, etc). Sannsynligvis legacy. Må håndteres egen i Fase 2.
- `sets` er **jsonb-array**, ikke fremmednøkler til en raddtabell.
- **Ingen** `strength_sets`-tabell, ingen separate set-rader, ingen
  fremmednøkler å validere mot.

**Implikasjon for Fase 2:**
- BE-endepunktet (`POST /scan/ingest/strength`) må **PATCHe**
  `strength_session.sets`-jsonb (append eller replace), ikke INSERT i en tabell.
- Reversibilitet: jsonb-PATCH er litt mindre granulært, men en undo-strategi
  kan beholde gammel `sets`-versjon (last-write-wins eller versjonsstemplet).
- Spør Mayo: «Skal skannede sett **erstatte** eller **legges til** eksisterende
  `sets`-array på den valgte sessionen?» Default-antakelse: append (han logger
  flere sett underveis).

`session_insight` (annen tabell) refererer til `strength_session(id)` med
fremmednøkkel — den må ikke brytes ved PATCH.

---

## 3. Epley 1RM-util — kun i FE

**Backend:** ingen Epley-util. Hverken i `db_api/`, `modules/health/`, eller
PT-laget. Strength-modulen er i hovedsak FE.

**Frontend:**
- `src/lib/strength.js:13` — `export const epley = (weight, reps) => weight * (1 + reps / 30);`
- `src/mobile/pages/PageStyrke.jsx:152` — `const _epley = (weight, reps) => (weight || 0) * (1 + (reps || 0) / 30);`

**Implikasjon for Fase 2:** OCR → parsed sets (kg + reps) lagres som de er;
Epley-beregning skjer fortsatt FE-side ved fremstilling. Ingen BE-Epley nødvendig.

Note: `src/lib/strength.js:620` har en kommentar «dropp e1RM, vis faktisk
maksløft». Dette indikerer at Mayo har bevisst nedprioritert Epley i UI — vi
trenger ikke aktivere flere e1RM-paths i skann-flyten.

---

## 4. reminders — modul finnes ✅

- **Tabell** `reminder`: 18 kolonner inkl. `title`, `description`, `due_at`,
  `tags[]`, `priority`, `list_name`, `completed`. iCloud-synket
  (`icloud_uid`, `icloud_etag`, `last_synced_at`).
- **user_id**: uuid (konsistent med item/note).
- **Endepunkter** i `db_api/reminders_module.py`:
  - `GET /reminders`
  - `GET /reminders/lists`
  - `POST /reminders` (linje 134 — sannsynligvis det Fase 4 trenger)
  - `POST /reminders/sync`
  - `POST /reminders/bulk-sync`

**Implikasjon for Fase 4:** `POST /reminders` finnes; kan kalles direkte fra
scan_ingest med parsed linje + dato + område. Påminnelser-modulen kjører i
produksjon, ingen ny scope-utvidelse trengs.

**FE-flate**: Reminders er allerede i Apple Reminders + sync til `item`-tabell
(via `item.source='reminder'`). Det finnes ikke en dedikert «PageReminders» —
de vises i `/tasks`/Livsplan via unified-feeden. Fase 4 må enten:
- (a) bygge ny `PageReminders` for skann-knappen, eller
- (b) legge skann-knappen i kalender-/dato-arket der reminder opprettes idag.
Anbefaling: **spør Mayo i Fase 4-start** hvilken han foretrekker.

---

## 5. Vault-skrivesti — gjenbruk eksisterende

`note_module.py::_write_to_vault` (linje ~81):
```python
NOTES_DIR.mkdir(parents=True, exist_ok=True)
path = NOTES_DIR / f"{note_date}.md"
# header = "# {title}\n\n" hvis tittel finnes
path.write_text(header + body_md, encoding="utf-8")
```

`NOTES_DIR` = `MayoVault/notater/`. Idempotent overskriving. Filer ligger med
`<note_date>.md`-navn (siste fil sett: `2026-06-26.md`).

**Implikasjon for Fase 3 (Kladd-skann):** når Mayo bekrefter preview → kall
eksisterende `POST /notes` med `body_md` = parsed OCR-tekst. `note_module`
tar seg av vault-skriving automatisk via `_write_to_vault`. **Ingen ny vault-
skrivesti trengs.**

**`MayoVault/skann/`** for ad-hoc Shortcut-stien (Fase 5): kan opprettes ved
behov, men anbefal Mayo skipper det og lar alt ligge i `notater/` med en
`#skann`-tag i front-matter (enklere søk).

---

## 6. PageStyrke.jsx — fil-struktur ✅

- Fil: `src/mobile/pages/PageStyrke.jsx`
- Routes: `src/routes/strength/Strength.jsx` (desktop) + `src/lib/strength.js` (util)
- PageStyrke er ~2700 linjer (stor). Topp-handlingsrad må **identifiseres
  presist** i Fase 2 — den er nede i komponent-treet, ikke åpenbart eksponert i
  topp-grepet jeg gjorde.
- Anbefal: når Fase 2 starter, naviger PageStyrke i nettleser først,
  identifiser visuelt hvor «Ny økt»/«Historikk»-knappene sitter, så match til
  kode med en mer presis grep eller `git blame`.

---

## Anbefalinger for Fase 1

1. **Module-layout:** lag `modules/scan_ingest/` med:
   - `__init__.py`
   - `ocr_engine.py` — `async def transcribe(image_bytes: bytes, *, hint: str = "") -> str`
   - `cli.py` — `python -m scan_ingest.ocr <bilde-sti>`
2. **Engine-impl:** kall LiteLLM-gateway `scan-ocr`-alias med
   `image_url`/`base64`-content. Hint sendes som system-prompt-suffix.
3. **CLI:** ta filsti(er) som arg, print rå transkripsjon for hver. Mayo
   trenger denne for å sende 3-5 ark og dømme kvalitet.
4. **Ingen DB-skriving i Fase 1.** Bare OCR-motor + CLI.

**STOP-gate (handover §7):** etter Fase 1 må Mayo kjøre CLI mot test-ark og
godkjenne tyde-kvalitet før Fase 2.

## Konkret Fase 1-plan

a. Legg til `scan-ocr`-alias i `infra/litellm/config.yaml` (Claude Haiku 4.5).
b. `docker restart mayo-litellm` + curl-smoke-test mot vision-aliaset.
c. Lag `modules/scan_ingest/` med `ocr_engine.py` + `cli.py`.
d. Skriv en kort `README.md` med CLI-bruksanvisning.
e. Commit + push + STATE.

Avventer Mayos signal før commit av §1 (Fase 1) hvis preferansen er å se
recon-funnene først; ellers kan jeg fortsette direkte siden Mayo allerede
forhåndsgodkjente Fase 1.

— Elmars 2026-06-29
