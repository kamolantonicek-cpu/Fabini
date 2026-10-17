# Ověření předaného řešení

Datum kontroly: 16. 10. 2026.

## Podklady

Kontrola vychází ze sedmi dodaných JSON exportů workflow, dvou archivních CSV exportů tabulek, tří přímých JSON exportů výstupu AI Agenta a dřívějších textových výpisů a screenshotů manuálních běhů v autorově n8n. Screenshoty nejsou součástí balíčku.

Tři soubory v `examples/` jsou obsahově i po bajtech shodné s posledními dodanými exporty `01(2).json`, `02(2).json` a `03(2).json`; změnily se pouze názvy. Obsahují `output` a `intermediateSteps`, včetně `messageLog`. Nejde o přepsané shrnutí ani o archiv celé execution. Dotazy pro reprodukci jsou v [SCENARIOS.md](SCENARIOS.md); samotné tři exporty neobsahují původní uživatelský `chatInput`.

Archivní `data/observed-*.csv` pocházejí z předchozích běhů. Dokládají dřívější opravu a incidenty včetně `INC-3`, nikoli aktuální incident `INC-4`. Základem aktuálních tří ukázek jsou skutečné výsledky nástrojů v `examples/`.

## Ověřeno v manuálních bězích

- Diagnostika `100246`: čtyři čtecí nástroje potvrdily `pending`, `on_hold` a `not_started`; agent neprovedl zápis.
- Oprava `100245`: čtení → lidské schválení → jeden retry → dvě následná čtení. V přiloženém běhu má zápis čas `2026-10-16T12:33:23.525+02:00`; výsledek obsahuje `success`, `received` a počitadlo `1`.
- Výpadek `100247`: simulovaná výjimka byla převedena na `TOOL_UNAVAILABLE`; agent zachoval nejistotu o příčině a neopakoval synchronizaci.
- Podmíněná eskalace při výpadku: přiložený výsledek `create_ticket` má neprázdné `description`, `ok: true`, `created: true`, `ticket_id: INC-4`, `request_id: 151`. Doložený finální JSON téhož běhu obsahoval chybu logů v poli `errors`.
- V předchozím běhu 146 model neposlal povinné `description`. Nástroj odmítl vstup chybou schématu a agent nevydával incident za vytvořený. Po doplnění pokynu pro parametr následoval úspěšný běh 151. Jeden úspěšný běh nezaručuje bezchybnost všech dalších rozhodnutí LLM.
- Dřívější manuální kontroly doložily zamítnutí obou zápisů bez nového schvalování v témž běhu, ukončení po `ORDER_NOT_FOUND`, prázdný vstup a vyžádání chybějícího čísla objednávky bez nástroje.
- Agent nevolal další retry pro již přijatou objednávku.
- Dřívější sekvenční test stejného `request_id` vrátil existující `INC-1` s `created: false`; pozdější export deduplikační logiky byl zkontrolován staticky.

## Zkontrolováno z exportů

- Sedm workflow i tři aktuální příklady jsou platné JSONy. Názvy uzlů jsou v jednotlivých workflow jednoznačné a propojení míří na existující uzly.
- Každý z šesti nástrojů odkazuje na odpovídající pomocné workflow. Namapované vstupy odpovídají jeho vstupnímu schématu.
- `retry_sync` i `create_ticket` jsou připojené přes Chat Human review. V exportu není jejich přímé napojení do AI Agenta, které by tuto cestu obcházelo.
- AI Agent má `maxIterations: 10`, `returnIntermediateSteps: true`. Workflow nemají připnutá testovací data.
- `request_id` incidentu je výraz `$execution.id` z hlavního workflow a není doplňován modelem.
- Čtecí workflow a retry kontrolují prázdné ID i nenalezenou objednávku. Get je omezený na jeden řádek a odkazy v chybových uzlech míří na správné vstupní uzly.
- Retry vyžaduje současně `paid`, `not_received`, `failed` a číselné počitadlo `0`. Update mapuje pouze čtyři zamýšlená pole a znovu filtruje počitadlo `0`.
- Oba zapisující uzly mají `continueErrorOutput`, chybu připojenou do `Return write error` a vypnuté automatické retry. Výsledek chyby obsahuje `WRITE_RESULT_UNKNOWN` a `write_result: unknown`.

## Úpravy připraveného balíčku

Při předání byly odstraněny nepřenositelné odkazy na přístup k modelu a metadata, workflow byla deaktivována pro import a výslovně bylo zapnuto ukládání executions. Vazby se po importu převažují podle README.

V hlavním workflow jsou navíc dvě konkrétní úpravy proti původně dodanému exportu:

1. Parametr `description` v nástroji `create_ticket` obsahuje podrobný pokyn pro model. Není nahrazen pevným popisem incidentu; hodnotu stále vytváří model a následně ji schvaluje člověk.
2. `Build result` zahrnuje do `errors` také nativní výsledky ve tvaru `{"error": "zpráva"}`. Bez této úpravy zůstala chyba schématu v historii kroků a slovním souhrnu, ale chyběla v souhrnném seznamu chyb. Upravený uzel ji označuje `TOOL_ERROR` a zachovává skutečnou zprávu.

Podmínky zapisujících workflow ani propojení nástrojů se těmito úpravami nemění. Nové balíčkové soubory samy nemění otevřenou instanci autora.

## Lokální kontrola JavaScriptu a souborů

Aktuální `Build result` byl ověřen nad všemi třemi přiloženými výstupy AI Agenta. Kontrola porovnává pořadí, názvy, parametry a skutečné výsledky nástrojů, zachování odpovědi modelu, chybu `TOOL_UNAVAILABLE` a vynechání objemných `messageLog` z finálního objektu.

Zvlášť byl znovu přehrán dříve zachycený případ chyby schématu s chybějícím `description`: aktuální agregace musí zaznamenat výpadek logů i nativní chybu nástroje. Kontroly zamítnutí, blokování a chybějících `intermediateSteps` ověřují rozlišení chyby od neprovedené akce. Jde o lokální vykonání JavaScriptu nad uloženými daty, nikoli nové spuštění n8n nebo LLM.

Dřívější lokální kontrola ověřila také kód simulace výpadku, podmínky retry a vybrané validační výrazy. Při aktualizaci balíčku byly zkontrolovány relativní odkazy dokumentace, vazby workflow, shoda aktuálních příkladů s dodanými soubory a integrita ZIPu.

## Co tímto ověřeno není

- Import a spuštění připraveného balíčku ve druhé čisté instanci n8n. Převázání ID tabulek, workflow a přístupu k modelu je popsané v README.
- Nový běh upravené agregace nativních chyb uvnitř n8n. Její JavaScript byl ověřen lokálně nad skutečně zachycenou chybou.
- Skutečné ERP/WMS, platby, síťové timeouty nebo reálné obchodní operace.
- Souběžné zakládání incidentů a závody mezi kontrolou a zápisem retry.
- Runtime vynucení chyby samotného zápisu do Data Tables. Chybové větve byly zkontrolované z konfigurace.
- Samostatné manuální spuštění všech validačních větví každého pomocného workflow.
- Globální náhradní odpověď při chybě orchestrace, vyčerpání iterací nebo dlouhém čekání na schválení.

Slovní pole `summary` vytváří LLM. Označení akce jako „blokováno“ může znamenat i rozhodnutí nástroj vůbec nevolat. Skutečná volání dokládají `steps` ve finálním objektu nebo `intermediateSteps` v přiložených exportech. Změnu systému potvrzuje výsledek nástroje a případná následná kontrola.
