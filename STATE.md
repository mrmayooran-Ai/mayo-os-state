# STATE.md — Mayo OS levende tilstand

> **Dette er sannheten.** Elmars oppdaterer denne som SISTE handling i hver oppgave.
> Planleggeren (claude.ai) leser denne FØRST i hver økt, via public speil `mayo-os-state`.
> Aldri secrets/PII her — kun `<SET>`-markører.

**Sist oppdatert:** 2026-06-05 · **Av:** Elmars · **Versjon:** v0.5 Aurora

---

## 🟢 Live & verifisert
Tjenester som kjører og er bekreftet fungerende (med dato for siste verifisering).

| Komponent | Status | Sist verifisert | Notat |
|---|---|---|---|
| frontend (nginx 8086) | 🟢 | 2026-06-05 | mayooran.com · build 115cc7c |
| db-api (8001) | 🟢 | 2026-06-05 | |
| Whoop-integrasjon | 🟢 | 2026-05-20 | direct-fetch, token `<SET>` |
| Strava-integrasjon | 🟢 | 2026-05-20 | direct-fetch, token `<SET>` |
| Telegram-bot | 🟢 | 2026-05-25 | Postgres chat_history |
| LiteLLM-gateway (4000) | 🟢 | 2026-06-04 | |
| Google Calendar (skriv) | 🟢 | 2026-06-05 | OAuth re-auth: write-scope + `/calendar-auth`-callback (db-api) + ny refresh-token. PT→kalender aktiv. |
| Styrkelogg (`/strength`) | 🟢 | 2026-06-05 | Ekte logging → `strength_session`-tabell (JSONB-sett). Faner: I dag / Stats / Program. |
| PT øktvalg-regelbok | 🟢 | 2026-06-05 | `okt_logikk.evaluate_request` (Del D/E/F), 71 grønne tester. Live: `/training?action=evaluate`. |

## 🟡 Pågår / delvis
- **#2 Ekte recovery + «uka så langt» i `/strength` «I dag»** — gjenstår. Skjermen viser fortsatt mock (51 %/uke). Skal kobles til `/whoop` + `/strava`. Ekte recovery vises allerede riktig på `/health → Program`.
- **Regelbok → app-anbefaling** — `evaluate_request` er live som endepunkt, men ikke koblet inn i `/strength` sitt anbefaling-kort ennå.

## 🔴 Åpne problemer
Kjente feil som blokkerer eller irriterer. Med dato oppdaget.

- **E-post (gmail send/compose) nede** (2026-06-02) — OAuth-token døde; gjenopprettes med Mayos ENE consent-klikk (4-scope re-auth). Kalender funker uavhengig.
- **`/strength` «I dag» mock-data** (2026-06-05) — recovery/anbefaling/uke er mock til #2 er gjort.

## 📋 Backlog (prioritert)
- **#2:** ekte recovery/uke + regelbok-anbefaling inn i `/strength`
- Enable Banking / finance-modul (6-fase plan klar)
- Lokal modell-oppgradering 3b → 14b/32b
- Obsidian-class markdown-editor i mayooran.com
- Auto-enrichment pipeline (silent tagging, psykolog-refleksjon)

## 🕐 Siste commits
Nyeste øverst. Format: `hash — beskrivelse (dato)`

**Backend (`mayo-ai-os`):**
- `d4b41dd` — PT øktvalg-regelbok (Del D pull-skille + Del E fase-gate + Del F T1–T6, 71 tester) (2026-06-05)
- `da325ef` — `strength_session` ekte logging (JSONB) (2026-06-05)
- `42806c4` — `/training?action=coach` (PT-beslutning fra pt_logg) (2026-06-05)
- `f2bcd98` — Google Calendar OAuth re-auth + PT→kalender + gmail-scope (2026-06-05)
- `e991dec`/`e430b98` — OpenClaw read-only recon-rapport (2026-06-05)

**Frontend (`mayo-os`):**
- `115cc7c` — styrkelogg ekte logging (DB) (2026-06-05)
- `a43c08a` — Helse → Program-fane (ekte PT-coach) (2026-06-05)
- `adda625` — full styrkelogg-port (I dag/Stats/Program) (2026-06-05)

## 📝 Til planleggeren (claude.ai)
Beskjeder fra Elmars til claude.ai som påvirker neste planlegging.

- **Styrkelogg:** logging er EKTE (Postgres). Stats/PR/volum bygges på loggførte økter (tomt til Mayo logger). Recovery/uke/anbefaling i `/strength` er fortsatt MOCK (#2 gjenstår). Ekte recovery: `/health → Program`.
- **Regelboka (øktvalg)** er implementert deterministisk + testet (71 grønne) + live: `GET /training?action=evaluate&q=<forespørsel>` bruker dagens ekte gating. Tung markløft nedgraderes til RDL 4×6 RIR3 utenfor Q4-peak-fasen.
- **Gmail re-auth** venter på Mayos consent-klikk.
- **Public state-mirror (`mayo-os-state`):** PENDING — venter på Mayos godkjenning av publisering (steg 3 i setup-flyt).
