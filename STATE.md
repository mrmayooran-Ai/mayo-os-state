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
| frontend (nginx 8086) | 🟢 | 2026-06-06 | mayooran.com · build 4e126df |
| db-api (8001) | 🟢 | 2026-06-05 | |
| Whoop-integrasjon | 🟢 | 2026-05-20 | direct-fetch, token `<SET>` |
| Strava-integrasjon | 🟢 | 2026-06-06 | direct-fetch, token `<SET>`. **Kalori-berikelse:** detalj-endpoint → Postgres-cache (`strava_activity_cache`) + hourly backfill-cron (rate-limited, 429-trygg). |
| Telegram-bot | 🟢 | 2026-05-25 | Postgres chat_history |
| LiteLLM-gateway (4000) | 🟢 | 2026-06-04 | |
| Google Calendar (skriv) | 🟢 | 2026-06-05 | OAuth re-auth: write-scope + `/calendar-auth`-callback (db-api) + ny refresh-token. PT→kalender aktiv. |
| Styrkelogg (`/strength`) | 🟢 | 2026-06-06 | v3.1: Mayos EKTE øvelsesbibliotek (17 øvelser) + baselines + PPL×2 + **progresjonsmotor** (dobbel progresjon, justeringsregel, stagnasjonsflagg, «sist:»-tall, coaching-banner pr øvelse). **I-dag UX-batch (06.06):** RecoveryCard redesignet (1-linje HRV ▲/▼ vs 30d-baseline, søvn-pil vs i går + måneds-snitt, dyp/REM i t:m fra /api/whoop); anbefaling klikkbar (plan+grunn) + **recency-fiks** (ben/RDL=ben, anbefaler mest uthvilte gruppe fra loggen — ikke Strava-tittel); «Valgfritt»-knapp m/ frekvens-fargede øvelser; klikkbare uke-økter → økt-stats. |
| Health → Logg (`/health`) | 🟢 | 2026-06-06 | Periode-stats-flis 4→6 (Økter, Tid, Distanse, Kalorier, Snitt puls, Maks puls). **3mnd/YTD-databug fikset:** frontend hentet /strava uten `days` → backend 90d-default → YTD (~157d) undertalte; nå `?days=400`. Snitt puls kun over økter m/ pulsdata. **Kalorier nå ekte** (Strava detalj-endpoint → cache + backfill-cron); «—» kun til cachen er fylt. |
| Assistent (Jarvis) (`/assistent`) | 🟢 | 2026-06-06 | **JARVIS-skin** (reaktor-orb + holo-HUD). **Agentisk persona** (proaktiv multi-ekspert: coach/formue/IVF/helse/psykolog/tech) som HANDLER via verktøy, ikke bare råder. **30 verktøy** inkl. web_search (DuckDuckGo, nøkkelfri) + get_stock_price/get_stock_history (Yahoo, inline cyan-graf) + get_daily_workout/check_training_rule (PT-motor) + kalender/oppgaver-skriv. **Anti-fabrikasjon-guardrail.** Kjører Claude (Gemini/Lokal-velger = inc 2). |
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
- **Jarvis-bygg (pågår):** ✅ inc 0 JARVIS-skin + agentisk multi-ekspert-persona + 30 verktøy (web/aksje/graf/PT-motor/kalender-handlinger) + anti-fabrikasjon-guardrail er **live**. Gjenstår (spec `handoff/JARVIS_v1_DESIGN.md`, plan `docs/superpowers/plans/2026-06-06-jarvis-v1.md`): **inc 1** minne (profil-MD + DB-historikk = «aldri glemme»), **inc 2** Gemini/Claude/Lokal-velger + C-knapp + llm_router-omskriving (🔴→lokal-VPS), **inc 3** anonymiser-round-trip, **inc 4** auto-rutede skills, **inc 5** RAG. Suverenitet bestemt: A default (anonymiser→sky) + C-knapp; M1 kun Obs BYGG. Neste: minne (inc 1) + log_workout-verktøy.

## 🕐 Siste commits
Nyeste øverst. Format: `hash — beskrivelse (dato)`

