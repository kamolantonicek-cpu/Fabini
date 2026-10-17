# Ukázky běhů

Všechny scénáře používají mock data. Výsledky pocházejí z manuálních běhů v n8n dne 16. 10. 2026. Soubory v `examples/` jsou nezměněné JSON exporty výstupu uzlu **AI Agent** se stejnými poli `output` a `intermediateSteps`; nejde o archiv celé execution. Obsahují odpověď modelu i skutečné parametry a výsledky nástrojů, včetně původních `messageLog`.

Dotazy pro reprodukci jsou uvedené níže u každého scénáře. V přiložených JSONech původní `chatInput` není. Doslovný vstup konkrétního uloženého běhu lze dohledat v **User input → Output → JSON → chatInput**. Finální zjednodušený objekt s poli `request_id`, `environment`, `summary`, `steps` a `errors` sestavuje následující uzel **Build result**.

Soubory `data/observed-*.csv` jsou archivní snímky ze starších běhů; aktuální ukázky níže vycházejí přímo z přiložených JSONů.

## 1. Úspěšná diagnostika nezaplacené objednávky

Výchozí objednávka: **100246**. Nevyžaduje reset mezi opakováními, protože scénář neprovádí zápis.

```text
Zjisti, proč objednávka 100246 nebyla expedována. Proveď pouze diagnostiku, nic neměň.
```

| Nástroj | Ověřený výsledek |
|---|---|
| `get_order` | Objednávka existuje, `order_status = new`. |
| `get_payment` | `payment_status = pending`. |
| `get_fulfillment` | `fulfillment_status = on_hold`. |
| `get_sync_logs` | `sync_status = not_started`, log potvrzuje čekání na zaplacení. |

Agent popsal čekání na úhradu a doporučil po úhradě ověřit další zpracování. Přiložený výpis obsahuje čtyři čtecí volání, žádnou zápisovou akci a žádnou chybu nástroje.

Přímý výstup AI Agenta: [01-unpaid-diagnosis.json](../examples/01-unpaid-diagnosis.json). ID hlavní execution není součástí tohoto exportu a není dodatečně doplňováno.

## 2. Úspěšná oprava synchronizace po schválení

Výchozí objednávka: **100245**, zaplacená, `not_received`, `failed`, počitadlo `0`.

```text
Zjisti, proč objednávka 100245 nebyla expedována. Pokud ověříš selhanou synchronizaci a splnění podmínek pro opravu, proveď retry_sync. Nevytvářej incident.
```

Před potvrzením chat zobrazí nástroj `retry_sync` a parametr `order_id: 100245`. Zvol **Schválit**.

Pozorovaný průběh:

1. `get_order` potvrdil existenci a stav `processing`.
2. `get_payment` potvrdil `paid`.
3. `get_fulfillment` vrátil `not_received`.
4. `get_sync_logs` vrátil `failed` a `WMS_TIMEOUT (mock)`.
5. Po schválení proběhl `retry_sync`.
6. Agent znovu načetl logy a fulfillment.

| Pole | Před opravou | Po opravě |
|---|---|---|
| `sync_status` | `failed` | `success` |
| `fulfillment_status` | `not_received` | `received` |
| `sync_retry_count` | `0` | `1` |

V přiloženém běhu zůstal v logu původní důvod a přibyl záznam **2026-10-16T12:33:23.525+02:00 – Opakovaná synchronizace úspěšná; objednávka přijata skladem (mock).** Změnu i hodnotu počitadla potvrzuje výsledek `retry_sync`; dvě následná čtení potvrzují stav `success` a `received`. Archivní CSV zachycuje starší opakování scénáře. Odeslání zákazníkovi potvrzené není.

Výsledky nástrojů a následných kontrol: [02-approved-retry.json](../examples/02-approved-retry.json).

### Reset pro další ukázku

Před opakováním opravného scénáře uprav existující řádek `100245`:

