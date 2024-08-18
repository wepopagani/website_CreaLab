+++
title = 'Lsass-Extraktor'
date = 2023-09-16T16:39:49+02:00
draft = false
image = '/posts/lsass-extractor/static/cover.png'
+++

## Einführung

Bei TeamFence sind wir fest davon überzeugt, dass es für einen Offensiv-Operator heutzutage immer wichtiger ist, die Details hinter den Techniken und Werkzeugen zu verstehen.

Aus diesem Grund werden wir in diesem Blogbeitrag untersuchen, wie Mimikatz Anmeldeinformationen aus LSASS-Dumps extrahiert und wie wir diese Logik in ein Tool integrieren können, das auf unsere zukünftigen Bedürfnisse zugeschnitten ist.

**Hinweis**: Alle Analysen, die im Rahmen dieses Projekts durchgeführt wurden, basieren auf einer Windows 11 Enterprise Evaluation-Workstation.

## Warum LSASS?

Bevor wir in die technischen Details eintauchen, ist es wichtig, die Frage zu stellen: Warum tun wir das?

Wenn ein Hacker Zugriff auf eine Zielmaschine mit Windows erhält, besteht einer der entscheidenden ersten Schritte darin, Anmeldeinformationen zu erlangen. Windows bietet mehrere potenzielle Ziele, aber zwei Hauptquellen stechen hervor: die lokale Sicherheitsdatenbank (Security Account Manager) und der LSASS-Prozess (Local Security Authority Subsystem Service).

In diesem Beitrag konzentrieren wir uns auf LSASS, das als zentrales Element in Windows fungiert und die Authentifizierung überwacht. Als zentrale Anlaufstelle für Authentifizierungsanforderungen verschiedener Dienste spielt es eine entscheidende Rolle bei der Optimierung des Authentifizierungsflusses. Dieser Prozess implementiert verschiedene Authentifizierungspakete wie NTLM, Kerberos, WDigest und andere. Folglich ist es ein wertvolles und informationsreiches Ziel für Hacker.

## NTLM-Hashes

Wie wir im vorherigen Abschnitt gesehen haben, verwaltet LSASS mehrere Authentifizierungspakete, darunter NTLM. NTLM ist ein Challenge-Response-Authentifizierungsprotokoll, das in Windows-Umgebungen weit verbreitet ist. Wenn sich ein Benutzer bei einem Windows-Rechner anmeldet, speichert das System die Anmeldeinformationen des Benutzers im Speicher. Diese Informationen beinhalten das Passwort des Benutzers in Form eines NTLM-Hashes.

In diesem Beitrag konzentrieren wir uns darauf, diese NTLM-Hashes aus dem LSASS-Prozess zu extrahieren, um die zugrunde liegende Logik besser zu verstehen.

Im folgenden Bild sehen wir die interne LSASS-Struktur, die hervorhebt, wo die NTLM-Hashes gespeichert sind.

![lsass](/posts/lsass-extractor/static/lsass-structure.png)

## Wo befinden sich die einzelnen Teile?

Zu Beginn ist es wichtig zu verstehen, wo die Informationen gespeichert sind und in welchem Format. Der LSASS-Prozess ist ein kritischer Bestandteil, und nicht alle Informationen werden im Klartext gespeichert. In diesem Fall sind die NTLM-Hashes verschlüsselt. Um sie zu extrahieren, müssen wir die Schlüssel zur Entschlüsselung finden. Das folgende Diagramm veranschaulicht, welche Module an der Verwaltung der Schlüssel und der Anmeldeinformationen beteiligt sind.

<img src="/posts/lsass-extractor/static/information-schema.png" alt="wo sind die Teile" style="display: block; margin-left: auto; margin-right: auto; width: 500px; height: auto;">

Es ist wichtig zu beachten, dass das kryptografische Material des LSASS-Prozesses Laufzeitdaten sind, die sich nach jedem Neustart ändern. Daher müssen wir sie jedes Mal extrahieren, um aktuelle Informationen zu erhalten.

```text
"Es ist interessant zu beachten, dass das Verschlüsselungsmanagement über LsaProtectMemory erfolgt, das ein Wrapper für LsaEncryptMemory ist, das wiederum ein Wrapper für BCryptEncrypt ist, eine gängige Verschlüsselungsfunktion, die in bcrypt.h vorhanden ist."
```

## Modultemplates

An diesem Punkt haben wir ein klares Verständnis davon, in welchem Bereich des Speichers sich alle Teile befinden. Jetzt müssen wir einen Weg finden, sie zu extrahieren. Durch die Analyse des Codes von PyPyKatz, einer Python-Version von Mimikatz, können wir den Ansatz verstehen, der verwendet wird, um die Schlüssel und Anmeldeinformationen zu extrahieren.

Zunächst müssen wir einige Informationen über das System extrahieren: ProcessorArchitecture, BuildNumber und aus dem "ModuleListStream" den TimeDateStamp von "lsasrv.dll".

