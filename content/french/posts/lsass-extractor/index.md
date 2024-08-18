+++
title = 'Extracteur de Lsass'
date = 2023-09-16T16:39:49+02:00
draft = false
image = '/posts/lsass-extractor/static/cover.png'
+++

## Introduction

Chez TeamFence, nous croyons fermement qu'il est de plus en plus important pour un opérateur offensif de comprendre les détails derrière les techniques et les outils.

Pour cette raison, dans cet article de blog, nous allons examiner comment Mimikatz extrait les identifiants à partir de dumps de lsass, ce qui nous permettra de réimplémenter la logique dans un outil adapté à nos besoins futurs.

**Remarque** : toutes les analyses effectuées au cours de ce projet ont été réalisées sur une station de travail Windows 11 Enterprise Evaluation.

## Pourquoi LSASS ?

Avant d'entrer dans les détails techniques, il est important de se demander : pourquoi faisons-nous cela ?

Lorsqu'un hacker accède à une machine cible exécutant Windows, l'une des étapes cruciales consiste à obtenir les identifiants. Windows offre plusieurs cibles potentielles, mais deux dépôts principaux se distinguent : la base de données de sécurité locale (Security Account Manager) et le processus LSASS (Local Security Authority Subsystem Service).

Dans cet article, nous allons travailler sur LSASS, qui joue un rôle central dans Windows en supervisant l'authentification. En tant que plaque tournante des demandes d'authentification provenant de divers services, il joue un rôle clé dans la rationalisation du flux d'authentification. Ce processus implémente divers packages d'authentification tels que NTLM, Kerberos, WDigest, entre autres. Par conséquent, il représente une cible précieuse et riche en informations pour les hackers.

## Hashs NTLM

Comme nous l'avons vu dans la section précédente, LSASS gère plusieurs packages d'authentification, y compris NTLM. NTLM est un protocole d'authentification challenge-response largement utilisé dans les environnements Windows. Lorsqu'un utilisateur se connecte à une machine Windows, le système stocke les identifiants de l'utilisateur en mémoire. Ces informations incluent le mot de passe de l'utilisateur sous forme de hash NTLM.

Dans cet article, nous nous concentrerons sur l'extraction de ces hashs NTLM à partir du processus LSASS, ce qui nous permettra de comprendre plus facilement la logique sous-jacente.

Dans l'image suivante, nous pouvons voir la structure interne de LSASS, mettant en évidence l'endroit où les hashs NTLM sont stockés.

![lsass](/posts/lsass-extractor/static/lsass-structure.png)

## Où sont toutes les pièces ?

Pour commencer, il est crucial de comprendre où sont stockées les informations et sous quel format. Le processus LSASS est un composant critique, et toutes les informations ne sont pas stockées en texte clair. Dans ce cas, les hashs NTLM sont cryptés. Donc, pour les extraire, nous devons localiser les clés de décryptage. Le schéma suivant illustre les modules impliqués dans la gestion des clés et des identifiants.

<img src="/posts/lsass-extractor/static/information-schema.png" alt="où sont les pièces" style="display: block; margin-left: auto; margin-right: auto; width: 500px; height: auto;">

Il est important de se rappeler que le matériel cryptographique du processus LSASS est une donnée d'exécution, ce qui signifie qu'il change après chaque redémarrage. Par conséquent, nous devons l'extraire à chaque fois pour garantir des informations à jour.

```text
"Il est intéressant de noter que la gestion du cryptage est effectuée via LsaProtectMemory, qui est un wrapper pour LsaEncryptMemory, qui à son tour est un wrapper pour BCryptEncrypt, une fonction de cryptage courante présente dans bcrypt.h"
```

## Modèles de modules

À ce stade, nous avons une compréhension claire de la zone de la mémoire où toutes les pièces sont stockées. Maintenant, nous devons trouver un moyen de les extraire. En analysant le code de PyPyKatz, une version Python de Mimikatz, nous pouvons comprendre l'approche utilisée pour extraire les clés et les identifiants.

Tout d'abord, nous devons extraire certaines informations sur le système : ProcessorArchitecture, BuildNumber, et à partir du "ModuleListStream" le TimeDateStamp de "lsasrv.dll".

