# Co jsem ověřil

Manuální testy v n8n proběhly 16. 10. 2026. Přiložil jsem tři výstupy AI Agenta a CSV tabulek po testech.


| Scénář               | Výsledek                                                                                                                         |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Diagnostika `100246` | Agent zjistil čekání na platbu a nic nezměnil.                                                                                   |
| Oprava `100245`      | Po schválení proběhl jeden retry. Kontrolní čtení potvrdila `success` a `received`, počitadlo je `1`.                            |
| Výpadek u `100247`   | Agent zachytil `TOOL_UNAVAILABLE` a po schválení vytvořil **INC-4**. Synchronizaci neopakoval a příčinu nevydával za potvrzenou. |


Zkoušel jsem také zamítnutí obou zapisujících akcí, neexistující objednávku, chybějící a prázdné ID, další retry již přijaté objednávky a opakované volání incidentu se stejným `request_id`. Zamítnuté akce agent v témž běhu znovu nevyžadoval. Stejné ID požadavku vrátilo existující incident.

## Kontrola souborů

Z exportů byly zkontrolované odkazy mezi workflow, schvalování zápisů, limit 10 iterací a podmínky retry. Zapisující uzly mají vypnuté automatické opakování a připojené chybové větve.

JavaScript uzlu **Build result** byl samostatně ověřen nad uloženými výstupy. Kontrola zahrnovala historii nástrojů, vstupy, výsledky a sběr chyb. Tato část proběhla lokálně mimo n8n. Zkontrolovaný je také obsah a integrita ZIPu.

## Podklady k návodu po importu

[Návod po importu](TROUBLESHOOTING.md) vychází z potíží při importu a dokumentace n8n. Popsané opravy nejsou dokladem kompletního importu a spuštění všech scénářů v druhé čisté instanci. Přesná verze použité instance není v podkladech zaznamenaná; uvedené verze uzlů ji nenahrazují.

## Co zatím otestované není

Import do druhé čisté instance, reálné systémy, souběžné zápisy ani vynucená chyba zápisu do Data Tables. Všechny validační větve pomocných workflow nebyly samostatně spuštěné.

Dotazy a přiložené výsledky jsou v [SCENARIOS.md](SCENARIOS.md).