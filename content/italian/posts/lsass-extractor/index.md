+++
title = 'Estrattore di Lsass'
date = 2023-09-16T16:39:49+02:00
draft = false
image = '/posts/lsass-extractor/static/cover.png'
+++

## Introduzione

In TeamFence, crediamo fermamente che sia sempre più importante per un operatore offensivo comprendere i dettagli delle tecniche e degli strumenti.

Per questo motivo, in questo post del blog, esploreremo come Mimikatz estrae le credenziali dai dump di lsass, permettendoci di reimplementare la logica in uno strumento su misura per le nostre esigenze future.

**Nota**: tutte le analisi condotte durante questo progetto sono state eseguite su una workstation Windows 11 Enterprise Evaluation.

## Perché LSASS?

Prima di entrare nei dettagli tecnici, è importante chiedersi: perché lo stiamo facendo?

Quando un hacker ottiene l'accesso a una macchina target che esegue Windows, uno dei primi passaggi cruciali consiste nell'ottenere le credenziali. Windows offre molte potenziali destinazioni, ma due principali repository si distinguono: il database di sicurezza locale (Security Account Manager) e il processo LSASS (Local Security Authority Subsystem Service).

In questo post ci concentreremo su LSASS, che funge da componente centrale in Windows, supervisionando l'autenticazione. Come centro nevralgico per le richieste di autenticazione provenienti da vari servizi, svolge un ruolo fondamentale nel semplificare il flusso di autenticazione. Questo processo implementa vari pacchetti di autenticazione come NTLM, Kerberos, WDigest, tra gli altri. Di conseguenza, rappresenta un obiettivo prezioso e ricco di informazioni per gli hacker.

## Hash NTLM

Come abbiamo visto nella sezione precedente, LSASS gestisce diversi pacchetti di autenticazione, incluso NTLM. NTLM è un protocollo di autenticazione challenge-response ampiamente utilizzato negli ambienti Windows. Quando un utente accede a una macchina Windows, il sistema memorizza le credenziali dell'utente in memoria. Queste informazioni includono la password dell'utente sotto forma di hash NTLM.

In questo post ci concentreremo sull'estrazione di questi hash NTLM dal processo LSASS, il che ci permetterà di comprendere più facilmente la logica sottostante.

Nell'immagine seguente possiamo vedere la struttura interna di LSASS, evidenziando dove vengono memorizzati gli hash NTLM.

![lsass](/posts/lsass-extractor/static/lsass-structure.png)

## Dove si trovano tutti i pezzi?

Per iniziare, è fondamentale comprendere dove sono memorizzate le informazioni e in quale formato. Il processo LSASS è un componente critico, e non tutte le informazioni sono memorizzate in chiaro. In questo caso, gli hash NTLM sono crittografati. Pertanto, per estrarli, dobbiamo individuare le chiavi per la decrittazione. Il diagramma seguente illustra quali moduli sono coinvolti nella gestione delle chiavi e delle credenziali.

<img src="/posts/lsass-extractor/static/information-schema.png" alt="dove sono i pezzi" style="display: block; margin-left: auto; margin-right: auto; width: 500px; height: auto;">

È importante ricordare che il materiale crittografico del processo LSASS è una data runtime, il che significa che cambia dopo ogni riavvio. Di conseguenza, dobbiamo estrarlo ogni volta per garantire informazioni aggiornate.

```text
"È interessante notare che la gestione della crittografia viene effettuata tramite LsaProtectMemory, che è un wrapper per LsaEncryptMemory, che a sua volta è un wrapper per BCryptEncrypt, una funzione di crittografia comune presente in bcrypt.h"
```

## Modelli di moduli

A questo punto, abbiamo una chiara comprensione di quale area della memoria contenga tutti i pezzi. Ora dobbiamo trovare un modo per estrarli. Analizzando il codice di PyPyKatz, una versione Python di Mimikatz, possiamo comprendere l'approccio utilizzato per estrarre le chiavi e le credenziali.

Prima di tutto, dobbiamo estrarre alcune informazioni sul sistema: ProcessorArchitecture, BuildNumber e dal "ModuleListStream" il TimeDateStamp di "lsasrv.dll".

Una volta ottenute queste informazioni, possiamo utilizzarle per identificare il modello corretto da usare per l'estrazione. I modelli sono classi che contengono la firma e gli offset che ci permettono di sapere come muoverci all'interno della memoria per estrarre le chiavi e le credenziali.

Nel nostro caso, che è una Windows 11 Enterprise Evaluation, il modello per la parte lsasrv.dll è il seguente:

