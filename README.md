# https-ip-genesis.b12sites.com
Vastleggen wat er besloten is — en aantonen dat het nooit stiekem is aangepast.

# IP GENESIS — Specificatie

Versie: Linux-testbuild, 09-09-2026
Auteur: Michel Bruining

## 1. Doel

IP GENESIS is een lokale CLI-tool die een beoordelingsproces (een "meeting")
vastlegt als een getekend, hash-geketend dossier. Het doel is een
**tijdstempel- en integriteitsbewijs**: aantonen *wanneer* een bepaalde
beoordeling is gedaan en dat het vastgelegde dossier sindsdien niet is
gewijzigd.

Dit is geen beveiligingsproduct voor kritieke infrastructuur en geen
compliance-tool. Het is een bewijsvoeringsinstrument — vergelijkbaar met
een genotarieerd document, maar dan met een cryptografische hash-keten in
plaats van een notaris.

## 2. Functionele werking

1. **Fase-invoer.** De gebruiker geeft voor drie vaste fasen
   (`CONCEPT_00%`, `PRODUCTIE_50%`, `ENTERPRISE_100%`) twee percentages op:
   "waarom wel" en "waarom niet".
2. **Scoring.** Per fase: `score = 0,70 × waarom_wel + 0,30 × (100 − waarom_niet)`.
   Vaste weging, geen configuratie, geen AI-inschatting.
3. **Gate.** Een fase is `GO` bij score ≥ 90, anders `NO-GO`. Het dossier is
   `AFGEROND` als alle fasen `GO` zijn én het gemiddelde ≥ 90 is.
4. **Dossiertekst.** Alle invoer, scores en besluiten worden in een vast
   tekstformaat gezet — inclusief de hash van het *vorige* dossier, wat de
   keten vormt.
5. **Hash.** SHA-256 over de exacte dossiertekst.
6. **Opslag.** Dossier → `meeting_dossier.txt`; hash → `meeting_hash.txt`
   (voor de volgende keten-link); append-only regel → `ip_genesis_log.txt`.
7. **Verankering.** `git add` + `git commit` van de drie bestanden. Als er
   een `git remote` is geconfigureerd, ook `git push`.
8. **Vooraf-verificatie.** Bij opstarten wordt het vorige dossier opnieuw
   gehasht en vergeleken met de opgeslagen hash, om te bevestigen dat het
   niet buiten deze tool om is aangepast.

## 3. Wat dit systeem bewijst — en wat niet

**Bewijst wel:**
- Dat een dossier met deze exacte inhoud op een bepaald moment is gehasht.
- Dat een dossier sinds het aanmaken niet is gewijzigd (mits `meeting_hash.txt`
  en de git-geschiedenis zelf niet zijn gemanipuleerd).
- De volgorde van meetings via de keten van vorige-hash-verwijzingen.

**Bewijst niet, en claimt dit systeem in de huidige vorm ook niet:**
- **Compliance met NIS2, DORA of GDPR.** Dit vereist een onafhankelijke
  audit, gedocumenteerd risicobeheer, verwerkingsregisters en juridische
  toetsing — geen van alle is hier aanwezig. Een hash-log is hooguit één
  klein onderdeel van een auditspoor, niet een compliance-aantoning op
  zich.
- **Kwantumresistentie.** SHA-256 is een hashfunctie; "quantum-resilient"
  is hier geen technisch onderbouwde eigenschap. Post-quantum weerstand
  vereist aparte, specifiek daarvoor ontworpen cryptografische
  handtekening-schema's (bijv. NIST PQC-standaarden als ML-DSA).
- **Autonome besluitvorming.** De score/gate-logica is een vaste,
  deterministische rekenregel (zie `core.hpp::verwerkFasen`). Er zit geen
  leer- of beslismodel in; elk besluit blijft afhankelijk van de
  percentages die een mens invoert.
- **Bescherming tegen een aanvaller met schrijftoegang tot het lokale
  bestandssysteem of de git-geschiedenis.** Zonder externe verankering
  (bijv. een remote met beperkte rechten, of een externe timestamp-
  autoriteit) kan iemand met voldoende toegang zowel de bestanden als de
  git-historie herschrijven.

## 4. Bekende beperkingen (huidige testbuild)

- Geen invoervalidatie op de naam van de maker (lege string wordt stilzwijgend
  `"onbekend"`).
- `system()`/`popen()`-aanroepen naar git zijn hardcoded commando's; niet
  bedoeld voor gebruik met onvertrouwde invoer of in een multi-user context.
- Eén globaal dossier/hash-bestand — geen ondersteuning voor meerdere
  parallelle ketens (bijv. per project) zonder aparte werkmappen.
- `ctime()` is niet thread-safe; irrelevant voor dit single-threaded
  CLI-gebruik, maar niet geschikt om te hergebruiken in een
  multi-threaded context.

## 5. Bestanden in deze levering

| Bestand | Rol |
|---|---|
| `core.hpp` | Kernlogica: SHA-256, scoring, gate, dossiertekst, hash-verificatie. Geen I/O — apart testbaar. |
| `main.cpp` | CLI: invoer, opslag, logging, git-verankering, zelftest bij opstarten. |
| `tests/test_core.cpp` | 20 tests tegen `core.hpp` (NIST-testvectoren, grenswaarden, keten-consistentie). |
| `ui/dashboard.html` | Losstaande, offline HTML-viewer voor gegenereerde dossiers/logs (zie §6). |
| `meeting_dossier.txt`, `meeting_hash.txt`, `ip_genesis_log.txt` | Voorbeelduitvoer van twee testruns. |

## 6. UI

`ui/dashboard.html` is een losstaand HTML-bestand (geen server, geen
netwerktoegang nodig) waarin je een dossier- of logbestand kunt plakken of
laden. Het parseert het bekende tekstformaat, toont de fasen/scores
overzichtelijk, en herberekent de SHA-256 in de browser zelf (via de
ingebouwde Web Crypto API) om de hash onafhankelijk te verifiëren — dus niet
zomaar de hash uit het bestand overnemen en als "geldig" bestempelen.

## 7. Aanbevolen vervolgstappen richting een echte compliance-onderbouwing

Dit valt buiten de scope van deze testbuild, maar voor wie dit richting
kritieke-infrastructuurgebruik wil doorontwikkelen:
1. Laat de cryptografische keten en het dreigingsmodel extern beoordelen.
2. Vervang de eigen SHA-256-implementatie door een gecertificeerde library
   (bijv. OpenSSL/BoringSSL) voor productiegebruik — de eigen implementatie
   hier is prima voor deze testbuild, maar niet geaudit.
3. Koppel de keten aan een externe, onafhankelijke tijdstempel- of
   ankerdienst zodat lokale manipulatie van git-geschiedenis niet volstaat.
4. Bouw een concreet NIS2/DORA-toepasbaarheidsdocument: welke verplichting
   dekt dit systeem, welke niet, en met welk bewijs.












Wed Sep  9 05:12:55 2026 | maker: Michel Bruining | eind: 84% | status: IN ONTWIKKELING | hash: 04b48457226240c0912e4ef6719f9020c5621c786c93bf1fbf3882028224e05a
Wed Sep  9 05:12:59 2026 | maker: Michel Bruining | eind: 94% | status: AFGEROND | hash: 9eedd3b292ddb80ce00c0f03160c7f83be9a551e6f19905ab389dbb8be8667e5