Une fois que nous avons ces informations, nous pouvons les utiliser pour identifier le modèle correct à utiliser pour l'extraction. Les modèles sont des classes qui contiennent la signature et les décalages qui nous permettent de savoir comment nous déplacer dans la mémoire pour extraire les clés et les identifiants.

Dans notre cas, qui est un Windows 11 Enterprise Evaluation, le modèle pour la partie lsasrv.dll est le suivant :

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

Jetez un œil au [code source](https://github.com/skelsec/pypykatz/blob/c91dcdc09289ad2e93c475e7c640d0f90906a7c0/pypykatz/lsadecryptor/lsa_template_nt6.py#L301) de PyPyKatz pour voir tous les modèles.

## Extraction du vecteur d'initialisation

Pour extraire les clés, nous devons d'abord extraire le vecteur d'initialisation (IV). L'IV est un nombre aléatoire utilisé pour garantir que le même message en texte clair ne donne pas lieu au même texte chiffré. L'IV est stocké en mémoire, et nous pouvons l'extraire en cherchant dans la mémoire du module lsasrv.dll le motif présent dans le modèle.

Une fois que nous avons l'emplacement du motif, nous pouvons utiliser les décalages pour nous déplacer dans la mémoire et extraire l'IV.

<img src="/posts/lsass-extractor/static/iv-schema.png" alt="schéma pour extraire l'IV" style="display: block; margin-left: auto; margin-right: auto; width: 600px; height: auto;">

Comme nous pouvons le voir sur le schéma, une fois que nous avons l'emplacement du motif, nous devons sauter un décalage prédéfini (présent dans le modèle) de 67 octets pour trouver une autre valeur, qui est un autre décalage. Ce décalage est l'emplacement de l'IV.

## Extraction des clés

Il est maintenant temps d'extraire les clés. L'approche est similaire à celle utilisée pour extraire l'IV. Nous devons trouver le même motif dans la mémoire, puis utiliser les décalages pour extraire les clés.

<img src="/posts/lsass-extractor/static/des-schema.png" alt="schéma pour extraire l'IV" style="display: block; margin-left: auto; margin-right: auto; width: 600px; height: auto;">

Cette fois, l'utilisation du pointeur ne conduit pas directement à la valeur souhaitée, mais plutôt à une structure BCRYPT_KEY_HANDLE, qui, dans bcrypt.h, est un type de données qui permet de référencer et de manipuler des clés cryptographiques dans le cadre CNG. Dans pypykatz, la classe KIWI_BCRYPT_HANDLE_KEY réplique ce type.

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

À ce stade, nous avons toutes les informations nécessaires pour décrypter les hashs NTLM. Nous avons l'IV et les clés. Nous pouvons maintenant commencer la recherche des LogonSessions.

## LogonSessions

Les LogonSessions sont les structures qui contiennent les informations sur les utilisateurs connectés au système. Les LogonSessions sont stockées en mémoire dans le module Msv1_0.dll, et nous pouvons les extraire en recherchant le motif présent dans le modèle spécifique, qui dans notre cas est le suivant :

```python
elif WindowsBuild.WIN_11_2022.value <= sysinfo.buildnumber < WindowsBuild.WIN_11_2023.value: #20348
    template.signature = b'\x45\x89\x34\x24\x4c\x8b\xff\x8b\xf3\x45\x85\xc0\x74'
    template.first_entry_offset = 24
    template.offset2 = -4
```

Il n'est pas possible de connaître le nombre exact de LogonSessions présentes en mémoire, nous devons donc d'abord récupérer le compteur.

#### Compteur de LogonSessions et adresses

Avec une approche

 similaire à celle utilisée pour extraire l'IV et les clés, nous devons trouver le motif en mémoire, puis utiliser les décalages pour extraire le compteur.

Ce compteur nous permet de parcourir toutes les adresses de LogonSessions, où chaque entrée se trouve 16 octets après la précédente.

<img src="/posts/lsass-extractor/static/counter-addresses.png" alt="schéma pour extraire l'IV" style="display: block; margin-left: auto; margin-right: auto; width: 600px; height: auto;">

En convertissant ces adresses en un char*, elles deviennent un pointeur qui pointe vers l'emplacement en mémoire où les entrées LogonSessions sont stockées. Cela permet d'accéder aux données associées à la LogonSession, permettant de récupérer des informations pertinentes.

#### Nom d'utilisateur et domaine

Maintenant que nous avons les adresses des LogonSessions, nous pouvons commencer à extraire les informations. En sautant de 144 octets à partir de l'adresse, plusieurs champs sont présents, y compris la longueur du nom d'utilisateur, la longueur maximale et un pointeur vers le nom d'utilisateur réel.
La même logique s'applique au domaine, qui est stocké en mémoire après le pointeur vers le nom d'utilisateur.

Il est maintenant possible d'extraire le nom d'utilisateur et le domaine de l'utilisateur en suivant les pointeurs et en lisant en mémoire le nombre exact d'octets stockés dans le champ de longueur maximale.

<img src="/posts/lsass-extractor/static/logon-structure.png" alt="schéma pour extraire l'IV" style="display: block; margin-left: auto; margin-right: auto; width: 650px; height: auto;">

#### Liste des identifiants

Pour la dernière partie de l'extraction, nous devons nous concentrer sur les listes d'identifiants. À partir du pointeurToDomain vu dans la section précédente, en avançant de 96 octets, l'adresse mémoire où se trouve la première 'Liste des identifiants' peut être trouvée. Une fois la valeur à l'adresse pointée lue en tant qu'uint64_t et interprétée comme une adresse mémoire, elle fournira l'adresse mémoire où se trouve la deuxième 'Liste des identifiants', et ce processus se poursuit de manière itérative. Étant donné qu'il s'agit d'une liste chaînée circulaire, lorsque la valeur correspond à l'adresse mémoire de la première structure, cela indique que toute la liste a été parcourue.

Chaque entrée de la liste chaînée 'Liste des identifiants' contient sa propre liste chaînée associée, appelée 'Liste d'identifiants primaires'. Pour récupérer l'adresse de la première entrée de cette liste chaînée, il est nécessaire d'avancer de 16 octets et de lire cette zone mémoire comme un pointeur char.

Le schéma suivant illustre la structure des listes :
<img src="/posts/lsass-extractor/static/credlist-struct.png" alt="schéma pour extraire l'IV" style="display: block; margin-left: auto; margin-right: auto; width: 650px; height: auto;">

Dans cette structure finale, le hash NTLM chiffré peut être trouvé. Tout comme la liste chaînée 'Liste des identifiants', interpréter le nombre à l'adresse de la première entrée de la Liste primaire comme une adresse conduit à la découverte de la structure suivante, en suivant une liste circulaire. En suivant la structure, il est possible de récupérer la taille du NTLM chiffré et l'adresse mémoire où ce NTLM est situé, permettant sa lecture et son extraction ultérieures.

## Décryptage NTLM

L'algorithme de cryptage utilisé pour chiffrer le hash ne peut pas être déterminé à l'avance. Pour identifier l'algorithme, le NTLM chiffré est divisé par 8 et son reste est vérifié pour voir si la longueur du NTLM est divisible par 8. Si la longueur du NTLM est divisible par 8, le cryptage AES est utilisé. Si la longueur du NTLM n'est pas divisible par 8, indiquant une longueur irrégulière, le cryptage 3DES est utilisé.

Le pseudocode suivant représente la logique de sélection de l'algorithme :

```c
if ( encryptedNTLMLength % 8 != 0) {
 // L'identifiant chiffré était vide ?
} else {
 // Préparer les fonctions d'algorithme et l'IV
 if ( encryptedNTLMLength % 8) {
    // Effectuer le décryptage AES en utilisant la clé fournie
 } else {
    // Effectuer le décryptage 3DES en utilisant la clé fournie
 }
}
```

Une fois que l'algorithme utilisé a été identifié, la tâche restante consiste à appliquer les informations acquises en utilisant la bibliothèque bcrypt.h, en suivant la même approche que MSV. En configurant les variables nécessaires pour utiliser le bon algorithme et en invoquant ensuite la fonction BCrypt-Decrypt avec la clé chiffrée et l'IV récupéré, il devient possible de récupérer le hash NTLM en texte clair.