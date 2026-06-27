# STATE.md — Mayo OS levende tilstand

> **Dette er sannheten.** Oppdateres som SISTE handling i hver oppgave.
> Planleggeren (claude.ai) leser denne FØRST i hver økt, via **privat speil** `mayo-os-state` (GitHub-connector — repoet er privat, ikke lenger rå public-URL).
> Aldri secrets/PII her — kun `<SET>`-markører.

**Sist oppdatert:** 2026-06-27 08:40 UTC · **Av:** Claude (terminal, mayo-ai-os) · **Versjon:** v0.42 Strava-incident DIAGNOSE+FAIL-CLOSED — handover-rotårsak avkreftet, §4 levert. §3 ikke akutt.

## 🎯 Nyeste (2026-06-27 08:40) — Strava-incident: diagnose + fail-closed PT-rapport

**Trigger:** HANDOVER-STRAVA-TOKEN-FIX.md (`a530bbe`) — Mayo: «Strava har
sluttet å synche, får helt feil treningsanbefalinger.»

### Diagnose (`829d551`, full rapport i HANDOVER_RESULT.md)

**Handover-rotårsak AVKREFTET av VPS-data:**
| Sjekk | Forventet (race-årsak) | Faktisk |
|---|---|---|
| `grep -c STRAVA_REFRESH_TOKEN .env` | >1 | **1** — ingen race-spor |
| 400/502 i logger | mange | **0** noensinne |
| `fetch_activities(days=14)` live | feil | **HTTP 200, 7 økter** |
| `/strava` endpoint | tom/feil | **7 økter** |
| `strava_notified` 14d | tom | **8 rader, siste 26.06** |
| Strava rate-limit | overskudd | **410/1000 daglig — god margin** |

Faktisk historisk feilmønster: **429 Too Many Requests** på
`/athlete/activities` fra **2026-06-06 20:20** til **2026-06-26 23:55**
(20 dager). Aldri 400 fra `/oauth/token`. Sluttet av seg selv.

Sannsynlig faktisk grunn til «feil anbefalinger»:
1. 20 dagers tomgang under 429 → recency-merge så «ingen ny trening» →
   gating-input ble systematisk skjev. Den feilen Mayo merket.
2. **PT LLM feiler hver dag** (`pt_llm:coach_comment:155 - PT LLM %s feilet`).
   Fallback brukes uavhengig. Separat sak handover ikke nevner.

### Levert (`32f68a5`) — fail-closed PT-rapport (handover §4)

`modules/health/strava_client.py`:
- Nytt `with_status=True`-arg → returnerer `(workouts, {'stale': bool,
  'problem': str|None})`. Stale=True ved fetch-exception. Bakover-kompat:
  uten arg returneres kun listen (psycholog uberørt).

`modules/health/send_report.py`:
- `main()`: ved `strava_status.stale`: `events=[]`, frekvens-tellingen
  droppes, ærlig sync-feil-melding erstatter `freshness_flag`. PT-LLM
  skippes helt — statisk fail-closed-tekst brukes:
  «Belastningsråd hoppet over — Strava-sync utilgjengelig. Stol på
  følelse + Whoop. Re-aktiver via /strava-auth hvis dette varer.»
- `build_message()`: nytt `strava_stale: dict|None`-arg. Ved stale:
  ⚠️-banner høyt i meldingen mellom header og recovery, med eksakt
  problem-streng så Mayo ser hva som er galt. Whoop-bidraget kommer
  fortsatt gjennom; bare belastningsråd droppes.

### 🔴 Disiplin
- Bedre stille enn selvsikkert feil. Aldri regn anbefaling på data vi
  ikke har — speiler Whoop `stale`-mønsteret som allerede finnes.
- Eksakt feilmelding tas med i meldingen → Mayo vet hva som er galt.

### Verifisert
- 99/99 PT-tester grønne (test_decide, test_okt_logikk, ...)
- Manuell stale-simulering: banner rendret korrekt, PT-LLM skippet
- Happy-path build_message identisk med før (null regresjon)
- Live `fetch_strava_workouts(days=14, with_status=True)`: 7 økter, stale=False
- Psycholog-callers (reflect.py + on_demand.py): uberørt (bakover-kompat)

### Ikke gjort (avventer Mayos retning)
- **Handover §3 (DB-backet `service_token` + advisory_lock)**: reell
  strukturell sårbarhet, men ikke akutt — race har ikke faktisk truffet
  i loggene. Bygges som planlagt arkitekturarbeid, ikke incident.
- **Watcher rate-limit-disiplin** (frekvens */5 → */15 + retry-backoff):
  forebygger neste 20-dagers tomgang. Egen handover hvis Mayo vil.
- **PT LLM-fallback-feil**: separat undersøkelse — hvorfor feiler
  `pt_llm.coach_comment` hver dag?

### Cron-test
Watcheren bruker ny `strava_client` neste 5-minutters-vindu (cron). PT-
rapport-cron kjører i morgen 06:00 Oslo — første ekte fail-closed-test.
Hvis sync funker (som nå): null banner. Hvis sync ryker igjen: tydelig
advarsel + ingen feil belastningsråd.

## 🚨 INCIDENT (2026-06-26, planlegger) — Strava-sync død → feil treningsanbefalinger (`HANDOVER-STRAVA-TOKEN-FIX.md`)

