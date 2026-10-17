# Testovací scénáře

Ověřeno 16. 10. 2026. Všechny tři ukázky používají mock data.

Ve složce `examples/` jsou výstupy uzlu **AI Agent**: odpověď v `output` a skutečná volání v `intermediateSteps`. Dotazy pro opakování jsou níže. Původní vstup uloženého běhu je v **User input → Output → JSON → chatInput**; finální zjednodušený JSON je v **Build result**.

## 1. Diagnostika bez změny

```text
Zjisti, proč objednávka 100246 nebyla expedována. Proveď pouze diagnostiku, nic neměň.
```

Agent použil čtyři čtecí nástroje. Zjistil `pending`, `on_hold` a `not_started`; log potvrdil čekání na zaplacení. Nic neměnil. Tento test lze opakovat bez resetu dat.

Výstup: [01-unpaid-diagnosis.json](../examples/01-unpaid-diagnosis.json).

## 2. Oprava synchronizace se schválením

```text
Zjisti, proč objednávka 100245 nebyla expedována. Pokud ověříš selhanou synchronizaci a splnění podmínek pro opravu, proveď retry_sync. Nevytvářej incident.
```

Pro tento test musí být objednávka zaplacená, synchronizace selhaná a sklad ji nesmí mít přijatou. V chatu potvrďte `retry_sync` tlačítkem **Schválit**.

Agent nejdřív ověřil data a chybu `WMS_TIMEOUT`. Po schválení provedl jeden retry a znovu načetl logy a fulfillment:

| Pole | Před opravou | Po opravě |
|---|---|---|
| `sync_status` | `failed` | `success` |
| `fulfillment_status` | `not_received` | `received` |
| `sync_retry_count` | `0` | `1` |

Výstup: [02-approved-retry.json](../examples/02-approved-retry.json).

**Před opakováním** upravte existující řádek `100245` v tabulce objednávek:

- `payment_status`: `paid`
- `fulfillment_status`: `not_received`
- `sync_status`: `failed`
- `sync_retry_count`: číslo `0`
- `sync_logs`: `Přenos do skladu selhal: WMS_TIMEOUT (mock).`

Bez resetu se další oprava nemá provést. Nepřidávejte znovu celé CSV do stejné tabulky, aby nevznikly duplicitní objednávky.

## 3. Výpadek služby a vytvoření incidentu

```text
Zjisti, proč objednávka 100247 nebyla expedována. Pokud narazíš na nedostupnou službu, vytvoř incident s ověřenými zjištěními. Synchronizaci neopakuj.
```

U `100247` pomocné workflow záměrně simuluje výpadek služby logů. Agent ověřil objednávku, platbu a fulfillment, ale logy vrátily `TOOL_UNAVAILABLE`. Přesnou příčinu nepředání skladu proto nepotvrdil a synchronizaci neopakoval.

V chatu potvrďte **create_ticket**. V přiloženém běhu vznikl incident **INC-4**, `created: true`, `request_id: 151`. Na jiné instanci se ID může lišit. Odpověď má vždy uvádět ID vrácené nástrojem.

Výstup: [03-outage-with-incident.json](../examples/03-outage-with-incident.json).

Třetí ukázka je chybový scénář: služba je nedostupná, agent chybu zachytí a předá problém k řešení přes incident.

Další kontroly a rozsah ověření jsou v [VERIFICATION.md](VERIFICATION.md).
