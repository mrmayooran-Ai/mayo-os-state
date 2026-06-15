# STATE.md — Mayo OS levende tilstand

> **Dette er sannheten.** Oppdateres som SISTE handling i hver oppgave.
> Planleggeren (claude.ai) leser denne FØRST i hver økt, via **privat speil** `mayo-os-state` (GitHub-connector — repoet er privat, ikke lenger rå public-URL).
> Aldri secrets/PII her — kun `<SET>`-markører.

**Sist oppdatert:** 2026-06-14 sent kveld · **Av:** Claude (full audit + UX-rapport) · **Versjon:** v0.11 stabilitet + v1.2 ferdig + UX-handover

---

## 🆕 Siste (2026-06-14 sent kveld) — Nattaudit: stabilitet, v1.2, UX-rapport

Mens Mayo sov, gikk Claude Code gjennom hele systemet for optimalisering,
feilretting og UX-forbedringer. Tre konkrete handlinger venter på Mayo
i morgen, alt annet er ferdig + committet + pushet.

### 🛡️ Server-stabilitet — ✅ ANVENDT 2026-06-15 08:33 (commit `c994a6e`)

Mayo kjørte `sudo bash infra/scripts/fix-server-stability.sh` 08:33:43.
Resultat (alt grønt):
- Swap: **4.0 Gi total**, 179 Mi allerede i bruk av kernel.
- MemoryHigh: 3 GB, MemoryMax: 4 GB (cgroup-cap).
- db-api: PID 131884 (én prosess), health OK.
- Gammel unit sikkerhetskopiert til `/etc/systemd/system/db-api.service.bak.20260615`.
- `/etc/fstab` har swap-linjen (overlever reboot).

**Rotårsaken som ble fanget:**

