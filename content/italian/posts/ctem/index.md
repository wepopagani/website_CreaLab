+++
title = 'CTEM su Red Teaming'
date = 2024-07-29T17:04:31+02:00
draft = false
image = '/posts/ctem/static/cover.jpg'
+++

## Introduzione

In TeamFence cerchiamo sempre nuovi modi per migliorare la postura di sicurezza dei nostri clienti. In questo post del blog, discuteremo del framework di Gestione Continua dell'Esposizione alle Minacce (CTEM) e di come possa aiutare le organizzazioni a minimizzare l'esposizione agli attacchi informatici. Confronteremo anche il CTEM con il Red Teaming e gli attacchi simulati per comprendere i vantaggi di adottare un approccio proattivo alla sicurezza.

## Cos'è CTEM?

La Gestione Continua dell'Esposizione alle Minacce (CTEM) è un framework dinamico e continuo in cinque fasi progettato per minimizzare l'esposizione agli attacchi informatici. Questo approccio proattivo aiuta le organizzazioni a identificare le vulnerabilità, mapparle a possibili percorsi di attacco, dare priorità in base al rischio per gli asset critici e monitorare i progressi degli sforzi di mitigazione. Le aziende di tutto il mondo stanno adottando CTEM per gestire efficacemente le esposizioni e rafforzare la loro postura di sicurezza.

Il CTEM comporta una valutazione approfondita dell'intero ecosistema di un'organizzazione, comprese reti, sistemi, asset e altro ancora, per rilevare esposizioni e debolezze. L'obiettivo principale è ridurre la probabilità che queste vulnerabilità vengano sfruttate dagli aggressori. L'implementazione di un programma CTEM garantisce un miglioramento continuo delle misure di sicurezza identificando e affrontando aree potenzialmente problematiche prima che possano essere sfruttate.

L'aspetto "continuo" di CTEM enfatizza la relazione iterativa tra il programma CTEM e gli sforzi di mitigazione del rischio. I dati generati da entrambi gli elementi si informano a vicenda, facilitando decisioni sempre più ottimali sulla gestione dei rischi di esposizione.

Sfruttando CTEM, le organizzazioni possono garantire un framework di sicurezza resiliente che si adatta alle minacce in evoluzione, proteggendo continuamente i loro asset critici dagli attacchi informatici.

## I cinque passaggi del framework CTEM

![CTEM](static/steps.png)

#### Fase 1 – Definizione del perimetro

La fase iniziale consiste nel comprendere le superfici di attacco e determinare l'importanza aziendale di ciascun asset, riconoscendo che questi fattori si evolveranno nel tempo. Questo include l'identificazione delle superfici di attacco chiave con il contributo di vari decisori, come responsabili di IT, Legale, GRC, Sviluppo, R&D, Prodotto e Operazioni aziendali.

#### Fase 2 – Scoperta

Durante la fase di scoperta, ciascun asset viene valutato per identificare le potenziali esposizioni e analizzare i rischi associati. Questo va oltre l'identificazione di vulnerabilità isolate per includere altri tipi di esposizioni, come i rischi legati a Active Directory, identità e configurazione, e considera come queste esposizioni potrebbero essere collegate per creare percorsi di attacco verso gli asset.

#### Fase 3 – Prioritizzazione

Nella fase di prioritizzazione, le esposizioni vengono analizzate per determinarne il livello di minaccia in base a incidenti reali noti e all'importanza degli asset coinvolti. Questo passaggio è cruciale perché le organizzazioni spesso affrontano più esposizioni di quante possano gestire a causa del grande volume e degli ambienti in continua evoluzione. CTEM aiuta a prioritizzare le mitigazioni che riducono più efficacemente il rischio per gli asset critici, tenendo conto di tutti i tipi di esposizioni, comprese identità e configurazioni errate.

#### Fase 4 – Validazione

La fase di validazione esamina come possono verificarsi gli attacchi e la loro probabilità, utilizzando vari strumenti per diversi scopi. A volte, la validazione aiuta nella prioritizzazione come nella Fase 3, mentre in altri casi viene utilizzata per testare continuamente i controlli di sicurezza o automatizzare i test di penetrazione periodici.

#### Fase 5 – Mobilitazione

La fase di mobilitazione assicura che tutti comprendano i loro ruoli e responsabilità nel contesto del programma. Una mobilitazione efficace richiede che sia i team di sicurezza che quelli IT coinvolti negli sforzi di mitigazione abbiano chiarezza sul valore della riduzione del rischio delle loro azioni e possano riferire sull'andamento generale dei miglioramenti nella postura di sicurezza nel tempo.

## Il CTEM è meglio del Red Teaming?