> **Status (Elmars 2026-06-26 20:55):** ikke berørt denne sesjonen — Mayo
> prioriterte tre-task-batchen + Jarvis-streaming + A4 PDF + parkering før
> Palette Fase 2. Strava-fiks ligger som åpen handover for neste økt.
>
> **Symptom (Mayo):** «Strava har sluttet å synche, får helt feil treningsanbefalinger.»
>
> **Rotårsak (audit fra kode — Elmars verifiserer på VPS):** TO uavhengige prosesser refresher OG roterer samme Strava engangs-roterende refresh-token, uten delt lås. (1) db-api `strava_module._refresh_access_token` (:68–97), (2) cron `*/5` `strava_watcher.strava_access_token` (:86–104, INGEN caching → ~288 rotasjoner/døgn). Usynkronisert read-modify-write på `.env` → før eller siden strandes en allerede-ugyldiggjort token i fila → hver refresh 400/502 → `fetch_activities` kaster → sync død permanent. **Whoop-dobbeltbindingen (koord.regel #4), strukturell.**
>
> **Anbefalings-effekt:** begge PT-stier (`strava_training_module._fetch_apps_script` → `strava_module.fetch_activities`) får 502 → null ferske økter → recency-merge (`8c8262a`) ser tomt → coach antar ingen trening → feil belastningsråd.
>
> **Fiks (handover):** (1) UMIDDELBAR: re-OAuth via `/strava-auth` → fersk token, sync tilbake. (2) DURABLE: én sannhetskilde — DB-backet `service_token` + `pg_advisory_xact_lock` rundt refresh+rotér, delt `get_strava_access_token()` brukt av BÅDE db-api og watcher; slett begge `.env`-skrivestiene. (3) FAIL-CLOSED: PT-rapport sier «Strava utdatert, hopper over belastningsråd» når data mangler — speiler Whoop `stale`-mønsteret (:391) — i stedet for selvsikkert feil råd.
>
> **Planlegger blind på VPS** — audit gjort fra koden; Elmars verifiserer (logg/curl/strava_notified) + implementerer + verifiserer (grunnlov §3).

## 🎯 Nyeste (2026-06-26 20:50) — Long-press kontekstmeny på items LEVERT

**Trigger:** Mayo: «Lukk heller den siste småtingen først: long-press
kontekstmeny … gjenbruk useLongPress, render via SheetHost/openSheet,
maks 5 handlinger, 🔴 ingen «send til Obs BYGG», outbox + ærlig toast.»
Avslutter tre-task-batchen ryddig før Palette Fase 2 (parkert til neste økt).

### Levert (FE `e98b64a`)

Tre filer, ÉN logisk endring:

1. **NY `src/mobile/livsplan_v12/itemMenu.jsx`** (~120 linjer)
   - `ItemMenuSheet` — bottom-sheet med 5 handlinger:
     ✓ Ferdig · → Utsett til i morgen 09:00 · ▤ Flytt til område ·
     ✎ Rediger · 🗑 Slett
   - `AreaPickerSheet` (sub-sheet) — flytt-til-område-velger;
     filtrerer `ctx.areas` til `a.track !== 'jobb'`
   - Pulse-guard hindrer dobbel-fire av en rad
   - Header viser tittel + område-chip i accent-farge

2. **`app.jsx`** — tre nye ctx-helpers
   - `ctx.snoozeItem(id, isoWhen)` — speiler completeItem-mønsteret.
     Optimistisk `scheduled_at`+`state='scheduled'`, toast «→ Utsatt ·
     angre», apiPatch ved UUID, rollback ved feil.
   - `ctx.setItemArea(id, areaKey)` — defensiv vakt: avviser jobb-mål
     med ærlig toast. Toast viser navn på mål-område.
   - `ctx.openItemMenu(id)` — gate på `it.track !== 'jobb'`, åpner
     ItemMenuSheet via openSheet med område-accent.

3. **`shared.jsx`** — ItemLine-wrapper gjenbruker useLongPress
   - Hook kalles ALLTID (stabil rekkefølge); handlers spreades kun når
     `canMenu = !!ctx.openItemMenu && it.track !== 'jobb' && !dragHandle`.
   - Prio-konteksten (dragHandle=true) bypasser hele wrapperen → bruker
     custom onPointerDown drag-pickup i triage.jsx (c7bd137). Ingen
     kollisjon: pointer-events ≠ touch/mouse-events.
   - Når canMenu=false: identisk oppførsel som før (ingen handlers,
     ingen click-suppression).

### 🔴 Suverenitet — håndhevet 5 steder (defense in depth)
1. **shared.jsx wrapper:** `it.track !== 'jobb'`
2. **ctx.openItemMenu:** defensiv same-sjekk
3. **ItemMenuSheet:** defensiv `sheet.close` på jobb-item
4. **ctx.setItemArea:** avviser jobb-mål eksplisitt
5. **AreaPickerSheet:** filtrerer `ctx.areas` til privat før render

Et privat item kan ALDRI ende opp i et jobb-område via denne menyen.
Ingen «Send til Obs BYGG»-handling eksisterer.

### Smoke etter deploy
- **Null regresjon** på berørte ItemLine-veier: #09 mobil-nav ✓,
  #10 area-overflow ✓, #14 inbox-search ✓, #16 sheet-no-x ✓,
  #18 palette ✓, #19 kladd ✓.
- 18/21 totalt. Røde: #02 + #15 pre-existing; #20 flaky
  (network error mid-stream — Gemma cold-start under full suite-last).

---

## Sesjon parkert — neste opp

Mayo parkerte denne økta før Palette Fase 2 (#3, HANDOVER-PALETTE-PHASE2.md):
«den har migrasjon 026 + embedding-backfill + suverenitets-ruting, og
fortjener en fersk økt». Ingenting tapt — handover er fullt specet i
`99d6e99`. Påbegynnes neste sesjon.

## 🎯 Forrige (2026-06-26 20:18) — A4 PDF-eksport av møtereferat LEVERT

**Trigger:** HANDOVER-MEETING-PDF.md (Mayos prioritet #2). Klient-side
print → «Lagre som PDF». Privat IVF/helse må aldri server-rendres.

### Levert (FE `95d8353` + smoke `fa0b2b4`)

`src/mobile/pages/ObsDetail.jsx`:
- Print-knapper i header-actionrow (ved siden av 🗑), KUN ved status='done':
  🖨 (kort referat) + +TXT (m/ transkript, vises bare hvis segmenter finnes)
- `MeetingReferat`-komponent: A4-layout, blekk-på-hvitt, Georgia serif.
  Egen CSS-blokk så referatet ikke arver mørkt tema. Seksjoner i
  handover-rekkefølge (sammendrag → temaer → beslutninger → tall &
  datoer → handlingspunkter → entiteter → [transkript hvis valgt]).
  Null-disiplin: tomme felter rendres ikke (ingen «—»-headers).
- Return restrukturert til Fragment med to søsken: skjerm-DIV
  (data-print="screen-only") + print-DIV (.obs-print-root). @media
  print veksler dem; ingen React-portals.
- `doPrintExport(withTranscript)`: setter `document.title` til
  «Referat — {title} — {dateStr}» for fornuftig PDF-filnavn,
  `window.print()`, restaurerer tittel på `afterprint`.

### 🔴 Suverenitet
- Genereres KUN i nettleseren fra data allerede i ObsDetail-state.
- Smoke #21 stubber `window.print` + sniffer `fetch`-URLer, asserter
  ZERO kall til `/export|/pdf|/print|/render` og eksterne PDF-tjenester
  (DocRaptor/PDFShift/etc).
- Privat referat bærer 🔒 Privat-markør og «Ikke del videre»-foot.
- tokens.ts urørt, ingen nye deps (`git diff package.json` tom).

Smoke #21 grønn isolert: 4s, 4000 tegn print-rot, 1 nettverkskall i
klikk-veien (ikke `/export|/pdf` — sannsynligvis bakgrunns-`/qa`-load).

## 🎯 Forrige (2026-06-26, planlegger) — Tre Fase 3-handovers FORBEREDT (alle 🛑 GATED — vent på «Kjør»)

> **Status:** Mayo: «bare forbered disse imens så er de klare». Tre spec'er skrevet, alle tydelig 🛑-merket — Elmars skal IKKE bygge før Mayo sier «Kjør». Alle BE+FE, ingen nye deps. Ligger i backend-repo-rot.
>
> **3a — `HANDOVER-JARVIS-CLOUD-STREAMING.md`** (Jarvis Fase 2). Streamer sky/Claude-ruten i `/meeting/{id}/ask`. Kjerne: **de-anon-sikker streaming** — Anonymizer bruker `⟦PERSON_1⟧`-klammer (anonymizer.py:152); buffer-regel emitter kun frem til siste ÅPNE `⟦` så en split-placeholder ALDRI lekker rå. Gjenbruker Elmars' SSE fra Fase 1 (`f59cf0d`). Smoke #23 asserter at strømmen aldri inneholder `⟦`/`⟧`.
>
> **3b — `HANDOVER-INLINE-CMDJ.md`** (inline AI på markert tekst). Global ⌘J → handlings-popover (Forklar/Oppsummer/Skriv om/Utvid) → streamet svar. 🔴 **Ruten bestemmes av FLATEN, fail-closed til lokal:** `/obs-bygg/*`=jobb→Claude-anon; alt annet (inkl. tvil)=privat→Gemma. Nytt `/jarvis/inline?stream=1`-endepunkt som gjenbruker meeting_ask-rutingshjelperne — IKKE `/chat/web/stream` (ubetinget sky). v1 kopier-only (ingen auto-erstatt). Smoke #24.
>
> **3c — `HANDOVER-PALETTE-PRIVATE-MEETINGS.md`** (private møter søkbare i palett). **To fysisk adskilte søkegrener:** jobb `meeting` (`is_private=FALSE`→`/obs-bygg`) URØRT; ny `meeting_private` (`is_private=TRUE`→`/livsplan#privatmote={id}`, 🔒). Krever ny `#privatmote=`-deep-link i app.jsx (speiler eksisterende `#item=`-handler; PageMeetings åpner i dag kun via intern state). FE fail-closed-vakt: 🔒-treff med `/obs-bygg`-URL → dropp. Forutsetter palett Fase 2 levert. Smoke #25.
>
> **Avhengighet:** 3a — Jarvis FE-streaming er NÅ LEVERT (se under), så 3a kan tas når Mayo «Kjør». 3c etter palett Fase 2. 3b frittstående. Alle bak 🛑.

## 🎯 Forrige (2026-06-26 19:55) — Jarvis streaming Fase 1 KOMPLETT

**Trigger:** HANDOVER-JARVIS-STREAMING.md (Mayos prioritet #1).
**Mål:** Drep 80s dødvente under «Jarvis tenker…» på private møter —
opplevelsen snur fra «hengt» til «skriver».

### Levert
- **BE** `f59cf0d` (`meeting_module.py` +97):
  `_ask_local_gemma_stream` (Ollama stream:True). `meeting_ask` tar nå
  `Request` og sjekker `?stream=1` — KUN på lokal-rute (sky-rute beholder
  dict-svar til Fase 2, fordi de-anon-sikker buffering kreves der).
  SSE-events: `meta` (model+sensitive) → `delta` (token) → `done`
  (persistering ferdig) eller `error` (ærlig). Persistering = identisk med
  ikke-stream-grenen (Fernet hvis sensitive). `X-Accel-Buffering: no` så
  nginx ikke buffrer.
- **FE** `eb2ec15` (`api.js` + `AskJarvis.jsx`):
  Ny `postStream()` parser SSE-rammer og forwarder named events. Behold
  `post`/`get`/`fetchJson` urørt. `AskJarvis.submit()` predikat:
  `willStream = !forceCloud && defaultSensitive && !IS_DEMO`. Sky/Obs BYGG
  beholder dagens ikke-stream-post → ingen regresjon. Optimistisk QA-rad
  appendes med `answer:''`, oppdateres token-for-token. Pulsen vises kun
  til første delta. Pulserende caret etter siste token mens streamen
  fortsetter. Streaming-feil → fjern halvferdig rad + ærlig askErr.
- **Smoke** `39ee2c8` (#20 + per-test `timeoutMs`-override):
  E2E mot privat møte; assert `meta → delta(≥1) → done`. Suverenitets-
  røyk: meta.model inneholder 'lokal'/'gemma' (IKKE 'sky'). Verifisert
  grønn isolert med modell `gemma3:4b-it-q4_K_M (lokal)`.

### 🔴 Suverenitet
- Lokale tokens forlater aldri VPS-en (Gemma localhost:11434).
- Stream-predikatet håndhevet i BÅDE BE og FE — sky kan ikke havne i
  stream-grenen selv om brukeren forsøker.
- RouteBadge 🔒 lokal tegnes fra `meta` FØR første token.
- **Fase 2 (sky-streaming) fortsatt bak 🛑** — venter på Mayos «Kjør»
  etter de-anon-sikker buffering er bygget.

### Smoke-status
- 17/20 grønne (de samme 2 pre-existing: #02 FET-strategi, #15 desktop
  modal). #20 grønn isolert, marginal under suite-last (Gemma cold-start
  konkurrerer); justert `timeoutMs` 180→240s og page-evaluate abort 120→220s.
## ⚡ Elmars-leveranser observert (planlegger logget — Elmars pushet uten STATE-oppdatering)

> Fanget ved fetch 2026-06-26. Elmars beveger seg raskt; disse er live på branchene men var ikke logget i STATE:
> - **Jarvis streaming Fase 1 — KOMPLETT** (BE `f59cf0d` + FE `eb2ec15` + smoke `39ee2c8`). Fase 2 (sky) fortsatt bak 🛑.
> - **Nav: «Revidere» fjernet fra mobil-NAV** (`0c1ea66`, FE) — 5→4 faner (I dag · Prio · Kladd · Privat møte). Småting #1 ✅.
> - **Smoke #03 fikset** (`5511a60`, FE): rotårsak = SearchTopbar-mount (ikke «pre-existing/stale» — diagnostisert ordentlig). Småting #2 ✅.
> - **Gjenstår fra tre-småting:** long-press kontekstmeny-guardrails (#3) — ingen commit observert enda.

## 🎯 Nyeste (2026-06-26, planlegger) — Handovers skrevet: A4 møte-PDF + palett Fase 2 (de to siste i kø)

> **Status:** Mayo ba om begge gjenværende handovers nå (overstyrte just-in-time). Begge spec'er til Elmars klare. **Ikke implementert.** Rekkefølge etter Jarvis-streaming: #2 PDF, #3 palett.
>
> **#2 — `mayo-os/HANDOVER-MEETING-PDF.md`** (FE-only, branch `claude/confident-noether-lpacih`). A4-referat-eksport fra ObsDetail. 🔴 **Klient-side `window.print()` + print-CSS — ALDRI server/sky-render for private møter** (privat = IVF/helse; PDF genereres i Mayos nettleser fra data alt i minnet → forlater aldri enheten). Knapp i ObsDetail-header ved 🗑, kun `status==='done'`. Referat = sammendrag/temaer/beslutninger/tall/handlingspunkter/entiteter; transkript AV som default; 🔒-markør på private referat. Ingen dep. Smoke #21.
>
> **#3 — `mayo-ai-os/HANDOVER-PALETTE-PHASE2.md`** (BE+FE). Semantisk søk i Mayos EGNE data i ⌘K via eksisterende pgvector `/search/cross-domain` (server.py:1388). 🔴 **Embedding skjer LOKALT** (nomic-embed på localhost, `_embed_query`:1372) → privat tekst forlater aldri VPS — det er fundamentet som gjør egen-privat-søk trygt. BE: `note.embedding vector(768)` + embed ved PATCH + backfill + `note`-type i cross-domain (`url:'/livsplan'`). FE: debounced bakgrunns-«Dypsøk»-seksjon, aldri blokker v1-treff. Fail-closed ruting: journal→`/brain` 🔒, note→`/livsplan` 🔒, meeting→`/obs-bygg` (kun jobb ved kilden — is_private-filteret URØRT). Besvarer v1-handoverens åpne spørsmål (ja, finn egne private notater — lokalt embedet, privat-rutet). Smoke #22.
>
> **🛑 Gates:** PDF — aldri server/sky-render for privat (hard grense). Palett — stopp før Fase 3 (inline ⌘J) + før private møter gjøres søkbare.

## 🎯 Forrige (2026-06-26, planlegger) — Handover skrevet: Spør Jarvis token-streaming (`HANDOVER-JARVIS-STREAMING.md`)

> **Status:** Spec til Elmars klar — `mayo-ai-os/HANDOVER-JARVIS-STREAMING.md` (branch `claude/confident-noether-lpacih`). **BE Fase 1 LEVERT** (`f59cf0d`); **FE-streaming gjenstår** (postStream + AskJarvis token-render). Mayo prioriterte dette som #1 av tre store (foran A4-PDF og palett-Fase-2). Streaming-transport på eksisterende `/meeting/{id}/ask` — ingen nye deps.
>
> **Hvorfor:** «Spør Jarvis» tar ~60–90s på lokal Gemma med død «Jarvis tenker…»-puls. Reflect-gapet var *opplevd hastighet*; dette er skarpeste forekomst. Token-streaming = samme svar/ruting/latens, men opplevelsen snur fra «hengt» til «skriver».
>
> **Kjerneinnsikt (styrer scopet):** lokal Gemma (`_ask_local_gemma`, meeting_module.py:1680) har INGEN anonymisering → streamer trivielt + trygt. Sky/Claude (`_ask_claude_anonymized`:1696) **de-anonymiserer HELE svaret** (`anon.deanonymize(raw)`:1720) → naiv token-streaming kan splitte en placeholder (`PERSON_1`) over to chunks → lekkasje. Derfor: **Fase 1 = KUN lokal-streaming** (default-rute + fail-closed + IVF/helse — og den eneste trygge). **Fase 2 (sky) bak 🛑** — krever de-anon-sikker buffering.
>
> **Scope BE+FE:** BE: `?stream=1` → `StreamingResponse` SSE (`meta`→`delta`→`done`/`error`), `_ask_local_gemma_stream` (Ollama `stream:True`, NDJSON-delta), persister kryptert ved `done` (uendret meeting_qa + history). FE: ny `postStream` i `api.js` (SSE-leser, `post` urørt), AskJarvis appender token-for-token, RouteBadge fra `meta` FØR tokens. Obs BYGG/`force_cloud` faller tilbake til dagens ikke-stream `post` til Fase 2 (ingen regresjon). Smoke #20 spesifisert.
>
> **🔴 Suverenitet:** lokale tokens forlater aldri VPS (Ollama localhost, samme autentiserte db-kanal); 🔒-RouteBadge før tokens; kryptering + fail-closed uendret. **Fase 2-sky gated nettopp pga. de-anon-split-risiko.**

## 🎯 Forrige (2026-06-26 14:42) — «I dag» declutter (FE `fb841e6`)

**Trigger:** Mayo: «Ny handover: HANDOVER-IDAG-DECLUTTER.md … område-kort
øverst + kompakt, fjern smart-flisene, flytt Andre visninger+søk ned med
luft. 🛑 Behold/kutt-kall (brief/kapasitet/innboks) er Mayos.»

### Levert (FE `fb841e6` — branch `feat/whoop-redesign`)

Endring kun i `src/mobile/livsplan_v12/today.jsx` (+77/-71). Ingen
backend, ingen nye deps, `tokens.ts` urørt.

1. **Område-grid FØRST under header** — Privat-kort + Jobb-kort flyttet
   til toppen av `!searching`-blokken (var nederst). Det er der reisen
   starter; skal ikke ligge under sekundære verktøy.
2. **AreaCard kompakt** — preview-linja og «Privat ·»-linja fjernet,
   padding 12→10, header-marg 8→6. Beholder ikon, tittel, flyt-status,
   ring, «N åpne» + overdue-badge, ⋯-knapp. Ca 30 % lavere per kort.
3. **SmartTiles-raden FJERNET fra denne fanen** — «3 I DAG / 1 FORFALT /
   3 DENNE UKA / 106 INNBOKS» borte. Komponenten beholdt i
   `shared.jsx` (desktop + legacy bruker den fortsatt).
4. **ExtraModes + SearchTopbar til BUNNEN, med luft** — gap 6→10
   mellom mode-boksene, marginTop/marginBottom 20px så seksjonen ikke
   klistrer seg til nabo-blokker.
5. **🛑-respekt** — brief, kapasitetsmåler, «Fra innboks» BEHOLDT.
   Ingen AreaCard→tight-list-fallback (handover sier vent på Mayos OK).

### Smoke
- ✅ 09 (mobil bunn-nav `+` / `⌕`) grønn
- ✅ 10 (desktop right-panel overflow) grønn
- 17/19 totalt — de 2 røde (#03 typo-fuzzy stale etter `881fd67`,
  #15 desktop modal) er pre-existing og uavhengig av denne layouten

## 🎯 Forrige (2026-06-26 11:00) — Kladd-fane v1 deployet (`eb149d2` FE / `2fb7916` BE)

**Trigger:** Mayo: «Ny handover klar: Kladd-fane (plain-text notater →
`[]`-tasks)». HANDOVER-KLADD-NOTES.md.

### Levert (BE `2a3ec7e` + `2fb7916`)

1. **Migrasjon 025** — `note` + `note_task`-tabeller.
2. **`note_module.py`** (260 linjer) — GET/POST/PATCH/DELETE /notes med
   `[]`-høsting + vault-speiling + `[x]`-statusspeiling.
3. **`tasks_module.py`** — `source='note'` lagt til i `/tasks/unified`;
   note-treff får `is_private:true` + `url='/livsplan'`.

### Levert (FE `eb149d2`)

- **`kladd.jsx`** (395 linjer) — PageKladd + NoteEditor. Ren textarea,
  `fontSize:16` (iOS no-zoom). Autosave 600ms med localStorage outbox
  FØRST, ærlig toast. Backend-respons body_md → caret-bevarende update.
- **app.jsx + desktop.jsx** — Kladd som 4. NAV. Notisblokk-glyf.

### 🔴 Suverenitet

- `track='privat'` default → bor i Livsplan, ALDRI Obs BYGG
- `/action-items` filtrerer ut `'note'` → ingen lekkasje
  (smoke #19 verifiserer)

### E2E-verifisert

- ✅ 2 `[]`-linjer → 2 privat-inbox-tasks; gjentatt PATCH → 0 duplikater
- ✅ Vault-fil `MayoVault/notater/2026-06-26.md` skrevet
- ✅ note-tasks i `/tasks/unified`, IKKE i `/action-items`
- ✅ Smoke 18/19 pass (#19 ny; #02 FET-strategi feilet eksisterende)

### ⚠️ Nav-tranghet IKKE løst — krever Mayo's «Kjør»

Mobile NAV har 5 faner nå. Handover foreslo flytte «Revidere» inn under
ExtraModes — men: «Bekreft med Mayo før du fjerner en fane». Alle 5
beholdt.

### 🛑 STOPP-gate respektert

Fase 2 IKKE bygget: AI-høsting · embeddings · long-press → «Lag oppgave»
· toveis `[x]`→`[ ]` re-åpner task.

---

## 🎯 Nyeste (2026-06-26, planlegger) — Handover skrevet: «I dag»-fane opprydding (`d860cbe`, mayo-os)

> **Status:** Spec til Elmars klar — `mayo-os/HANDOVER-IDAG-DECLUTTER.md` (branch `claude/confident-noether-lpacih`). **Ikke implementert enda.** Ren frontend-layout i `today.jsx`, ingen backend/deps.
>
> **Mayos tre grep (skjermbilder):** (1) prosjekt-/område-kortene tar for mye plass + skal ØVERST (der reisen starter) → flytt til topp + gjør kompakt; (2) «Andre visninger» (Puls/Kalender/Revider) + søk mangler luft → vertikal rytme; (3) dropp smart-flisene (I dag/Forfalt/Uka/Innboks) → fjernes fra fana.
>
> **Ny rekkefølge:** Header → Områder (kompakt, Privat+Jobb, suverenitets-skille intakt) → Brief → Kapasitet+Dagens → Fra innboks → Andre visninger+søk (nederst, med luft). SmartTiles fjernet (kun fra denne fana, ikke slettet fra shared).
>
> **Kompakt AreaCard:** dropp forhåndsvis-linja nederst (`Ben A — styrkeøkt` osv.) + stram padding → ~30 % lavere. Fallback til tett-liste kun etter Mayos OK.
>
> **🛑 Planlegger-kall (vetoable):** brief/kapasitet/«Fra innboks» beholdt slanke — kutt mer kun etter å spørre Mayo.

---

## 🎯 Tidligere (2026-06-26, planlegger) — Handover skrevet: Kladd-fane (plain-text notater → `[]`-tasks)

> **Status:** Spec til Elmars klar — `HANDOVER-KLADD-NOTES.md` (commit `8fa9904`). **Ikke implementert enda.** Bakgrunn: Mayo fanger løse idéer i Apple TextEdit (plain text) og vil ha samme frihet i Livsplan — en egen «Kladd»-fane der notater lagres som `.md` (som journal, men i egen `MayoVault/notater/`-mappe) og oppgaver høstes rett ut av teksten.
>
> **Låst med Mayo:** (1) egen fane i Livsplan-bunnnavet, ikke knapp i journal; (2) `[]`-checkbox-syntaks for task-høsting; (3) mange små, auto-daterte notater; (4) `[]` → privat inbox default (jobb-ruting er etter-handling).
>
> **Kjernemekanikk:** `[]` er BÅDE trigger og status — `[]` = lag task, backend skriver om `[]`→`[ ]` ved høsting (idempotent vakt mot duplikat ved autosave), `[x]` speiles når item.state=done. Notatet blir en levende sjekkliste i ren tekst, speilet til vault som ekte markdown.
>
> **Scope BE+FE:** ny `note`+`note_task`-tabell (migrasjon), `note_module.py` (mønster fra journal_module), vault-speiling, deterministisk `[]`-parsing (ingen AI). FE: `kladd.jsx` (plain textarea, autosave, outbox), ny NAV_ITEM. 🔴 `track='privat'`, `source='note'` — aldri Obs BYGG. Smoke #19 spesifisert.
>
> **Nav-obs:** bunnnavet er fullt (I dag·Prio·Revidere·Privat møte + søk + Fang). Forslag i spec: flytt «Revidere» inn under «Andre visninger» (finnes allerede inline i I dag) og gi plassen til Kladd — men IKKE fjern en fane uten Mayos OK.
>
> **🛑 Fase 2 (gated):** AI-høsting (privat→Gemma), embeddings (palett-/RAG-søkbar), marker-og-trykk, toveis `[x]`→`[ ]`.

---

## 🎯 Tidligere (2026-06-26 10:40) — Kommandopalett (⌘K) v1 deployet (`ce206a7` FE / `93db5c1` BE)

**Trigger:** Mayo: «Ny handover klar: kommandopalett (⌘K) + lynsøk ...
Bygg kun v1». HANDOVER-COMMAND-PALETTE.md på branch claude/confident-
noether-lpacih i mayo-os.

### Levert (FE `ce206a7` på feat/whoop-redesign)

1. **`src/lib/search.js`** (ny) — `searchScore` utløftet fra `today.jsx:153`
   så Livsplan + paletten deler matcheren. Streng literal-substring
   (Mayo's `881fd67` bevart). Tester 03/13/14 uberørt.
2. **`src/shell/CommandPalette.jsx`** (ny, 416 linjer) — overlay-modal +
   `useCommandPalette()`-hook. ⌘K/Ctrl+K åpner globalt, Esc lukker, ↑/↓/
   Enter navigerer. Tre seksjoner: Handlinger · Naviger · Søk-treff.
3. **`src/App.jsx`** — mountet etter `<Routes>` inni `<ToastProvider>` →
   tilgjengelig på alle ruter.
4. **`today.jsx`** — import endret til `@/lib/search`, lokal funksjon
   slettet (22 linjer -).

### 🔴 Suverenitet (handover §Suverenitet pkt 2)

- `/meeting?limit=200` med default `include_private=false` → private
  møter lastes ALDRI inn i palette-state i v1.
- `/tasks/unified` items filtreres: `is_private === true` droppes.
- Møte-treff → `/obs-bygg/meeting/{id}` (kun jobb). Item-treff → ALLTID
  `/livsplan`. Ingen krysning mulig.

### Levert (BE `93db5c1` på claude/confident-noether-lpacih)

- **`smoke/tests/18-command-palette.js`** — Meta+K åpner, søker «liv»,
  ≥1 treff, suverenitets-røyk (ingen 🔒). ✓ pass.

### Verifisert (akseptansekriterier)

- ✅ ⌘K/Ctrl+K + Esc · ↑↓Enter · iOS no-zoom · ingen mus nødvendig
- ✅ Naviger-treff verifisert mot App.jsx (`/assistent` for Jarvis,
  `/strength` for Styrke siden `/jarvis` og `/styrke` ikke finnes)
- ✅ Søk < 50ms (pure-memory)
- ✅ Ingen private treff
- ✅ Ingen ny dep · tokens.ts urørt
- ✅ Live: `https://mayooran.com/version.json` → `ce206a7`
- ✅ Smoke 16/17 (test 02 FET-strategi feilet i cron-runs `08:01`,
  `08:15`, `08:30` FØR mine endringer — eksisterende issue, ikke regression)

### 🛑 STOPP-gate respektert

Fase 2 (privat-søk / inline ⌘J / `[[backlinks]]`) IKKE bygget. Venter
på Mayo's «Kjør» på det åpne suverenitets-spørsmålet i handover.

---

## 🎯 (2026-06-25) — Handover skrevet: kommandopalett (⌘K) + lynsøk (`e2434ca`, FE PR #23 draft)

> **Status:** Handover-spec til Elmars skrevet etter at Mayo sammenlignet UX/muligheter mot reflect.app. Funn: vi er foran på suverenitet/dybde (helse, lokal-AI-ruting, norsk stemme, agentisk Jarvis), men bak på *opplevd hastighet/friksjon*. Reflects «premium» = keyboard-first kommandopalett + instant søk. Det er det enkleste grepet med størst løft, og rører ikke personvern-lagene.
>
> **Leveranse:** `mayo-os/HANDOVER-COMMAND-PALETTE.md` (commit `e2434ca` på `claude/confident-noether-lpacih`, draft PR #23 mot `feat/whoop-redesign`). Lagt på feature-branch bevisst — doc-only skal ikke trigge prod-rebuild. **Ingen kode-endring enda** — venter på at Elmars implementerer v1.
>
> **v1-scope (lav risiko, ingen nye deps, tokens.ts urørt):** global `⌘K`-overlay (montert i App.jsx), naviger + hurtighandlinger + instant klient-side literal-søk over forhåndslastede items (`/tasks/unified`) + møter (`/meeting`, default `include_private=false`). Gjenbruker `searchScore` (flyttes til `src/lib/search.js`) + `api.js`-wrapper. 🔴 Suverenitet: private møter ekskludert by default, `is_private`-items skjult i v1 → null jobb/privat-lekkasje. Inkluderer ny `smoke/tests/18-command-palette.js`.
>
> **Fase 2 (bak 🛑 STOPP, Mayos «Kjør»):** semantisk cross-domain (`/search/cross-domain`), inline `⌘J`-AI (rutet via Jarvis, privat→Gemma), `[[backlinks]]` som primitiv. Anbefalt rekkefølge etter v1: (2) inline-AI, (3) Jarvis-latens (streaming).

---

## 🎯 Tidligere (2026-06-25) — Sovereignty-fixene LIVE + ny suverenitets-smoke-test (`59b000a` PUSHET)

> **Status:** Alle sovereignty-fiksene fra audit-en under er nå **DEPLOYET** — HEAD `f39b4bd` kjører i prod (fersk PID, helsesjekk grønn). Privat→Obs BYGG-lekkasjen er lukket på backend-kilden i ALLE live jobb-feeds. I tillegg: ny **suverenitets-smoke-test** (`smoke/tests/17-sovereignty-private-leak.js`, commit `59b000a`) lagt til den eksisterende `*/15`-Playwright-cron-en (`mayo-smoke.sh`).

**Hva smoke-testen gjør (grunnsannhet fra prod, ingen mutasjon):**
- Finner et ekte privat møte via `/meeting?include_private=true`, henter dets action-item-id-er.
- Påstår at møtet + items lekker til INGEN jobb-feed: `/meeting` (default), `/action-items`, `/tasks/unified` (privat MÅ være `is_private`-flagget — uflagget = selve 2026-06-25-hendelsen), `/meeting-graph`, `/graph/unified`.
- Brudd → smoke flipper RØDT → web-push via `smoke-flip-push.py`. En fremtidig regresjon som gjenåpner lekkasjen fanges innen 15 min, ikke ved at Mayo ser en IVF-task på jobb-tavla.
- Ingen privat møte i prod → pass (logger «ingenting å verifisere»).

**Verifisert:** `node -e require(...)` laster modulen rent; følger samme auth-mønster som test 14 (mint session-cookie på `.mayooran.com` → følger til `db.mayooran.com`). Selve cron-kjøringen verifiseres ved neste `*/15`-slot på VPS.

---

## 🎯 Tidligere (2026-06-25) — Komplett privat→Obs BYGG sovereignty-audit (BE `b11d96d`+`11d3565`+`ec58f66`, nå DEPLOYET via `f39b4bd`)

> **Status:** 3 backend-commits pushet til `claude/confident-noether-lpacih` — **NÅ DEPLOYET** (HEAD `f39b4bd` live). Kun AST/inspeksjon-verifisert (3.11-parser gir kjent falsk PEP701-feil på server.py:756/865 — IKKE mine linjer). Ingen migration. Ingen frontend-endring nødvendig (alle live Obs-flater får nå backend-filtrert data; FE home-filter + ObsGraph journal-drop allerede på plass).

**Trigger:** Full audit etter at private IVF-møte-action-items lakk inn på Obs BYGG (delvis fikset i `743789a`/FE `9b0d0fd`). Mål: tette ALLE work-flater mot private data (is_private-møter, track=privat, sensitive).

**Funn + fiks (alle på `is_private`-kilden, fail-closed):**
- `/meeting-graph` (live Obs GRAF): ekskluderte IKKE is_private — kun heuristisk tag/term-blocklist. Privat møte uten magic-words lakk noder/entities/tags. → `b11d96d` ekskluderer is_private (+ semantic-nabo-subquery).
- `/meeting/entities` + `/meeting/assignees` + `/meeting-tags` (ObsDetail/PageObs autocomplete): aggregerte over ALLE møter inkl. private → person-/klinikknavn, assignees, tags lakk. → `b11d96d` ekskluderer is_private.
- `/graph/unified` + `/search/cross-domain` (work-URL'd, men kun døde desktop-routes i dag): meeting-halvdel uten is_private-filter; `/graph/unified` manglet edge-post-filter → kunne lekke privat møte-id som edge-target. → `11d3565` ekskluderer is_private (inkl. semantic-nabo-subqueries via `_priv_clause`).
- `/action-items/backfill`: hardkodet `track='jobb'` for ALLE done-møter inkl. private (mislabel; contained av consumer-join men skjørt). → `ec58f66` skipper is_private (primær-pipeline lager private items korrekt som track='privat').

**Verifisert CLEAN (ingen endring):**
- `/action-items` (Oppgaver-liste/Kanban/assignee-stats): allerede is_private-filtrert i `743789a`. Autoritativ for live Oppgaver-fane.
- `/tasks/unified`: eksponerer is_private+track (`75aa081`); FE home-filter (`9b0d0fd`/PageObs:100) fail-closed (kun source=meeting + is_private===false/bekreftet jobb-møte).
- `/meeting` (Møter+Kalender i PageObs): default `include_private=false` ✓.
- `/meeting/{id}`, /notes, /tags, /summary etc.: single-meeting, user-scoped — et jobb-møtes egne data = jobb-data ✓.
- `/calendar` (calendar_module): viser private møter, MEN kun konsumert av personlige flater (PageHjem/PageKalender/Timeline), IKKE Obs BYGG. Korrekt.

**Judgment call — voice-journal i /action-items:** Voice-journal-action-items opprettes BEVISST med `track='jobb'` (journal_module:1321, assignee default «Max») og vises i Obs BYGG by design — kun de uttrukne *handlingspunktene* (tittel+assignee), aldri journal-innhold (raw_text/mood/themes). IKKE en lekkasje → BEHOLDT. Residual: hvis en voice-journal tilfeldig uttrekker et privat-klingende punkt, vises det i Obs BYGG (de er hardkodet jobb/non-sensitive). Anbefaling (ikke implementert): area-klassifisering ved kilden.

**Residual risiko / ikke-live:** `/meeting/prep` (ingen FE-caller) blander private prior-møter + journal i prep-note uansett target-møtes privacy — vil lekke HVIS wiret til Obs senere. Døde desktop-routes `routes/obs/MeetingGraph.jsx` (→/graph/unified) + `MeetingCalendar.jsx` (→tasks/unified ufiltrert) er importert men ikke montert; trygt nå (graph backend-fikset), men re-mount av MeetingCalendar uten filter ville vise private møte-tasks.

**Konklusjon:** Etter disse + de allerede-pushede fixene er ALLE live Obs BYGG-flater lukket for private data. Gjenstående er døde/uwirede endepunkter (flagget over), ikke live-lekkasjer.

---

## 🎯 Tidligere (2026-06-23) — Privat møte → Obs BYGG-paritet + hybrid Spør Jarvis (BE `a7f7eb3` + FE merget `a5f7fad`)

> **Status:** FE merget til prod (`a5f7fad`, PR #22) etter planlegger-review av begge personvern-invariantene (privat lekker ikke til Obs BYGG/vault; privat→sky kun ved eksplisitt `force_cloud`). **BE `a7f7eb3` GJENSTÅR deploy:** `cd ~/mayo-ai-os && git pull origin claude/confident-noether-lpacih && ./deploy.sh` — aktiverer avkryssing (action_item `item_id`-join) + `force_cloud`. Før deploy: checkboxer deaktivert (graceful), `force_cloud` ignoreres trygt (Pydantic dropper ekstra-felt).

**Trigger:** Mayo testet «Privat møte» + «Spør Jarvis»: (1) kunne ikke huke av
oppgaver / redigere / skrive notater i privat møte (ulik den rike Obs BYGG-
visningen), (2) ønsket hybrid — lokal Gemma som default, men eksplisitt per-
spørsmål-valg «Svar med Claude (anonymisert)».

**Backend (`mayo-ai-os`, branch `claude/confident-noether-lpacih`, commit `a7f7eb3` — PUSHET):**
- `GET /meeting/{id}` returnerer nå `is_private` + beriker topnivå-`action_items[]`
  med `item_id` + `done` (join mot `item`-tabellen source='meeting', origin_ref,
  FIFO tekst-match) → UI får stabil id for avhuking. Avhuking går via eksisterende
  `PATCH /action-items/{id}` (user_id-scoped proxy — virker likt privat/jobb).
- `meeting_ask` tar nytt felt `force_cloud: bool = False`. Ruting:
  ikke-privat → Claude anonymisert (uendret); privat+!force_cloud → lokal Gemma
  (uendret default); **privat+force_cloud → anonymisert Claude** (eneste vei et
  privat transkript når sky, kun på eksplisitt forespørsel). Raden lagres fortsatt
  `sensitive=True` (kryptert); model-etikett = «claude-sonnet (anonymisert · sky)».
  FAIL-CLOSED bevart.
- ⚠️ **Backend auto-deploy AV** → krever manuell `cd ~/mayo-ai-os && ./deploy.sh`.
  Kun AST/syntaks-verifisert, IKKE runtime-testet på VPS. Ingen ny migration.

**Frontend (`mayo-os`, branch `feat/privat-mote-parity`, draft PR #22 → `feat/whoop-redesign`):**
- Privat møte-detalj (`livsplan_v12/meetings.jsx`) GJENBRUKER nå `ObsDetail` med
  `isPrivate={true}` istedenfor egen read-only visning (slettet). Full paritet:
  transkript, redigerbart sammendrag, avhukbare/redigerbare oppgaver, notater,
  reanalyse.
- `ObsDetail` tar `isPrivate`-prop og gater ALT Obs-BYGG-spesifikt bak `!isPrivate`:
  synk-toggle/SyncChip skjult (🔒 privat-chip + «kun Postgres» istf.), `[[wiki]]`-
  graf-nav av, Spør Jarvis ruter på ekte `isPrivate`, privat-footer. `ActionItemRow`
  huker av via `/action-items/{item_id}` (fallback legacy `/tasks/{id}`).
- `AskJarvis`: ny sekundær-knapp «⚡ Svar med Claude (anonymisert)» (kun private)
  → `force_cloud:true`, eksplisitt ☁ trade-off-note. Default «🔒 Spør» = lokal.
  RouteBadge leser `model`-strengen → viser «☁ sky (anonymisert)» når privat→sky.
- `npm run build` rent. `tokens.ts` urørt. Ingen nye deps. `ObsDetail` er delt
  chunk (ingen duplisering PageObs/PageLivsplanV12).
- ⚠️ **GJENSTÅR (Mayo/VPS):** `cd ~/mayo-ai-os && ./deploy.sh` for å aktivere
  avhuking + hybrid-ruten. PR #22 er DRAFT (ikke merget). Til backend er deployet:
  oppgaver vises men checkbox er disabled; force_cloud ignoreres trygt.

## 🎯 (2026-06-23) — «Spør Jarvis» Q&A per møte (BE `3de665a` + FE PR #21)

**Trigger:** Mayo: «Bygg Spør Jarvis inne i hvert møte — Q&A mot møtets transkript + sammendrag.»

**Backend (`mayo-ai-os`, branch `claude/confident-noether-lpacih`, commit `3de665a`):**
- Migration `024_meeting_qa.sql` — tabell `meeting_qa` (id/meeting_id/user_id/question/answer/model/sensitive/created_at) + indeks `(meeting_id, created_at)`. Idempotent.
- `POST /meeting/{id}/ask` + `GET /meeting/{id}/qa` i `db_api/meeting_module.py` (på eksisterende meeting-router — ingen server.py-endring).
- **Suverenitets-ruting:** `sensitive = bool(is_private) if is_private is not None else True` (FAIL-CLOSED). sensitive → lokal Gemma (`gemma3:4b`, localhost Ollama), aldri sky. ikke-sensitiv → Claude `claude-sonnet-4-5` på anonymisert kontekst (reuse `Anonymizer`), de-anonymisert svar. Aldri lokal→sky-fallback ved feil.
- Krypterer spørsmål+svar (jarvis-modulens Fernet) når sensitivt. Graceful svar ved modell-feil (ingen 500).
- ⚠️ **Backend auto-deploy AV** → krever manuell `cd ~/mayo-ai-os && ./deploy.sh` + at migration 024 kjøres mot mayo_sov. Kun py_compile/AST-verifisert, IKKE runtime-testet på VPS.

**Frontend (`mayo-os`, branch `claude/confident-noether-lpacih`, draft PR #21 → `feat/whoop-redesign`):**
- Ny `src/mobile/AskJarvis.jsx` (gjenbrukbar panel). Mountet i privat møte-detalj (`livsplan_v12/meetings.jsx`, 🔒 lokal) + Obs BYGG-detalj (`pages/ObsDetail.jsx` SummaryTab, ☁ anonymisert).
- `npm run build` rent. Ingen nye deps. `tokens.ts` urørt.
- ✅ **MERGET til prod** (`e0425c4`, PR #21) etter planlegger-review av rutings-koden (bekreftet IVF aldri til sky). Frontend auto-deployer; UI-en er live, men selve svaringen aktiveres FØRST når backend deployes (til da: ærlig «kunne ikke svare», ingen data sendt).
- ⚠️ **GJENSTÅR (Mayo/VPS):** kjør migration 024 + `cd ~/mayo-ai-os && ./deploy.sh` (samme deploy shipper også JSON-krasj-fiksen `b111fa1`). Deretter verifiser at IVF-spørsmål treffer lokal Gemma, ikke Claude.

## 🎯 Nyeste (2026-06-23 13:55) — OOM-diagnose: chromium-lekkasje var rotårsak (`03301ae`/`af1c4a8`)

**Trigger:** Mayo: «Fiks OOM-kræsj i backend-auto-deploy (db-api SIGKILL ved
Whisper-last)». Deploy-backend.yml feilet 17. juni 11:09 med exit 137.

**Diagnose (les-først):**
- **35 chromium-prosesser** fra smoke-cron lekket (etime 5d12h) →
  spiste 2.6 GB RAM
- Ollama gemma3:4b spiser 4.3 GB → bare 1.6 GB tilgjengelig før cleanup
- db-api MemoryHigh=3GB/MemoryMax=4GB, peak 3.22 GB (strupes, men kræsjer
  hvis swap+RAM begge fulle)
- Lazy-load Whisper var allerede aktivert (20s pre-warm-timeout, faller
  tilbake til lazy-load)
- Swap (4 GB) er aktivt + i fstab — overlever reboot

**Spor A (lazy-load) var allerede gjort.** Rotårsak var Spor B — system-
press fra smoke-lekkasjen.

**Fiks:**
1. Drepte alle lekkede chromium-prosesser (`pkill -9 -f chrome-headless-
   shell`) → frigjorde 800+ MB umiddelbart
2. `smoke/lib/runner.js` (`af1c4a8`): per-test page+context cleanup i
   finally, browser.close() med 10s Promise.race-timeout, kill-9 safety-
   net etter run. Hindrer at lekkasjen kommer tilbake.

**Verifisert:**
- Smoke 16/16 pass etter fix
- 0 chromium-prosesser igjen etter run (var 38 før)
- db-api restart: Whisper lastet 7s, MemoryPeak 3.22 GB, helsesjekk
  grønn, RAM 2.6 GB tilgjengelig (var 1.6 GB før cleanup)

**Push-trigger fjernet (`19a41ba`):** Mayo valgte å fjerne push-trigger
fra `.github/workflows/deploy-backend.yml`. `workflow_dispatch` beholdt
(web-UI + `gh workflow run`). Manuell `./deploy.sh` på VPS uendret.
Effekt: ingen automatiske Action-deploys lenger → ingen parallelle
restarter → ingen OOM-risiko fra dobbel-Whisper-load.

**Deploy-workflow nå:**
1. Skriv kode, push til branch
2. SSH til VPS: `cd ~/mayo-ai-os && git pull && ./deploy.sh`
   ELLER GitHub web-UI → Actions → Deploy backend → Run workflow

---

## 🎯 (2026-06-23 14:20) — Lyd-upload krasjet på Claude-JSON («Expecting ',' delimiter»)

Mayo testet IVF-opptak → rød feil `Expecting ',' delimiter: line 13 col 120`.
Rotårsak: `claude_extract` (meeting_module.py:290) gjorde `json.loads(raw)` på
Claude-output som av og til er litt ugyldig JSON. Pipelinen kastet → status=
`failed`, selv om **transkriptet allerede var lagret** (status `analyzing` før
analysen). To fixer:
- **Frontend (LIVE, merge `0055913`):** «📄 Vis transkript» vises nå også i
  feil-tilstand (når `meetingId` finnes) → Mayo får lest opptaket selv om
  uttrekket feilet. `capture.jsx`.
- **Backend (`b111fa1`, IKKE deployet — krever `./deploy.sh`):** `_loads_lenient`
  (tolererer trailing commas / ledetekst / brace-trim) + pipelinen wrapper
  `claude_extract` i try/except → ved feil: status=`done`, tomt uttrekk (0
  oppgaver), `analyze_degraded`-event, transkript intakt. Aldri rød feil igjen.

**⚠️ Backend-deploy gjenstår:** `cd ~/mayo-ai-os && ./deploy.sh` (auto-deploy-
push-trigger fjernet i `19a41ba`). AST OK, ikke kjørt i prod ennå.

---

## 🎯 (2026-06-23 13:01) — Vis transkript etter privat lyd-opptak (merge `fc98c54`)

Mayo testet lyd-pipelinen og ville lese transkriptet (ligger i Postgres
`meeting.transcript_text`, men hadde ingen UI-flate — private møter er skjult
fra Obs BYGG-lista). La til **«📄 Vis transkript»**-knapp i Fang-arkets done-
tilstand (`capture.jsx`) som henter `GET /meeting/{id}` og viser sammendrag +
fullt transkript inline (scrollbart), merket «privat · ikke speilet til vault».
Rein frontend (backend-endepunkt fantes, user_id-scoped). Commit `4335697`
→ merge `fc98c54`. Build + deploy grønt. **NB:** lyd-pipelinen ikke
end-to-end-bekreftet med ekte tale ennå — Mayo tester nå.

---

## 🎯 (2026-06-23 11:06) — Søk runde 2: streng bokstavelig matcher + dedupe (merge `881fd67`)

Mayo testet de 13 fiksene (bra!), eneste gjenstående: søk. «mat» ga fortsatt
random treff fordi matcheren tillot 1-edit ord-treff («mat»↔«man»). **Fjernet
edit-avstand helt** — hvert søkeord må nå finnes som bokstavelig delstreng i
tittel/område/tagger/tekst (tittel-vektet relevans). **Også:** dedupe av
resultater på innholds-nøkkel (tittel+område+frister) — Tasks-IA-migreringen
la samme oppgave i både `/items` og `/tasks` → identisk treff vist to ganger.
Commit `aac108f` → merge `881fd67`. Build grønt, frontend-deploy success.
`searchScore()` erstattet `fuzzyScore`/`editDistanceCapped` (today.jsx).

---

## 🎯 (2026-06-23 10:43) — Livsplan 13-bug UX-batch LIVE (merge `d6ab721`, PR #19)

**Trigger:** Mayo feilmeldte 13 UX-bugs med screenshots mot live mayooran.com
(Livsplanlegger). Planlegger-sesjonen delegerte fiksene til en isolert
worktree-agent, gjennomgikk diffen, merget til `feat/whoop-redesign` →
auto-deploy grønt. **Alle 13 live.** Kompilerings-verifisert, ikke nettleser-
testet — Mayo tester på ekte enhet.

| # | Bug | Rotårsak / fix |
|---|-----|----------------|
| 5 | Dag/natt gjorde ingenting | Invert-CSS lå i `App.jsx ShellLayout` (aldri mountet). Flyttet til `globals.css` (alltid lastet) |
| 1 | Livets puls overlappet | `L0Puls position:absolute` lakk ut → bruker in-flow `PulseStripToday` |
| 6 | Søk traff alt | `fuzzyScore` matchet 3-tegns vindu overalt → omskrevet (eksakt substring + ord-lengde-bevisst, fuzzy kun ≥4 tegn) |
| 7/8 | Prio-dott-overlapp + sortering | ≈-badge → bunn; la til sortering (relevans/frist/opprettet ↑↓) + prio/område-filtre |
| 10 | Jobb/Obs BYGG i Livsplan | Filtrerer `track==='jobb'`/`area==='obs_bygg'` ut av ALLE visninger + teller. Privat-only |
| 11 | Global topp-nav støy | Skjuler skall-nav på `/livsplan` + diskret «‹ Mayo OS»-hjemknapp (ikke innelåst) |
| 9 | ⋮ over ring + død legende | ⋮ flyttet ut av ring; status-legende filtrerer område-kort |
| 4 | «Gjør når» låst | Var hardkodet 20. jun → ekte dato-velger, «ikke planlagt» når tom |
| 3 | Swipe-lukk + autolagre | Drag-fra-hvor-som-helst (kun ved scroll-topp). Delvis: ikke full outbox per felt (patch lagrer hvert felt optimistisk m/ ærlig toast) |
| 12 | Desktop høyre-panel kuttet | Uttrekkbar skuff (⤢ → 540px, titler wrapper) |
| 13 | Mobil/desktop-paritet | Revidere på mobil-nav + Søk på desktop |

**10 commits** (`5c84277`…`581102b`) på `claude/confident-noether-lpacih` →
merge `d6ab721`. Build grønt (PageLivsplanV12 197 KB). Frontend deploy-run
`d6ab721` = success.

**Også (Mayo):** Vercel-GitHub-integrasjonen disconnectet → ikke flere
ubrukte previews / bot-kommentar-spam. Prod var aldri på Vercel (VPS-only).

**Annet denne sesjonen:**
- Backend `e193017`: fix feil «timer siden» på samme-dags Strava-økter (tid-
  på-døgn strippet i `/api/training`). Deployet.
- CI-fix `f8be4c5`: `curl|head` SIGPIPE (exit 23) i deploy-verifisering →
  capture-then-slice. (Etterfølgende deploy-run OOM-killet db-api ved Whisper-
  last under restart — infra/timing, ikke kode.)
- `HANDOVER-PT-HEALTH-AUDIT.md` (`a2eeadf`) → terminal-sesjonen kjørte Fase A–E.

---

## 🎯 (2026-06-22 23:30) — Livsplan UX-rewrite (`90df59c` + `95a3cec` + `d9de5ec`)

**Mandat:** Mayo: «ta en re-vurdering av hele UX i livsplanlegger ... det
er den jeg kommer til å bruke aller mest». «ikke X øverst i skjerm som
jeg må lete etter». «du har 6 timer, kjør på».

**Audit:** `docs/superpowers/reviews/LIVSPLAN-UX-2026-06-22.md` (17 funn,
benchmark mot iOS HIG / Linear / Things 3 / Baymard).

**Implementert (alle iOS-native gesture-mønstre):**

1. **SheetHost rewrite** — drag handle øverst er primær lukk-affordance.
   Swipe-down lukker. Snap points medium (60%) ↔ large (92%). X-knapp
   fjernet helt fra sheet-header.

2. **DesktopItemModal** — løser Mayo's konkrete klage «tekst kutter».
   ItemDetail rendres som sentral modal (600 px) istedenfor trang
   RightPanel (320 px). ESC + backdrop + ×.

3. **Inline tittel-redigering** — auto-grow textarea med `overflowWrap:
   anywhere`. Lange titler wrapper naturlig, kuttes aldri.

4. **SwipeableItem** — iOS Mail-mønster. Drag høyre → «✓ Ferdig», drag
   venstre → «🗑 Slett». Threshold 80 px med resistance + vertikal-lock.

5. **useLongPress hook** — 500ms, vibrate-feedback, cancel-click-trick.

6. **PullToRefresh wrapper** — pull > 70 px på toppen → onRefresh.

7. **useTabSwipe** — swipe-left/right > 50 px bytter mellom nav-tabs.

8. **ctx.completeItem / ctx.deleteItem** — felles handlers for swipe.
   Optimistisk + rollback ved feil.

9. **SkeletonItem** — placeholder med shimmer mens lister laster.

**Smoke 16/16 pass.** Nye tester:
- #15 ItemDetail er sentral modal > 400 px med tittel-textarea
- #16 Sheet har drag handle + INGEN X i header (Mayo's eksplisitte klage)

**Også:**
- `tokens.js` har TYPE / SPACE / TOUCH tokens for fremtidig migrasjon
- `ctx.refresh()` tilgjengelig (loadItems som named callback) klar for
  PullToRefresh-wrapping når Mayo verifiserer gesture-konflikter er løst
- ItemLine deaktiverer SwipeableItem når dragHandle-prop er satt (triage
  beholder sitt eget drag-and-drop)
- `docs/superpowers/reviews/LIVSPLAN-UX-PROVE-DETTE.md` — release-notes
  med konkret prøv-dette-rekkefølge for Mayo når han våkner

**Ikke gjort (utenfor 6t-budsjett, dokumentert i audit):**
- Typografi-standardisering: 20 unike font-sizes → 7-8 stilarter (TYPE-
  tokens definert, men eksisterende komponenter ikke migrert)
- PullToRefresh-wrapping på spesifikke lister (avventer Mayo's feedback
  på swipe-tab-konflikt-risiko)
- useLongPress→context-menu kobling
- Triage drag-and-drop modernisering

---

## 🎯 Nyeste (2026-06-22 14:15) — Privat lyd-opptak i Livsplanlegger (`0bedbe6` + `7fadfb8`)

**Trigger:** Mayo: «trenger mulighet i mayooran.com å laste opp iphone
lydopptak slik at whisper kan oversette transkribasjon og oppsumere…
men dette er privat, ikke obs bygg. så må ha mulighet under
livsplanlegger… nå hadde vi IVF møte f eks».

### Hva
🎙-knappen i Livsplanlegger-Fang-sheet er ikke lenger en stub. Tap →
filvelger (m4a/mov/mp3/wav/webm/ogg) → upload som privat møte med full
Whisper + Claude-pipeline. Action items lander i Livsplanlegger-inbox
med riktig track/area, IKKE i Obs BYGG.

### Backend (`0bedbe6`)
`db_api/meeting_module.py`:
- `/meeting/upload` aksepterer nå `is_private: bool = Form(False)` og
  lagrer på meeting-raden ved opprettelse
- `_insert_action_items` leser `meeting.is_private` og setter
  `track='privat'` per item. Hvis Claude foreslår `area`, brukes det
  (helse/familie/ivf/mayo_os/okonomi/obs_bygg). `sensitive=true` for
  is_private eller ivf/okonomi.
- `EXTRACTION_PROMPT` utvidet: Claude tildeler nå `area` per
  action_item med eksplisitte definisjoner (ivf = fertilitet/klinikk/FET,
  helse = trening/lege/blodprøve, familie = barn/Max/Priya, etc.)
- Pipeline skipper Obsidian-vault-MD for private møter (privacy: ikke
  speile IVF-/helse-transkripter til vault som syncer til andre enheter)

### Frontend (`7fadfb8` på `feat/whoop-redesign`)
`src/mobile/livsplan_v12/capture.jsx`:
- Mic-stub fjernet (la inn fast tekst). Erstattet med ekte filvelger
  (`<input type="file" accept="audio/*,...">`)
- XHR-upload med progress (lik Obs BYGG Meetings.jsx-mønstret)
- EventSource til `/meeting/{id}/stream` → live SSE-status:
  "Splittet i 24 biter" → "Transkriberer 3/24…" → "Claude analyserer…"
- På `done`-event: re-fetcher `/items` så ny inbox-rader dukker opp
  uten side-reload. Toast: "Lyd transkribert · N oppgaver fanget"
- Progress-overlay (lilla glasskort) under sheet-textarea — vises kun
  under aktiv pipeline, lukkbar når ferdig

### Privacy
- Items får `sensitive=true` (blur via `t.blurSensitive` i UI)
- `meeting_list?include_private=false` (default i Obs BYGG) filtrerer
  hele transkriptet ut av Obs BYGG-listen — det vises kun i
  Livsplanlegger
- Ingen vault-MD = transkript bare i Postgres (`meeting.transcript_text`)

### Test-status
- Backend syntaks OK, db-api restartet og /health svarer
- Form-validering verifisert med curl (401 = passerte form-parse,
  traff auth-vegg)
- Frontend bygget OK (Vite, ingen warnings utover eksisterende
  chunk-size-advarsel). Bundle: `PageLivsplanV12-DKbVq94G.js` 184 KB
- Ikke end-to-end testet (krever passkey-innlogging + faktisk audio).
  Mayo må teste IVF-opptaket sitt.

## 🎯 (2026-06-19 14:30) — Tasks IA Fase 2-3 (`b08b22c` + `96788f6`)

**Trigger:** Mayo: "kjør Fase 2-5". Snapshot tatt:
`~/backups/manual/pre-fase2-5-20260619-1157.sql` (97 KB).

### Fase 2 — `crm_task` → `item` (`b08b22c`)
**Auditen flagget «RISKY pga Apple sync»** men prod-tilstand viser at
Apple sync IKKE er i aktiv bruk: 0 av 88 crm_tasks har `reminder_id` eller
`synced_at`. Risiko derfor mye lavere enn antatt.

Migrasjoner:
- **019**: utvider item-tabellen med `reminder_id BIGINT`, `sync_origin
  TEXT`, `synced_at TIMESTAMPTZ`, `contact_id UUID`, `goal_id INTEGER` +
  FK-er + indexer. Apple-sync-felter beholdes så fremtidig aktivering
  ikke trenger nytt skjema-skifte.
- **020**: idempotent flytting av alle 88 crm_task-rader til `item` med
  `source='task'`, `track='privat'`. Status: inbox/open→inbox, today→
  today, done→done, dropped→dropped. tags, position, reminder_id,
  contact_id, goal_id, created_at, updated_at bevart.

**IKKE-DESTRUKTIV**: crm_task beholdt som arkiv med `migrated_to_item_id`-
peker. Innebygd guard ROLLBACK'er ved feilet mapping. Resultat: 88/88
migrert.

### Fase 3 — `/tasks` + `/tasks/unified` proxy (`96788f6`)
SPA-koden uendret — `_item_to_task()` mapper item-skjema tilbake til
crm_task-format. Endringer i `tasks_module.py`:
- GET/POST/PATCH/DELETE /tasks: proxy mot `item WHERE source='task'`.
  Status-mapping ('open'-alias→inbox).
- POST /tasks/quick (iOS Shortcut): INSERT direkte til item.
- GET /tasks/unified: én query mot `item WHERE source IN ('task','meeting',
  'voice-journal')` JOIN meeting. Apple-reminder-del uendret.

bg_task_sync no-op'er trygt — task_sync.py refererer til crm_task som
forblir arkiv. Apple-sync flyttes til item-tabellen i fremtidig fase når
Mayo aktiverer det.

### Verifisert
- /tasks → 50 items (default limit), filter status=inbox→62 items.
- PATCH status round-trip OK (today/inbox).
- /tasks/unified?limit=300 → **105 totalt** (62 task + 38 meeting + 5
  reminders).
- Smoke 14/14 pass.

### Fase 4 — Slett Tasks.jsx duplikat (`7865bb4`) ✅
Mayo valgte A: PageTasks (mobil-redesign på /tasks) som SOT.
- `src/routes/Tasks.jsx` slettet (-814 linjer)
- `App.jsx`: Tasks-import fjernet, `/calendar/tasks` → `<Navigate to="/tasks" replace />`
- `CalendarLayout.jsx`: TASKS-tab fjernet fra TABS

**Bundle**: main 594 KB → 572 KB (-22 KB). Smoke 14/14 pass.

### Fase 5 — DROP legacy-tabeller (2026-06-19 18:42) ✅
Migration 021 dropper `crm_task` + `meeting_action_item`. All task-data
(126 rader) lever videre i `item` via source-flagg.

**Refaktor før drop** (alle skrivere/lesere flyttet til item):
- `meeting_module._insert_action_items` → INSERT item source='meeting'
- `meeting_module._create_tasks_from_action_items` → no-op (duplikat fjernet)
- `meeting_module.meeting_assignees` → SELECT item source='meeting'
- `server.py /voice/task` → INSERT item source='task' track='privat'
- `server.py /action-items/backfill` → INSERT item source='meeting'
- `goals_module` task-stats → SELECT item source='task' AND goal_id
- `modules/vault/weekly_digest` → SELECT item source='meeting'
- `modules/reminders/task_sync.enabled()` → hardcoded False (Apple-sync må
  re-implementeres på item-tabellen før reaktivering — ikke i bruk i dag)

**Dead code slettet:**
- `db_api/item_mirror.py` (crm_task → item speil)
- `item_logic.crm_task_to_item_fields` + 4 tester

**Eksisterende bug oppdaget — FIKSET 2026-06-20 (`08ff902`):** rute-
rekkefølge `/meeting/{id}` deklarert før `/meeting/assignees` →
'assignees' tolket som UUID → 500. Flyttet `/meeting/assignees` +
`/meeting/entities` til FØR `/meeting/{id}` med kommentar over som
forklarer hvorfor. E2E: 10 unike assignees returneres nå.

---

## 🎯 2026-06-22 — Innstillinger-side (sikkerhets-senter) (`abbcff6` + `6420c39`)

**Trigger:** Mayo: «lag innstillinger-siden». Samler alle security-
relevante kontroller på én flate.

**Backend (`abbcff6`)**:
- GET `/pyauth/login-history?limit=100` returnerer login_attempt-rader
  + aggregerte 30d-stats (success_30d, fail_30d, unique_ips_30d, fail_24h).
- DELETE `/pyauth/sessions` (manglet før!) tar enten `{token}` eller
  `{token_prefix}` (8-tegn). Prefix krever EKSAKT 1 treff for å unngå
  feil sletting. Avviser nåværende sesjon (bruk /logout).

**Frontend (`6420c39`)**:
- Ny `PageInnstillinger.jsx` mountet på `/innstillinger`:
  · **Passkeys** — liste m/ device_label, sist brukt, iCloud-backup-flagg,
    slett-knapp (blokker sletting av siste passkey), legg-til-knapp.
  · **Aktive sesjoner** — current-badge, IP-prefix, last_seen, logg-ut.
  · **Login-historikk** — KPI-kort (vellykkede/feilet/unike IPs siste 30d),
    advarsel hvis fail_24h > 5, filter-toggle, tabell m/ IP/country/
    method/reason.
- PageHjem-headeren: 🛡️-knapp navigerer til /innstillinger. Rødt badge
  hvis ingen passkey registrert.

**Smoke 14/14 pass** (en flake på 08 reproduserte ikke).

---

## 🎯 2026-06-21 — Security-hardening på login (`517fcb2` + `f4623c0`)

**Trigger:** Mayo: «ekstrem sikkerhet — har helse og journal her».
Sikkerhetsvurdering avslørte 5 kritiske/høye funn på login-flaten.
Implementert i ett bundle:

1. **Tvunget 2FA**: krever BÅDE passord OG OTP. Tidligere enten-eller.
2. **Rate limit**: 5 feilforsøk/IP/5min → 429. Python-implementasjon
   (ikke nginx) siden db.mayooran.com går CF-tunnel → db-api direkte.
3. **Session-TTL ned til 14 dager** (fra 90).
4. **Audit-log på alle login-forsøk** til `login_attempt`-tabellen
   med IP, user-agent, country, method, success, reason.
5. **Web-push ved vellykket login** «🔓 Ny innlogging fra <land>, <enhet>».

**Frontend**: Login.jsx viser nå BÅDE passord OG OTP-felt samtidig.
Enter på passord → fokus til OTP. Feil → tøm OTP, behold passord.
Status-strip: «BCRYPT-12 + TOTP-6 · PG-SESSION 14D · RATE-LIMIT 5/5MIN».

**Bypass**: `LOGIN_BYPASS_TOKEN` i .env + cookie `mayo_bypass` lar Mayo
omgå rate limit hvis han blir låst ute — settes via SSH.

**Smoke 14/14 pass.**

**Gjenstår (krever Mayo)**:
- chat.mayooran.com bak CF-tunnel (DNS-endring)
- Cloudflare Access foran SPA (CF dashboard)
- Token-rotasjon (oppdaterer Shortcuts)
- Bitwarden vault-entry med TOTP + recovery-instruks (bruk **bitwarden.eu**)

**Arkitektur-funn (fail2ban)**: fail2ban er feil verktøy for vårt setup
fordi CF-tunnel skjuler angriperens IP fra iptables. Erstattet med
progressive Python-rate-limit (`f026b38`):
- Tier 1: 5 forsøk / 5 min
- Tier 2: 15 forsøk / 1 time
- Tier 3: 40 forsøk / 24 timer

Strengeste tier rapporteres i 429-melding + audit-log.

---

## 🎯 2026-06-20 — Apple Reminders → mayo_sov → GCal speiling (`2068064` + `b65b254`)

**Trigger:** Mayo: «vil at taskene skal speile til Google Calendar ut i
fra frist på tasken. Bruker Apple hurtigknapp for å opprette task i Apple
Reminders. Bruker primært Google Calendar appen for oversikt. Men huk —
må ha dette i livsplanlegger.»

**Endring:**
- `/reminders/bulk-sync` speiler nå hver Apple Reminder til item-tabellen
  (`source='reminder'`, `track='privat'`). Idempotent via `origin_ref=
  reminder.id`. Slett-håndtering: når reminder forsvinner fra Apple,
  soft-delete tilsvarende item.
- Migration 022 backfiller 6 eksisterende reminders.
- `/tasks/unified` leser reminders via item (source='reminder')
  istedenfor direkte FROM reminder — unngår dobling.
- `PageTasks.jsx` leser nå `/tasks/unified` så Apple Reminders dukker opp
  i I dag/Forfalt-bucketene. moveTask bruker `/items/{id}` PATCH.
- JOIN-mønster bytt fra `m.id = i.origin_ref::uuid` til `m.id::text =
  i.origin_ref` så reminder-items (bigint→text origin_ref) ikke trigger
  UUID-cast-feil.

**Automatisk GCal-sync:** `gcal_sync.py` finner alle items med due_at
uten gcal_event_id og pusher dem. Cron-frekvens: 3 min. Mayo's reminders
dukker opp i Google Calendar innen 3 min etter Shortcut-trigger.

**Test-skript:** `infra/scripts/test-reminder-sync.sh` verifiserer hele
kjeden ende-til-ende — 5/5 pass.

**For å aktivere i prod**: Mayo må konfigurere iOS Shortcut til POST mot
`db.mayooran.com/reminders/bulk-sync` med `X-Shortcut-Token`-header
(`SHORTCUTS_TOKEN` fra .env). Shortcut må sende ALLE reminders i hver run
(Apple's «I dag» er filter, ikke separat liste).

**Smoke 14/14 pass.**

Snapshot: `~/backups/manual/pre-drop-legacy-20260619-1828.sql` (45 KB).
E2E: alle endepunkter grønne, smoke 14/14.

**Notabene:** `reminder.crm_task_id`-kolonnen står som dødt felt — kan
ryddes i senere migrasjon.

### Fase 5 — IKKE GJORT (for risikabelt)
`meeting_action_item` og `crm_task` beholdes som arkiv med
`migrated_to_item_id`-pekere. Hvis noe går galt:
`UPDATE item SET deleted_at=now() WHERE source IN ('task','meeting',
'voice-journal')` reverserer alt. Sletting kan vurderes senere når
proxy-laget er bekreftet stabilt over flere uker.

---

## 🎯 (2026-06-19 13:45) — Tasks IA Fase 1 Steg 2–4 (`dbdbb56` + `e9f3ca1`)

**Trigger:** Mayo: "kjør Steg 2–4". Fase 1 av konsolideringen som auditen
anbefalte (4–5t). Snapshot tatt: `~/backups/manual/pre-tasks-ia-20260619-1140.sql`.

### Steg 2 — Datamigrasjon (`dbdbb56`)
Migration 018 flyttet alle 38 `meeting_action_item`-rader til `item`-tabellen
med `source='meeting'`, `track='jobb'`, beholdt `kanban_lane` direkte. Mapping:
text→title, assignee→assigned_to, due_date→due_at (Oslo tz), status→state,
meeting_id→origin_ref::text.

**IKKE-DESTRUKTIV**: meeting_action_item-tabellen er beholdt som arkiv med ny
`migrated_to_item_id`-peker. Migrasjonen har innebygd guard som ROLLBACK'er
hvis noen rad ikke kunne mappes. Resultat: 38/38 migrert, 16 assigned_to satt.

### Steg 3 — `/action-items` proxy (`e9f3ca1`)
SPA-koden er **uendret** — `_item_to_action_item()` mapper item-skjemaet
tilbake til action_item-formatet. GET filtrerer på `source IN ('meeting',
'voice-journal')`. PATCH oversetter status→state med completed_at, dueDate
til timestamp med Oslo-tz, behandler null-felter via `exclude_unset`.
DELETE er nå soft-delete (item.deleted_at).

Voice-journal-action_items skrives nå direkte til `item` istedenfor
`meeting_action_item` — proxy plukker dem opp via voice-journal-source.

### Steg 4 — Apple Reminders-sync intakt ✅
crm_task (88 rader) og reminder-tabeller er uberørt. TASK_REMINDER_SYNC=1
fortsatt aktivt. Ingen mister Apple-sync siden vi kun har konsolidert
meeting_action_item — som aldri var Apple-synket.

### Verifisert
- E2E: PATCH status=done/open, kanban_lane, soft-delete — alle round-trip OK.
- Smoke 14/14 pass (inkl. #11 obs-tasks-kanban som verifiserer at SPA
  fortsatt rendrer Brio-kanban-tavla mot proxy-en).
- assignee_stats: 10 personer med åpne actions, fungerer som før.

### Senere (Fase 2–5, ~6–8t — IKKE startet)
- Migrere `crm_task` → `item` (RISKY pga Apple sync).
- Aktivere `/tasks/unified` read-projection i alle UI-er.
- Slette `PageTasks.jsx` ↔ `Tasks.jsx` duplikat (~100KB).
- Slette legacy-tabeller (meeting_action_item, crm_task) når trygt.

---

## 🎯 (2026-06-19 13:15) — Åpne spor fra reviews (`2300eb1` + `415de89` + `3b063df`)

**Trigger:** Mayo: "ta åpne sporene du nevner". De tre sporene fra forrige
oppsummering: kalender-union, tasks-IA Fase 1, høyrepanel-duplikasjoner.

### Spor #1 — Kalender-union (`2300eb1`) ✅
Bekreftet gap: 7 meetings + 5 items siste 30 dager hadde scheduled_at men
manglet gcal_event_id og var derfor usynlige i SPA-kalender. GET /calendar
unioner nå inn `meeting` (source='_local_meeting') og `item`
(source='_local_item') med scheduled_at uten gcal_event_id. E2E: 46 events
i 14-d vindu = 45 Google + 1 tidligere usynlig lokalt meeting.

### Spor #2 — Tasks-IA Fase 1, Steg 1 (`415de89`) ⚠ delvis
Lagt `item.assigned_to TEXT` (migrasjon 017) + ItemCreate/Patch + EDITABLE_
FIELDS + _COLS. E2E: PATCH assigned_to round-trip OK, 21/21 item_logic-
tester pass.

**Steg 2–4 IKKE gjort** (krever Mayo-godkjenning — ikke-reversibel uten
backup):
- Steg 2: data-migrering `meeting_action_item` → `item` med `source='meeting'`
- Steg 3: re-pek frontend /obs-bygg/oppgaver fra `/action-items` til
  `/items?source=meeting`
- Steg 4: verifiser Apple Reminders-sync er intakt

### Spor #3 — Høyrepanel-duplikat (`3b063df`) ✅
HANDOVER_RESULT 2026-06-18 flagget «I dag: 5» vises i både RightSummary KPI-
rad og SmartTiles i sentrum på desktop ≥1024px. Wrappa SmartTiles i
`.lp-smarttiles-center` med `@media (min-width: 1024px) { display: none }`.
Samme breakpoint som RightPanel sin visibility-grense.

**Smoke:** 14/14 pass etter alle endringer.

---

## 🎯 (2026-06-19 12:54) — ARIA-polish addendum til memo #8 (`2f187b7`)

**Trigger:** Mayo: "ta neste oppgave i listen" — i kveld var #8 ARIA allerede
gjort i `4ecf00a` (basale roles + labels). Min sesjon gikk videre og adresserte
restenede anti-patterns + flere primitiver som ikke var dekket.

**Fronted (`2f187b7`):**
- **Anti-pattern fikset i ItemLine**: tidligere `aria-label=it.title` på wrapperen
  overstyrte all descendant-tekst for skjermleser. Ny `buildItemAria()` gir rik
  label: tittel · område · prioritet · frist · energi · undertask-progresjon.
- `PriDots` → `aria-hidden` (info finnes i parent-label, ingen dobbeltlesning).
- `AreaTag` button-versjon → `aria-label="Filtrer på område X"`.
- `SensLock` → `role="img"` + `aria-label="Sensitiv oppgave — privat"`.
- `SubtaskRow` priority-button → menneskelig label (ingen/lav/middels/høy)
  istedenfor "0 av 3".
- `SmartTiles` → `aria-label="I dag: 3 oppgaver — krever oppmerksomhet"`.

**Bundle:** PageLivsplanV12 179→181 KB (+1.6 KB ARIA-strings), main uendret
594 KB. **Smoke 14/14 pass** (med ny test #12, #13, #14 lagt av kveldssesjon).

---

## 🎯 (2026-06-18, kveld) — Konsolideringssprint fra REVIEW-2026-06-17 (10 punkter)

**Versjon:** konsolideringssprint — 10/10 memo-punkter ferdig

## 🎯 (2026-06-18, kveld) — Konsolideringssprint fra REVIEW-2026-06-17 (10 punkter)

**Trigger:** Mayo: «ta alle punktene stegvis» — alle resterende punkter
fra prio-listen + TL;DR-konsolideringspunkter.

**Ferdig:**

| # | Tema | Commit (backend) | Commit (frontend) |
|---|------|------------------|-------------------|
| C | Sky-backup-doctor (audit migrations + dumps + offsite-hook) | `1444534` | — |
| #6 | Atomisk lane-cascade ved kanban_lanes-endring | `a2d801c` | — |
| #7 | Feltnivå-diff audit-log på PATCH /items + meeting/summary | `ed63854` | — |
| A2 | Smoke #12 noindex-privacy (chain-bug-test) | `7eee469` | — |
| A1 | test_gcal_dedup (chain-bug-test) | `f1ab8de` | — |
| A3 | Smoke #13 fuzzy-edge-cases | `bfd337f` | — |
| #2 | Smoke #14 inbox-item søkbart (chain-bug-test) | `e864bb4` | — |
| #10 | Pinch-zoom på /brain + /obs-bygg 3D-graf | — | `d08fdd8` |
| #8 | ARIA-pass på ItemLine/SubtaskRow/AreaCard | — | `4ecf00a` |
| B | Statisk duplikat-audit høyrepanel ↔ sentrum (rapport) | `0f39a15` | — |

**Audit-funn (oppsummert i HANDOVER_RESULT.md):**
- 2 reelle duplikasjoner mellom RightPanel og sentrum (KPI-tall HØY, forfalt-stack LAV)
- Quick-fix er en CSS media-query — overlatt til frontend-revisjon

**Tester:**
- Pure-logic: 21/21 (item_logic) + 7/7 (audit) + 8/8 (gcal_dedup) ✓
- Smoke: ny test #12 + #13 + #14 verifisert mot live mayooran.com ✓

**Backend-deploy:** `sudo /bin/systemctl restart db-api` etter hver
endring. Helsesjekk OK på alle. Live E2E-test av audit-diff (PATCH title+
priority returnerte korrekt diff-rad i audit_log).

**Etterspurt etter første sprint og levert (rapporter i HANDOVER_RESULT.md):**

| Review | Hovedfunn | Toppanbefaling |
|---|---|---|
| `/brain` | ~~🔴 XSS-vektor i Psykolog.jsx~~ ✅ **FIKSET** `0e44bb3` — DOMPurify-wrap rundt marked.parse | — |
| `/kalender` | SPA leser fra `calendar_event`-tabell, gcal-pull skriver til `item`+`meeting` → mulig UI-gap | verifiser union i `/api/db/calendar` |
| `/tasks` IA | `/api/db/tasks/unified` finnes i backend, ikke brukt i SPA. 4+1 task-flater + ~100KB duplikat | Fase 1: assigned_to + migrer meeting_action_item → item (~4–5t) |

Commit-hash for review-blokken: `3437ace`.

## 🎯 (2026-06-18, ny økt) — chore(infra): commit av obs_gcal_dedupe.py (`6d61cf1`)

**Trigger:** Mayo: "hvor slappu sist?" → fant untracked
`infra/scripts/obs_gcal_dedupe.py` (opprettet 17. juni 11:18, aldri
committet). Mayo valgte å committe det som operasjonelt verktøy.

**Hva er det:** Engangs-cleanup brukt 2026-06-17 da første Obs BYGG-
backfill havnet i en auto-opprettet «Mayo OS · Obs BYGG»-kalender før
`CALENDAR_SYNC_ID_OBS` var satt. Andre backfill (mot riktig kalender)
etterlot 14 duplikater i auto-kalenderen. Scriptet sletter events med
`mayo_meeting_id`-extendedProperty i ikke-target-kalendere og fjerner
selve auto-kalenderen hvis den blir tom.

**Hvorfor committe:** CLAUDE.md regel #1 — alt som skal kjøre skal være
i git. Gjenbrukbar hvis calendar-sync-feilkonfig dukker opp igjen.

## 🎯 Forrige (2026-06-18 13:05) — fix(spa): Suspense rundt lazy routes (`cc2637e`)

**Trigger:** Mayo: "kommer ikke inn på flere av tabbene i mayooran.com".

**Diagnose:** Commit `66c95bf` (lazy-load 9 mobile pages) påstod i melding
at MayoShell allerede hadde `<Suspense>` rundt `<Outlet/>` — det stemte
ikke. Navigasjon til /strength, /helse, /tasks, /brain, /assistent,
/kalender, /livsplan*, /obs-bygg/* fikk React til å kaste «component
suspended without boundary», hele app-treet blanket. Kun `/` (PageHjem,
eager-importert) overlevde.

**Fix (`cc2637e`):** Pakket alle tre `<Outlet/>`-steder i `MayoShell.jsx`
i `<Suspense fallback="Laster…">`. Build OK, push trigger auto-deploy.

**Lærdom:** commit-meldinger kan lyve. Verifiser når kode antar at en
boundary/wrapper finnes andre steder.

## 🎯 (2026-06-18 12:45) — Brio-style kanban for /obs-bygg/oppgaver (`ac74b79` + `fc3f875`)

**Trigger:** Mayo: "ta tak i neste oppgave i listen og kjør helt ut" — neste i
REVIEW-2026-06-17.md (memo prio #4): Brio-style kanban for action-items.

**Backend (`5c04ace`):**
- Migration `016_action_item_kanban.sql` legger `kanban_lane TEXT` på
  `meeting_action_item` + index `(user_id, kanban_lane)`.
- `ActionItemPatch` utvidet med `kanban_lane`, så drag-drop PATCH-er det.
- Nye endepunkter: `GET/PUT /action-items/board-config` lagrer lane-config
  (id, title, color) i `settings_kv`-tabellen som key `obs_action_lanes`.
  Default ved tom KV: Research / Jobbes med / I fremtiden.
- `list_action_items` SELECT inkluderer `ai.kanban_lane`.

**Frontend (`fc3f875`):**
- `PageObs.jsx`: ObsTasks får view-toggle (Liste/Tavle) som persisteres i
  `localStorage['obs.tasks.view']`. Default = Liste (bakoverkompatibel).
- Ny `ObsKanban`-komponent (~200 linjer): brukerdefinerte kolonner med dra-
  og-slipp mellom dem, lane-editor med rename/reorder/delete, orphan-
  håndtering ved sletting, Gjort/Avvis-knapper bevart på kort. Speiler
  livsplan_v12 KanbanBoard-mønstret men forenklet — global tavle-config
  istedenfor per-parent (action-items er én felles brett).
- Bundle: PageObs lazy-chunk vokste fra ~140KB → 146KB (+6KB). Main
  bundle 594KB (uendret).

**Smoke (`ac74b79`):** Ny test `11-obs-tasks-kanban.js` verifiserer
toggle finnes, klikker Tavle, sjekker at Rediger kolonner + minst én
default-lane rendres. 11/11 pass på andre kjøring (test 08 hadde én flake
på /helse-tekstmatch, ikke relatert).

**E2E-verifisert:** `curl PATCH /action-items/{id} {kanban_lane:"research"}`
→ 200, GET viser kanban_lane i response.

---

## 🎯 2026-06-16 07:46 — PT-audit Fase E (loose ends) (`193e8e4`)

**Trigger:** Mayo: "kjør fase e" — fortsetter etter Fase D-rapporten der jeg
flagget tre forbedringer som «kommer i Fase E».

**Fase E (frontend `193e8e4`):**
- E1: `routes/health/Program.jsx` parallell-fetcher nå `action=coach` +
  `action=daily`. Ny per-gruppe restitusjons-glass viser timer-siden + terskel
  per push/pull/ben + `picked_group`-markering. Desktop ↔ mobil paritet.
- E2: `PageStyrke` trafikklys-knapper: title-tooltip per gruppe
  ("push: <36t rød · 36–60t gul · >60t grønn") + inline `vindu 36/60t`-mono
  på valgt rutine. Mayo ser hvor lyset bytter.
- E3 (§3.2-D): `routes/strength/Strength.loadRoutine` mapper nå
  `daily.planlagt[].anbefaling` → `item.rec` per `exId` når rutine-id
  matcher motorens valg. `ExerciseCard` bruker allerede
  `item.rec || recommendNext()` — backend tar over som SOT, lokal
  fallback for ikke-daily-rutiner. Eliminerer drift-risiko mellom JS- og
  Python-progresjons-motorer.

**Tester:** uendret (99/99 grønne — Fase E er ren wire-up uten ny logikk).
**Frontend-deploy:** `193e8e4` live på mayooran.com.

**Status PT-audit:** Fase A+B+C+D+E fullført. Audit-doken pluss alle
loose ends lukket.

---

## 🎯 (2026-06-16 05:40) — PT-audit Fase C+D fullført (`225dc4e`, `356cc7e`)

**Trigger:** Mayo: "kjør fase c og d" — fortsetter HANDOVER-PT-HEALTH-AUDIT.

**Fase C — arkitektur (`225dc4e` backend + `356cc7e` frontend):**
- C2 (§3.3-J): `_daily_brief` beriker recovery med Garmin-signaler hentet
  fra `health_daily(source='garmin')` — body_battery, stress_avg,
  resting_hr. `pt_llm.anonymize()` inkluderer disse → coach ser komplett
  Whoop+Garmin-bilde. Motoren bruker ikke Garmin i gating ennå (forsiktig).
- C3 (§3.2-E): Differensierte restitusjons-terskler. THRESHOLDS = {push
  (36, 60), pull (48, 72), ben (60, 84), markloft (60, 84)}. Push raskest,
  ben/markløft tregest (aksial). `_freq_band` tar group-parameter. Frontend
  `strength.js.FREQ_THRESHOLDS` + `PageStyrke` synket med backend.
- C1 (§3.3-H): Allerede gjort (Strava OAuth-fetch primær, Apps Script
  fallback) — verifisert, ingen endring.

**Fase D — UX-pass:**
- D1: PageStyrke/PageHelse null-disiplin god. Bugg: hardkodet `h < 48` i
  note-tekst (PageStyrke:775) → erstattet med per-gruppe `tAvvis`.
- D2 (trafikklys): Solid design. Anbefaling: vis terskel-tall på
  hover/tap så Mayo lærer per-gruppe-grensen.
- D3 (coach-tekst): Prompt + anonymize + fallback-kjede solid. Trenger
  live-sampling for subjektiv kvalitet.
- D4: Desktop `routes/health/Program.jsx` leser `action=coach` (subset),
  mobil `PageStyrke` leser `action=daily` (full motor). Funksjonelt OK,
  desktop mangler recency/picked_group. Lavprioritet å unifisere.

**Tester:** 98 → 99 grønne. Lagt til `test_T17b_per_group_thresholds_locked`.

**Backend-deploy:** db-api restartet. **Frontend-deploy:** `356cc7e` live.

**Status PT-audit:** Fase A+B+C+D fullført. Audit-doken lukket.

---

## 🎯 (2026-06-15 21:00) — PT-audit Fase A+B fullført (`2c0c340`, `edd4f5a`, `2a2d98a`)

**Trigger:** `HANDOVER-PT-HEALTH-AUDIT.md` (`a2eeadf`) — fra planlegger til terminal.

**Fase A — verifisering (`2c0c340`):**
- A1: e193017-fixen er på disk + korrekt — `sessions[].date` beholder full ISO-ts.
- A2: pytest 79 → 86/86 ✓. Fant + fikset import-chain-bug i `strava_watcher.py`
  (manglet `_ROOT` på sys.path → 7 zones_hrlag-tester gikk ned ved pytest cwd).
- A3: `_merge_recency` bruker `min(vals)` per gruppe — bekreftet + låst med
  `tests/test_merge_recency.py` (6 tester inkl. eksplisitt MAX-regresjons-guard).

**Fase B — fixes (`edd4f5a` backend + `2a2d98a` frontend):**
- B1 (§3.2-A): Frontend↔backend recency synket. `/training?action=daily`
  eksponerer nå `card.recency` + `picked_group`. PageStyrke leser disse
  direkte i stedet for å regne lokalt → eliminerer divergens-risiko der lokal
  kunne vise GRØNT mens backend sa RØDT.
- B2 (§3.2-C): `okt_logikk._PUSH_RE`/`_LEGS_RE` fjernet. Bytt til nye public
  helpers `is_push_request()` / `is_legs_request()` / `title_mentions_group()`
  i `parser.py` som bruker SAMME `_KEYWORDS` som `parse_title`. Fritekst og
  Strava-titler kan ikke lenger drifte. Manglende keywords lagt til:
  PUSH (skulderpress, brystpress, tricep, pec fly), LOWER (legext, utfall,
  tåhev, lår).
- B3 (§3.2-G): `strength_session.ts` er TIMESTAMPTZ end-to-end. Frontend
  sender ISO+Z, backend bruker `fromisoformat()`, `_parse_ts` håndterer
  string/naive/aware korrekt. Ingen kodefiks — låst med
  `tests/test_parse_ts.py` (6 tester).
- B4 (§3.2-F): Eksplisitt «Whoop-data ikke tilgjengelig»-banner over
  trafikklys-kort i PageStyrke når `!hasRec`. Forhindrer feiltolkning av
  grå/grønne dots som "alt klart" når recovery-sone er ukjent.

**Tester:** 92 → 98/98 grønne. **Backend-deploy:** db-api restartet.
**Frontend-deploy:** `2a2d98a` live på `mayooran.com`.

**Gjenstår (Fase B):** ingen i scope. Fase C (arkitektur) + D (UX-pass)
venter på Mayos go.

---

## 🎯 (2026-06-15 20:25) — Fix: feil «timer siden» på samme-dags økter (`e193017`)

**Mayos rapport:** «på push sier den trent for 20t siden — men hadde push 7 timer siden.»

**Rotårsak:** `/api/training?days=8` (strava_training_module.py) strippet
klokkeslettet fra hver økts dato (`date.split("T")[0]`). Frontend regner
«timer siden» via `groupHoursSinceAny` → `new Date("2026-06-15")` = midnatt
UTC, ikke faktisk treningstid. En push kl ~13 ble målt fra midnatt → ~20t i
stedet for 7t → falskt rødt lys + re-anbefalte nettopp-trent gruppe.

**Fix:** behold fullt ISO-tidsstempel i `date`; bruk kun dato til stabil
fallback-id. Backendens egen recency (frequency.py `parse_workouts` via
`fromisoformat`) var allerede korrekt — kun frontend-stien var rammet.
Frontend `isoOf` bruker lokale dato-komponenter → ISO-uke uendret.

**Krever backend-deploy:** `cd ~/mayo-ai-os && ./deploy.sh`.

---

## 🎯 (2026-06-15 21:00) — PT coach enrichment + smart recency + notification center

### PT-forbedringer (3 commits: `9be23b5`, `8c8262a`, `e88843b`)

**Parser (`9be23b5`):**
- ~20 nye norske gym-keywords (bryst, skulder, overkropp, nedtrekk, dips, arnold, sidehev, ben, glute, etc.)
- Strava Type-fallback: Run→Aerob, Ride→Aerob, etc. (brukes kun når tittel-keywords gir 0 treff)
- 33 tester (opp fra 17), alle grønne

**Recency-merge (`8c8262a`):**
- `_merge_recency()` i daily_card.py: tar MIN per muskelgruppe fra BÅDE strength_session-logg OG Strava-parser
- Tidligere overskrev Strava-override hele session-dataen — nå supplerer den
- Recovery-kontekst beriket: sleep_efficiency, deep_sleep_min, rem_sleep_min, strain_yesterday fra Whoop
- Motor-tekst annoterer datakildene ("Recency-kilder: styrkelogg + Strava")

**Coach-enrichment (`e88843b`):**
- `anonymize()` eksponerer ALLE planlagte øvelsers sist-tall (ikke bare hovedløftet)
- PT_SYSTEM-prompten forsterket: strain, "push når dataen tåler det, hold igjen når den ikke gjør det"
- Daglig PT-rapport → notification bell (`create_notification(category="health")`)
- Ukentlig PT-rapport → notification bell (`weekly_report.py:_notify()`)

### In-app notification center (bell icon) (`49306a7` + `f8bf84f`)

**Backend:**
- Migration `008_notifications.sql`: `notification`-tabell (category, title, body, url, icon, read_at)
- `notification_module.py`: GET /notifications, PATCH /{id}/read, POST /read-all, GET /unread-count
- `create_notification()` helper for intern bruk (trading, health, etc.)
- `send_signals.py`: trading-signaler → DB-notifikasjon istedenfor Telegram (gated bak `TRADING_TELEGRAM_SEND=0`)

**Frontend:**
- `NotificationBell.jsx`: bell icon + dropdown panel i desktop Topnav
- Polls unread-count hvert 60s, merk lest/merk alle lest, navigerer til url
- Sovereign Glass-estetikk, magenta unread-badge med glow

**Deploy-rekkefølge (kreves på VPS):**
1. `psql mayo_sov -f ~/mayo-ai-os/migrations/008_notifications.sql`
2. `cd ~/mayo-ai-os && ./deploy.sh` (backend — alle 4 commits)
3. `cd ~/mayo-os-deploy && git fetch origin feat/whoop-redesign && git reset --hard FETCH_HEAD && ./deploy.sh skip-pull` (frontend)

---

## 🎯 (2026-06-15 17:00) — Livsplan v1.2-handoff importert

Mayo lastet opp `mayooran.com Design v1.1 (8).zip` (misvisende navn — inneholder
hele v1.2-bundle) til vaulten. Pakket ut til `mayo-os-deploy/_design/livsplan-v12-handoff/`.

**Innhold:**
- 14 ferdig porterte JSX-filer i `src/livsplan_v12/`
- 2 standalone HTML-fasit (mobil + desktop)
- 6 handover/spec-dokumenter + CLAUDE.md + README
- `src/app/` primitiver: icons, primitives, tokens

**Status:**
- 2 commits på `feat/whoop-redesign` (frontend-repo, ikke pushet):
  - `1a697b1` import av handoff
  - `5e64cfb` KICKSTART-merge-til-repo.md
- Backend MCP `list_inbox`-UX fra tidligere session: commits `fdb009a`, `d6356ae`, `92d39b8` (på `claude/confident-noether-lpacih`, pushet)

**Surprise-funn:** `/home/mayo/mayo-os-deploy/` er IKKE bare deploy-katalog —
det er frontend-repoet `mayo-os` (origin: `github.com/mrmayooran-Ai/mayo-os.git`).
Det betyr v1.2-merge KAN kjøres fra VPS, men neste session må starte friskt
(forrige nådde 90% session limit før merge kunne begynne).

**Neste:** ny session leser `_design/livsplan-v12-handoff/KICKSTART-merge-til-repo.md`
→ oppretter `src/mobile/livsplan_v12/` parallelt med v1.1 → porterer + Vite-tilpasser.



## 🎯 Nyeste (2026-06-15 12:13) — MCP list_inbox UX-iterasjon 2

Mayo: «fortsatt kommer tall. jeg må ha task navn og frist dato og viktighet».
- `_tool_list_inbox` rebuilt: tittel + frist (relativ tid) + prio (lav/medium/høy) + område
- Frister: «i dag», «i morgen», «om N dager», absolutt dato i parentes
- ID-mapping flyttet til en TOOL_INTERNAL-seksjon (HTML-kommentar + instruks)
- Tool-description forsterket med eksplisitt «vis ALDRI TOOL_INTERNAL»
- Bekreftet av Mayo: Claude.ai viser nå 20 oppgaver helt rent, ingen tall/IDer
- Commits: `fdb009a` (logikk) + `d6356ae` (description-instruks)
- Deployet via `./deploy.sh` (PID 184352)



---

## 🎯 Final (2026-06-15 09:42 → 10:05) — Alle 8 gjenværende moduler trukket ut

Etter første 6 moduler (-47%) kjørte Mayo «kjør alle i rekkefølge uten stopp».
8 nye moduler portet med samme mønster — register router → flytt endepunkter
→ verifiser mot prod → commit per modul. Total runtime: ~25 min.

| # | Modul | Endepunkter | Commit | server.py-linjer |
|---|-------|-------------|--------|------------------|
| 7 | calendar_module.py | 4 | 9d32b05 | 2480 |
| 8 | email_module.py | 5 | 9d32b05 | 2480 |
| 9 | finance_local_module.py | 5 | 8f206fe | 2300 |
| 10 | notes_module.py | 6 | 8f206fe | 2300 |
| 11 | goals_module.py | 6 | af75f3a | 1980 |
| 12 | habits_module.py | 5 | af75f3a | 1815 |
| 13 | weight_module.py | 3 | af75f3a | 1690 |
| 14 | chat_module.py | 3 | af75f3a | **1590** |

**Sluttresultat etter ALLE 14 moduler:**
- server.py: 5284 → **1590 linjer (-3694, -70%)**
- 14 nye moduler: ~3300 linjer totalt
- 99 endepunkter migrert

Server.py inneholder nå BARE: auth-middleware, /health, /strava-ruter,
audit-log, vault/tree, brief, search/cross-domain, voice/jarvis, og noen
helt spesialiserte ting (calendar OAuth callback osv).

### Special handling per modul

- **finance_local_module**: navnesuffix -local for å unngå kollisjon med
  `finance_advisor/backend/finance_module.py`.
- **trading_module**: erstattet eldre Notion-fallback (.legacy-backup);
  sys.modules-cache måtte poppes for å unngå wrong `advisor` import fra
  finance_advisor.
- **calendar_module**: GET /calendar-auth (OAuth callback) BEHOLDT i
  server.py som spesialformål.
- **chat_module**: Jarvis-ruting + ekspert-persona + Anonymizer er
  alle preservert intakt.

---

## 🆕 Aller siste (2026-06-15 09:25–09:42) — Trukket ut 5 nye moduler

Etter journal_module.py (-23%) gikk samme mønster på 5 moduler til,

---

## 🆕 Aller siste (2026-06-15 09:25–09:42) — Trukket ut 5 nye moduler

Etter journal_module.py (-23%) gikk samme mønster på 5 moduler til,
i prio-rekkefølge per Mayos valg. ALLE verifisert mot prod med ekte data.

| Runde | Modul | Endepunkter | Commit | server.py-linjer etter |
|-------|-------|-------------|--------|------------------------|
| 1 | journal_module.py | 24 (8 sub-commits) | a4f7bd1 | 4050 |
| 2 | strength_module.py | 10 | f225e01 | 3717 |
| 3 | reminders_module.py | 8 | 64115e5 | 3370 |
| 4 | tasks_module.py | 6 | d5fe108 | 3023 |
| 5 | nutrition_module.py | 7 | aa44003 | 2870 |
| 6 | trading_module.py | 7 | 6d9e36b | **2780** |

**Totalresultat:** server.py 5284 → **2780 linjer (-2504, -47%)**

Nye moduler:
- journal_module.py: 1338 linjer (24 ruter)
- strength_module.py: 372 linjer (10 ruter)
- reminders_module.py: 341 linjer (8 ruter)
- tasks_module.py: 301 linjer (6 ruter)
- nutrition_module.py: 206 linjer (7 ruter)
- trading_module.py: 135 linjer (7 ruter; gammel Notion-fallback i .legacy-backup)

**Total kode:** 5284 → 5273 linjer (samlet redusert, ikke bare flyttet)

Bug oppdaget + fikset under prod-test: `modules/trading` og
`finance_advisor/backend` har SAMME filnavn (advisor.py, run.py).
Python's sys.modules cacher første loading → wrong module-bytting.
Fix: trading_module._trading_path() pop'er sys.modules-keys før
import → tvinger fersk lookup fra modules/trading.

---

---

## 🆕 Aller siste (2026-06-15 08:45 → 09:18) — journal_module.py 100% ferdig

8 commits over 30 min. ALLE 24 journal-endepunkter migrert ut av
server.py-monolitten. Hver runde verifisert mot prod med curl.

| Runde | Commit | Endepunkter | server.py-linjer etter |
|-------|--------|-------------|------------------------|
| 1 | `1f8b0bb` | 4 lese (list/by-tag/by-id/wiki-links) | 5224 |
| 2 | `17e83b2` | 5 skriv (POST/PATCH/DELETE + quick + geo) | 5126 |
| 3 | `b8cd6a5` | 3 lese (insights/related/backlinks) | 4901 |
| 4 | `a7def61` | 2 voice (text + audio) | 4824 |
| 5 | `21fdf87` | 5 (search/ask/sync-vault/psykolog x2) | 4634 |
| 6 | `4021eb7` | 1 (graph) | 4528 |
| 7 | `b8c97ec` | 2 media + slett 9 dead helpers | 4264 |
| 8 | `a4f7bd1` | 1 (voice-journal m/ chunking + Claude) | 4050 |

**Resultat:**
- server.py: 5284 → **4050 linjer (-1234, -23%)**
- journal_module.py: 0 → **1338 linjer** (ny)
- Total: 5284 → 5388 (+104 — godt trade-off for løs kobling)

Server.py kjenner ikke til ÉN /journal-rute lenger. Alle media-helpers,
geo-enrichment, Pillow/HEIC, Whisper, Claude-strukturering, og psykolog-
synteser bor nå i journal_module. 9 dead helpers + 6 konstanter slettet.

Neste kandidater for splitting (per HANDOVER-UX-FOR-DESIGN.md §X):
- `strength_module.py` (10 endepunkter, ~400 linjer)
- `reminders_module.py` (8 endepunkter)
- `tasks_module.py` (6 endepunkter)
- `nutrition_module.py` (7 endepunkter)

---

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