```python
class LSA_x64_6(LsaTemplate_NT6):
	def __init__(self):
		LsaTemplate_NT6.__init__(self)
		self.key_pattern = LSADecyptorKeyPattern()
		self.key_pattern.signature = b'\x83\x64\x24\x30\x00\x48\x8d\x45\xe0\x44\x8b\x4d\xd8\x48\x8d\x15'
		self.key_pattern.IV_length = 16
		self.key_pattern.offset_to_IV_ptr = 67
		self.key_pattern.offset_to_DES_key_ptr = -89
		self.key_pattern.offset_to_AES_key_ptr = 16
				
		self.key_struct = KIWI_BCRYPT_KEY81
		self.key_handle_struct = KIWI_BCRYPT_HANDLE_KEY
```

Dai un'occhiata al [codice sorgente](https://github.com/skelsec/pypykatz/blob/c91dcdc09289ad2e93c475e7c640d0f90906a7c0/pypykatz/lsadecryptor/lsa_template_nt6.py#L301) di PyPyKatz per vedere tutti i modelli.

## Estrazione del vettore di inizializzazione

Per estrarre le chiavi, dobbiamo prima estrarre il Vettore di Inizializzazione (IV). L'IV è un numero casuale utilizzato per garantire che lo stesso messaggio in chiaro non risulti nello stesso testo cifrato. L'IV è memorizzato in memoria, e possiamo estrarlo cercando nella memoria del modulo lsasrv.dll il pattern presente nel modello.

Una volta trovato il pattern, possiamo usare gli offset per spostarci all'interno della memoria ed estrarre l'IV.

<img src="/posts/lsass-extractor/static/iv-schema.png" alt="schema per estrarre l'IV" style="display: block; margin-left: auto; margin-right: auto; width: 600px; height: auto;">

Come possiamo vedere dallo schema, una volta individuato il pattern, dobbiamo saltare un offset prefissato (presente nel modello) di 67 byte per trovare un altro valore, che è un altro offset. Questo offset è la posizione dell'IV.

## Estrazione delle chiavi

Ora è il momento di estrarre le chiavi. L'approccio è simile a quello utilizzato per estrarre l'IV. Dobbiamo trovare lo stesso pattern in memoria e poi utilizzare gli offset per estrarre le chiavi.

<img src="/posts/lsass-extractor/static/des-schema.png" alt="schema per estrarre l'IV" style="display: block; margin-left: auto; margin-right: auto; width: 600px; height: auto;">

Questa volta, utilizzando il puntatore, non si arriva direttamente al valore desiderato, ma piuttosto a una struttura BCRYPT_KEY_HANDLE, che, in bcrypt.h, è un tipo di dati che consente di fare riferimento e manipolare chiavi crittografiche all'interno del framework CNG. In pypykatz, la classe KIWI_BCRYPT_HANDLE_KEY replica questo tipo.

```python
class KIWI_BCRYPT_HANDLE_KEY :
    def __init__ ( self , reader ) :
        self.size = ULONG( reader ).value
        self.tag = reader.read(4) #'UUUR'
        self.hAlgorithm = PVOID(reader).value
        self.ptr_key = PKIWI_BCRYPT_KEY(reader)
        self.unk0 = PVOID (reader).value
[...]
```

A questo punto, abbiamo tutte le informazioni necessarie per decriptare gli hash NTLM. Abbiamo l'IV e le chiavi. Possiamo ora iniziare la ricerca delle LogonSessions.

## LogonSessions

Le LogonSessions sono le strutture che contengono le informazioni sugli utenti connessi al sistema. Le LogonSessions sono memorizzate nella memoria del modulo Msv1_0.dll, e possiamo estrarle cercando il pattern presente nel modello specifico, che nel nostro caso è il seguente:

```python
elif WindowsBuild.WIN_11_2022.value <= sysinfo.buildnumber < WindowsBuild.WIN_11_2023.value: #20348
    template.signature = b'\x45\x89\x34\x24\x4c\x8b\xff\x8b\xf3\x45\x85\xc0\x74'
    template.first_entry_offset = 24
    template.offset2 = -4
```

Non è possibile conoscere il numero esatto di LogonSessions presenti in memoria, quindi dobbiamo prima recuperare il contatore.

#### Contatore delle LogonSessions e indirizzi



Con un approccio simile a quello utilizzato per estrarre l'IV e le chiavi, dobbiamo trovare il pattern in memoria e poi utilizzare gli offset per estrarre il contatore.

Questo contatore ci permette di iterare su tutti gli indirizzi delle LogonSessions, dove ogni voce si trova 16 byte dopo la precedente.

