# Import a řešení potíží

Základní postup je v [README](../README.md#spuštění). Tento přehled vychází z problémů při importu z 16. 10. 2026 a dokumentace n8n. **Nejde o potvrzení kompletního spuštění v druhé čisté instanci.**

Přesná verze použité instance n8n není v podkladech zaznamenaná. Export obsahuje AI Agent **3.1**, Call n8n Workflow Tool **2.2**, Chat Human review **1.3** a Data Table **1.1**. To jsou verze uzlů, ne celé aplikace. Názvy tlačítek se mohou podle verze n8n lišit.

## 1. Tabulky: původní ID a „Error fetching options“

Tabulky i workflow musí být ve stejném projektu. Tabulky v n8n pojmenujte **Fabini - orders** a **Fabini - incidents**.


| Kde                           | Uzel                        | Vybraná tabulka    |
| ----------------------------- | --------------------------- | ------------------ |
| Pomocná workflow `02` až `06` | Find order                  | Fabini - orders    |
| `06-retry-sync.json`          | Apply sync retry            | Fabini - orders    |
| `07-create-ticket.json`       | Find ticket a Insert ticket | Fabini - incidents |


Export obsahuje původní ID. **V poli Data table znovu vyberte svoji tabulku, i když už je zobrazený stejný název.** Při duplicitních názvech ověřte ID nové tabulky v její adrese; v uzlu ho lze zadat přes **By ID**.

Při hlášce **„Error fetching options“** nebo **„Data table with ID … could not be found in this project“**:

1. Vyberte nově vytvořenou tabulku z aktuálního projektu.
2. U pole **Column** otevřete tři tečky a zvolte **Refresh List**, tedy obnovení seznamu sloupců.
3. Zkontrolujte podmínku podle tabulky níže. Hodnoty s `{{ … }}` vložte jako **Expression**.


| Uzel        | Column                | Condition | Value                                       |
| ----------- | --------------------- | --------- | ------------------------------------------- |
| Find order  | `order_id (string)`   | Equals    | `{{ String($json.order_id ?? '').trim() }}` |
| Find ticket | `request_id (string)` | Equals    | `{{ $json.request_id }}`                    |


V těchto čtecích uzlech ponechte **Return All vypnuté** a limit **1**. Pokud možnosti zůstávají staré, zavřete a znovu otevřete uzel.

`order_id` **a** `request_id` **mají být String.** Pokud CSV vytvoří `order_id` jako Number a při importu nelze typ nastavit, založte tabulku ručně podle [schématu](../data/table-schema.json) a vložte tři objednávky z [výchozích dat](../data/orders-initial.json).

Zapisující uzly ponechte podle exportu. **Apply sync retry** má filtr **All Conditions**: správné `order_id` a počitadlo `sync_retry_count = 0`. Mění jen `sync_status`, `fulfillment_status`, `sync_retry_count` a doplňuje `sync_logs`. **Insert ticket** zapisuje `request_id`, `status` a `description`. Automatické opakování zápisu zůstává vypnuté a chybový výstup připojený.

## 2. Pomocná workflow a přesné vstupy

V hlavním workflow znovu vyberte odpovídající **nově importované** pomocné workflow:


| Nástroj           | Soubor                    |
| ----------------- | ------------------------- |
| `get_order`       | `02-get-order.json`       |
| `get_payment`     | `03-get-payment.json`     |
| `get_fulfillment` | `04-get-fulfillment.json` |
| `get_sync_logs`   | `05-get-sync-logs.json`   |
| `retry_sync`      | `06-retry-sync.json`      |
| `create_ticket`   | `07-create-ticket.json`   |


Při stejných názvech ověřte ID nového workflow v jeho adrese; lze ho zadat i místo výběru ze seznamu.

**Po každé změně cílového workflow zkontrolujte Workflow Inputs.** Přes **Refresh**, tedy obnovení vstupů, načtěte schéma a doplňte prázdná pole. Všechny níže uvedené vstupy jsou **String**:


| Nástroj v hlavním workflow                                                   | Pole          | Hodnota v režimu Expression                                                                                                                   |
| ---------------------------------------------------------------------------- | ------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `get_order`, `get_payment`, `get_fulfillment`, `get_sync_logs`, `retry_sync` | `order_id`    | `{{ $fromAI('order_id', 'Číslo objednávky z požadavku uživatele.', 'string') }}`                                                              |
| `create_ticket`                                                              | `description` | `{{ $fromAI('description', 'Povinný neprázdný popis incidentu: objednávka, ověřená zjištění, chyba, nejistoty a další postup.', 'string') }}` |
| `create_ticket`                                                              | `request_id`  | `{{ $execution.id }}`                                                                                                                         |


U `order_id` a `description` lze místo ručního výrazu kliknout na **✨**: zobrazí se **Defined automatically by the model**. `request_id` doplňuje workflow, proto u něj ponechte uvedený výraz bez AI.

Popisy uvnitř `$fromAI` jsou doporučené pro obnovení nastavení. Mohou se lišit od stručnějších popisů v exportu; názvy parametrů a typy zůstávají stejné.

**„No parameters are set up to be filled by AI“**: obnovte vyplňování `order_id`, případně `description`, podle tabulky.

Když se pole nezobrazí ani po obnovení, zkontrolujte první vstupní uzel pomocného workflow: soubory `02` až `06` mají `order_id`; `07` má `request_id` a `description`, vždy String. Pak vstupy nástroje obnovte znovu.

`$fromAI` patří do nástrojů hlavního agenta. Uvnitř pomocných workflow zůstávají výrazy s `$json`. `retry_sync` a `create_ticket` ponechte připojené přes schvalovací uzel **Chat**.

## 3. „Workflow is not active and cannot be executed“

1. Ověřte, které pomocné workflow nástroj skutečně volá.
2. Otevřete tuto nově importovanou kopii a publikujte ji přes **Publish**; ve starších rozhraních může být aktivace označená **Active**.
3. Pro produkční chat publikujte všech šest pomocných workflow a potom hlavní workflow. Po dalších změnách znovu publikujte upravenou verzi.
4. Spusťte nový dotaz z hlavního chatu a ověřte výsledek v historii běhu.

Uložení změn a publikování jsou různé kroky. Před testem ověřte, že editor potvrzuje uložené změny; pokud rozhraní nabízí **Save**, použijte ho. Produkční volání používá publikovanou verzi.

Dokumentace n8n: [Data Tables](https://docs.n8n.io/data/data-tables/), [parametry z AI](https://docs.n8n.io/advanced-ai/examples/using-the-fromai-function/), [Call n8n Workflow Tool](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolworkflow/) a [publikování](https://docs.n8n.io/workflows/publish/).