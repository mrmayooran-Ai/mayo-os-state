# STATE.md — Mayo OS levende tilstand

> **Dette er sannheten.** Oppdateres som SISTE handling i hver oppgave.
> Planleggeren (claude.ai) leser denne FØRST i hver økt, via public speil `mayo-os-state`.
> Aldri secrets/PII her — kun `<SET>`-markører.

**Sist oppdatert:** 2026-06-08 · **Av:** Claude (autonom Jarvis-økt) · **Versjon:** v0.6 Jarvis

---

## 👉 HVA MAYO MÅ GJØRE (åpne handlinger)
Disse låser opp ferdigbygde features — alt annet kjører.

1. **iOS-varsler (web push):** På iPhone → åpne mayooran.com i Safari → Del → «Legg til på Hjem-skjerm» → åpne appen FRA hjem-skjerm-ikonet → Jarvis-fanen → trykk «🔔 Varsler» → Tillat. (iOS 16.4+.) Si ifra → jeg sender test-push. Da får du push fra dagbok-vokter + trend-vakt + (snart) morgenbrief.
2. **iOS voice-shortcut → Jarvis:** Lag en Shortcut: *Record Audio* → *Get Contents of URL* (POST, `https://db.mayooran.com/voice/jarvis`, Header `X-Mobile-Token: <MOBILE_API_TOKEN>`, Request Body = Form, felt `audio` = opptaket) → *Get Dictionary Value* `response` → *Speak Text*. Legg på Hjem-skjerm/«Hey Siri». Endepunktet er live + testet.
3. **Enable Banking (bank):** Koble banken → låser opp abonnement-detektor + (kommende) inkasso-vakt + skatte-/likviditetsmotor. `finance.transactions` er tom til dette.
4. **Vær-hjemsted:** Defaulter til Oslo (WEATHER_LAT/LON i .env). Si fra om annet sted.
5. **Mac-Whisper-tunnel** (127.0.0.1:11436) er NEDE → møte-transkribering bruker treg VPS-Whisper. Start Mac-tjenesten + reverse-tunnel for rask transkribering.
6. **IVF-spor** (sensitivt): når du vil, gi input på *tonen* → da bygger jeg fase-bevisste påminnelser. Ikke gjettet autonomt.

---

## 🟢 Live & verifisert