**Backend (`mayo-ai-os`):**
- `a815111` — agentisk Jarvis-persona + PT-motor-verktøy (get_daily_workout/check_training_rule), 30 verktøy (2026-06-06)
- `852c835` — get_stock_history + chart-event i strøm-loop (graf-plotting) (2026-06-06)
- `debabee` — nøkkelfri web_search (DuckDuckGo) + get_stock_price (Yahoo) (2026-06-06)
- `dded011` — web_search (Tavily) + anti-fabrikasjon-guardrail (2026-06-06)
- `1ab35c9` — Strava kalori-berikelse (detalj-endpoint → cache + rate-limited backfill-cron) (2026-06-06)
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
- `9cb0eaf` — inline JARVIS-graf (chart-SSE-event → cyan linje-graf) (2026-06-06)
- `f4a2066` — fiks dobbel «analyserer»-indikator (2026-06-06)
- `09e5813` — JARVIS-skin: reaktor-orb + holo-HUD (Jarvis inc 0) (2026-06-06)
- `4fdc640` — Health/Logg periode-stats (kcal+maks puls) + fiks 3mnd/YTD-data (E) (2026-06-06)
- `c6b370d` — RecoveryCard redesign: HRV-trend + søvn-piler + dyp/REM (A) (2026-06-06)
- `5c3e43c` — klikkbare uke-økter → stats (D) (2026-06-06)
- `ab4ee7d` — «Valgfritt» + frekvens-fargede øvelser (C) (2026-06-06)
- `419a2b6` — anbefaling klikkbar (plan+grunn) + recency-fiks ben/RDL (B) (2026-06-06)
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
- **✅ inc 4 LLM ferdig (06.06):** `GEMINI_API_KEY` lagt inn + litellm reloadet → daglig brief kjører gratis på Gemini 2.5 Flash (pt-daily), m/ fallback pt-weekly→claude-haiku→motor. Ukentlig analyse på pt-weekly (Claude).
- **⚠️ db-api restart-lærdom:** db-api bruker ~15s å boote (NB-Whisper). Restart KUN via `sudo -n systemctl restart db-api` ÉN gang + poll til oppe. Rask gjentatt restart = boot-overlapp → krasj-loop. `stop`/`start`/`reset-failed` er IKKE NOPASSWD (kun `restart`).
- **Gmail re-auth** venter på Mayos consent-klikk.
- **Public state-mirror (`mayo-os-state`):** 🟢 live — les STATE.md på `raw.githubusercontent.com/mrmayooran-Ai/mayo-os-state/main/STATE.md`.
- **I-dag/Logg UX-batch ferdig (06.06):** RecoveryCard (HRV-trend/søvn-piler/dyp-REM), klikkbar anbefaling + **recency-fiks** (motor/LLM ser nå styrkeloggen, ikke Strava-tittelen — fikser «ben i dag når jeg trente ben i går»), «Valgfritt»-velger, klikkbare uke-økter, Logg periode-stats + **3mnd/YTD-databug fikset** (days=400). kcal: **berikes nå** fra Strava detalj-endpoint → Postgres-cache + hourly backfill-cron (rate-limited, 429-trygg). 95/317 cachet umiddelbart (alle Mayos periode-vinduer dekket); resten fylles av cron. Verifisert 30d/90d/YTD = alle aktiviteter har kalorier.
- **Jarvis-bygg (natt 06.06):** assistenten er nå **agentisk** — JARVIS-skin + multi-ekspert-persona som HANDLER (kalender/oppgaver/trening) + analyserer på tvers; **30 verktøy** inkl. nøkkelfri web_search (DuckDuckGo) + aksjekurs/-graf (Yahoo, inline cyan-graf) + PT-motor (dagens økt + regelbok) + anti-fabrikasjon-guardrail. Mayo godkjente JARVIS-vibben + ba om «smartere + gjør ting i mayooran.com» → levert. Gjenstår inc 1–5 (minne «aldri glemme» / Gemini-velger+ruting / anonymisering / skills / RAG) — se backlog + plan. Kjører Claude (din Gemini-nøkkel brukes til PT-daglig-kort).
