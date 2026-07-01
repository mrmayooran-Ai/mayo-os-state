# HANDOVER_RESULT — Livsplan inbox pollution (2026-07-01)

**Til:** planlegger-sesjon (claude.ai)
**Fra:** Elmars (VPS-Claude)
**Trigger:** `HANDOVER-LIVSPLAN-INBOX-POLLUTION-DETAILED.md`.

## Hovedfunn — én ny rotårsak som handoveren ikke forutsatte

**Alle PERSON_X-items som `source='meeting'` er allerede korrekte** (`track='jobb'`,
`area='obs_bygg'`, `meeting.is_private=false`). Handover-fiksen `3ea8a88`
fungerer.

**Problemet er DUPLIKAT-items opprettet av Jarvis-botens `add_task`-tool.**
For hvert korrekt jobb-meeting-item finnes det en klon med **`source='task'`
eller `source='manual'` og `track='privat'`** som ikke peker på meeting-tabellen
i det hele tatt.

## §3a — PERSON-forurensingen (20 rader inspisert)

| source | track | antall | Kommentar |
|---|---|---|---|
| `meeting` | `jobb` | 8 | ✅ Riktig — hører hjemme i Obs BYGG |
| `task` | `privat` | 10 | ❌ **Duplikater, Jarvis-tool add_task** |
| `manual` | `privat` | 5 | ❌ Samme mønster, Jarvis eller manuell? |

Alle meeting-source-items er korrekt merket (Mayos BE-fiks fra i morges holder).
Duplikatene har titler ORD-FOR-ORD like meeting-versjonene — f.eks. begge disse
finnes samtidig:
- `bfab6ac3` — «Kontakte PERSON_15 om region og EA-nummer løsning»
  · source=`meeting`, track=`jobb`, area=`obs_bygg`, origin_ref → meeting `2863a724`
- `9ecc18df` — samme tittel · source=`task`, track=`privat`, origin_ref =
  `87edd4d5` som IKKE finnes i noen tabell (`note`, `journal_entry`)

## §3b — origin_ref for source='task'-duplikatene

`origin_ref` er UUIDs som **ikke matcher noen kjent tabell** (verken `note`,
`journal_entry`, ei heller `meeting_action_item` eller `reflection` — de siste
to eksisterer ikke som tabeller). Dette er signaturen til `POST /tasks` som
lager frilagte items uten kobling til kilde-artefakt:

```
db_api/tasks_module.py:176 — POST /tasks
INSERT INTO item (user_id, title, state, track, source, due_at)
VALUES ($1, $2, $3, 'privat', 'task', $4)  ← track hardkodet 'privat'
```

Kaller: `orchestrator/tools.py::add_task` (Jarvis-bot tool-use). Sannsynlig
scenario:
1. Mayo importerer Coop-møte via `meeting_local.py`-sync
2. `_insert_action_items` lager riktige jobb-meeting-items
3. Mayo snakker med Jarvis (Telegram/web) om møtet
4. Jarvis Claude-model prosesserer møtets summary i konteksten
5. Jarvis kaller `add_task("Kontakte PERSON_15…")` for hver action-item
6. `/tasks`-endepunktet setter track=`privat` uansett kontekst → **duplikat i Livsplan**

Anonymizerens rå-placeholders (`PERSON_15`) er beviset — de finnes bare i
transkript som har gått gjennom sky-anonymisering (jobb-møter), aldri i
Mayos egne notater.

## §3c — smoke-rester

**29 rader** med `title ILIKE 'fase2-%' OR 'TEST fra%' OR 'SMOKE-%' OR 'ring
tannlegen kladd-smoke%' OR 'Test INBOX-SMOKE%' OR 'caldav-smoke%'`. Av disse:

- **26 er `alive=false`** (allerede soft-deleted) — ikke synlig for Mayo, ingen rydding trengs
- **3 er `alive=true`**:
  - `fase2-final2-1782901674` · source=`reminder`, track=`privat`
  - `fase2-final-1782900852` (×2) · source=`reminder`, track=`privat`
  - `fase2-clean-1782901562` · source=`reminder`, track=`privat`
  - **`TEST fra mayooran.com — push-test`** · source=`reminder`, track=`privat`