| Komponent | Status | Sist verifisert | Notat |
|---|---|---|---|
| frontend (nginx 8086) | 🟢 | 2026-06-08 | mayooran.com · build 08fd963 |
| db-api (8001) | 🟢 | 2026-06-08 | **Whisper pre-warm-timeout lagt til** (henger ikke lenger på oppstart) |
| Whoop / Strava / Telegram / LiteLLM (4000) / Google Calendar | 🟢 | 2026-06-08 | uendret, fungerer |
| **Gmail (lese + skrive)** | 🟢 | 2026-06-08 | re-auth gjort (gmail.readonly). Jarvis kan nå LESE/søke innboks (`search_emails`/`read_email`) + lage/sende utkast. E-post anonymiseres før sky. |
| **Jarvis Inc 1 — minne** | 🟢 | 2026-06-08 | chat-historikk i Postgres (overlever enheter); profil-MD `jarvis_memory` (wiret inn). |
| **Jarvis Inc 2 — ruting + C-knapp** | 🟢 | 2026-06-08 | rute-matrise: 🔴 ivf/økonomi → lokal Gemma 3 4B (aldri sky), resten → Claude. Pill-velger Auto/Claude/Gemini/Lokal + 🔒/☁-indikator. Gemini = gemini-2.5-flash. |
| **Jarvis Inc 3 — anonymizer** | 🟢 | 2026-06-08 | norsk NER (spaCy) + regex scrubber ALL sky-trafikk (inkl. tool-resultater i Claude-loopen). Bevist 0 rå PII ut. `entities.txt` always-scrub. |
| **iOS Web Push (VAPID)** | 🟢 (backend/frontend) | 2026-06-08 | self-hosted, kryptert, ingen tredjepart. `/sw.js` servert. Mangler kun Mayos abonnement (se TODO #1). |
| **Voice-router `/voice/jarvis`** | 🟢 | 2026-06-08 | tale → Whisper → Jarvis (ruting+anonymizer+34 verktøy) → ett kort svar. Jarvis avgjør reminder/kalender/mail/økonomi/trening selv. Testet (kalender-kommando). Mangler Shortcut (TODO #2). |
| **Proaktiv dagbok-vokter** | 🟢 | 2026-06-08 | cron 30 min: nye dagbok-entries → lokal Gemma henter action items → de-dup → crm_task + Telegram/push. 🔴-trygt (lokal). Verifisert: 5 reelle oppgaver. |
| **Trend-vakt (helse)** | 🟢 | 2026-06-08 | cron 07:30 UTC: Whoop recovery/HRV/RHR vs 7d-baseline → Telegram/push ved overtrening/sykdom-signal. |
| **Morgenbrief-motor** | 🟢 | 2026-06-08 | 08:00-helserapporten samler nå recovery→økt + kalender (m/ konflikt) + topp-3 tasks + vær. |
| **Abonnement-detektor** | 🟡 dvalende | 2026-06-08 | `/finance/advisor/subscriptions` — ferdig+testet, men `finance.transactions` tom (TODO #3). |
| **Vær-verktøy (get_weather)** | 🟢 | 2026-06-08 | yr/MET, default Oslo. |
| Styrkelogg / PT-regelbok / Health-Logg | 🟢 | 2026-06-08 | uendret + **PT-fiks:** daglig-kort bruker Strava-recency (ikke tom manuell logg) → anbefaler ikke gruppe trent i går. |
| Assistent (Jarvis) `/assistent` | 🟢 | 2026-06-08 | **34 verktøy** (la til search_emails, read_email, get_weather). Helse-nav lander på «Klar for økt». |
| Public state-mirror | 🟢 | 2026-06-08 | `mayo-os-state` · planleggeren leser den |

## 🟡 Pågår / delvis
- Web push + voice-router venter på Mayos engangs-oppsett (TODO #1, #2).
- Abonnement-detektor dvalende til bank kobles (TODO #3).

## 🔴 Åpne problemer
- **`finance.transactions` tom** — Enable Banking ikke koblet → finans-features dvalende (TODO #3).
- **Mac-Whisper-tunnel nede** (127.0.0.1:11436) → møte-transkribering bruker treg VPS-Whisper (TODO #5).
- **crm_task auto-task-bug fikset 06-08** (manglet `tags`-kolonne) — alle møte-tasks feilet stille; kolonne lagt til.

## 📋 Backlog (prioritert)
- **Jarvis-bygg:** Inc 0–3 ✅ live. Inc 4 (base-persona + 8 ekspert-skills + persona-ruter), Inc 5 (RAG) gjenstår.
- **Proaktive lag bygd 06-08:** dagbok-vokter, trend-vakt, morgenbrief, web push, voice-router ✅.
- IVF-tidslinje (lokal, krever Mayos tone-input) · inkasso-vakt + skatte-/likviditetsmotor (krever bank) · per-person møteforberedelse (krever Inc 5 RAG).
- Enable Banking-kobling · lokal modell-oppgradering · Obsidian-class editor.

## 🕐 Siste commits (nyeste øverst)
**Backend (`mayo-ai-os`):**
- `01a3f03` — voice-router /voice/jarvis (tale → Jarvis → verktøy) (06-08)
- `57b7a8d` — web push backend (VAPID) + wir inn i proaktive features (06-08)
- `6b148b5` — proaktiv dagbok-vokter (action items → tasks + Telegram) (06-08)
- `df4c716` — morgenbrief-motor (kalender+tasks+vær i 08:00-rapporten) (06-08)
- `4a61871` — trend-vakt (overtrening/sykdom-tidligvarsel) (06-08)
- `3d9aa59` — get_weather-verktøy (yr/MET) (06-08)
- `17f7da4` — abonnement-detektor (finance) (06-08)
- `ae3b8ea` / `5901c23` — Gmail-lesing (search_emails/read_email) + metadata-fiks (06-08)
- `77d42fe` — Whisper pre-warm-timeout (db-api henger ikke på oppstart) (06-08)
- `45fa1ef` — Gemini → gemini-2.5-flash + tool-fri prompt for plain-modeller (06-08)
- `6a132eb` / `b152131` — Inc 2 rute-matrise + C-knapp + lokal Gemma + pen feilmelding (06-08)
- `f358794` — Inc 3 anonymizer-round-trip (norsk NER, 0 rå PII) (06-08)
- `7c71ab3` — PT daily-kort bruker Strava-recency (Push-bug) (06-08)
- `1813cec` — Telegram-ukedag-fiks + dropp vaner/mat-reminders (06-08)
- `d9255cf` — monitor: morning-sjekk → calendar-briefing-daily (falsk alarm) (06-08)
- `ec9fdd8` / `2086624` — Inc 1 DB-historikk + fjern duplikat profile.py (06-08)

**Frontend (`mayo-os`):**
- `08fd963` — web push (service worker + abonnement + «Varsler»-knapp) (06-08)
- `20c4489` — Helse-nav → «Klar for økt» + Helse-dashboard-lenke (06-08)
- `eb273fc` — modell/rute-velger + transparens-indikator (Inc 2) (06-08)
- `f1ba4d2` — DB-historikk i Assistent (Inc 1) (06-08)

## 📝 Til planleggeren (claude.ai)
- **Stor Jarvis-økt 06-08 (autonom + interaktiv):** Inc 1–3 ferdig+live (minne, ruting+lokal Gemma, anonymizer m/ norsk NER). Det **proaktive laget** bygget: dagbok-vokter (journal → auto-task + varsel, lokal/🔴-trygt), trend-vakt (Whoop-tidligvarsel), morgenbrief-motor (én samlet 08:00-leveranse). **Gmail-lesing** + **iOS Web Push** (VAPID, self-hosted) + **voice-router** (`/voice/jarvis`: tale → Jarvis → verktøy). 5 bugfikser (PT Push-bug, Telegram-ukedag, vaner/mat, Helse-nav, monitor-alarm) + Whisper-oppstartsfiks + crm_task.tags-bug + 2 stuck Obs-Bygg-møter gjenopprettet.
- **Mønstre/lærdom:** SSH-multipleks satt opp (`~/.ssh/config`, ControlMaster) — unngår rate-ban ved mange koblinger. db-api restart: KUN `sudo -n systemctl restart db-api` + poll. Heredoc med `$1`/quotes brytes av shell → bruk scp'd script-filer. 🔴 = ivf/økonomi aldri rått til sky (lokal Gemma eller anonymisert). Obs BYGG-møter = jobb (Claude OK, ikke 🔴).
- **Venter på Mayo:** se «👉 HVA MAYO MÅ GJØRE» øverst.
- **IVF:** nedprioritert i kommunikasjon (Mayos ønske) — viktig fremover, men tonen må kalibreres med ham, ikke gjettes.