Sobald wir diese Informationen haben, können wir sie verwenden, um die richtige Vorlage zur Extraktion zu identifizieren. Die Templates sind Klassen, die die Signatur und die Offsets enthalten, die uns ermöglichen, uns im Speicher zu orientieren, um die Schlüssel und Anmeldeinformationen zu extrahieren.

In unserem Fall, einer Windows 11 Enterprise Evaluation, lautet die Vorlage für den lsasrv.dll-Teil wie folgt:

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

Werfen Sie einen Blick auf den [Quellcode](https://github.com/skelsec/pypykatz/blob/c91dcdc09289ad2e93c475e7c640d0f90906a7c0/pypykatz/lsadecryptor/lsa_template_nt6.py#L301) von PyPyKatz, um alle Vorlagen zu sehen.

## Initialisierungsvektor-Extraktion

Um die Schlüssel zu extrahieren, müssen wir zuerst den Initialisierungsvektor (IV) extrahieren. Der IV ist eine Zufallszahl, die sicherstellt, dass dieselbe Klartextnachricht nicht zum selben Chiffretext führt. Der IV wird im Speicher gespeichert, und wir können ihn extrahieren, indem wir im Speicher des Moduls lsasrv.dll nach dem im Template vorhandenen Muster suchen.

Sobald wir den Standort des Musters haben, können wir die Offsets verwenden, um uns im Speicher zu bewegen und den IV zu extrahieren.

<img src="/posts/lsass-extractor/static/iv-schema.png" alt="Schema zur Extraktion des IV" style="display: block; margin-left: auto; margin-right: auto; width: 600px; height: auto;">

Wie im Schema zu sehen ist, müssen wir, nachdem wir den Standort des Musters haben, einen vordefinierten Offset (im Template vorhanden) von 67 Bytes überspringen, um einen weiteren Wert zu finden, der ein weiterer Offset ist. Dieser Offset ist der Standort des IV.

## Schlüssel-Extraktion

Nun ist es an der Zeit, die Schlüssel zu extrahieren. Der Ansatz ist ähnlich wie bei der Extraktion des IV. Wir müssen dasselbe Muster im Speicher finden und dann die Offsets verwenden, um die Schlüssel zu extrahieren.

<img src="/posts/lsass-extractor/static/des-schema.png" alt="Schema zur Extraktion des IV" style="display: block; margin-left: auto; margin-right: auto; width: 600px; height: auto;">

Dieses Mal führt uns der Zeiger nicht direkt zum gewünschten Wert, sondern zu einer BCRYPT_KEY_HANDLE-Struktur, die in bcrypt.h ein Datentyp ist, der das Referenzieren und Verwalten kryptografischer Schlüssel innerhalb des CNG-Frameworks ermöglicht. In pypykatz repliziert die Klasse KIWI_BCRYPT_HANDLE_KEY diesen Typ.

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

An diesem Punkt haben wir alle notwendigen Informationen, um die NTLM-Hashes zu entschlüsseln. Wir haben den IV und die Schlüssel. Jetzt können wir mit der Suche nach den LogonSessions beginnen.

## LogonSessions

Die LogonSessions sind die Strukturen, die die Informationen über die Benutzer enthalten, die im System angemeldet sind. Die LogonSessions werden im Speicher des Moduls Msv1_0.dll gespeichert, und wir können sie extrahieren, indem wir nach dem im spezifischen Template vorhandenen Muster suchen, das in unserem Fall wie folgt aussieht:

```python
elif WindowsBuild.WIN_11_2022.value <= sysinfo.buildnumber < WindowsBuild.WIN_11_2023.value: #20348
    template.signature = b'\x45\x89\x34\x24\x4c\x8b\xff\x8b\xf3\x45\x85\xc0\x74'
    template.first_entry_offset = 24
    template.offset2 = -4
```

Es ist nicht möglich, die genaue Anzahl der im Speicher vorhandenen LogonSessions zu kennen, daher müssen wir zuerst den Zähler abrufen.

#### LogonSessions-Zähler und Adressen

Mit einem ähnlichen Ansatz wie bei der Extr

aktion des IV und der Schlüssel müssen wir das Muster im Speicher finden und dann die Offsets verwenden, um den Zähler zu extrahieren.

Dieser Zähler ermöglicht es uns, alle LogonSessions-Adressen durchzugehen, wobei jeder Eintrag 16 Bytes neben dem vorherigen liegt.

<img src="/posts/lsass-extractor/static/counter-addresses.png" alt="Schema zur Extraktion des IV" style="display: block; margin-left: auto; margin-right: auto; width: 600px; height: auto;">

Durch das Umwandeln dieser Adressen in einen char*-Pointer wird es zu einem Zeiger, der auf die Speicheradresse zeigt, an der die LogonSessions-Einträge gespeichert sind. Dies ermöglicht den Zugriff auf die mit der LogonSession verbundenen Daten und damit das Abrufen relevanter Informationen.

#### Benutzername und Domäne

Jetzt, da wir die Adressen der LogonSessions haben, können wir mit der Extraktion der Informationen beginnen. Durch einen Sprung von 144 Bytes von der Adresse aus befinden sich mehrere Felder, darunter die Länge des Benutzernamens, die maximale Länge und ein Zeiger auf den tatsächlichen Benutzernamen.
Die gleiche Logik gilt für die Domäne, die im Speicher nach dem Zeiger auf den Benutzernamen gespeichert ist.

Nun ist es möglich, den Benutzernamen und die Domäne des Benutzers zu extrahieren, indem man den Zeigern folgt und die genaue Anzahl der Bytes liest, die im Feld der maximalen Länge gespeichert sind.

<img src="/posts/lsass-extractor/static/logon-structure.png" alt="Schema zur Extraktion des IV" style="display: block; margin-left: auto; margin-right: auto; width: 650px; height: auto;">

#### Anmeldeinformationen-Liste

Für den letzten Teil der Extraktion müssen wir uns auf die Liste der Anmeldeinformationen konzentrieren. Ausgehend vom pointerToDomain, den wir im vorherigen Abschnitt gesehen haben, können wir durch einen weiteren Sprung von 96 Bytes die Speicheradresse finden, an der die erste 'Anmeldeinformationen-Liste' gespeichert ist. Sobald der Wert an der angegebenen Adresse als uint64_t gelesen und als Speicheradresse interpretiert wird, erhalten wir die Speicheradresse der zweiten 'Anmeldeinformationen-Liste', und dieser Prozess setzt sich iterativ fort. Da es sich um eine zirkuläre verkettete Liste handelt, zeigt die Übereinstimmung des Werts mit der Speicheradresse der ersten Struktur an, dass die gesamte Liste durchlaufen wurde.

Jeder Eintrag in der verketteten Liste der 'Anmeldeinformationen-Liste' enthält seine eigene verkettete Liste, die als 'Primäre Anmeldeinformationen-Liste' bezeichnet wird. Um die Adresse des ersten Eintrags in dieser verketteten Liste abzurufen, muss ein Offset von 16 Bytes vorangeschritten und dieser Speicherbereich als char*-Pointer gelesen werden.

Das folgende Schema veranschaulicht die Struktur der Listen:
<img src="/posts/lsass-extractor/static/credlist-struct.png" alt="Schema zur Extraktion des IV" style="display: block; margin-left: auto; margin-right: auto; width: 650px; height: auto;">

In dieser letzten Struktur kann der verschlüsselte NTLM-Hash gefunden werden. Ähnlich wie bei der verketteten Liste der 'Anmeldeinformationen-Liste' führt die Interpretation der Zahl an der Adresse des ersten Eintrags der Primären Anmeldeinformationen-Liste als Adresse zur Entdeckung der nächsten Struktur, wobei eine zirkuläre Liste verfolgt wird. Durch das Befolgen der Struktur ist es möglich, die Größe des verschlüsselten NTLM-Hashes und die Speicheradresse, an der sich dieser NTLM befindet, zu ermitteln, um ihn anschließend zu lesen und zu extrahieren.

## NTLM-Entschlüsselung

Der Verschlüsselungsalgorithmus, der zum Verschlüsseln des Hashs verwendet wird, kann im Voraus nicht bestimmt werden. Um den Algorithmus zu identifizieren, wird der verschlüsselte NTLM durch 8 geteilt, und der Rest wird überprüft, um festzustellen, ob die Länge des NTLM durch 8 teilbar ist. Wenn die Länge des NTLM durch 8 teilbar ist, wird AES-Verschlüsselung verwendet. Wenn die Länge des NTLM nicht durch 8 teilbar ist, was auf eine unregelmäßige Länge hinweist, wird 3DES-Verschlüsselung verwendet.

Der folgende Pseudocode stellt die Logik der Algorithmusauswahl dar:

```c
if ( encryptedNTLMLength % 8 != 0) {
 // Verschlüsselte Anmeldeinformationen waren leer?
} else {
 // Algorithmusfunktionen und IV vorbereiten
 if ( encryptedNTLMLength % 8) {
    // AES-Entschlüsselung mit dem bereitgestellten Schlüssel durchführen
 } else {
    // 3DES-Entschlüsselung mit dem bereitgestellten Schlüssel durchführen
 }
}
```

Sobald der verwendete Algorithmus identifiziert wurde, besteht die verbleibende Aufgabe darin, die gewonnenen Informationen mit der bcrypt.h-Bibliothek anzuwenden, indem der gleiche Ansatz wie MSV verfolgt wird. Durch die Konfiguration der erforderlichen Variablen zur Verwendung des richtigen Algorithmus und anschließendem Aufrufen der BCrypt-Decrypt-Funktion mit dem verschlüsselten Schlüssel und dem wiederhergestellten IV wird es möglich, den NTLM-Hash im Klartext abzurufen.