1. **Crash-loop Jun 11 09:12-09:15** — stale uvicorn-prosess holdt port 8001 →
   `Errno 98 Address already in use` hvert 13. sekund i 5 minutter. Hver
   gang lastet Whisper-modellen før den krasjet på bind → minne-press +
   risiko for Whoop-token-revokering (CLAUDE.md regel #4).
2. **Null swap** — 8 GB RAM, 0 swap. Linux OOM-killer fyrer brutalt når
   Whisper+Ollama+Postgres+db-api+litellm tilsammen topper. Forklarer
   Mayos «RAM-bruken er lav ofte i løpet av dagen».

Begge fanget av samme script (idempotent), nå anvendt i prod.

### 🎨 Livsplanlegger v1.2 — ferdig (commits `380ddae` → `ba44115`)

Alle 5 v1.2-deler portet og live på mayooran.com:

- **Items:** «Forfaller»/«Gjør når»-labels, ↩ Gjenåpne for done-items,
  «→ Til Prio», **FristChooser** (hurtigchips + mini-kalender + klokkeslett).
- **Prio:** Triage→Prio rename (route-id beholdt), bane-navn redigerbare
  (✎/dobbeltklikk), mobil-drag-fix (250ms long-press + scroll-lås +
  tekst-markering-blokk).
- **Oversikt (desktop):** Smart-fliser (I dag/Forfalt/Uke/Innboks),
  momentum-fremdriftsring i AreaCard, «I dag» som default-landing.
- **Oversikt (mobil):** Ny MobileOverview med Kort/Puls-bryter.
- **Kalender:** Egen flate (Måned/Tidslinje/Gantt/År), flyttet til
  hovedmeny ved siden av «I dag» (Mayos valg 2026-06-14).
- **Fang:** Share/Siri/Widget-strip fjernet.
- **Bonus:** Styrke-modulen fikk synlig autolagre-status + sticky
  Fullfør-knapp + helt fikset bug der draften aldri ble lagret.

### 📋 UX-rapport til Claude Design — `HANDOVER-UX-FOR-DESIGN.md`

10 prioriterte UX-funn med konkrete forslag, basert på faktiske
bug-rapporter + kode-audit. Topp 3 (P1 ⭐⭐⭐):

1. **Data-sikkerhet er ikke synlig nok** — strength-bug + meeting-sync-bug
   begge skyldes at lagring/sync-state er usynlig til det feiler. Foreslår
   sentral «Statusbar / Safety-indicator»-chip alltid synlig.
2. **«Hvor er datoen?»** — items uten dato har bare bane-flytting som vei.
   FristChooser implementert i v1.2; trenger fortsatt «+ dato»-pill direkte
   på ItemLine i lister.
3. **Bunn-nav-tetthet** — 5 items + Fang-knapp = 6 ting på 375px. Foreslår
   adaptiv tetthet (skjul labels under 360px).

Resterende (P2/P3) ligger i handover-fila. Gi den til Claude Design.

### ✅ Koblings-audit (alt i orden)

- **systemd:** db-api, telegram-bot, cloudflared, nginx, docker, ollama — alle running.
- **Postgres:** accepting connections, 54 MB DB, største tabell `health_sample` 31 MB.
- **ChromaDB / Ollama:** heartbeat OK, Ollama har gemma3:4b lastet.
- **HTTPS:** mayooran.com 200, db.mayooran.com health OK.
- **Tokens:** Whoop refresh-token satt (87 bytes), Strava (40 bytes),
  Google Calendar (gjenbrukbar — calendar.events scope).
- **Backups:** daglig pg-dump + vault-tar siste 14 dager (siste 03:30 i dag).
- **Cron:** 21 aktive linjer.
- **Manglende indekser:** ingen kritiske (`calendar_event` har gode indekser,
  høyt seq_scan-tall er mest fra full-table-resyncs).

### 🧹 Dead code / vedlikehold

- 6 `.bak`-filer ikke i git (server.py.bak, transcribe.py.bak, etc.). Mayo
  kan trygt slette: `rm db_api/server.py.bak* modules/whisper/transcribe.py.bak modules/news/digest.py.bak infra/.env.bak infra/docker-compose.yml.bak.20260515-1134`
- `server.py` er 5284 linjer — moden for å splittes i moduler (item_module/
  meeting_module finnes; kandidater: jarvis_module, biometrics_module). Risikabelt
  å gjøre uattendert, så ikke gjort.
- Frontend bundle: 1 MB unminified (271 KB gz). Vite advarer om >500 KB.
  Code-splitting kan implementeres for bedre TTI, men er ikke kritisk nå.

### 🔚 Mayos huskeliste når han våkner

1. **Kjør:** `sudo bash /home/mayo/mayo-ai-os/infra/scripts/fix-server-stability.sh`
   (engangs — fikser swap + crash-loop-vern).
2. **Verifiser:** `free -h` viser swap ≠ 0; `systemctl show db-api -p MemoryMax`
   viser 4 GB.
3. **Trigg synk** i Mac-appen for møtene fra 11. og 12. juni (de finnes
   lokalt; bryteren var av, nå er den på).
4. **Hvis tid:** gå gjennom `HANDOVER-UX-FOR-DESIGN.md` og send til Claude
   Design for neste runde polish.
5. **Hvis vil rydde:** slett .bak-filer som listet over.

---

## 🆕 Tidligere (2026-06-14, kveld) — App→Google Calendar synk (Fase 1) staget

Backend (`mayo-ai-os`, `claude/confident-noether-lpacih`, fire commits):

- **Migrasjon 007** (`e722e59`): `item.gcal_event_id` + `gcal_calendar_id` +
  `gcal_synced_at` + `sync_state` + `sync_error` + 2 indekser. Idempotent.
- **Sync-laget** (`cdc6ff2`): `modules/calendar/gcal_sync.py` — rene
  hjelpere (`pick_dt`, `title_for`, `event_payload`, `calendar_name_for_track`),
  `ensure_calendar` (auto-oppretter «Mayo OS» + «Mayo OS · Jobb»), `sync_item`
  (idempotent insert/patch/delete; respekterer 60s undo; kalender-flytt =
  delete+insert), `reconcile_once` (3 køer). `schedule_sync` fire-and-forget-
  hook i POST/PATCH/DELETE /items (eksceptions svelges — API-svar blokkeres
  aldri). `ItemCreate` fikk `scheduled_at`.
- **Løkke-vern + 21 tester** (`d7a75a1`): leseren (`modules/calendar/sync.py`)
  filtrerer events med `extendedProperties.private.mayo_item_id`. Tester
  dekker I1-negativ (privat aldri til jobb-kalender, og motsatt), idempotent
  ✓-prefiks, sensitiv-modus (`full`/`masked`/`skip`), 60s-undo-prinsipp,
  scheduled_at-foran-due_at.
- **Cron `*/3 min`** (`mayo-gcal-sync.sh`): staget, no-op til `CALENDAR_SYNC=1`.

**🔴-til-sky dispensasjon (bevisst, Mayo 2026-06-14):** IVF + private manuelle
oppgaver synkes med FULL tittel til Google. Konfigurerbart via
`CALENDAR_SENSITIVE_MODE=full|masked|skip` (default `full`). Dette er ENESTE
sted appens 🔴-prinsipp bevisst fravikes — `masked` = «Privat», `skip` = ingen
synk. Eksisterende Google-token har allerede `calendar.events` write-scope —
ingen re-consent nødvendig (motbevises kun hvis token ble opprettet før
scope-utvidelse).

**Status (2026-06-14 19:44 UTC):** **AKTIVERT** — `CALENDAR_SYNC=1`. Første
manuelle sweep gikk grønt: 1 privat item insertet → «Mayo OS»-kalenderen
(item `e3fa6f47-92f3-47bb-8035-17a839e75308`, «Re-autoriser Whoop OAuth…»).
Re-kjørt sweep er idempotent (0 operasjoner). Cron `*/3 min` håndterer
sweep videre.

**Pre-opprettet kalender** — tokenet har `calendar.events` + `calendar.readonly`,
men IKKE `calendar` (full), så `events.insert` virker men ikke
`calendars.insert`. Mayo opprettet «Mayo OS» manuelt i Google UI (2026-06-14)
og kalender-ID-en er pinnet i `.env` som `CALENDAR_SYNC_ID_PRIVAT=<satt>`.
`ensure_calendar` sjekker env-override FØR den prøver å opprette → ingen
403. Re-consent kan gjøres senere hvis det blir behov for å auto-opprette
nye kalendere.

**Spec §9.1 svart (Mayo 2026-06-14):** Obs BYGG (jobb) skal IKKE synkes —
de eier sin egen kalender; unngår dobbel-booking. Default
`CALENDAR_SYNC_SKIP_JOBB=1`. Overridbart med `=0` hvis ønsket senere.
Sweep fjerner allerede synkede jobb-events fra Google når flagget er på.

---

## 🆕 Tidligere (2026-06-14) — Livsplanlegger frontend live på mayooran.com

Frontend (`mayo-os`, branch `feat/whoop-redesign`, auto-deploy via `deploy-frontend.yml`):
- **Fase 1-flater** (PR #15, `b03f79c`): I dag · Triage (2-modus + drag + WIP-3) · Fang · Item/Monday · L0 Kart. Datalag `livsplan/store.js` (ekstern store + localStorage-outbox, ærlig lagring I3).
- **Zoombar oversikt L0→L5** (PR #16, `64a2977`): wheel/pinch/skinne/snap zoom-motor + klient-fallback.
- **Desktop master/detail** (PR #17): venstre skinne + kommando-kart (3-kol kort, sparkline, status-farge, drill) + detalj-panel. Responsiv via `useIsDesktop` (≥1024px).
- **Eget funnel-nav + Revidere** (PR #18, `c98a04f`): `/livsplan` er full-bleed, app-skallets globale nav skjult; funnel (I dag/Triage/Oversikt/Revidere) + Fang.
- **Fullført-arkiv + område-CRUD** (`3bf9638`, VPS-Claude): rename/farge/sone/slett + fullført-arkiv (oppgaver beholdes ved sletting).
- **CLAUDE.md DEFINITION OF DONE** (frontend `d4c84ad`): varig regel — en oppgave er ikke ferdig før commit + push + STATE.md + PR.

Backend (`mayo-ai-os`, `claude/confident-noether-lpacih`):
- **`GET /overview`** (`552127b`): L0/L1/L2-aggregat (`item_logic.build_overview`), 5/5 tester. **IKKE deployet enda** (token-403 på workflow_dispatch) → kjør `cd ~/mayo-ai-os && ./deploy.sh` for å aktivere; til da bruker frontend klient-fallback.

**Utestående:** område-CRUD trenger backend-endepunkter (`life_area` er fast seed i dag — frontend lagrer lokalt i localStorage, persisterer ikke på tvers av enheter). Recovery-ring (Whoop), tweaks-panel, Fraunces-titler per mobil-flate = polish.

**⚠️ Forbehold:** alle frontend-deploys i natt er bygd grønt men **ikke device-verifisert** (web-Claude har ingen browser). Sjekk mobil + desktop. Rask tilbakerulling: revert merge-commit + redeploy.

---

## 👉 HVA MAYO MÅ GJØRE (åpne handlinger)
Disse låser opp ferdigbygde features — alt annet kjører.

1. **iOS-varsler (web push):** På iPhone → åpne mayooran.com i Safari → Del → «Legg til på Hjem-skjerm» → åpne appen FRA hjem-skjerm-ikonet → Jarvis-fanen → trykk «🔔 Varsler» → Tillat. (iOS 16.4+.) Si ifra → jeg sender test-push. Da får du push fra dagbok-vokter + trend-vakt + (snart) morgenbrief.
2. **iOS voice-shortcut → Jarvis:** Lag en Shortcut: *Record Audio* → *Get Contents of URL* (POST, `https://db.mayooran.com/voice/jarvis`, Header `X-Mobile-Token: <SET>`, Request Body = Form, felt `audio` = opptaket) → *Get Dictionary Value* `response` → *Speak Text*. Legg på Hjem-skjerm/«Hey Siri». Endepunktet er live + testet.
3. **Enable Banking (bank):** Koble banken → låser opp abonnement-detektor + (kommende) inkasso-vakt + skatte-/likviditetsmotor. `finance.transactions` er tom til dette.
4. **Vær-hjemsted:** Defaulter til Oslo (WEATHER_LAT/LON i .env). Si fra om annet sted.
5. **Mac-Whisper-tunnel** (127.0.0.1:11436) er NEDE → møte-transkribering bruker treg VPS-Whisper. Start Mac-tjenesten + reverse-tunnel for rask transkribering. (Coop-opptakeren har egen Whisper på Tailscale — denne TODO gjelder mayooran.com-pipelinen.)
6. **IVF-spor** (sensitivt): når du vil, gi input på *tonen* → da bygger jeg fase-bevisste påminnelser. Ikke gjettet autonomt.
8. **🔄 Re-autoriser Whoop (1 gang):** aapne **https://db.mayooran.com/whoop-auth** i nettleser → logg inn Whoop → godkjenn. Token-kjeden ble revokert av en reuse-race (naa fikset med laas, 9f6ad11). Etter dette virker Whoop-data + trend-vakt igjen.
7. **🧹 Slett 5 junk-test-møter i `meeting`-tabellen** — laget under diagnostikk 06-10 (`b85703de…`, `31c3c7ef…`, `3b8eed98…`, `9a37b689…`, `d7b34391…`). DELETE-SQL klar i handover (`~/mayo-whisper/HANDOVER-obsbygg.md` §5.2). Sletting av rader er regel-messig brukerens å kjøre selv.
9. **🔗 Aktiver Tasks↔Apple Reminders sync** (ny, 2cc8e0c — feature-flagget AV): (a) kjør `migrations/005_task_reminder_sync.sql`, (b) lag Apple-liste **«Mayo OS»** på iPhone (eller sett `TASK_SYNC_LIST` i .env), (c) sett `TASK_REMINDER_SYNC=1` i .env + `systemctl restart db-api`. Da speiles oppgaver begge veier (nyeste vinner, speilet sletting). (d) Valgfritt: utvid iOS-Shortcuten til å prosessere `delete_queue` fra `/reminders/bulk-sync` så sletting også når Apple. Detaljer i HANDOVER_RESULT.

---

## 🟢 Live & verifisert

| Komponent | Status | Sist verifisert | Notat |
|---|---|---|---|
| frontend (nginx 8086) | 🟢 | 2026-06-08 | mayooran.com · build 13a91e5 |
| db-api (8001) | 🟢 | 2026-06-10 | Whisper pre-warm-timeout + **graceful Claude-fail-håndtering** (5abace9) |
| Whoop / Strava / Telegram / LiteLLM (4000) / Google Calendar | 🟢 | 2026-06-08 | uendret, fungerer |
| **Gmail (lese + skrive)** | 🟢 | 2026-06-08 | re-auth gjort (gmail.readonly). Jarvis kan nå LESE/søke innboks (`search_emails`/`read_email`) + lage/sende utkast. E-post anonymiseres før sky. |
| **Jarvis Inc 1 — minne** | 🟢 | 2026-06-08 | chat-historikk i Postgres (overlever enheter); profil-MD `jarvis_memory` (wiret inn). |
| **Jarvis Inc 2 — ruting + C-knapp** | 🟢 | 2026-06-08 | rute-matrise: 🔴 ivf/økonomi → lokal Gemma 3 4B (aldri sky), resten → Claude. Pill-velger Auto/Claude/Gemini/Lokal + 🔒/☁-indikator. Gemini = gemini-2.5-flash. |
| **Jarvis Inc 3 — anonymizer** | 🟢 | 2026-06-08 | norsk NER (spaCy) + regex scrubber ALL sky-trafikk (inkl. tool-resultater i Claude-loopen). Bevist 0 rå PII ut. `entities.txt` always-scrub. |
| **iOS Web Push (VAPID)** | 🟢 (backend/frontend) | 2026-06-08 | self-hosted, kryptert, ingen tredjepart. `/sw.js` servert. **Stille-timer 23–07** (vekker ikke). Mangler kun Mayos abonnement (se TODO #1). |
| **Voice-router `/voice/jarvis`** | 🟢 | 2026-06-08 | tale → Whisper → Jarvis (ruting+anonymizer+34 verktøy) → ett kort svar. Jarvis avgjør reminder/kalender/mail/økonomi/trening selv. Testet (kalender-kommando). Mangler Shortcut (TODO #2). |
| **Proaktiv dagbok-vokter** | 🟢 | 2026-06-08 | cron 30 min: nye dagbok-entries → lokal Gemma henter action items → de-dup → crm_task + Telegram/push. 🔴-trygt (lokal). Verifisert: 5 reelle oppgaver. |
| **Trend-vakt (helse)** | 🟢 | 2026-06-08 | cron 07:30 UTC: Whoop recovery/HRV/RHR vs 7d-baseline → Telegram/push ved overtrening/sykdom-signal. |
| **Morgenbrief-motor** | 🟢 | 2026-06-08 | 08:00-helserapporten samler nå recovery→økt + kalender (m/ konflikt) + topp-3 tasks + **innboks (uleste 24t)** + vær — og sender også **iOS-push**. Schedulert (cron 06,07 UTC). |
| **In-app «I dag»-brief** | 🟢 | 2026-06-08 | kollapsbart kort øverst i Jarvis-visningen: kalender + tasks + innboks + vær (samme data som morgenbriefen, når som helst). Endepunkt `GET /brief/today`. |
| **Viktig-e-post-flagging** | 🟢 | 2026-06-08 | morgenbrief + in-app-brief flagger **⚠️ viktige uleste** (penger/frist) via nøkkelord-match. **Lokalt — ingen LLM/sky.** Fanget en buried eFaktura. |
| **Abonnement-detektor** | 🟡 dvalende | 2026-06-08 | `/finance/advisor/subscriptions` — ferdig+testet, men `finance.transactions` tom (TODO #3). |
| **Vær-verktøy (get_weather)** | 🟢 | 2026-06-08 | yr/MET, default Oslo. |
| Styrkelogg / PT-regelbok / Health-Logg | 🟢 | 2026-06-08 | uendret + PT-fiks (Strava-recency, ikke tom manuell logg). |
| Assistent (Jarvis) `/assistent` | 🟢 | 2026-06-08 | **34 verktøy**. Helse-nav lander på «Klar for økt». |
| Public state-mirror | 🟢 | 2026-06-10 | `mayo-os-state` · planleggeren leser den |
| **Coop møteopptaker (Mac, lokal :8765)** | 🟢 | 2026-06-10 | `meeting_local.py` i `~/mayo-whisper/`. **Alle 4 opprinnelige forespørsler live:** (a) responsivt fullhøyde-transkript, (b) speaker-diarisering m/ on-demand Ollama-knapp + editable navne-chips, (c) Notater-fane synket til Obs BYGG, (d) tag-autocomplete med prefiks-rangering + seedet norsk vokabular + "Opprett #X"-rad. Frontend single-file HTML/JS. Whisper: Tailscale 100.107.201.55:8081 (nb-whisper). Ollama: 100.107.201.55:11434 (qwen2.5:14b + llama3.1:8b fallback). |
| **Diariserings-persistering** | 🟢 | 2026-06-10 | Redigerte navn følger inn i lagret `.md` + Obs BYGG-synk. To garantier: (1) name-anchored prompt — navn puttes direkte i Ollama-prompten som `[Geir]:`-merker, ikke remappet via uavhengig nummerering (BLOCKER-fix); (2) ord-token-multiset-vakt 3% — fanger droppede setninger, kan ikke lures av LLM-padding (MAJOR-fix). **Live-verifisert mot ekte Ollama:** 0/203 tokens tap på 1013-tegns norsk transkript; Ollama-down → ren tekst-fallback uten hang; æøå/proper nouns/numre bevart tegn-for-tegn. |
| **launchd auto-start (Mac-opptaker)** | 🟢 | 2026-06-10 | `~/Library/LaunchAgents/com.mayo.meeting-recorder.plist`. KeepAlive (SuccessfulExit=false), ThrottleInterval=30s, ProcessType=Interactive. Logger til `~/Library/Logs/mayo-whisper/`. KeepAlive-testet 2x (kill -9 + SIGTERM → auto-restart). |
| **Obs BYGG `/meeting/import` graceful** | 🟢 | 2026-06-10 | `meeting_import` pakker `claude_extract` i try/except (commit 5abace9). Hvis Claude feiler (401/quota/timeout) → møtet lagres `status=done` med transkript+base_tags+user_notes intakt, AI-felter tomme. Tidligere 500 verifisert borte. Aksepterer nå `user_notes` + `tags` fra Mac-opptaker. |
| **`GET /meeting-tags`** | 🟢 | 2026-06-10 | Returnerer alle unike tagger på tvers av brukerens møter — for tag-autocomplete i Mac-opptakeren. Bindestrek (ikke `/meeting/tags`) for å unngå ruterkollisjon. Token-autentisert. |
| **Obs BYGG web §1.1-1.7 (design v1.1)** | 🟢 | 2026-06-11 | mayo-os frontend + VPS. §1.1 degradert-tilstand (amber-chip/re-analyse), §1.2 speaker-diariserings-UI (fargede chips, inline rename), §1.3 synk-opt-in (☁/⌂ toggle+chip), §1.4 tag-kuratering (autocomplete {tag,count}), §1.5 oppgaver inline-rediger, §1.6 vedlegg+Dokumenter-fane, §1.7 frittstående notater-fane. Aktiverte tidligere «snart»-faner. Live-verifisert i browser + 2 adversarielle review-runder (0 suverenitetsbrudd). Commits FE 4debfa7/03f40b8, BE d1262c5/21e8d26/4eeeed0. |
| **Journal psykolog-laget (§2)** | 🟢 | 2026-06-11 | `window.PSYCH` levende refleksjoner: per-entry refleksjon-pille (privat·lokal), PsychSummaryCard tidslinje-dividers, Innsikt-fane (emnetagger→per-uke-graf, delta-linjer, mood-prikker), touch()→«⟳ oppdaterer»→settle. PRIVAT-spor, kun lokal modell, bak OTP-gate, ALDRI blandet med jobb. Commit ce74dfc. ⚠️ Summary-kort/Innsikt = fortsatt seed-mock i `src/lib/psych.js`. |
| **Per-entry refleksjon-pille — backend-wiret (§2.1)** | 🟢 kode (krever VPS-deploy) | 2026-06-11 | **Rotårsak til «ingen pille» funnet:** `GET /journal` returnerte aldri `reflection`/`reflection_model`, så `isSafeReflection(e)` var alltid false. `psykolog_short` (lokal Ollama, enrich Phase 4b) ligger i vault-frontmatter, ikke Postgres. Fix `8c9bfe1`: `_attach_reflections` fester dagens refleksjon på dagens NYESTE entry som `reflection` + `reflection_model='local'` (sannferdig → 🔒 trygt også for 🔴 ivf/økonomi). Ingen DB-migrasjon. 5 enhetstester (AST-uttrekk, kjører uten fastapi/DB) grønne + `py_compile` OK under 3.12. **Deploy backend (master) + `systemctl restart db-api` → pillen vises på ekte entries.** |
| **DB: sync_enabled + meeting_attachment + work_note** | 🟢 | 2026-06-11 | Migrasjon 004 (idempotent). Vedlegg i `/home/mayo/MayoVault/obs-bygg/attachments/` m/ path-traversal-vakt. Alle queries user_id-filtrert (review fant+fikset manglende filter i action-item DELETE → 4eeeed0). |

## 🟡 Pågår / delvis
- **🚀 Livsplanlegger Fase 1 — motoren (BE `6895123`)** — kanonisk `item`-modell (migrasjon 006: item-tre + life_area 6-seedet + context_bucket), `item_logic.py` (board, today+soft-cap-3, L0-formel, SUVERENITETS-vegg: filter_track/assert_no_private_leak I1 + read-only synket jobb-item), `item_module.py`-router (POST/GET/PATCH/DELETE /items + /board //today /contexts /subtasks). 11 tester (40/40 totalt). **Additivt — rører IKKE journal-vokter/reminder-sync.** Beslutninger 2026-06-13: crm_task speiles, jobb read-only+lokal planlegging, kontekst seedet+egne, L0-formler v1. Design-handover lest mot mocks (aligner). ✅ **#1 crm_task→item speil** (`114085c`, `item_mirror.py` idempotent backfill + CLI). ✅ **Verifisert mot ekte Postgres 16** (`f00b763`): 006 kjører rent uten 005 (selv-tilstrekkelig crm_task.tags-ALTER), 14/14 SQL-assertions (seed/capture/board-WHERE/subtask-ekskl/tombstone/speil-mapping/idempotens) + 44/44 logikk-tester. **DEPLOYET & VERIFISERT 2026-06-13 18:59 UTC** (auto-deploy, run 27471142405, runner `mayo-vps`): migrasjon 006 anvendt, crm_task→item backfill `{created:67, total:67}`, `/items` + `/items/board` svarer med ekte data, helse `{ok:true}`, fersk PID. Self-hosted runner er nå live på VPS (systemd `actions.runner.*mayo-vps`) → ALLE fremtidige backend-pushes deployer seg selv. Ingen flere terminallinjer for Mayo. **Gjenstår Fase 1:** #2 journal-vokter-repoint (BEVISST UTSATT til Fase 3-flipp — speilet dekker vokter-tasks), #3 frontend-port (triage/inbox/«3 i dag» fra mocks på feat-grenen — der det blir synlig). 44/44 tester. Specs: `docs/superpowers/specs/2026-06-13-livsplanlegger*.md`.
- Web push + voice-router venter på Mayos engangs-oppsett (TODO #1, #2).
- Abonnement-detektor dvalende til bank kobles (TODO #3).
- Junk test-møter (5 stk) venter på Mayo's DELETE (TODO #7).
- **Refleksjon-pille backend-fix (8c9bfe1)** — ✅ **LIVE & verifisert 06-12:** `/journal` returnerer `reflection` på **18/31 entries** etter ren restart. Rotårsak var IKKE kode — gammel db-api-prosess (41 min) hadde aldri lastet ny kode; tidligere `systemctl restart` syklet den ikke. Fix: tøm `__pycache__` + restart → fersk prosess lastet koden. Pille vises i app etter hard-refresh.
- **Tasks↔Apple Reminders sync-layer (2cc8e0c)** — ✅ kode på master (PR #3 merget), **feature-flagget AV**. Venter aktivering på VPS (TODO #9: migrasjon 005 + `TASK_REMINDER_SYNC=1` + «Mayo OS»-liste). Pending-decision RESOLVED → B.
- ✅ **Journal entry-tekst = brukerens egne ord (ikke ai_summary)** + refleksjon-pille kollapset default (FE `1ffd5bb`, feat-grenen) — LIVE & verifisert i app 06-12.
- ✅ **Etappe 2 ferdig + infra (FE+BE)** — Kalender tapp-dag → bottom-sheet for ny/bakoverdatert entry (POST /journal m/ entry_date) → trigger ukes-regen for den uken (FE `fe57a75`). Innsikt: strukturert dagsrefleksjons-arkiv lazy-lasted fra `/notes/psykolog/history`, gruppert per ISO-uke, kollapserbart (FE `2bd3e1e`). Backend deploy.sh (BE `05ab4ca`) — én-kommando deploy med pycache-tømming + PID-verifisering. Whoop `?debug=1` (BE `1d27754`) viser nøyaktig redirect_uri/scopes til Whoop-dashboardet. Tasks↔Reminders aktiverings-script (BE `3a05484`) + iOS Shortcut delete_queue-doc. GitHub Action deploy-backend.yml (BE `55c1fd3`) — self-hosted runner gjør at Claude kan deploye via API. **Venter deploy: BE `./deploy.sh`, FE `./deploy.sh skip-pull` på VPS.**
- ✅ **Ekte lokal ukes-synteser — LIVE & verifisert 06-12** (BE-gren `claude/confident-noether-lpacih`, FE-gren `feat/whoop-redesign`). Erstatter «Uke 24»-mock med ekte analyse av entries+dagsrefleksjoner+kalender(+HRV/Whoop+Strava når Whoop er oppe). Lever/låser; regen ved bakoverdatert entry. `GET /journal/psykolog/weeks` + `psych.js`-fetch (FE `c2cc22f`). POST /journal tar `entry_date`. reflect.py Sunday-cron nå LOKAL (var Claude=sky-brudd). **Modell: `gemma3:4b-it-q4_K_M` på VPS** (privat-spor, 🔴-trygg) — 3B var for svak (metaprat), llama3.1:8b OOM'er (4.8>4.3 GiB). **Suverenitetssperre:** ALDRI jobb-Mac (Tailscale qwen2.5:14b = Obs BYGG). Retry m/ backoff på transiente Ollama-blips. Alle 8 uker backfill'et OK. Commits: `8c9bfe1`/`2cc8e0c` … `b289dcc`. **Gjenstår (Etappe 2, IKKE bygd): kalender tapp-dag→ny entry-UI, Innsikt strukturert dagsrefleksjons-visning.**

## 🔴 Åpne problemer
- **Whoop 502 — ROTAARSAK FUNNET + fikset (06-11), krever EN re-auth (TODO #8).** Refresh-token-reuse-race: samtidige /whoop-kall refresha access-token uten laas → Whoop revokerte hele token-kjeden → vedvarende 400 invalid_request. Fikset med dobbelsjekket asyncio.Lock (9f6ad11). Token-kjeden er fortsatt revokert → Mayo maa re-autorisere EN gang, deretter holder laasen den i live.
- **🔴 Whoop re-auth BLOKKERT (06-12):** `/whoop-auth` → Whoop avviser med `error=request_forbidden` («The request is not allowed») FØR innlogging. Ikke token-racet — det er en OAuth-config-avvisning på Whoop sin side. Mest sannsynlig: redirect-URI ikke registrert eksakt (`https://db.mayooran.com/whoop-auth`) i Whoop Developer Dashboard, eller appen er disabled/under review (kan ha blitt flagget av reuse-racet). Sjekk OGSÅ scopes (`read:recovery read:sleep read:cycles read:profile offline`) + at `WHOOP_CLIENT_ID` matcher appen. Fiks ligger i Whoop-dashboardet, ikke i koden. (Tilbud: `?debug=1`-diagnostikk-endepunkt kan bygges.)
- ✅ **Frontend DEPLOYET (06-11 16:31)** — mayooran.com serverer naa `28cbc97` (Obs BYGG §1.1-1.7 + Journal §2 LIVE inkl. design-fidelity-fikser). Deploy-repo `~/mayo-os-deploy` byttet main → feat/whoop-redesign (fetch-refspec var kun main → maatte hente grenen eksplisitt). PR #14 staar fortsatt aapen for evt. senere merge til main.
- **`finance.transactions` tom** — Enable Banking ikke koblet → finans-features dvalende (TODO #3).
- **Mac-Whisper-tunnel nede** (127.0.0.1:11436) → mayooran.com-pipelinen bruker treg VPS-Whisper (TODO #5). Coop-opptakeren har egen tunnel via Tailscale og påvirkes ikke.
- **crm_task auto-task-bug fikset 06-08** (manglet `tags`-kolonne) — alle møte-tasks feilet stille; kolonne lagt til.

## 📋 Backlog (prioritert)
- **Jarvis-bygg:** Inc 0–4 ✅ live. Inc 5 (RAG: pgvector journal semantic search → Jarvis chat) gjenstår.
- **Proaktive lag bygd 06-08:** dagbok-vokter, trend-vakt, morgenbrief, web push, voice-router ✅.
- ✅ **Coop-opptaker speaker-chips i Obs BYGG-frontend (06-11):** ferdig — §1.2 speaker-diariserings-UI med fargede chips + inline rename + on-demand Ollama. Live-diariseringsstatus persisteres fortsatt ikke ved page-refresh midt i opptak (mindre).
- IVF-tidslinje (lokal, krever Mayos tone-input) · inkasso-vakt + skatte-/likviditetsmotor (krever bank) · per-person møteforberedelse (krever Inc 5 RAG).
- Enable Banking-kobling · lokal modell-oppgradering · Obsidian-class editor.
- **Journal-spec gjennomgått (06-11):** §1–6 i praksis fullt bygd & live (audit på `feat/whoop-redesign`, se HANDOVER_RESULT). Gjenstår KUN 3 §7.3-punkter (Mayo-gated, IKKE bygd): mood-kurve over uker · «verktøy vi har øvd på»-liste · emnetagg→relaterte-entries drill-down. Kart-fanen = Fase-2-mockup (venter `{lat,lon}`). Frontend leser per-entry `entry.reflection` fra backend → refleksjon-fix `8c9bfe1` tenner pillen ved deploy.

## 🕐 Siste commits (nyeste øverst)
**Backend (`mayo-ai-os`):**
- `2cc8e0c` — feat(tasks): bidireksjonell crm_task↔Apple Reminders sync-layer (migrasjon 005 + task_sync.py + /tasks-hooks + bulk-sync revers + reconcile). Feature-flag `TASK_REMINDER_SYNC=0`. Branch `claude/confident-noether-lpacih`, IKKE aktivert enda (06-11)
- `8c9bfe1` — fix(journal): surface per-dag psykolog-refleksjon i GET /journal (reflection + reflection_model='local') → refleksjon-pillen vises. Branch `claude/confident-noether-lpacih`, IKKE deployet enda (06-11)
- `9f6ad11` — fix(whoop): async-laas rundt token-refresh (reuse-race revokerte kjeden) + manglende asyncio-import (06-11)
- `c909796` — fix(ops): demp falsk evening-alarm + privacy-import (news/psykolog) + trend-vakt data-klar-gate hver 30 min (06-11)
- `4eeeed0` — fix(security): filter meeting_action_item DELETE by user_id (06-11)
- `21e8d26` — feat: design v1.1 §1.3/1.6/1.7 backend — sync-opt-in, vedlegg, work-notes (06-11)
- `d1262c5` — feat: reanalyze + rename-speaker + tag frequency {tag,count} (06-11)
- `5abace9` — meeting/import: user_notes + extra tags fra Mac-opptaker + graceful Claude-fail + GET /meeting-tags (06-10)
- `19e7807` — fix(privacy): robust anonymizer-import (dual-path) — fikset db-api oppstart-krasj
- `86f3a3b` — feat(meeting): PATCH /meeting/{id}/summary — lagre redigert sammendrag + beslutninger
- `536d623` — fix(privacy): Fase 1b — anonymiser 13 gjenværende sky-call-sites
- `27c4a89` — docs: synk STATE.md — Inc 4 (persona-ruter) live
- `30d0256` — feat(jarvis): Fase 3 — ekspert-persona-ruter med 9 personas
- `e195a54` — flagg viktige uleste e-poster (penger/frist) i brief (06-08)
- `675644f` — GET /brief/today (06-08)
- `288caad` — push stille-timer 23-07 + morgenbrief sender også push (06-08)
- `539175f` — innboks-sammendrag i morgenbriefen (06-08)
- `01a3f03` — voice-router /voice/jarvis (06-08)
- `57b7a8d` — web push backend (VAPID) (06-08)
- `6b148b5` — proaktiv dagbok-vokter (06-08)
- `df4c716` — morgenbrief-motor (06-08)
- `4a61871` — trend-vakt (06-08)
- `3d9aa59` — get_weather-verktøy (06-08)
- `17f7da4` — abonnement-detektor (06-08)
- `ae3b8ea` / `5901c23` — Gmail-lesing + metadata-fiks (06-08)
- `77d42fe` — Whisper pre-warm-timeout (06-08)
- `f358794` — Inc 3 anonymizer-round-trip (06-08)

**Frontend (`mayo-os`):**
- `28cbc97` — feat(obs): §1.5 oppgave-omgrupperings-toast + §1.7 notater wiki-nav til Graf (06-11)
- `1a12e6c` — fix(journal): design-fidelity §2 — refleksjon AAPEN default, ukekort aapen for live, 🔴-suverenitetsguard (13 gap-fikser fra 46-agent design-audit) (06-11)
- `03f40b8` — feat(obs): §1.3/1.5/1.6/1.7 — synk-opt-in, oppgave-edit, vedlegg, notater (06-11)
- `ce74dfc` — feat(journal): §2 psykolog-laget — levende refleksjoner window.PSYCH (06-11)
- `4debfa7` — feat(obs): §1.1/1.2/1.4 — degradert-tilstand, speaker-diarisering, tag-kuratering (06-11)
- `13a91e5` — flagg viktige e-poster (penger/frist) i «I dag»-kortet (06-08)
- `a202ac2` — in-app «I dag»-brief (06-08)
- `08fd963` — web push (service worker) (06-08)
- `20c4489` — Helse-nav → «Klar for økt» (06-08)
- `eb273fc` — modell/rute-velger (Inc 2) (06-08)
- `f1ba4d2` — DB-historikk i Assistent (Inc 1) (06-08)

**Mac-side (uncommitted, lokalt på Mac-en — `/Users/mayo/mayo-whisper/`):**
- `meeting_local.py` — diariserings-persistering m/ navn-anchored prompt + token-vakt + dual-format `_apply_speaker_names`; tag-autocomplete prefiks-rangering + seedet norsk vokabular + "Opprett #X"-rad; alle 4 opprinnelige feature-forespørsler implementert. (Ingen Git-repo på Mac-en for denne — kun lokal fil. Handover-dokument: `HANDOVER-obsbygg.md`.)
- `~/Library/LaunchAgents/com.mayo.meeting-recorder.plist` — launchd auto-start, installert + KeepAlive-verifisert.
- `~/.ssh/config` — `Host 37.27.248.55: KexAlgorithms -mlkem768x25519-sha256,sntrup761x25519-sha512@openssh.com` (permanent SSH-fiks — OpenSSH 10.2 default PQ-KEX hang med Ubuntu OpenSSH 9.6 pga PMTU).

## 📝 Til planleggeren (claude.ai)

### Obs BYGG web + Journal design v1.1-økt (2026-06-11) — handover ferdigstilt
Implementerte HELE design-handoveren (`HANDOVER-design-obsbygg-journal.md`): Obs BYGG §1.1-1.7 + Journal §2.
- **Obs BYGG web §1.1-1.7**: degradert-tilstand, speaker-diariserings-UI, synk-opt-in (☁/⌂), tag-kuratering ({tag,count}), oppgaver inline-rediger, vedlegg+Dokumenter-fane, frittstående notater-fane. Aktiverte de tidligere «snart»-fanene.
- **Journal §2 psykolog-laget**: `window.PSYCH` levende refleksjoner (pub/sub + touch/settle), refleksjon-pille, PsychSummaryCard tidslinje-dividers, Innsikt|Samtale. PRIVAT-spor — kun lokal modell, bak OTP-gate, ALDRI blandet med jobb-sporet.
- **DB+backend**: migrasjon 004 (sync_enabled + meeting_attachment + work_note) + 9 nye endepunkter, alle session/user_id-autentisert.
- **Kvalitet**: 2 adversarielle review-runder (16+16 agenter) → 0 suverenitetsbrudd; fikset kritisk user_id-isolasjon (action-item DELETE), XSS-sanering, autosave-race, paste-guard. Live-verifisert alle flater i browser.
- **Pushet til GitHub**: mayo-os `feat/whoop-redesign` + mayo-ai-os `master` (38 upushede commits synket). Git-identitet fikset → `mr.mayooran@gmail.com`.
- **Gjenstår**: handover §4 (mood-kurve, drag-reschedule, vedlegg-noder) — eksplisitt «avklar med Mayo», ikke bygd. Journal §2.3 backend-regenerering (event-drevet lokal modell) — seed-mock i dag.

### Design-fidelity-fikser (2026-06-11 16:31)
Mayo flagget at Journal-siden ikke matchet design-handover-prototypen. Adversariell design-audit (46 agenter, 7 dimensjoner) fant 13 reelle avvik. Alle fikset i en batch:
- §2.1 ReflectionPill: AAPEN som default (var lukket); pille `+ refleksjon ▾` aapen / `✦ refleksjon ▸` lukket; sitatkort: tekst OEVERST, badges (privat + lokal modell) NEDERST; ny prop reflectionModel (local → 🔒, ukjent → ⚠).
- §2.1 PsychSummaryCard: AAPEN for live-perioder, lukket for historiske (useState(s.live)); live-footer basert paa N entries + separator.
- §2.3 + suverenitet: 🔴-domener (ivf/økonomi) faar INGEN refleksjon-pille uten reflection_model=local. Skjuler heller pillen enn aa lyve med 🔒-merket. Backend-kontrakt dokumentert.
- §1.5/§1.7: oppgave-omgrupperings-toast + notater wiki-nav til Graf.
- Deployet 16:31, live-bundle bekreftet med nye strenger.

### Drift/ops-fikser (2026-06-11)
Trigget av Telegram-monitor-alarm + Mayo-rapport om manglende endringer/Whoop:
- **Falsk evening-alarm dempet**: kveldsmeldingen ble bevisst disablet 06-08 (personvern), men monitoren voktet fortsatt loggen → fjernet sjekken.
- **To tjenester fikset**: `news-digest` + `psycholog-connect-daily` feilet daglig med `ModuleNotFoundError: privacy` → repo-rot paa sys.path.
- **Trend-vakt ombygd** (Mayo-krav): hver 30 min 05-10 Oslo, DATA-KLAR-gate (sender ALDRI for dagens recovery er publisert), en gang/dag, 502-retry.
- **AAPENT**: Whoop-502 rotaarsak + frontend-deploy (se Aapne problemer). Git-identitet fikset → mr.mayooran@gmail.com. `mayo-os-state` boer settes privat (GitHub-connector dekker planleggeren).


### Coop møteopptaker-økt (2026-06-10) — alt levert
1. **Alle 4 opprinnelige forespørsler ferdig & live** på Mac-opptakeren (`localhost:8765/?mode=fysisk`):
   - Responsivt fullhøyde-transkript (flex column, viewport-tilpasset).
   - Speaker-diarisering: on-demand Ollama-knapp ("🔍 Identifiser høyttalere"), `[Person N]:` markører i live-visning, fargede editable navne-chips som propagerer gjennom transkriptet.
   - Notater-fane med auto-lagring (2s debounce), synket til Obs BYGG via `user_notes`-feltet.
   - Tag-autocomplete: prefiks-rangering (eksakt > prefiks > delstreng → kortest → alfabetisk), seedet norsk vokabular (44 tagger inkl. `ukesstart`, `oppfølging`, `beslutning`), "➕ Opprett #X"-rad, prefiks uthevet.

2. **Diariserings-persistering (kritisk fix)** — redigerte navn følger nå inn i lagret `.md` + Obs BYGG-synk. To garantier verifisert mot ekte Ollama med adversariell workflow (13 agenter, 5 vinkler):
   - **Name-anchored prompt** (BLOCKER-fix): navn puttes direkte i prompten som `[Geir]:`-merker — modellen mapper etter innhold, ikke uavhengig rekkefølge → ingen name-swap mulig.
   - **Ord-token-multiset-vakt** 3% (MAJOR-fix): erstattet 0.85 lengde-sjekk som lot LLM-padding skjule droppede setninger.
   - Live-test: 0/203 tokens tap; Ollama-down → ren tekst-fallback uten hang.

3. **VPS-fiks deployet** (commit `5abace9`): `meeting/import` pakker `claude_extract` i try/except (tidligere 500-bugen borte). Nytt `GET /meeting-tags` for autocomplete. `user_notes` og `tags` aksepteres fra Mac-opptaker.

4. **Driftsforbedringer:**
   - **launchd auto-start** for Mac-opptakeren (KeepAlive, ThrottleInterval=30s, logger til `~/Library/Logs/mayo-whisper/`).
   - **SSH port 22 permanent fix** — OpenSSH 10.2 sin default `mlkem768x25519-sha256` PQ-KEX henger mot Ubuntu OpenSSH 9.6 (PMTU). Fikset i `~/.ssh/config`.

5. **ANTHROPIC_API_KEY falsk alarm** — tidligere "401" var fra auth-middleware (manglet shortcut-token), ikke fra Anthropic. Live verifisert: `claude-sonnet-4-5` svarer; full `meeting/import` returnerer `action_items: 2`, anonymizer kjører (PERSON_N), `claude_cost_usd` metrert.

### Mønstre/lærdom oppdatert
- **Mac-only komponenter** (Coop-opptaker): single-file Python+HTML+JS i `~/mayo-whisper/meeting_local.py`. Ingen Git-repo lokalt. Hold separat fra mayooran.com — denne er **jobb-spor (Obs BYGG)**, ikke privat journal.
- **Adversariell verifisering** før deploy av høy-risiko endringer (data-tap) lønner seg. Token-multiset-vakt > lengde-vakt for LLM-output-validering.
- **SSH-feildiagnose** når TCP-koblingen lykkes men handshake henger → mistenk KEX-algoritme-mismatch eller PMTU på store key-pakker.

### Venter på Mayo
- TODO #7: Slett 5 junk-test-møter fra DB (DELETE-SQL klar).
- Øvrige TODOs (1-6) uendret fra forrige økt.