Il CTEM e il Red Teaming sono entrambi strumenti preziosi per migliorare la postura di sicurezza di un'organizzazione, ma servono a scopi diversi. Se il nostro obiettivo principale è migliorare la postura di sicurezza di un'organizzazione, CTEM è la strada da percorrere. Vediamo perché:

#### Red Teaming

Un'attività di Red Teaming mira a identificare le vulnerabilità negli asset di un'azienda simulando un attacco per prendere il controllo dei sistemi critici. Questo processo coinvolge un team di hacker etici che utilizzano un approccio opportunistico per trovare debolezze e sfruttarle, concentrandosi esclusivamente sul raggiungimento del loro obiettivo senza considerare altri problemi o percorsi alternativi.

La seguente immagine potrebbe rappresentare una comune iterazione di Red Teaming:
![CTEM](static/red-gantt.png)
<br>
Sebbene il Red Teaming fornisca informazioni preziose sulle potenziali falle di sicurezza, ha alcune limitazioni notevoli.

***In primo luogo***, offre un'istantanea nel tempo piuttosto che una valutazione continua. Le vulnerabilità identificate durante un esercizio di Red Teaming possono diventare rapidamente obsolete man mano che emergono nuove minacce e l'ambiente dell'azienda si evolve.

***In secondo luogo***, il Red Teaming non fornisce una visione completa di tutte le possibili vulnerabilità. Si concentra su vettori di attacco e percorsi specifici, il che può lasciare alcune aree critiche di esposizione non rilevate. Questo focus ristretto può portare a non rilevare alcune vulnerabilità, in particolare quelle che non si allineano con le tattiche specifiche utilizzate dal team rosso.

***Infine***, come possiamo vedere nell'immagine sopra, il Red Teaming è un processo che richiede tempo e risorse, il che potrebbe non essere fattibile per le organizzazioni con budget limitati o vincoli operativi. L'elevato costo e l'impegno necessari per condurre esercizi di Red Teaming possono limitare la loro frequenza ed efficacia nell'identificazione e risoluzione delle vulnerabilità. Inoltre, il risultato di un esercizio di Red Teaming è un elenco di vulnerabilità anziché un elenco prioritizzato di esposizioni, spesso costringendo i clienti a scoprire da soli come implementare le misure necessarie.

### CTEM

D'altra parte, CTEM risolve molti dei problemi che un'organizzazione potrebbe affrontare quando cerca di migliorare la propria postura di sicurezza:

***Assorbimento dei risultati***: molte organizzazioni lottano per assorbire i risultati del Red Teaming o di altri esercizi di simulazione di attacchi, trovando i risultati travolgenti e difficili da capire e prioritizzare. CTEM include una fase di mobilitazione che garantisce che i team interni comprendano completamente i rischi, consentendo loro di dare priorità e implementare misure di mitigazione in modo efficace.

***Tempo e denaro***: dobbiamo considerare che questo tipo di attività sono servizi B2B e sono costosi. Il CTEM è un approccio più conveniente alla sicurezza, poiché la fase di enumerazione degli attacchi simulati è sostituita da uno sforzo collaborativo tra i team di sicurezza e IT per identificare tutti gli asset e le esposizioni nell'organizzazione.

***Copertura degli asset***: CTEM consente una copertura del 100% degli asset, poiché la fase di scoperta non è limitata agli asset che il team rosso può trovare, ma è uno sforzo collaborativo tra i team di sicurezza e IT. Ciò significa che tutti gli asset vengono presi in considerazione e il rischio viene calcolato in base all'importanza dell'asset.

***Continuo***: CTEM è un processo continuo, il che significa che l'organizzazione è sempre consapevole dei rischi e può agire di conseguenza. Il team di sicurezza può monitorare i cambiamenti dell'infrastruttura, seguire le nuove minacce e vulnerabilità emergenti e mantenere la postura di sicurezza aggiornata. Questo allevia gran parte della responsabilità della sicurezza dal team IT e il cliente può concentrarsi sul business.

## Conclusione

In conclusione, sebbene il Red Teaming sia utile per scoprire determinate vulnerabilità e testare l'efficacia delle difese di un'azienda contro scenari di attacco specifici, le sue limitazioni evidenziano la necessità di valutazioni di sicurezza più continue, complete e consapevoli del contesto.

In TeamFence, crediamo che CTEM offra un approccio più efficace e sostenibile alla sicurezza fornendo la sicurezza come servizio che monitora e gestisce continuamente l'esposizione di un'organizzazione alle minacce informatiche. Adottando il CTEM, le aziende possono identificare e affrontare proattivamente le vulnerabilità, prioritizzare gli sforzi di mitigazione e mantenere una postura di sicurezza resiliente che si adatta alle minacce in evoluzione.

Contattaci per saperne di più su come CTEM può aiutare la tua organizzazione a stare al passo con le minacce informatiche e proteggere i tuoi asset critici.
