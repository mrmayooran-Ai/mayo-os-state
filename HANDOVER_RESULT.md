# HANDOVER_RESULT — Tasks ↔ Reminders Fase 0 recon (2026-07-01)

**Til:** planlegger-sesjon (claude.ai)
**Fra:** Elmars (VPS-Claude)
**Trigger:** `HANDOVER-TASKS-REMINDERS-SYNC-REBUILD.md` (revidert til CalDAV-polling).
Mayo bekreftet at app-specific password «Mayo OS CalDAV» ble generert 2026-05-22.

## Sammendrag

**Fase 0 grønn — Mayo kan si «kjør Fase 1».** Ingen creds-arbeid gjenstår.

Tre mindre observasjoner å hensynta ved Fase 1-bygging (dokumentert nedenfor):
- Funksjonsnavn i handover matcher ikke faktisk kode (`pull_all` finnes ikke,
  bruk `fetch_all_reminders`)
- List-dict-felt heter `name`, ikke `display_name`
- 4 av 6 «reminders» er Apples egne onboarding-tekster — trenger filter

---

## 1. iCloud-creds ✅

```
ICLOUD_APPLE_ID=<SET>
ICLOUD_APP_PASSWORD=<SET>
```

Begge er lastet i `.env`. Passordet er antakelig det som ble generert
2026-05-22 for «Mayo OS CalDAV» — funker uansett.

## 2. CalDAV-tilgang: LIVE og virker ✅

```python
from modules.reminders.caldav_client import list_reminder_lists, fetch_all_reminders
lists = list_reminder_lists()
todos = fetch_all_reminders()
```

**3 lister (iCloud Reminders):**

| Navn | URL |
|---|---|
| Påminnelser ⚠️ | `caldav.icloud.com/53220125/calendars/9e7515ef…` |
| Handleliste ⚠️ | `caldav.icloud.com/53220125/calendars/a71274f3…` |
| Me time | `caldav.icloud.com/53220125/calendars/e14c0867…` |

**6 reminders totalt:**

| Status | Tittel | Liste |
|---|---|---|
| [ ] | Hvor er påminnelsene mine? | Påminnelser ⚠️ |
| [ ] | Oppretteren av denne listen har oppgradert disse påminnelsene. | Påminnelser ⚠️ |
| [ ] | Hvor er påminnelsene mine? | Handleliste ⚠️ |
| [ ] | Oppretteren av denne listen har oppgradert disse påminnelsene. | Handleliste ⚠️ |
| [x] | Handleliste | Me time |
| [ ] | TEST fra mayooran.com — push-test | Me time |

Ingen 401/403. Autentisering + TLS + WebDAV-parsing alt OK.

**Merk:** de to «Hvor er påminnelsene mine?»- og «Oppretteren av denne listen…»-radene
er **Apples egne standard-onboarding-tekster** — de dukker opp automatisk i alle
lister som ikke har blitt endret siden Reminders-oppgraderingen (typisk i 2018).
Fase 1 må filtrere bort disse så de ikke lager støy-items i Mayo OS. Enkleste
regel: `if title in {"Hvor er påminnelsene mine?", "Oppretteren av denne listen
har oppgradert disse påminnelsene."}` → skip.

Alternativ: Mayo sletter disse manuelt i Reminders-appen — men de kommer sannsynligvis
tilbake ved neste iCloud-sync til flere enheter. Bedre å filtrere server-side.

## 3. `task_sync.enabled()` — bekreftet `False` ✅

```python
def enabled() -> bool:
    """Leses ved kall-tid (ikke import) så testene/VPS kan toggle uten reimport.

    2026-06-19 Fase 5: hardcoded False fordi crm_task-tabellen er droppet og
    sync-laget peker på den. Apple Reminders-sync må re-implementeres på
    item-tabellen før dette kan reaktiveres (TODO ved bevisst valg fra Mayo).
    """
    return False
```

Rørene mot `crm_task` finnes fortsatt i modulen. Fase 1 må retargete til
`item`-tabellen (og teste at `enabled()` kan snus til `True` via env-flagg —
eventuelt fjerne enabled-gate helt hvis Mayo vil at sync alltid er på).

## Merknader til Fase 1-bygging

### Handover-spec-navn ↔ faktisk kode

Handover-instruksene refererer til funksjons-/felt-navn som ikke matcher
`caldav_client.py`. Ingen bug — men Fase 1-koden må bruke faktiske navn:

| Handover-navn | Faktisk navn i `caldav_client.py` |
|---|---|
| `pull_all()` | `fetch_all_reminders()` (linje 154) |
| `list.display_name` | `list["name"]` |
| `_create_vtodo` | `_build_vtodo` (linje 189) |
| `update_vtodo` | `push_update` (linje 247) |
| `delete_vtodo` | `push_delete` (linje 264) |

Fase 1-koden må referere til de faktiske navnene, ellers får vi ImportError
(som jeg fikk i første tørrkjøring).

### Suverenitet i sync-mapping

Når Fase 1 mapper reminder → item, trenger vi en policy for hvilken liste
som havner i hvilken track/area. Naiv default:

- «Påminnelser ⚠️» og «Handleliste ⚠️» → track='privat', area=null (Livsplan-inbox)
- «Me time» → track='privat', area='helse' eller similar

Mer solid: kartlegg iCloud-list-URL → Mayo OS-area i en config-tabell eller
en dict i `task_sync.py`. Ingenting fra iCloud skal automatisk lande i
Obs BYGG (jobb-flate) — samme rail som meeting-fiksen 3ea8a88 håndhever.

### 3-lister-realiteten

Bare 3 iCloud-lister eksisterer. Mayos handover snakker om «dagens liste»
— sannsynligvis meningen med «Påminnelser ⚠️»-lista (Apples default). Fase 1
bør ha en config-linje `MAYO_DEFAULT_LIST = "Påminnelser ⚠️"` (eller list-URL)
så nye tasks som opprettes i Mayo OS pusher til rett liste.

## Definition of Done — Fase 0

- [x] Creds lastet (`ICLOUD_APPLE_ID` + `ICLOUD_APP_PASSWORD` i `.env`)
- [x] CalDAV-autentisering fungerer live
- [x] `list_reminder_lists()` + `fetch_all_reminders()` returnerer data
- [x] `task_sync.enabled()` bekreftet `False` og krever retargeting
- [x] Skjema-mismatch mellom handover-spec og faktisk kode dokumentert
- [x] Onboarding-tekst-filter identifisert
- [x] HANDOVER_RESULT.md skrevet + commit + speil

**Fase 1 klar til «Kjør».** Ingen blokkerende avhengigheter, ingen
creds-arbeid gjenstår for Mayo.

— Elmars 2026-07-01 10:00 UTC