| Pole | Výchozí hodnota |
|---|---|
| `payment_status` | `paid` |
| `fulfillment_status` | `not_received` |
| `sync_status` | `failed` |
| `sync_retry_count` | `0` jako Number |
| `sync_logs` | `Přenos do skladu selhal: WMS_TIMEOUT (mock).` |

Upravuj stávající řádek. Opakovaným přidáváním celého CSV by vznikaly duplicity `order_id`. Na čisté instanci lze založit tabulku z `orders-initial.csv`.

Bez resetu má agent další opravu odmítnout na základě již úspěšného stavu. V doloženém běhu se tak zachoval a `retry_sync` znovu nezavolal.

## 3. Chybový scénář: výpadek služby a schválený incident

Výchozí objednávka: **100247**, `processing`, `paid`, `not_received`. Pro toto ID pomocné workflow `get_sync_logs` vždy simuluje výpadek služby vyhozením výjimky v Code uzlu.

```text
Zjisti, proč objednávka 100247 nebyla expedována. Pokud narazíš na nedostupnou službu, vytvoř incident s ověřenými zjištěními. Synchronizaci neopakuj.
```

Pozorovaný průběh:

1. Agent ověřil objednávku, platbu a fulfillment.
2. `get_sync_logs` vrátil `ok: false`, `status: error`, `error_code: TOOL_UNAVAILABLE`.
3. Agent zachoval nejistotu o příčině neexpedování a neopakoval selhané volání.
4. Naplnil podmíněný požadavek vytvořením incidentu přes Human review.
5. Po schválení nástroj vrátil `ok: true`, `created: true`, `ticket_id: INC-4`, `ticket_status: open`, `request_id: 151`.

Incident **INC-4** je doložen skutečným výsledkem `create_ticket` v přiloženém JSONu; stejnou hodnotu uvádí odpověď agenta. Vstup nástroje obsahuje konkrétní `description` s ověřenými údaji, nedostupností logů a doporučeným postupem. `retry_sync` nebyl proveden. Původní `TOOL_UNAVAILABLE` zůstal v `intermediateSteps` a byl zachycen také v doloženém finálním výstupu `Build result` pro běh 151.

Výsledky nástrojů: [03-outage-with-incident.json](../examples/03-outage-with-incident.json).

Na nové instanci s prázdnou tabulkou bude první incident obvykle mít jiné ID. Očekává se shoda odpovědi agenta s ID skutečně vráceným nástrojem, nikoli pevná hodnota `INC-4`.

## Další doložené kontroly

| Vstup / situace | Pozorované chování |
|---|---|
| Diagnóza selhané `100245` bez požadavku na opravu | Čtyři čtecí nástroje, návrh řešení, žádný zápis. |
| Zamítnutí `retry_sync` | Nativní Human review akci zrušil, agent ji v témž běhu neopakoval. |
| Zamítnutí `create_ticket` při výpadku | Incident se neprovedl, chyba logů zůstala zachycená. |
| Objednávka `999999` | Jedno `get_order`, `ORDER_NOT_FOUND`, žádná navazující diagnostika. |
| Chybějící číslo | Po upřesnění promptu pouze doplňující otázka, `steps: []`, `errors: []`, execution **116**. |
| Neplatný prázdný vstup nástroje `get_order` | `VALIDATION_ERROR`; ověřeno během předchozí verze chování agenta. |
| Neplatná hodnota `not_receive` v datech | Pomocné `retry_sync` vrátilo `blocked`. Překlep byl následně opraven na `not_received`. |
| Chybějící `description` při volání incidentu | V dřívějším běhu 146 nástroj odmítl vstup podle schématu a incident nevznikl. Po upřesnění pokynu parametr obsahoval popis; přiložený běh 151 vytvořil `INC-4`. |

Tyto kontroly nenahrazují testování souběhu nebo skutečných produkčních integrací. Přesný rozsah kontroly exportů uvádí [VERIFICATION.md](VERIFICATION.md).