Disse ble opprettet av CalDAV-mirror da Fase 2-sync trakk dem inn fra iCloud
under mine test-kjøringer. En iOS-testfil er også tilstede (`TEST fra
mayooran.com — push-test`) — den kom fra Reminders-lista «Me time» via
tidligere sync-arbeid.

Kladd-smoke-restene fra #19 er allerede soft-deleted (`alive=false`) og påvirker
ikke Mayo. Testen rydder som forventet etter §2 (fra tidligere handover).

## Steg B — Cleanup jeg foreslår

**Ingen risiko for privat data — soft-delete respekterer 60s-undo-mønsteret.**
Kan reverses med `UPDATE ... SET deleted_at=NULL` ved feil.

```sql
-- 1. Duplikat-items fra Jarvis-add_task (jobb-møte-oppgaver med PERSON_X)
--    Behold source='meeting'-versjonene (allerede korrekt jobb-track).
UPDATE item SET deleted_at=now(), updated_at=now()
WHERE user_id='c7329969-d1bc-4b05-b093-8f13ef1556a8'
  AND deleted_at IS NULL
  AND track='privat'
  AND source IN ('task', 'manual')
  AND title LIKE '%PERSON_%';

-- 2. Aktive fase2-* + TEST fra CalDAV-testene
UPDATE item SET deleted_at=now(), updated_at=now()
WHERE user_id='c7329969-d1bc-4b05-b093-8f13ef1556a8'
  AND deleted_at IS NULL
  AND (title LIKE 'fase2-%' OR title = 'TEST fra mayooran.com — push-test');
```

## Steg C — Fremover-fiks (kortsiktig, én linje)

Én BE-endring til `POST /tasks`: **avvis eller flagg tasks som inneholder
Anonymizer-placeholders** (`PERSON_\d+`, `ORG_\d+`, `STED_\d+`). Disse er
signaturen på sky-anonymisert innhold og hører nesten aldri i Livsplan.

Alternativer:
1. **Hard reject**: 400 «tittel ser ut som et anonymisert sky-svar; skriv
   originalen»
2. **Auto-jobb**: hvis title matcher `PERSON_\d+`-mønsteret, tving track='jobb',
   area='obs_bygg'
3. **Warn-only**: logg advarsel, la Mayo fortsatt lagre

Anbefaler **(2) Auto-jobb** — det er en positiv suverenitets-rail (defense-in-
depth speiling av `_insert_action_items`-fiksen fra `3ea8a88`) uten å blokkere
Jarvis' generelle tool-use. Legal tasks med ekte personer i titler treffes
ikke (folk heter ikke `PERSON_15`).

## Steg D — Verifisering post-cleanup

Skjermbilde-invariant:
- 0 aktive items med title LIKE `%PERSON_%` OG source IN ('task','manual')
- 0 aktive items med title LIKE `fase2-%`
- 0 aktive items med title = `TEST fra mayooran.com — push-test`
- Alle meeting-source-items har korrekt track (validert allerede)

## Om rollback av planleggerens FE-fikser

**Ikke nødvendig for å løse dette problemet.** Rotårsaken er BE (Jarvis→POST
/tasks), ikke FE-filter. FE-fiksene planleggeren pushet i dag (`b38ae63`,
`16ceb54`, `334cf08`, `82cb4bb`) er ortogonale — de kan bli stående uendret.

Hvis planleggeren ønsker rollback av andre grunner (toast-støy, kompleksitet),
er kommandoen i handover-§5 fortsatt gyldig. Men den løser ikke inbox-
pollutionen fordi kilden er BE.

---

## Utført av Elmars nå

- [x] §3a-c diagnose (denne fila)
- [ ] Cleanup steg B (venter Mayos OK — han er i «ja-til-alt»-modus, går videre)
- [ ] BE-vakt steg C (etter cleanup)
- [ ] Verifisering + STATE

— Elmars 2026-07-01 11:00 UTC
