# STATE.md — Mayo OS levende tilstand

> **Dette er sannheten.** Elmars oppdaterer denne som SISTE handling i hver oppgave.
> Planleggeren (claude.ai) leser denne FØRST i hver økt, via public speil `mayo-os-state`.
> Aldri secrets/PII her — kun `<SET>`-markører.

**Sist oppdatert:** 2026-06-06 · **Av:** Elmars · **Versjon:** v0.5 Aurora

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
| Styrkelogg (`/strength`) | 🟢 | 2026-06-06 | v3.1: Mayos EKTE øvelsesbibliotek (17 øvelser) + baselines + PPL×2 + **progresjonsmotor** (dobbel progresjon, justeringsregel, stagnasjonsflagg, «sist:»-tall, coaching-banner pr øvelse). Beholder v3-funksjoner. |
| Regelbok-sjekk i app | 🟢 | 2026-06-06 | «Sjekk økt mot regelboka» på /strength I dag → /training?action=evaluate (ekte gating+fase). |
| PT øktvalg-regelbok | 🟢 | 2026-06-06 | **v3.1 forenklet:** markløft progresjerer fritt (ingen Q4-gate, ingen langløp-sperre). `okt_logikk.evaluate_request`, 72 grønne tester. Live: `/training?action=evaluate`. |
| Public state-mirror | 🟢 | 2026-06-05 | `mayo-os-state` (public) · raw-URL 200 · planleggeren leser den |

## 🟡 Pågår / delvis

## 🔴 Åpne problemer
Kjente feil som blokkerer eller irriterer. Med dato oppdaget.

- **E-post (gmail send/compose) nede** (2026-06-02) — OAuth-token døde; gjenopprettes med Mayos ENE consent-klikk (4-scope re-auth). Kalender funker uavhengig.

## 📋 Backlog (prioritert)
- Enable Banking / finance-modul (6-fase plan klar)
- Lokal modell-oppgradering 3b → 14b/32b
- Obsidian-class markdown-editor i mayooran.com
- Auto-enrichment pipeline (silent tagging, psykolog-refleksjon)

## 🕐 Siste commits
Nyeste øverst. Format: `hash — beskrivelse (dato)`

**Backend (`mayo-ai-os`):**
- `25472a4` — PT regelbok v3.1: markløft fri (ingen Q4/langløp), 72 tester (2026-06-06)
- `4134a9a` — PPL×2-seed med Mayos ekte øvelser (2026-06-06)
- `6852e7d` — strength_routine CRUD + seed + strava_title (2026-06-05)
- `ee5e40f` — egne øvelser (strength_exercise) + PATCH /strength/sessions (2026-06-05)
- `d4b41dd` — PT øktvalg-regelbok (Del D pull-skille + Del E fase-gate + Del F T1–T6, 71 tester) (2026-06-05)
- `da325ef` — `strength_session` ekte logging (JSONB) (2026-06-05)
- `42806c4` — `/training?action=coach` (PT-beslutning fra pt_logg) (2026-06-05)
- `f2bcd98` — Google Calendar OAuth re-auth + PT→kalender + gmail-scope (2026-06-05)
- `e991dec`/`e430b98` — OpenClaw read-only recon-rapport (2026-06-05)

**Frontend (`mayo-os`):**
- `cb36d32` — progresjonsmotor v3.1 (dobbel progresjon + sist-tall + stagnasjon) (2026-06-06)
- `aad8fc0` — ekte øvelsesbibliotek + baselines + PPL×2 (PT v3.1) (2026-06-06)
- `2bc7067` — regelbok-sjekk i /strength I dag (Q4/RDL-verdikt synlig) (2026-06-06)
- `99d2870` — Program full-redigering + DB-persistens + Strava-tittel (2026-06-05)
- `7404db6` — styrkelogg v2: ekte recovery/anbefaling/uke + editering + egne øvelser (2026-06-05)
- `115cc7c` — styrkelogg ekte logging (DB) (2026-06-05)
- `a43c08a` — Helse → Program-fane (ekte PT-coach) (2026-06-05)
- `adda625` — full styrkelogg-port (I dag/Stats/Program) (2026-06-05)

## 📝 Til planleggeren (claude.ai)
Beskjeder fra Elmars til claude.ai som påvirker neste planlegging.

- **Styrkelogg (PT v3.1):** øvelsesbibliotek + PPL×2 = Mayos faktiske senter. **Progresjonsmotor** live: når Mayo loggfører vekt·reps·RIR, anbefaler appen neste økt (dobbel progresjon, +2.5/+5kg, stagnasjon→deload) med «sist:»-tall pr øvelse. Recovery/uke = ekte (Whoop+Strava). Gjenstår: daglig Claude-lag (seksjon 6, inc 4) — **UTSATT av Mayo 06.06** («vent, test inc 1-3 først»). Berører gjentakende Telegram-send; Mayo deaktiverte gamle morgen-brief 04.06. Bygges ikke før Mayo velger leveringskanal.
- **Regelboka (øktvalg) v3.1** — forenklet: ÉN kontinuerlig hypertrofi/styrke-fase, markløft progresjerer fritt (dobbel progresjon, ingen Q4-gate, ingen langløp-interferens). Testet (72 grønne) + live: `GET /training?action=evaluate&q=<forespørsel>`. GRØNN→full tung 4×6 · GUL→behold øvelse −volum/RIR · RØD→hvile.
- **Gmail re-auth** venter på Mayos consent-klikk.
- **Public state-mirror (`mayo-os-state`):** 🟢 live — les STATE.md på `raw.githubusercontent.com/mrmayooran-Ai/mayo-os-state/main/STATE.md`.