<img src="/posts/lsass-extractor/static/counter-addresses.png" alt="schema per estrarre l'IV" style="display: block; margin-left: auto; margin-right: auto; width: 600px; height: auto;">

Convertendo questi indirizzi in un char*, diventano un puntatore che punta alla posizione della memoria in cui sono memorizzate le voci delle LogonSessions. Ciò consente di accedere ai dati associati alla LogonSession, permettendo il recupero delle informazioni rilevanti.

#### Nome utente e dominio

Ora che abbiamo gli indirizzi delle LogonSessions, possiamo iniziare a estrarre le informazioni. Saltando all'offset di 144 byte dall'indirizzo, sono presenti diversi campi, tra cui la lunghezza del nome utente, la lunghezza massima e un puntatore al nome utente effettivo.
La stessa logica si applica al dominio, che è memorizzato in memoria dopo il puntatore al nome utente.

Ora è possibile estrarre il nome utente e il dominio dell'utente seguendo i puntatori e leggendo dalla memoria il numero esatto di byte memorizzati nel campo di lunghezza massima.

<img src="/posts/lsass-extractor/static/logon-structure.png" alt="schema per estrarre l'IV" style="display: block; margin-left: auto; margin-right: auto; width: 650px; height: auto;">

#### Lista delle credenziali

Per l'ultima parte dell'estrazione, dobbiamo concentrarci sulle liste delle credenziali. Partendo dal pointerToDomain visto nella sezione precedente, avanzando di un offset di 96 byte, è possibile trovare l'indirizzo di memoria in cui è situata la prima 'Lista delle credenziali'. Una volta letto il valore all'indirizzo puntato come uint64_t e successivamente interpretato come indirizzo di memoria, esso fornirà l'indirizzo di memoria in cui è situata la seconda 'Lista delle credenziali', e questo processo continua iterativamente. Poiché si tratta di una lista collegata circolare, quando il valore corrisponde all'indirizzo di memoria della prima struttura, ciò indica che l'intera lista è stata attraversata.

Ogni voce nella lista collegata della 'Lista delle credenziali' contiene la propria lista collegata associata, nota come 'Lista delle credenziali primarie'. Per recuperare l'indirizzo della prima voce in questa lista collegata, è necessario avanzare di un offset di 16 byte e leggere quest'area di memoria come puntatore char.

Il seguente schema illustra la struttura delle liste:
<img src="/posts/lsass-extractor/static/credlist-struct.png" alt="schema per estrarre l'IV" style="display: block; margin-left: auto; margin-right: auto; width: 650px; height: auto;">

All'interno di questa struttura finale, si può trovare l'hash NTLM crittografato. Proprio come nella lista collegata della 'Lista delle credenziali', interpretando il numero all'indirizzo della prima voce della Lista primaria come un indirizzo, si scopre la successiva struttura, seguendo una lista circolare. Seguendo la struttura, è possibile recuperare la dimensione dell'NTLM crittografato e l'indirizzo di memoria in cui si trova questo NTLM, consentendone la successiva lettura ed estrazione.

## Decrittazione NTLM

L'algoritmo di crittografia utilizzato per crittografare l'hash non può essere determinato in anticipo. Per identificare l'algoritmo, l'NTLM crittografato viene diviso per 8 e il resto viene controllato per vedere se la lunghezza dell'NTLM è divisibile per 8. Se la lunghezza dell'NTLM è divisibile per 8, viene utilizzata la crittografia AES. Se la lunghezza dell'NTLM non è divisibile per 8, indicando una lunghezza irregolare, viene utilizzata la crittografia 3DES.

Il seguente pseudocodice rappresenta la logica di selezione dell'algoritmo:

```c
if ( encryptedNTLMLength % 8 != 0) {
 // La credenziale crittografata era vuota?
} else {
 // Preparare le funzioni dell'algoritmo e l'IV
 if ( encryptedNTLMLength % 8) {
    // Eseguire la decrittazione AES utilizzando la chiave fornita
 } else {
    // Eseguire la decrittazione 3DES utilizzando la chiave fornita
 }
}
```

Una volta identificato l'algoritmo utilizzato, il compito rimanente è applicare le informazioni acquisite utilizzando la libreria bcrypt.h, seguendo lo stesso approccio di MSV. Configurando le variabili necessarie per utilizzare l'algoritmo corretto e successivamente invocando la funzione BCrypt-Decrypt con la chiave crittografata e l'IV recuperato, diventa possibile recuperare l'hash NTLM in testo semplice.