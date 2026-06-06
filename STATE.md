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
| PT øktvalg-regelbok | 🟢 | 2026-06-06 | **v3.1 forenklet:** markløft fritt + **søvn-gating relaksert** (6-7t nedgraderer ikke grønn dag; <6t = eneste søvn-terskel, §4.1). `okt_logikk`+`gating`, **88 grønne**. Pull/Push/Bein/markløft svarer alle (gating-nivå). **Frekvens-vakt** <48t (markløft + push/bein-gruppe, fra styrkeloggen) → AVVIS. Live: `/training?action=evaluate`. |
| PT LLM-lag (inc 4) | 🟢 | 2026-06-06 | Daglig motor-kort (PPL×2 + progresjon) + anonymisert LLM-kommentar live på `/strength` + `/training?action=daily`. Kjører på **gratis Gemini 2.5 Flash** (pt-daily) m/ fallback pt-weekly→claude-haiku→motor. **Telegram:** daglig (morgenrapport, Gemini, 08:00) + **ukentlig analyse** (søndag 20:00 Telegram + **i Stats-fanen** via /training?action=weekly, cachet, pt-weekly/Claude, hopper over hvis 0 økter). |
| Public state-mirror | 🟢 | 2026-06-05 | `mayo-os-state` (public) · raw-URL 200 · planleggeren leser den |

## 🟡 Pågår / delvis

## 🔴 Åpne problemer
Kjente feil som blokkerer eller irriterer. Med dato oppdaget.

- **E-post (gmail send/compose) nede** (2026-06-02) — OAuth-token døde; gjenopprettes med Mayos ENE consent-klikk (4-scope re-auth). Kalender funker uavhengig.

## 📋 Backlog (prioritert)
- **Lokal 14b-oppgradering — IKKE trygt mulig nå** (recon 06.06): VPS 4.5Gi ledig (14b≈9GB), M1 sover uforutsigbart. Krever M1 always-on (launchd keep-alive) for 14b-default — Mayos avgjørelse. Dagens 3b-VPS + 14b-M1(privat, eksplisitt) + sky-fallback er hw-riktig.
- Enable Banking / finance-modul (6-fase plan klar)
- Lokal modell-oppgradering 3b → 14b/32b
- Obsidian-class markdown-editor i mayooran.com
- Auto-enrichment pipeline (silent tagging, psykolog-refleksjon)

## 🕐 Siste commits
Nyeste øverst. Format: `hash — beskrivelse (dato)`

**Backend (`mayo-ai-os`):**
- `f880657` — phases.py/decide.py → v3.1 single-phase (Q4/Race/1RM-test fjernet) (2026-06-06)
- `e6e2d86` — /training?action=weekly (cachet ukesanalyse) (2026-06-06)
- `1f57467` — ukentlig PT-analyse (søndag-cron, Claude) (2026-06-06)
- `9a41691` — Push/Bein i regelboka + frekvens-vakt generalisert (2026-06-06)
- `b228d9e` — morgenrapportens PT-blokk → v3.1 daglig-kort + Gemini (inc 4 cron) (2026-06-06)
- `3382e8c` — inc 4: daglig motor-kort + LLM-lag (anonymisert) + /training?action=daily (2026-06-06)
- `057c782` — markløft frekvens-vakt FØR gating (Vei C styrkelogg) (2026-06-06)
- `bd3fc77` — relaks søvn-gating v3.1 §4.1 (6-7t holder grønt) (2026-06-06)
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
- `83b65fa` — gull-MM-logo + app-ikoner (login-orb + header + favicon) (2026-06-06)
- `79e48ac` — markdown-editor: live preview + wikilinks (punkt 2) (2026-06-06)
- `7c23246` — reelle beste tall i stats (dropp est. 1RM/Epley) (2026-06-06)
- `fa943e8` — ukentlig PT-analyse i Stats-fanen (2026-06-06)
- `34c0da2` — vis daglig motor-kort + LLM-kommentar i I dag (erstatter hip-thrust-wart) (2026-06-06)
- `dcc6d74` — ekte dato i datostripe + PPL×2-ukeplan (ikke mock) (2026-06-06)
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
- **Regelboka (øktvalg) v3.1** — forenklet: ÉN kontinuerlig hypertrofi/styrke-fase, markløft progresjerer fritt (dobbel progresjon, ingen Q4-gate, ingen langløp-interferens). Testet (72 grønne) + live: `GET /training?action=evaluate&q=<forespørsel>`. GRØNN→full tung 4×6 · GUL→−volum/RIR · RØD→hvile. **Frekvens-vakt:** markløft <48t siden (fra styrkeloggen) → AVVIS uansett farge (erector ikke restituert). I dag: markløft trent 14t siden → AVVIST.
- **decide.py/phases.py ryddet (06.06):** Q4/Race/1RM-test-blueprint fjernet → én kontinuerlig hypertrofi/styrke-fase (v3.1 §4.2). ROTATION-øvelseslista (hip thrust o.l.) er fortsatt der men DORMANT — erstattet av v3.1 daglig-kort både i app (`/strength`) og Telegram (send_report). Kjører fortsatt for gating-nivået + skriver pt_logg-narrativ (ikke brukervendt). Opprinnelig: (f.eks. «hip thrust», «leg curl») som IKKE er i v3.1-biblioteket. FORTSATT i Telegram-helsebrief (send_report.py). I appen er den ERSTATTET av inc 4-kortet (daily). Resten av appen (Program/logger/progresjon/regelbok) er v3.1-korrekt. Increment 4 bygger om decide.py → PPL×2 + Mayos bibliotek (UTSATT av Mayo 06.06).
- **⏳ Mayo må gjøre (for å fullføre inc 4 LLM):** (1) legg `GEMINI_API_KEY` i `infra/litellm/litellm.env` (gratis fra Google AI Studio) → daglig brief blir gratis i stedet for claude-haiku. (2) restart litellm (`mayo-litellm.service`, Docker — Elmars mangler NOPASSWD) for å laste `pt-daily`/`pt-weekly`-aliasene. Inntil da kjører alt på claude-haiku (virker fint, bare ikke gratis).
- **⚠️ db-api restart-lærdom:** db-api bruker ~15s å boote (NB-Whisper). Restart KUN via `sudo -n systemctl restart db-api` ÉN gang + poll til oppe. Rask gjentatt restart = boot-overlapp → krasj-loop. `stop`/`start`/`reset-failed` er IKKE NOPASSWD (kun `restart`).
- **Gmail re-auth** venter på Mayos consent-klikk.
- **Public state-mirror (`mayo-os-state`):** 🟢 live — les STATE.md på `raw.githubusercontent.com/mrmayooran-Ai/mayo-os-state/main/STATE.md`.
