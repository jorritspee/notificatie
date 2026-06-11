# Notificatie & Subscripties op events in zorgsystemen

author: "Ministerie van VWS"
docversion: "0.1"
docversiondate: "2026-06-07"
docstatus: "concept"
---

### Inhoudsopgave

1. [Inleiding](#inleiding)
2. [Functionele doelstellingen](#functionele-doelstellingen)
3. [Privacy](#privacy)
4. [Solution overview](#solution-overview)
5. [Actoren](#actoren)
6. [Onderwerp: keuze van SubscriptionTopic-codering](#onderwerp-keuze-van-subscriptiontopic-codering)
7. [Transacties](#transacties)
8. [Notificatie-inhoud](#notificatie-inhoud)
9. [Relatie tot andere specificaties](#relatie-tot-andere-specificaties)
10. [Security](#security)
11. [Voorbeeld use cases](#voorbeeld-use-cases)
12. [Roadmap en open punten](#roadmap-en-open-punten)


### Inleiding

Notificeren in de zorg is geen op zichzelf staand technisch mechanisme, maar een functionele behoefte en afspraak tussen partijen. Het vertrekpunt is de informatiebehoefte van de ontvanger: een arts, afdeling, patiënt of ondersteunend systeem wil tijdig weten dat een relevante gebeurtenis heeft plaatsgevonden. De verzender faciliteert die behoefte op vooraf afgesproken onderwerpen en filters. Notificaties zijn daarmee ontvanger-gedreven, proportioneel en altijd ingebed in de geldende kaders voor identificatie, authenticatie, autorisatie, toestemming en adressering.

Deze specificatie beschrijft hoe organisaties in de Nederlandse zorg zulke notificaties eenduidig en veilig uitwisselen. Als technische basis gebruiken we het [FHIR R5 Subscription Framework](https://hl7.org/fhir/R5/subscriptions.html), toegepast in R4 via de [Subscriptions R5 Backport IG](https://hl7.org/fhir/uv/subscriptions-backport/). Voortbouwend op het werk van de werkgroep "geharmoniseerde notified pull" legt dit document de functionele uitgangspunten en de technische keuzes vast die nodig zijn voor interoperabele, implementeerbare notificatie-uitwisseling.

#### Normatieve termen

In deze specificatie worden sommige woorden bewust in HOOFDLETTERS geschreven, zoals `MOET`, `MAG NIET`, `BEHOORT` en `MAG`. Dit markeert normatieve kracht en helpt om eisen, aanbevelingen en opties eenduidig te onderscheiden:

- `MOET` / `MOETEN` (`SHALL`): harde eis.
- `MAG NIET` (`SHALL NOT`): expliciet verbod.
- `BEHOORT` (`SHOULD`): sterke aanbeveling waarvan alleen met goede reden kan worden afgeweken.
- `MAG` (`MAY`): toegestane optie.


#### Scope

Binnen scope:

- Het uitwisselen van betekenisarme (`id-only`) notificaties tussen zorgaanbieder-organisaties.
- De keuze en codering van notificatie-onderwerpen (SubscriptionTopic).
- De relatie tot adressering (GF Adressering), autorisatie (GF Autorisatie) en patiënttoestemming (Mitz/EHDS), zonder uitwerking van een adresseringsprofiel.

Buiten scope:

- De inhoudelijke pull die op een notificatie kan volgen (zie Notified Pull, Clinical Order Workflow).
- De interne verwerking en routering bij de ontvanger na aankomst van de notificatie (zie TA Routering).
- De technische invulling van toestemmingsregistratie (zie OTV/Mitz en aankomende EHDS-verordening).
- De concrete registratie, publicatie en resolutie van notificatie-endpoints (zie GF Adressering).


### Functionele doelstellingen

De volgende voorbeelden illustreren waarom een notificatie functioneel waardevol is. Ze keren in deze specificatie terug als concrete use cases (zie [Voorbeeld use cases](#voorbeeld-use-cases)).

1. **Lab-uitslag voor de huisarts.** Een huisarts wil een bericht zodra een door haar aangevraagde laboratoriumuitslag voor patiënt X bij het ziekenhuislab beschikbaar is, zodat zij die op tijd kan beoordelen en eventueel met de patiënt kan bespreken.
2. **OK-verplaatsing voor de thuismonitoring.** Een thuismonitoring-zorgaanbieder wil weten wanneer een geplande operatie van een patiënt in het ziekenhuis verplaatst is, zodat de monitoring-planning daarop aangepast kan worden.
3. **Dossierinzage voor de patiënt.** Een patiënt wil bericht ontvangen wanneer iemand haar dossier heeft ingezien, zodat ze achteraf weet dat een raadpleging heeft plaatsgevonden en daar zo nodig op kan reageren.
4. **Verwijzing van zorgaanbieder A naar zorgaanbieder B.** Een verwijzende zorgaanbieder wil de ontvangende zorgaanbieder direct informeren wanneer een verwijzing is aangemaakt of inhoudelijk is gewijzigd, zodat de vervolgzorg zonder onnodige vertraging kan starten.

#### Signaal-moeheid en specificiteit

Zorgverleners hebben een eindige hoeveelheid aandacht. Het is een breed gedragen wens, zowel bij zorgverleners zelf als bij de overheid, om die aandacht efficiënt in te zetten. Een notificatiesysteem dat te grofmazig werk, produceert *signaal-moeheid*: Ontvangers leren de meldingen te negeren en daarmee verdwijnt ook de waarde van de meldingen die wél relevant zijn. Het tegenovergestelde (geen melding sturen) laat de patiënt tussen wal en schip vallen.

De ontwerp-implicatie is dat notificaties **specifiek genoeg moeten zijn** om voor een ontvanger relevant te zijn, zonder dat de specificiteit ten koste gaat van privacy. Specifiek betekent hier: per onderwerp (welk type gegeven), per zender (welke organisatie), en — waar van toepassing — per filter (welke patient, uitvoerder, specifieke lab-uitslag-codes of welke verfijning binnen het onderwerp). Hoe deze specificiteit technisch wordt bereikt staat in [Onderwerp: keuze van SubscriptionTopic-codering](#onderwerp-keuze-van-subscriptiontopic-codering).


### Privacy

Een notificatie draagt in deze specificatie geen klinische inhoud (zie [Notificatie-inhoud](#notificatie-inhoud)). Toch is een notificatie nooit volledig betekenisloos voor de privacy van de patiënt: het feit dát een lab-uitslag of een dossier-inzage plaatsvindt, koppelt al een patiënt aan een type zorgvraag en aan een zorgrelatie zorgrelatie met een bepaalde organisatie. Dat is, hoe dun ook, een dossierkenmerk.

#### Dataminimalisatie

Notificaties MOETEN een `id-only` payload gebruiken: de notificatie meldt dát er iets is gebeurd en geeft een referentie naar de bron, niet de inhoud zelf. De feitelijke gegevens worden door de ontvanger opgehaald via een aparte bevraging ('pull'), waarbij de verzender op dat moment opnieuw beoordeelt of toegang is toegestaan. Daarmee blijft de hoeveelheid persoonsgegevens die over het notificatiekanaal beweegt minimaal.

#### Patiënt en toestemming

Een patiënt moet kunnen verhinderen dat over haar gegevens notificaties tussen zorgaanbieders worden uitgewisseld, zelfs wanneer die notificaties dun zijn. De EHDS-verordening (Verordening (EU) 2025/327), artikel 9, bevestigt dit beginsel en breidt de transparantie- en logging-eisen uit naar elke vorm van toegang, ook geautomatiseerde. De technische invulling van dat blokkade-recht valt buiten deze specificatie: ze wordt geregeld door bestaande en aankomende toestemmingsmechanismen.

Voor de Nederlandse context is op dit moment **Mitz** de operationele invulling: opt-in toestemming op combinaties van zorgaanbieder-categorie van de bronhouder (in het geval van notificaties: verzender) en zorgaanbieder-categorie van de raadpleger (in het geval van notificaties: ontvanger). Een verzender MOET vóór het versturen van een notificatie controleren dat er voor de betrokken patiënt een geldige toestemming of andere grondslag is voor de uitwisseling met de ontvangende categorie. Zonder toestemming of andere grondslag MAG GEEN notificatie worden verstuurd.

Naar verwachting verschuift dit onder EHDS in de richting van doelgebonden (opt-out) toestemmingen en richting een patiënt-toegankelijk audit-log van álle toegang en uitwisseling.


### Solution overview

Een notificatie-uitwisseling tussen twee organisaties verloopt in een aantal stappen. Stappen 1 en 2 zijn voorbereidend (eenmalig per subscriptie/onderwerp); 3 en 4 zijn de operationele stappen die per gebeurtenis terugkeren. Stap 5 dekt herstel na een gemiste notificatie of een uitval van het kanaal.

1. **Subscription registratie.** Aan de zijde van de verzendende organisatie wordt een `Subscription` aangemaakt voor de ontvangende organisatie, gebonden aan een vooraf afgesproken `SubscriptionTopic`. De afspraak hierover ligt bij de ontvanger: zij wíl voor dit onderwerp genotificeerd worden. Hoe de Subscription technisch tot stand komt is per use case te bepalen:
    - in-band: door de ontvanger zelf gePOST; of
    - out-of-band: door de verzender ingericht op basis van een eerder gemaakte afspraak  Voor de out-of-band subscripties wordt het notificatie-endpoint van de ontvanger opgezocht via GF Adressering (`Endpoint.connectionType = hl7-fhir-rest`, `payloadType = Subscription`).
2. **Handshake.** Direct na registratie verstuurt de verzender een `handshake-notification` naar het notificatie-endpoint van de ontvanger om het kanaal te bevestigen en zet `Subscription.status` op `active`. *VRAAG: Is dit zowel bij in-band als out-of-banmd zo?*
3. **Event-notificaties.** Telkens wanneer aan de zijde van de verzender een gebeurtenis optreedt die past bij het topic (en het eventuele filter), verstuurt de verzender een `event-notification` Bundle. Elke notificatie krijgt een monotoon oplopend `event-number` zodat de ontvanger gemiste meldingen kan detecteren. De notificatie bevat geen klinische inhoud — alleen een `focus`-referentie (literal reference) naar de bron-resource bij de verzender.
4. **Heartbeats.** Op vooraf afgesproken intervallen verstuurt de verzender een `heartbeat-notification` zodat de ontvanger een uitval van het kanaal kan opmerken, ook wanneer er feitelijk geen gebeurtenissen plaatsvinden (optioneel als ontvanger dit opgeeft bij aanmaken van Subscription).
5. **Catch-up bij verstoring.** Wanneer de ontvanger een gat in de `event-number`-reeks ziet of een verwachte heartbeat mist, gebruikt zij de operaties `$event` en `$status` op de Subscription bij de verzender om gemiste notificaties op te halen of de status van de Subscription te verifiëren.

De ontvanger kan vervolgens de inhoud achter de `focus`-referentie ophalen via het FHIR-endpoint van de verzender. De inhoudelijke pull en de bijbehorende toegangscontrole vallen buiten deze specificatie (zie GF's Identificatien, Authenticatie, Autorisatie en Toestemming).


### Actoren

Twee actoren spelen in deze specificatie een operationele rol.

#### Subscription Server

De Subscription Server draait aan de zijde van de verzendende organisatie en is verantwoordelijk voor het beheer van Subscriptions en de aflevering van notificaties. De Subscription Server MOET:

- `Subscription`-resources aanmaken voor de afgesproken `SubscriptionTopic`(s) (o.b.v. vooraf afgesproken onderwerpen of als gevolg van een Subscription Client die een subscription vraagt);
- `handshake-notification` Bundles versturen wanneer `Subscription.status` op `requested` staat (met retry), en `Subscription.status` daarop bijwerken;
- `heartbeat-notification` Bundles op vastgestelde intervallen versturen (met retry);
- `event-notification` Bundles versturen waarbij `event-number` op een concurrency-veilige manier wordt opgehoogd;
- de operaties `$status` en `$event` op de Subscription ondersteunen, evenals `read` en `search` met zoekparameters `status`, `criteria`, `channel.endpoint`, `channel.type` en `channel.payload`;
- vóór elk versturen verifiëren dat er een geldige Subscription voor deze ontvanger is, en dat aan de toestemmingsvoorwaarden voor de betrokken patiënt is voldaan (zie [Privacy en toestemming](#privacy-en-toestemming)).

De Subscription Server MAG het aanmaken van Subscriptions door Subscription Clients ondersteunen (in-band beheerde Subscriptions). *VRAAG: is dit niet MOET als we use case 1 willen ondersteunen?*

#### Subscription Client

De Subscription Client draait aan de zijde van de ontvangende organisatie en consumeert notificaties. De Subscription Client MOET:

- notificatie-Bundles ontvangen op het notificatie-endpoint en deze intern doorgeven aan het verwerkende proces;
- elke binnenkomende Bundle controleren op continuïteit, gebruikmakend van het hoogst verwerkte `event-number`; gemiste notificaties worden opgehaald via de `$event`-operatie bij de verzender;
- (optioneel) op afgesproken intervallen controleren of een `heartbeat-notification` is uitgebleven; zo ja, dan MOET de Subscription-status via `$status` worden opgevraagd.

De Subscription Client MAG ook op eigen initief `Subscription`-resources aanmaken voor de afgesproken `SubscriptionTopic`(s) bij een Subscription server. *VRAAG: is dit niet MOET als we use case 1 willen ondersteunen?*

De Subscription Client is verantwoordelijk voor het signaleren en herstellen van foutieve communicatie, en BEHOORT daarom de toestand van haar Subscriptions bij te houden.


### Onderwerp: keuze van SubscriptionTopic-codering

Het onderwerp van een notificatie wordt vastgelegd in een `SubscriptionTopic`, waarvan de canonical-URL aan `Subscription.criteria` wordt gebonden (via de Backport-extension `backport-topic-canonical`). De vraag is: welke codering gebruiken we voor de topics in de Nederlandse zorgcontext?

Deze specificatie kiest voor de [NL GF Data Categories CodeSystem](https://minvws.github.io/generiekefuncties-docs/CodeSystem-nl-gf-data-categories-cs.html) als basis voor topic-codering. De argumentatie staat hieronder; concrete mapping van álle denkbare use cases naar codes is een implementatie-detail dat buiten deze specificatie valt.

#### Argument 1: voldoende specifiek voor de functionele doelstellingen

De data-categorieën onderscheiden onderwerpen op een niveau dat past bij de use cases uit [Functionele doelstellingen](#functionele-doelstellingen). Een lab-uitslag valt onder `ObservationLaboratory`; een verplaatste OK-afspraak onder `Procedure` (en `Encounter`); een dossier-inzage onder `Logging` (AuditEvent). Dit niveau is voldoende fijn om relevant te zijn voor de ontvanger, voldoende grof om als nationaal herbruikbaar onderwerp te dienen en voldoende grof om niet ten koste te gaan van privacy.

#### Argument 2: hergebruik van triggers vanuit GF Lokalisatie

Voor [GF Lokalisatie (NVI)](https://minvws.github.io/generiekefuncties-docs/) moeten verzenders (datahouders) deze data-categorieën sowieso al implementeren — als labels op de gegevens die ze beschikbaar maken voor lokalisatie-queries (aanmelding bij de nationale verwijsindex (NVI)). De interne triggers en events die daarbij horen (bijv. een nieuwe `ObservationLaboratory` die wordt aangemaakt, een `Encounter` die wordt gemuteerd) kunnen ongewijzigd worden hergebruikt voor het samenstellen van notificaties. Dezelfde data-categorieen worden ook gebruikt voor de typering van Endpoints in de generieke functie Adressering. Hergebriuk van deze data-categorieen in notificaties houdt de implementatielast bij verzenders beperkt en zorgt ervoor dat lokalisatie, adressering en notificatie semantisch consistent blijven.  

#### Voorbeeld: lab-uitslag voor de huisarts

Voor de eerste use case uit [Functionele doelstellingen](#functionele-doelstellingen) wordt het topic geconstrueerd rond de code `ObservationLaboratory` uit de NL GF Data Categories codeset. Wanneer een ontvanger niet álle lab-notificaties van deze verzender wil ontvangen (bijv. met het oog op de gegevensverwerkingsgrondslag of signaal-moeheid), kan de Subscription worden afgebakend met de Backport-extension `backport-filter-criteria`. Bijvoorbeeld:

- `backport-filter-criteria = patient=Patient/1234567890` beperkt notificaties tot één specifieke patiënt;
- `backport-filter-criteria = code=http://loinc.org|2951-2` beperkt tot één specifiek laboratoriumonderzoek (natrium in serum);
- combinaties zijn mogelijk om bijvoorbeeld alleen vitale-parameter-uitslagen voor een bepaalde monitoring-cohort te ontvangen.

Het topic blijft daarmee een gedeelde, nationaal afgesproken eenheid; de afbakening per Subscription is een lokale keuze van de ontvanger en is een belangrijk instrument om irrelevante notificaties te voorkomen. *vraag: is het slim om een eindige set aan toegestane filters af te spreken? Dus bijvoorbeeld MOET ondersteunen de volgende search/filter parameters: patient, code, category*

#### Open einde: Nictiz-review en mogelijke SNOMED/LOINC-verschuiving

De NL GF Data Categories codeset staat sinds enige tijd in review bij Nictiz en kan op onderdelen nog wijzigen. In de nieuwe [BgZ versie 2.0 (Technical IG, paragraaf 4.3)](https://informatiestandaarden.nictiz.nl/wiki/bgz:V2.0_Technical_IG_BgZ_MSZ) wordt voor data-categorisering gewerkt met SNOMED- en LOINC-codes. Het is denkbaar dat de uiteindelijke nationale codeset voor topic-codering in die richting verschuift; de onderwerpen die er semantisch in zitten zijn niet wezenlijk anders dan in de huidige NL GF Data Categories. Deze specificatie kan op dat punt zonder structurele wijzigingen worden bijgewerkt — alleen de canonical-URL's en displaynames van topics zouden meebewegen.

### Transacties

De onderstaande transacties verlopen tussen Subscription Server en Subscription Client over een mTLS-beveiligd kanaal (zie [Security](#security), na publicatie van de specificaties van Veilig Netwerk zal daarnaar worden verwezen). De wire-format en gedragsdetails volgen de [Subscriptions R5 Backport voor R4](https://hl7.org/fhir/uv/subscriptions-backport/); hieronder beperken we ons tot de eisen die in deze context aanvullend of beperkend zijn.

#### Subscription aanmaken

Een `Subscription` wordt aangemaakt aan de zijde van de verzender. Dit gebeurt 'out-of-band' (de verzender richt de Subscription in op basis van een, via GF Adressering, verkregen notificatie-endpoint van de ontvanger) of 'in-band' (de ontvanger POST een Subscription naar de Server). Een Subscription:

- MOET een vooraf afgesproken `SubscriptionTopic`-canonical bevatten op `Subscription.criteria` via de extension `backport-topic-canonical`;
- MAG een afbakenend filter dragen via de extension `backport-filter-criteria` (zie [voorbeeld](#voorbeeld-lab-uitslag-voor-de-huisarts));
- MOET `Subscription.channel.type = rest-hook` bevatten, met het notificatie-endpoint van de ontvanger;
- MOET `Subscription.channel.payload = application/fhir+json` bevatten, en de extension `backport-payload-content = id-only` (zie [Notificatie-inhoud](#notificatie-inhoud));
- MOET bij gebruik op `active` staan en bij retirement op `off` worden gezet (niet verwijderd).

Een Subscription is bedoeld als langlopende afspraak tussen Subscription Server & Subscription Client over een topic, niet als per-geval object.

#### Handshake-notification

Na aanmaak van een Subscription verstuurt de Subscription Server een `handshake-notification` Bundle naar het notificatie-endpoint van de ontvanger. Bij succes wordt `Subscription.status` op `active` gezet; bij falen wordt met exponentiële backoff geretryd, en bij blijvend falen op `error` gezet. *VRAAG: Is de handshake-notification ook verplicht voor 'out-of-band' Subscriptions? DIt zou dan een extra transactie zijn t.o.v. de huidige TANP/TAeO flows.*

#### Event-notification

Bij elke gebeurtenis die past bij het topic en het filter verstuurt de Server één `event-notification` Bundle naar het notificatie-endpoint van de ontvanger. De Bundle:

- MOET een monotoon oplopend `event-number` bevatten, op een concurrency-veilige manier toegekend;
- MOET een `notification-event.focus` bevatten, die een literal reference naar de bron-resource bij de verzender als waarde heeft;
- MAG GEEN klinische inhoud dragen (zie [Notificatie-inhoud](#notificatie-inhoud)).

#### Heartbeat-notification

Op vooraf afgesproken intervallen verstuurt de Subscription Server een `heartbeat-notification` Bundle. Dit dient als levensteken voor het kanaal en stelt de ontvanger in staat een stille uitval te detecteren ook wanneer er geen events optreden.

#### `$status` en `$event`

De ontvanger gebruikt:

- `$status` om de huidige toestand van een Subscription bij de Subscription Server op te vragen — typisch na een uitgebleven heartbeat;
- `$event` om een specifieke `event-number` of een bereik daarvan op te halen — typisch na detectie van een gat in de reeks.

#### Foutafhandeling

Alle transacties MOETEN bij fouten een FHIR `OperationOutcome` met passende HTTP-statuscode teruggeven. Subscription Clients MOETEN transient fouten (`5xx`, netwerkfouten) met exponentiële backoff retryen. Verwerking MOET idempotent zijn: het opnieuw afleveren van een eerder verwerkte notificatie MAG GEEN dubbele neveneffecten veroorzaken.


### Notificatie-inhoud

Een notificatie is een FHIR `Bundle` van type `history`, conform het Backport-profiel `backport-subscription-notification`. De Bundle bevat een `Parameters`-resource (SubscriptionStatus, conform `backport-subscription-status-r4`) met ten minste:

- `subscription` — referentie naar de geregistreerde Subscription;
- `status` — `active` of `off`;
- `type` — `event-notification`, `handshake-notification` of `heartbeat-notification`;
- `notification-event.event-number` — monotoon oplopend nummer om gemiste events te detecteren;
- `notification-event.timestamp` — moment waarop het event bij de verzender plaatsvond;
- `notification-event.focus` — referentie naar de bron-resource waar het event betrekking op heeft.

Omdat de payload `id-only` is, leert de ontvanger uit de Bundle alleen *dát* een resource bij de verzender is gewijzigd, niet de inhoud ervan. De inhoud wordt — desgewenst en onder de geldende toegangscontrole — door de ontvanger opgehaald van het FHIR-endpoint van de verzender. Daarmee blijft de hoeveelheid persoonsgegevens die over het notificatiekanaal beweegt minimaal, en wordt op het moment van inhoudelijke toegang opnieuw beoordeeld of die toegang is toegestaan.

Ten opzichte van een notificatie zonder resource-id heeft `id-only` een duidelijk operationeel voordeel. Zonder id moet de ontvanger binnen het topic zelf gaan zoeken welke wijziging bij de notificatie hoort (bijvoorbeeld via extra zoekopdrachten op tijdvenster, patiënt of status). Dat maakt de uitkomst potentieel ambigu wanneer meerdere resources kort na elkaar wijzigen, verhoogt de kans op verkeerde correlatie, belast zowel zender- als ontvangersystemen met extra queries en vergroot het dataverkeer op de lijn. Met `id-only` is meteen duidelijk waar de notificatie over gaat, terwijl de inhoud alsnog pas via geautoriseerde pull wordt opgehaald.

### Relatie tot andere specificaties

Deze specificatie levert een specificatie voor *notificaties*. Andere specificaties bouwen daarop voort om volledige workflows in te richten:

#### Consequentie voor eOverdracht (wijziging van §5.3.2 Notificatie)

Om de eOverdracht-notificatie uit de Nuts leveranciersspecificatie te laten aansluiten op deze specificatie, kan het huidige mechanisme (lege `POST` naar `<notification-endpoint-url>/<Task.id>`) functioneel gelijk blijven, maar moet het technisch anders worden ingevuld:

1. **Van pad-gecodeerde Task-id naar `id-only` event in de payload.**
	De Task-identificatie verhuist uit de URL (`/<Task.id>`) naar `notification-event.focus` in de notificatie-Bundle (bijv. `Task/<id>`). Het endpoint wordt daarmee een stabiel notificatie-endpoint per Subscription in plaats van een endpoint met resource-id in het pad.
2. **Van lege POST naar standaard Backport notificatie-Bundle.**
	In plaats van een lege body verstuurt de verzender een FHIR `Bundle` (`backport-subscription-notification`) met `Parameters` (`backport-subscription-status-r4`) met minimaal: `type`, `subscription`, `notification-event.event-number`, `notification-event.timestamp`, `notification-event.focus`.
3. **Expliete Subscription-afspraak vooraf.**
	De notificatie wordt vooraf door de verzender 'out-of-band' geactiveerd via een `Subscription`, met topic op `Request` en filters voor ontvanger/workflowcontext. Dit vervangt de impliciete afspraak dat elke POST op `notification/<Task.id>` een geldige notificatie is.
4. **Lifecycle-signalen toevoegen.** *VRAAG: In de eOverdracht use case wordt de data door ontvanger eenmalig opgehaaald bij verzender, dus ik weet niet of Lifecycle-signalen nodig zijn*
	Naast event-notificaties ondersteunt eOverdracht dan ook `handshake-notification` (kanaalbevestiging) en optioneel `heartbeat-notification` (beschikbaarheid kanaal), zodat storingen sneller detecteerbaar zijn.
5. **Gestandaardiseerd herstel bij missende notificaties.** *VRAAG: In de eOverdracht use case wordt de data door ontvanger eenmalig opgehaaald bij verzender, en zijn er maar heel weinig notificaties (1 bij aanmaak Task, 1 bij wijzigen Task.status) dus ik weet niet of herstel bij missende notificaties nodig is*
	De ontvanger detecteert gaten via `event-number` en haalt ontbrekende events op via `$event`; kanaalstatus wordt gecontroleerd via `$status`. Dit vervangt ad-hoc herstel op basis van alleen HTTP retries of timeouts.
6. **Foutafhandeling harmoniseren met FHIR R4 Backport.**
	Fouten blijven via HTTP-statuscodes lopen, met `OperationOutcome` als foutbody. Het semantische contract verschuift van "lege POST ontvangen" naar "Backport-notificatie valide verwerkt".

Voor eOverdracht betekent dit functioneel géén wijziging in het notified-pull principe: de ontvanger blijft na notificatie de `Task` en vervolgens het overdrachtsbericht ophalen onder bestaande autorisatie- en grondslagregels. De wijziging zit in verdergaande standaardisatie van datamodel en data-interacties en verbeterde robuustheid van het notificatiekanaal.

#### Consequentie voor BgZ-verwijzing
De BgZ-verwijzing gebruikt in de huidige Technical IG een Notified Pull-patroon met een notification-task die de ontvanger uitnodigt om vooraf gedefinieerde queries uit te voeren. Functioneel blijft dat patroon bruikbaar, maar technisch wijkt het af van de in deze specificatie beschreven notificatie-uitwisseling op basis van Subscription en event-notifications.

Voor aansluiting op deze specificatie is het daarom wenselijk dat BgZ-verwijzing gebruik gaat maken van een workflow met een medisch/functioneel gedreven request, vergelijkbaar met de benadering in eOverdracht. De inhoud van dit request representeert dan niet het container-begrip "BgZ-verwijzing" maar de daadwerkelijke reden van de verwijziging (bijv. "bloedonderzoek"). Daarbij kan aansluiting worden gezocht bij de Clinical Order Workflow IG en/of de eOverdracht IG voor de proceslaag (request, status, afhandeling), terwijl notificatie en pull als afzonderlijke gestandaardiseerde transacties worden toegepast zoals in dit document beschreven.

De concrete uitwerking van die workflowlaag (procesmodel, resources en statusovergangen) valt buiten scope van deze specificatie en wordt hier niet verder beschreven.

*gereviewd tot hier*

#### Consequentie voor notificaties binnen MedicatieOverdracht

Hoewel de MedicatieOverdracht-specificatie (MP9) meerdere transmurale data-update-transacties beschrijft (zowel PUSH Send/Receive als PULL Retrieve/Serve), is daarin geen apart notificatiemechanisme uitgewerkt. Daardoor heeft deze notificatiespecificatie op dit moment geen directe impact op de MedicatieOverdracht-specificatie: de bestaande MP9-transacties blijven inhoudelijk en technisch ongewijzigd.

#### Consequenties voor Medmij notificaties

Binnen de MedMij-extensie Abonneren/Notificeren bestaat al een werkend notificatiepatroon met een `Notification Client` en `Notification Server`, inclusief twee notificatietypen (resource-notificatie en abonnements-notificatie), JSON-berichten en vaste `POST` op `<base uri>/Notification`. Om aan deze specificatie te voldoen, zijn met name de volgende technische aanpassingen nodig.

1. **Van MedMij-specifieke Notification JSON naar FHIR Subscription-notification Bundle.**
	De huidige notification payload moet worden gemapt naar een Backport-conforme FHIR `Bundle` (`backport-subscription-notification`) met `SubscriptionStatus`-parameters (`subscription`, `type`, `notification-event.event-number`, `notification-event.timestamp`, `notification-event.focus`).
2. **Van twee interface-specifieke notificaties naar eenduidige event-typen.**
	MedMij's resource- en abonnementsnotificaties moeten semantisch worden afgebeeld op `event-notification` en, waar passend, op statuswijzigingen van de `Subscription` (bijv. `off` bij beëindiging), in plaats van op een MedMij-specifiek notificatieberichtmodel.
3. **Expliete Subscription-resource als bron van waarheid.**
	De abonnementstoestand moet niet alleen in proceslogica/OCL-context zitten, maar ook in een expliciete FHIR `Subscription`-resource aan de verzenderzijde, inclusief topic/filter en lifecycle (`requested`, `active`, `off`, `error`).
4. **Kanaalbetrouwbaarheid uitbreiden met handshake/heartbeat/catch-up.**
	De huidige timing- en response-eisen (zoals antwoord binnen 10 seconden) kunnen blijven, maar moeten worden aangevuld met `handshake-notification`, optionele `heartbeat-notification`, en herstel via `$event`/`$status` bij gemiste meldingen.
5. **Event-continuiteit afdwingen.**
	Elke notificatie moet een monotoon oplopend `event-number` krijgen zodat de ontvanger gaten kan detecteren; dit gaat verder dan alleen een succesvolle HTTP-response op de `POST /Notification`.
6. **Endpoint- en adresseringsmodel functioneel behouden, technisch herinterpreteren.**
	MedMij-adressering via OCL/Aanbiederslijst kan behouden blijven, maar het geadresseerde endpoint moet Backport-subscription-notificaties accepteren en valideren in plaats van uitsluitend MedMij Notification-JSON. Registratie van deze endpoints kan te zijner tijd in GF Adressering worden opgenomen, maar is voor dit patroon niet noodzakelijk: de Subscription ontstaat op initiatief van de ontvanger, waardoor het notificatie-endpoint in-band in de Subscription kan worden vastgelegd en niet out-of-band hoeft te worden gecommuniceerd.

Adressering zelf valt buiten scope van deze specificatie; alleen de functionele randvoorwaarde dat een geldig endpoint beschikbaar moet zijn is hier relevant.

Samengevat: het MedMij-communicatiepatroon (abonneren, notificeren, daarna gegevens ophalen) kan blijven bestaan, maar de notificatie-API en berichtinhoud moeten convergeren naar het FHIR Subscription Backport-model uit deze specificatie.

#### Consequentie voor TA Notified Pull

Het TA Notified Pull-principe blijft als communicatiepatroon ongewijzigd bestaan: eerst een notificatie, daarna een doelgerichte pull door de ontvanger. Wat wél wijzigt, is de technische afspraaklaag waaronder notificatie en pull als vast gekoppelde transacties in één patroon werden voorgeschreven.

In deze specificatie vervalt die vaste technische koppeling en daarmee de huidige Technische Afspraak. Notificatie en pull worden als afzonderlijke, herbruikbare transacties gestandaardiseerd, elk met een eigen contract, foutafhandeling en lifecycle. Koppeling tussen beide vindt plaats op functioneel niveau (use case, topic, filter en autorisatiegrond), niet meer als een hard voorgeschreven technisch transactiepaar.


### Security

Security op de transportlaag volgt de specificaties conform LDN Veilig Netwerk, de Generieke Functie Identificatie & Authenticatie en de Generieke Functie Autorisatie. Het notificatie-endpoint MOET mTLS afdwingen en gekwalificeerde certificaten van Qualified Trusted Service Providers (zoals PKIoverheid) vertrouwen. Authenticatie en autorisatie van de notificatie zelf en van de opvolgende pull op het FHIR-endpoint van de verzender volgen deze kaders.

Omdat de notificatie `id-only` payload gebruikt, draagt de Bundle zelf geen klinische inhoud en minimale persoonsgegevens. De inhoudelijke gegevens worden pas vrijgegeven wanneer de ontvanger ze ophaalt van het FHIR-endpoint van de verzender, op welk moment de verzender de autorisatie opnieuw beoordeelt.

#### Audit-events

Een uitgaande notificatie MOET aan de zenderzijde een audit-event opleveren waaruit achteraf is af te leiden dat de notificatie is verstuurd, op welk moment, naar welke organisatie en op welke patiënt of dossier zij betrekking had. Een binnenkomende notificatie MOET aan de ontvangerzijde een spiegelbeeldig audit-event opleveren. Beide events ontstaan onlosmakelijk uit de uitwisseling en zijn beschermd tegen eenzijdige wijziging of verwijdering door de betrokken organisatie. De audit-events vormen samen de basis voor de patiënt-toegankelijke transparantie die EHDS-artikel 9 voor toegang en uitwisseling voorschrijft.


### Voorbeeld use cases

Voor use cases 1 t/m 3 geldt dat de Subscription inhoudelijk door de ontvanger wordt geïnitieerd: de ontvanger bepaalt dat hij voor een bepaald onderwerp notificaties wil ontvangen en laat daarvoor een Subscription registreren bij de verzender (in-band of out-of-band).

#### Use case 1: lab-uitslag voor de huisarts

Een huisartsenpraktijk wil van een ziekenhuislab een notificatie ontvangen wanneer een door de praktijk aangevraagde laboratoriumuitslag voor één van haar patiënten beschikbaar komt.

- **Topic:** `ObservationLaboratory` (uit NL GF Data Categories).
- **Filter (`backport-filter-criteria`):** beperkt tot een door de praktijk gedefinieerd patiëntcohort, bijvoorbeeld via `requester=Practitioner/...` of `performer=Organization/...` zodat alleen door deze praktijk aangevraagde uitslagen melden.
- **Subscription:** door of namens de huisartsenpraktijk (ontvanger) geïnitieerd en geregistreerd bij de Subscription Server van het lab, met als ontvanger het notificatie-endpoint van de huisartsenpraktijk (uit GF Adressering).
- **Loop:** wanneer een matchende `Observation` met `category = laboratory` bij het lab vrijgegeven wordt, ontvangt de praktijk een `event-notification` met `focus → Observation/<id>`. De praktijk haalt de uitslag in een vervolgstap op bij het lab, onder GF Autorisatie.

#### Use case 2: OK-verplaatsing voor de thuismonitoring

Een thuismonitoring-aanbieder volgt patiënten die in een ziekenhuis een operatie ondergaan en wil weten wanneer zo'n geplande operatie verplaatst wordt, om de monitoring-planning aan te kunnen passen.

- **Topic:** `Encounter` of `Request` (passend bij hoe het ziekenhuis een geplande opname/ingreep modelleert).
- **Filter (`backport-filter-criteria`):** beperkt tot patiënten die bij deze thuismonitoring-aanbieder ingeschreven staan, en tot status-wijzigingen die een herplanning representeren (bv. `status=cancelled,entered-in-error,planned` of een datum-verschuiving).
- **Subscription:** door of namens de thuismonitoring-aanbieder (ontvanger) geïnitieerd en geregistreerd bij de Subscription Server van het ziekenhuis, met als ontvanger het notificatie-endpoint van de thuismonitoring-aanbieder.
- **Loop:** bij verplaatsing van de afspraak ontvangt de monitoring-aanbieder een `event-notification` met `focus → Encounter/<id>`. De aanbieder leest de nieuwe planning desgewenst op bij het ziekenhuis, en past het eigen schema aan.

#### Use case 3: dossierinzage voor de patiënt

Een patiënt wil bericht ontvangen wanneer iemand haar dossier bij een zorgaanbieder heeft ingezien. Dit sluit aan op de EHDS-eis dat patiënten op elk moment kunnen zien wie hun gegevens raadpleegt.

- **Topic:** `Logging` (de `AuditEvent`-categorie uit NL GF Data Categories).
- **Filter (`backport-filter-criteria`):** beperkt tot AuditEvents over deze patiënt (`patient=Patient/<id>` op basis van pseudoniem/BSN-mechanismen volgens GF Identificatie).
- **Subscription:** door of namens de patiënt(omgeving) als ontvanger geïnitieerd en geregistreerd bij de Subscription Server van de zorgaanbieder; het ontvangende endpoint is in dit geval typisch een PGO of patiënt-portaal namens de patiënt.
- **Loop:** bij elke registratie van een dossier-raadpleging ontvangt het patiëntkanaal een `event-notification` met `focus → AuditEvent/<id>`. De patiënt(omgeving) kan op haar gemak de details opvragen — bij voorkeur via een interface die deze raadplegingen samenvoegt tot een leesbaar logboek.

#### Use case 4: verwijzing van zorgaanbieder A naar zorgaanbieder B (zender initieert Subscription)

Zorgaanbieder A maakt een verwijzing voor zorgaanbieder B en wil B daar direct over notificeren. In dit scenario initieert de verzender (A) de Subscription, omdat de verwijzing een concrete, case-gebonden notificatiebehoefte creëert richting één specifieke ontvanger.

Voorbeeld: een huisarts (A) verwijst een patiënt naar de cardiologie-afdeling van een ziekenhuis (B). Zodra de `ServiceRequest` en/of `Task` is aangemaakt of geactualiseerd met nieuwe urgentie- of bijlage-informatie, verstuurt A een notificatie naar B zodat de triage en planning direct kunnen starten.

- **Topic:** `Request` (bijvoorbeeld op basis van `ServiceRequest` of `Task`, afhankelijk van de gekozen workflow/eOverdracht-specificatie).
- **Filter (`backport-filter-criteria`):** beperkt tot verwijzingen van A naar B, bijvoorbeeld op ontvangende organisatie, workflowstatus en/of patiëntcontext.
- **Subscription:** door zorgaanbieder A (verzender) aangemaakt bij de eigen Subscription Server, met als `channel.endpoint` het notificatie-endpoint van zorgaanbieder B (opgezocht via GF Adressering).
- **Loop:** zodra A de verwijzing publiceert of de status wijzigt naar "te behandelen door B", verstuurt A een `event-notification` naar B met `focus` naar de bron-resource bij A. B haalt daarna de inhoud op via de afgesproken pull en autorisatie.


### Roadmap en open punten

- **Codeset-evolutie.** De NL GF Data Categories staan in review bij Nictiz. Bij overgang naar SNOMED/LOINC-gebaseerde categorisering (conform [BgZ v2.0 §4.3](https://informatiestandaarden.nictiz.nl/wiki/bgz:V2.0_Technical_IG_BgZ_MSZ)) moeten topic-canonicals en displaynames meebewegen.
- **Identificatie en pseudonimisering.** Voor patiënt-gerichte notificatiekanalen (use case 3) is afstemming met GF Identificatie en pseudonimisering nodig om patiëntreferenties in filters veilig en consistent te ondersteunen.
