+++
title = 'CTEM par rapport au Red Teaming'
date = 2024-07-29T17:04:31+02:00
draft = false
image = '/posts/ctem/static/cover.jpg'
+++

## Introduction

Chez TeamFence, nous cherchons toujours de nouvelles façons d'améliorer la posture de sécurité de nos clients. Dans cet article de blog, nous discuterons du cadre de gestion continue de l'exposition aux menaces (CTEM) et de la manière dont il peut aider les organisations à minimiser l'exposition aux cyberattaques. Nous comparerons également le CTEM avec le Red Teaming et les attaques simulées pour comprendre les avantages d'une approche proactive de la sécurité.

## Qu'est-ce que le CTEM ?

La gestion continue de l'exposition aux menaces (CTEM) est un cadre dynamique et continu en cinq étapes conçu pour minimiser l'exposition aux cyberattaques. Cette approche proactive aide les organisations à identifier les vulnérabilités, à les associer à des chemins d'attaque potentiels, à les prioriser en fonction du risque pour les actifs critiques et à suivre les progrès des efforts de remédiation. Les entreprises du monde entier adoptent le CTEM pour gérer efficacement les expositions et renforcer leur posture de sécurité.

Le CTEM implique une évaluation approfondie de l'ensemble de l'écosystème d'une organisation, y compris les réseaux, les systèmes, les actifs, et plus encore, pour détecter les expositions et les faiblesses. L'objectif principal est de réduire la probabilité que ces vulnérabilités soient exploitées par des attaquants. La mise en œuvre d'un programme CTEM garantit une amélioration continue des mesures de sécurité en identifiant et en traitant les zones potentiellement problématiques avant qu'elles ne puissent être exploitées.

L'aspect "continu" du CTEM met l'accent sur la relation itérative entre le programme CTEM et les efforts de remédiation des risques. Les données générées par les deux éléments s'informent mutuellement, facilitant des décisions de plus en plus optimales sur la gestion des risques d'exposition.

En utilisant le CTEM, les organisations peuvent garantir un cadre de sécurité résilient qui s'adapte aux menaces évolutives, protégeant en permanence leurs actifs critiques contre les cyberattaques.

## Les cinq étapes du cadre CTEM

![CTEM](static/steps.png)

#### Étape 1 – Délimitation

La première étape consiste à comprendre vos surfaces d'attaque et à déterminer l'importance commerciale de chaque actif, en reconnaissant que ces facteurs évolueront au fil du temps. Cela inclut l'identification des surfaces d'attaque clés avec l'apport de divers décideurs, tels que les responsables de l'IT, du juridique, de la GRC, du développement, de la R&D, du produit et des opérations commerciales.

#### Étape 2 – Découverte

Au cours de la phase de découverte, chaque actif est évalué pour identifier les expositions potentielles et analyser les risques associés. Cela va au-delà de l'identification de vulnérabilités isolées pour inclure d'autres types d'expositions, telles que les risques liés à Active Directory, à l'identité et à la configuration, et examine comment ces expositions pourraient être enchaînées pour créer des chemins d'attaque vers les actifs.

#### Étape 3 – Priorisation

Lors de la phase de priorisation, les expositions sont analysées pour déterminer leur niveau de menace en fonction d'incidents réels connus et de l'importance des actifs touchés. Cette étape est cruciale car les organisations sont souvent confrontées à plus d'expositions qu'elles ne peuvent en traiter en raison de l'énorme volume et des environnements en constante évolution. Le CTEM aide à prioriser les remédiations qui réduisent le plus efficacement le risque pour les actifs critiques, en tenant compte de tous les types d'expositions, y compris les identités et les mauvaises configurations.

#### Étape 4 – Validation

La phase de validation examine comment les attaques peuvent se produire et leur probabilité, en utilisant divers outils pour différents objectifs. Parfois, la validation aide à la priorisation comme dans l'étape 3, tandis que dans d'autres cas, elle est utilisée pour tester en continu les contrôles de sécurité ou automatiser les tests de pénétration périodiques.

#### Étape 5 – Mobilisation

La phase de mobilisation garantit que tout le monde comprend ses rôles et responsabilités dans le cadre du programme. Une mobilisation efficace exige que les équipes de sécurité et IT impliquées dans les efforts de remédiation aient une vision claire de la valeur de réduction des risques de leurs actions et puissent rendre compte de la tendance générale des améliorations de la posture de sécurité au fil du temps.

## Le CTEM est-il meilleur que le Red Teaming ?

Le CTEM et le Red Teaming sont tous deux des outils précieux pour améliorer la posture de sécurité d'une organisation, mais ils servent des objectifs différents. Si notre objectif principal est d'améliorer la posture de sécurité d'une organisation, le CTEM est la voie à suivre. Voyons pourquoi :

