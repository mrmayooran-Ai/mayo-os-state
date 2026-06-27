# HANDOVER_RESULT — Strava-sync diagnose (2026-06-27)

**Til:** planlegger-sesjon (claude.ai)
**Fra:** Elmars (VPS-Claude)
**Trigger:** `HANDOVER-STRAVA-TOKEN-FIX.md` (a530bbe) — Mayo: «Strava har sluttet
å synche, får helt feil treningsanbefalinger.»

## Hovedfunn — handover-diagnose passer IKKE med data fra VPS

Den hypotiserte rotårsaken (dobbel refresh-token-rotasjon → strandet
ugyldiggjort token i `.env` → permanent 400/502) er **avkreftet** av live-
verifisering på VPS. Faktisk symptom var noe annet, og er allerede borte
ved diagnose-tidspunktet.

### Det jeg verifiserte (alle steg per handover §2 + utvidet)

| Sjekk | Forventet ved race-rotårsak | Faktisk |
|---|---|---|
| `grep -c STRAVA_REFRESH_TOKEN .env` | > 1 (race-spor) | **1** — ingen duplikat-linjer |
| db-api journal siste 24t | «Strava token refresh feilet», 400/502 | **Stille** — ingen Strava-relaterte feil |
| Cron-logg (`/home/mayo/.mayo-strava-watcher.log`) | 400/502 fra `/oauth/token` | **0 token-refresh-feil noensinne i loggen** |
| `SELECT MAX(start_at) FROM strava_notified` | Stoppet for lengst | **8 rader siste 14 dager, siste 2026-06-26** — sync har levert |
| `await fetch_activities(days=14)` live mot Strava | 400/502 | **HTTP 200, 7 aktiviteter** |
| Strava rate-limit-headers (live nå) | I overkant | **Read: 16/100 (15min), 410/1000 (daglig)** — komfortabel margin |
| `curl /strava?days=14` m/ X-VPS-Token | Tomt/feil | **7 aktiviteter, type WeightTraining** |

Refresh-tokenet er gyldig. Endepunktet fungerer. Sync-pipelinen er levende.

### Faktisk historisk feilmønster (lest fra loggen)

**Ikke** 400 fra `/oauth/token` — det var **429 Too Many Requests** fra
`https://www.strava.com/api/v3/athlete/activities?per_page=5`:

- **Første 429:** 2026-06-06 20:20
- **Siste 429:** 2026-06-26 23:55 (~7 timer før diagnose)
- **20 dager med 429** på `/athlete/activities`-kallene, men aldri på
  `/oauth/token` (refresh)

Sannsynlig årsak: read-rate-limit-burst på `/athlete/activities`. Watcheren
poller `per_page=5` hvert 5. minutt (~288×/døgn) + db-api `_fetch_apps_script`
+ kanskje en periode hvor en annen klient pushet over 100/15min eller
1000/døgn-grensen. 429-perioden sluttet av seg selv 26.06 23:55 —
sannsynligvis fordi Strava's read-rate-vindu rullet, eller noe annet
kall-mønster ble redusert.

Watcheren håndterte 429 ved å logge `ERROR Klarte ikke liste aktiviteter`
(strava_watcher.py:515) og returnere uten å re-prøve. Ingen retry-backoff
implementert.

### Konsekvens for «feil treningsanbefalinger»

PT-rapporten (`infra/scripts/mayo-health-report.sh` → `modules/health/send_report.py`)
kjørte hver morgen 06:00 også under 429-perioden:

| Dato | Gating | Forslag | LLM-status |
|---|---|---|---|
| 2026-06-24 | RØD | Z1 — gåtur 30-45 min eller hvile | `PT LLM feilet → fallback` |
| 2026-06-25 | GRØNN | Legs B — posterior | `PT LLM feilet → fallback` |
| 2026-06-26 | GUL | Z2 fastet 45–60 min | `PT LLM feilet → fallback` |
| 2026-06-27 | GRØNN | Legs B — posterior | `PT LLM feilet → fallback` |

To uavhengige problemer som forklarer Mayos klage:

1. **20 dagers tomgang på fersk treningsdata under 429-perioden** →
   recency-merge så «ingen ny trening» → coachens gating-input ble
   systematisk skjev. Mayo merket dette som «feil belastningsråd».
2. **PT LLM feiler hver dag** (`pt_llm:coach_comment:155 - PT LLM %s feilet`
   → fallback). Fallback-tekst brukes uavhengig av om Strava-data finnes.
   Dette er en separat sak handoveren ikke nevner.

## Hva jeg har IKKE gjort (avventer Mayos avgjørelse — diagnose endrer scope)

Handover §3 (DB-backet `service_token` + `pg_advisory_xact_lock`) løser en
**reell strukturell sårbarhet** (to rotatorer er fortsatt et latent
race-spor — vi var heldige i denne perioden). Men det er **ikke akutt**:
sync fungerer, og det var ikke token-rotasjonen som forårsaket symptomet.

Spurte ikke Mayo om å re-OAuthe — token er gyldig, det ville være no-op
med risiko (avbryter pågående gyldig refresh; ekstra `.env`-skriving som
selv kan trigge race vi vil unngå).

### Anbefaling for neste steg, i prioritet

1. **Fail-closed på PT-rapport** (handover §4) — fortsatt høyverdig. Speil
   Whoop `stale`-mønsteret i `strava_watcher.py:391` til PT-rapporten. Hvis
   Strava-data > 48t gammel: «Strava utdatert, hopper over belastningsråd»
   heller enn selvsikker fallback. **Dette er det som faktisk traff Mayo.**
2. **PT LLM-fallback-problem** — separat undersøkelse. Hvorfor feiler
   `pt_llm.coach_comment` hver dag? Sannsynligvis relatert til «feil
   anbefalinger» Mayo opplevde.
3. **Watcher rate-limit-disiplin:**
   - Reduser frekvens fra `*/5` til `*/15` eller `*/30` (288/døgn → 96/døgn
     eller 48/døgn), eller
   - Implementér retry-with-backoff på 429 + caching av activities-listen i
     30 min så vi ikke poller hver 5. min for samme data.
4. **Durable fiks (handover §3)** — bygges som planlagt arkitekturarbeid,
   ikke incident-respons. Den er fortsatt riktig på sikt; bare ikke akutt.

## Re-auth-URL (hvis Mayo VIL gjøre det allikevel)

`STRAVA_PUBLIC_BASE_URL=https://mayooran.com/api` → eksakt URL:

**https://mayooran.com/api/strava-auth**

Åpnes i nettleser → redirect til Strava → callback skriver fersk token.
Funksjonelt unødvendig (gyldig token er aktiv), men trygt å gjøre.

## STATE-oppdatering

Speiler dette funnet til STATE.md som siste handling, med klar markering
av at handover-diagnose er avkreftet og scope må omforhandles med Mayo.

— Elmars 2026-06-27 07:05 UTC