#### Red Teaming

Une activité de Red Teaming vise à identifier les vulnérabilités dans les actifs d'une entreprise en simulant une attaque pour prendre le contrôle des systèmes critiques. Ce processus implique une équipe de hackers éthiques qui utilisent une approche opportuniste pour trouver des faiblesses et les exploiter, en se concentrant uniquement sur l'atteinte de leur objectif sans considérer d'autres problèmes ou chemins alternatifs.

L'image suivante pourrait représenter une itération courante de Red Teaming :
![CTEM](static/red-gantt.png)
<br>
Bien que le Red Teaming fournisse des informations précieuses sur les failles de sécurité potentielles, il présente certaines limites notables.

***Premièrement***, il offre un instantané plutôt qu'une évaluation continue. Les vulnérabilités identifiées lors d'un exercice de Red Teaming peuvent rapidement devenir obsolètes à mesure que de nouvelles menaces apparaissent et que l'environnement de l'entreprise évolue.

***Deuxièmement***, le Red Teaming ne fournit pas une vue d'ensemble complète de toutes les vulnérabilités possibles. Il se concentre sur des vecteurs et des chemins d'attaque spécifiques, ce qui peut laisser certaines zones critiques d'exposition inaperçues. Ce champ d'application restreint peut entraîner l'absence de détection de certaines vulnérabilités, en particulier celles qui ne correspondent pas aux tactiques spécifiques utilisées par l'équipe rouge.

***Enfin***, comme nous pouvons le voir sur l'image ci-dessus, le Red Teaming est un processus chronophage et gourmand en ressources, qui peut ne pas être réalisable pour les organisations ayant des budgets limités ou des contraintes opérationnelles. Les coûts élevés et les efforts nécessaires pour mener des exercices de Red Teaming peuvent limiter leur fréquence et leur efficacité à identifier et à résoudre les vulnérabilités. De plus, le résultat d'un exercice de Red Teaming est une liste de vulnérabilités plutôt qu'une liste priorisée d'expositions, obligeant souvent les clients à déterminer eux-mêmes comment mettre en œuvre les mesures nécessaires.

### CTEM

Le CTEM, en revanche, résout de nombreux problèmes auxquels une organisation pourrait être confrontée lorsqu'elle tente d'améliorer sa posture de sécurité :

***Absorption des résultats*** : de nombreuses organisations peinent à absorber les résultats du Red Teaming ou d'autres exercices de simulation d'attaques, trouvant les résultats écrasants et difficiles à comprendre et à prioriser. Le CTEM inclut une phase de mobilisation qui garantit que les équipes internes comprennent pleinement les risques, leur permettant de prioriser et de mettre en œuvre les mesures de remédiation de manière efficace.

***Temps et argent*** : nous devons considérer que ce genre d'activités sont des services B2B, et elles sont coûteuses. Le CTEM est une approche plus rentable de la sécurité, car la phase d'énumération des attaques simulées est remplacée par un effort collaboratif entre les équipes de sécurité et IT pour identifier tous les actifs et expositions de l'organisation.

***Couverture des actifs*** : le CTEM permet une couverture à 100 % des actifs, car la phase de découverte ne se limite pas aux actifs que l'équipe rouge peut trouver, mais est un effort collaboratif entre les équipes de sécurité et IT. Cela signifie que tous les actifs sont pris en compte et que le risque est calculé en fonction de l'importance de l'actif.

***Continu*** : le CTEM est un processus continu, ce qui signifie que l'organisation est toujours consciente des risques et peut agir en conséquence. L'équipe de sécurité peut suivre les changements d'infrastructure, suivre les nouvelles menaces et vulnérabilités émergentes et maintenir la posture de sécurité à jour. Cela soulage en grande partie l'équipe IT de la responsabilité de la sécurité, et le client peut se concentrer sur son activité.

## Conclusion

En conclusion, bien que le Red Teaming soit utile pour découvrir certaines vulnérabilités et tester l'efficacité des défenses d'une entreprise contre des scénarios d'attaque spécifiques, ses limites mettent en évidence la nécessité d'évaluations de sécurité plus continues, complètes et contextuelles.

Chez TeamFence, nous pensons que le CTEM offre une approche plus efficace et durable de la sécurité en fournissant la sécurité en tant que service qui surveille et gère en continu l'exposition d'une organisation aux cybermenaces. En adoptant le CTEM, les entreprises peuvent identifier et résoudre les vulnérabilités de manière proactive, prioriser les efforts de remédiation et maintenir une posture de sécurité résiliente qui s'adapte aux menaces évolutives.

Contactez-nous pour en savoir plus sur la façon dont le CTEM peut aider votre organisation à anticiper les cybermenaces et à protéger vos actifs critiques.
