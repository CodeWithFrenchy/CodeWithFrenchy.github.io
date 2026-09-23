---
title: Comment orienter les choix d’architecture - monolithe modulaire, microservices, événementiel et cloud-native
date: 2026-12-31 19:00:00 -0400
categories: [architecture]
tags: [dotnet]
---

## Préambule

Une décision d’architecture engage bien davantage que l’organisation du code. Elle influence la vitesse de livraison, la cohérence des données, la manière de diagnostiquer une panne et les compétences nécessaires pour exploiter le produit. Ses conséquences deviennent souvent visibles plusieurs mois après les premiers développements, lorsque plusieurs équipes interviennent, que le trafic augmente ou que les règles métier évoluent.

Pourtant, les discussions commencent fréquemment par une préférence technologique : « il nous faut des microservices », « tout doit passer par des événements » ou « nous devons devenir cloud-native ». Ces propositions désignent des moyens. Elles ne précisent ni le problème à résoudre ni le coût que l’organisation accepte de supporter.

Cet article propose une démarche pour les architectes et les responsables techniques qui doivent justifier ces choix. Le principe directeur est simple : **retenir l’architecture la plus simple qui satisfait les contraintes connues, puis ajouter de la complexité lorsqu’un besoin concret la justifie**. Le monolithe modulaire constitue souvent un point de départ pertinent. Il ne représente ni une obligation ni une étape imposée avant les microservices.

Un exemple fictif accompagnera la réflexion : une plateforme de gestion de demandes comprenant l’instruction des dossiers, les décisions, la génération de documents et les notifications. Les exemples .NET et Azure servent à rendre les choix concrets. La démarche reste applicable à d’autres environnements.

## 1. Comparer les bonnes dimensions

Monolithe modulaire, microservices, événementiel et cloud-native ne répondent pas à la même question.

| Dimension | Question à résoudre | Choix possibles |
|---|---|---|
| Organisation et déploiement | Où placer les frontières du code, des responsabilités et des unités déployables ? | Application modulaire déployée ensemble, services autonomes, combinaison des deux |
| Communication | Quand et comment un composant sollicite-t-il un autre composant ? | Appel local, requête réseau, commande asynchrone, événement |
| Exploitation | Comment livrer, configurer, observer et faire fonctionner le système ? | Services managés, conteneurs, plateformes orchestrées, infrastructure administrée directement |

Un monolithe peut publier des événements et fonctionner sur une plateforme managée. Des microservices peuvent communiquer principalement par HTTP. La conteneurisation facilite la reproductibilité de l’environnement d’exécution, mais ne configure pas à elle seule le processus de livraison. Si l’équipe doit encore modifier manuellement la configuration, lancer les migrations de la base de données et vérifier le démarrage après chaque déploiement, ces opérations restent des sources de délai et d’erreur. La fiabilité de la livraison dépend aussi de son automatisation, des contrôles de santé et des mécanismes de retour arrière.

Le terme *cloud-native*, tel que présenté dans la [définition de la CNCF](https://github.com/cncf/toc/blob/main/DEFINITION.md), décrit une approche de construction et d’exploitation adaptée aux environnements cloud. Il ne suffit donc pas à déterminer la bonne granularité des services.

Cette distinction évite une erreur coûteuse : adopter simultanément plusieurs transformations alors qu’une seule répond au problème. Un traitement trop lent peut nécessiter une file et un worker, sans justifier le découpage de tout le domaine métier.

## 2. Partir des contraintes, puis formuler une hypothèse

Une architecture répond à une situation. Avant de dessiner des services, il faut comprendre les comportements attendus et les difficultés actuelles.

Les questions utiles portent notamment sur :

- **Le métier** : quelles opérations doivent réussir ensemble ? Quelles règles changent ensemble ? Où les mêmes termes prennent-ils des sens différents ?
- **La livraison** : qu’est-ce qui ralentit les changements ? La coordination des équipes, les tests, les approbations, le code ou le déploiement ?
- **La charge** : quelles fonctions consomment réellement les ressources ? Leur charge varie-t-elle indépendamment du reste ?
- **La disponibilité** : quelles capacités doivent rester accessibles lorsqu’une autre est indisponible ? Pendant combien de temps ?
- **L’exploitation** : qui intervient en cas d’incident ? Avec quels outils et quelle capacité de diagnostic ?
- **Les données** : quelles exigences de cohérence, d’accès, de résidence et de conservation faut-il respecter ?

Il faut ensuite distinguer une contrainte établie d’une anticipation. « La génération des documents sature les ressources pendant les campagnes » peut être étayé par des mesures. « Nous aurons peut-être cent services un jour » ne constitue pas une exigence actuelle.

Pour notre plateforme fictive, supposons que les règles d’instruction évoluent encore, que la décision doit être enregistrée de manière cohérente avec le dossier et que les documents peuvent être produits après cette décision. Ces éléments suggèrent de conserver une forte proximité entre dossier et décision, tout en laissant une possibilité de traitement différé pour les documents.

C’est une hypothèse de conception. On doit pouvoir expliquer ce qui la confirmerait et ce qui conduirait à la réviser.

## 3. Le monolithe modulaire : conserver la simplicité du déploiement

Un monolithe modulaire regroupe plusieurs modules métier dans une application déployée comme un ensemble. Chaque module expose des contrats explicites et protège son fonctionnement interne. La modularité repose sur des dépendances maîtrisées, pas seulement sur une arborescence de projets.

Dans une solution .NET, les dossiers, les documents et les notifications peuvent disposer de leurs propres composants applicatifs, modèles et accès aux données. Les références entre projets et la visibilité des types limitent les accès possibles. Des essais automatisés d’architecture permettent de vérifier que les dépendances respectent les règles convenues. Aucun médiateur ni framework ne crée automatiquement ces frontières.

### Ce qu’il facilite

Les appels internes évitent les délais et les modes de panne d’un réseau. Le diagnostic suit généralement un chemin plus direct. L’environnement de développement, les tests et la livraison comportent moins de pièces à coordonner.

Lorsque plusieurs modifications utilisent la même base de données et le même mécanisme transactionnel, une transaction locale peut préserver une règle métier sans protocole distribué. Cela ne rend pas toutes les transactions entre modules souhaitables : chacune crée une dépendance qu’il faut assumer. Mais imposer une cohérence différée à une opération qui exige une atomicité immédiate peut ajouter un problème au lieu d’en résoudre un.

Le monolithe facilite aussi l’ajustement des frontières lorsque le domaine reste mal compris. Déplacer une responsabilité entre deux modules demeure un changement significatif, mais implique moins de contrats réseau, de migrations et de coordination opérationnelle.

L’argument de [Martin Fowler en faveur d’un démarrage monolithique](https://martinfowler.com/bliki/MonolithFirst.html) est particulièrement pertinent dans ce contexte d’incertitude. Il s’agit d’un retour d’expérience utile, pas d’une preuve que tous les systèmes doivent commencer ainsi.

### Ce qu’il impose

Les modules partagent une unité de livraison et une partie de leur environnement d’exécution. Une régression, une consommation mémoire excessive ou une saturation des connexions peut affecter des fonctions sans rapport direct avec la modification initiale.

La mise à l’échelle horizontale reste possible : plusieurs instances de l’application peuvent servir le trafic. En revanche, on réplique habituellement l’ensemble, même si une seule fonction exige davantage de capacité. Les [architectures courantes d’applications web .NET](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures) illustrent ce compromis entre simplicité de déploiement et granularité de mise à l’échelle.

La modularité exige enfin une discipline continue. Des lectures directes dans les tables d’un autre module, un modèle partagé omniprésent ou une accumulation de dépendances circulaires peuvent transformer une application bien organisée en système difficile à faire évoluer.

Plusieurs équipes peuvent travailler sur un monolithe. La question est de savoir si le déploiement commun crée une coordination acceptable ou un blocage récurrent. Le nombre d’équipes, à lui seul, ne tranche pas ce choix.

### Vérifier les frontières avec des essais d’architecture

Une frontière dessinée dans un diagramme peut s’éroder au fil des changements. Les essais d’architecture transforment certaines décisions de structure en règles exécutables dans la suite de tests et le pipeline d’intégration continue.

Pour notre plateforme, on peut vérifier que Documents ne référence pas les types internes de Dossiers, que les échanges passent par les contrats prévus et que le cœur métier ne dépend pas des composants de persistance. Mon article sur les [essais automatisés d’architecture dans un projet .NET](https://codewithfrenchy.com/posts/essais-architecture-automatises-dotnet/) présente leur mise en œuvre avec ArchUnitNET et xUnit. Des intégrations existent aussi pour MSTest et NUnit.

Ces essais détectent rapidement certaines dérives et sécurisent les refactorisations. Leur portée reste limitée aux règles exprimées et aux dépendances analysables. Ils ne prouvent ni la pertinence du découpage métier ni l’absence d’accès indésirables par SQL dynamique ou par appel réseau. Les tests d’intégration et les contrôles d’accès à la base de données restent nécessaires.

Il faut privilégier les contraintes qui protègent une décision importante. Des règles trop nombreuses ou liées à des détails d’implémentation peuvent rigidifier le code. Lorsqu’une décision évolue, les essais correspondants doivent être révisés avec elle.

### Application au fil conducteur

Une première version de notre plateforme peut être une application ASP.NET Core modulaire, hébergée sur Azure App Service, avec une base de données relationnelle. Le module Dossiers contrôle l’instruction et l’enregistrement des décisions. Documents et Notifications exposent des contrats restreints.

Cette organisation réduit la charge opérationnelle initiale tout en donnant une place explicite aux responsabilités. Elle reste pertinente aussi longtemps que ses limites ne compromettent pas les objectifs du produit.

## 4. Les microservices : acheter de l’autonomie au prix de la distribution

Les microservices cherchent à rendre des capacités métier suffisamment autonomes pour évoluer et être déployées indépendamment. Leur intérêt dépend de la qualité des frontières et de la capacité de l’organisation à prendre en charge cette autonomie.

Un service n’est pas nécessairement petit en nombre de lignes. Il doit surtout posséder une responsabilité cohérente. Les contextes métier constituent un point de départ pour réfléchir au découpage, sans imposer une correspondance mécanique entre chaque contexte et un service.

### Les bénéfices possibles

Une frontière bien placée permet de livrer une capacité sans redéployer les autres. Elle peut aussi rendre possible une mise à l’échelle ciblée ou une politique d’exploitation adaptée à un besoin particulier.

Pour notre plateforme, un service documentaire pourrait devenir pertinent si plusieurs produits utilisent cette capacité, si une équipe en assume la responsabilité complète et si son cycle de livraison est indépendant de celui de l’instruction des dossiers.

L’isolation peut également protéger certaines fonctions. Toutefois, un service séparé ne garantit pas la continuité : si chaque consultation de dossier attend sa réponse, sa panne reste susceptible de bloquer le parcours. L’isolation dépend des dépendances réelles, des limites de ressources et du comportement prévu en cas d’échec.

### Les coûts incompressibles

La distribution introduit des situations absentes d’un appel local. Une requête peut expirer après que le destinataire a effectué l’opération. Une nouvelle tentative peut la reproduire. Plusieurs appels successifs cumulent les délais et multiplient les points de défaillance.

L’autonomie de déploiement suppose également des contrats compatibles pendant la coexistence de versions différentes, des pipelines fiables, une observabilité traversant les services et une responsabilité claire en production. Une équipe qui développe un service sans pouvoir le diagnostiquer ni le faire évoluer dispose d’une autonomie incomplète.

Les [compromis du style microservices](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/microservices) concernent ainsi autant la cohérence des données et les opérations que le développement.

Les coûts comprennent l’infrastructure, mais aussi les identités techniques, les droits, les secrets, les migrations, les environnements de test, les contrats et les interventions. Une baisse de consommation sur un composant peut être absorbée par cette charge permanente.

### Reconnaître le monolithe distribué

Un système composé de nombreux services peut conserver toutes les contraintes d’un monolithe : versions à livrer ensemble, tables modifiées par plusieurs applications, longues chaînes d’appels et changement métier réparti partout.

Il cumule alors coordination forte et pannes réseau. Ajouter de nouveaux services ne corrige pas nécessairement le problème. Revoir les frontières, rapprocher des responsabilités ou regrouper certains composants peut être plus efficace.

La bonne question devient : **quelle indépendance utile cette séparation procure-t-elle, et comment la vérifier ?**

## 5. L’événementiel : découpler dans le temps sans perdre la maîtrise du processus

L’asynchronisme permet de différer une opération. L’événementiel organise certaines interactions autour de faits déjà survenus. Ces deux notions se recoupent sans être équivalentes.

| Interaction | Signification | Exemple |
|---|---|---|
| Requête avec réponse | Obtenir un résultat pendant l’échange | Consulter l’état d’un dossier |
| Commande asynchrone | Demander une action à un responsable identifié | Générer le document d’une décision |
| Événement | Annoncer un fait que des consommateurs peuvent utiliser | Une décision a été rendue |

Placer une commande dans une file ne transforme pas automatiquement le système en architecture événementielle. De même, une notification en mémoire dans un processus ne possède pas les garanties d’un message durable entre applications.

### Ce que le découplage temporel apporte

Après l’enregistrement d’une décision, la plateforme peut différer la production du document. Un worker consomme les demandes et adapte son rythme aux ressources disponibles. Ce worker peut rester sous la responsabilité de la même équipe et dans le même cycle de livraison : disposer d’un processus supplémentaire ne suffit pas à créer un microservice autonome.

Le [nivellement de charge par file d’attente](https://learn.microsoft.com/en-us/azure/architecture/patterns/queue-based-load-leveling) permet d’absorber certains pics. La file ne résout toutefois pas une surcharge durable : si les arrivées dépassent continuellement la capacité de traitement, le retard augmente. Il faut surveiller notamment l’âge des messages et le délai métier, pas seulement la taille de la file.

Un événement de décision peut, quant à lui, intéresser plusieurs consommateurs sans que son producteur connaisse leurs traitements. Ce découplage facilite certains ajouts fonctionnels, à condition que le contrat publié représente un fait métier stable.

### Ce qu’il faut rendre explicite

L’utilisateur ne reçoit plus forcément un résultat final immédiatement. Le produit doit distinguer les états « accepté », « en cours », « terminé » et « en échec », puis définir ce qui se passe lorsque le délai attendu est dépassé.

Selon les garanties du transport et la conception, il faut gérer les doublons, les reprises, l’ordre des messages, les messages impossibles à traiter et l’évolution des schémas. Un traitement idempotent évite qu’une répétition produise deux fois le même effet métier. Une file de messages en erreur n’est utile que si quelqu’un peut analyser et corriger la situation.

Lorsqu’une transaction dans la base de données doit provoquer une publication fiable, une [Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html) peut réduire le risque d’enregistrer la modification sans publier son message. Elle implique aussi un relais, de la surveillance et une gestion des répétitions. Si une opération traverse plusieurs propriétaires de données, ses échecs et compensations éventuelles doivent être conçus au niveau métier.

Enfin, événementiel ne signifie ni Event Sourcing ni CQRS. Ces choix répondent à d’autres besoins et méritent leur propre justification. La [présentation de l’architecture événementielle](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/event-driven) fournit un cadre utile pour évaluer ce modèle sans en faire une règle universelle de communication.

## 6. Cloud-native : choisir une capacité d’exploitation adaptée

Une application unique sur une plateforme managée peut bénéficier d’une configuration externalisée, de déploiements automatisés, de contrôles de santé, de journaux centralisés et d’une mise à l’échelle adaptée. Ces capacités ne sont pas réservées aux microservices.

Dans notre exemple, App Service peut héberger l’application, tandis qu’un mécanisme de messagerie et un hébergement distinct prennent en charge les traitements documentaires. Azure Service Bus constitue une option pour des messages métier durables. Son choix doit dépendre des garanties requises, pas seulement de sa disponibilité dans le catalogue.

### Les avantages des services managés

Ils délèguent certaines tâches d’infrastructure et peuvent réduire le travail nécessaire pour livrer et exploiter le produit. Ils imposent en contrepartie des contraintes de fonctionnement, des quotas, des modèles tarifaires et des dépendances aux services du fournisseur.

La portabilité doit être évaluée concrètement. Un conteneur peut faciliter le déplacement d’un processus sans rendre interchangeables son identité, sa base de données, sa messagerie ou ses procédures d’exploitation.

### Quand une plateforme plus complexe se justifie

Kubernetes peut répondre à des besoins de contrôle, d’intégration réseau, d’orchestration ou de standardisation que l’organisation sait prendre en charge. Il ajoute également des responsabilités de configuration, de sécurité, de mise à niveau et de diagnostic, même lorsque le plan de contrôle est managé.

Il n’existe pas de nombre universel de services à partir duquel Kubernetes devient rentable. Le choix dépend aussi des compétences disponibles, des contraintes du système et de la plateforme déjà exploitée. La [comparaison des options de calcul Azure](https://learn.microsoft.com/en-us/azure/architecture/microservices/design/compute-options) aide à examiner ces différences.

Le coût total doit intégrer le travail humain et les incidents, en plus de la facture cloud. Une facturation à l’usage peut convenir à une charge intermittente, tandis qu’une charge soutenue demande une autre comparaison. Les étiquettes « serverless » ou « conteneur » ne permettent pas de conclure seules.

## 7. Les données déterminent les frontières réellement tenables

Le découpage du code peut sembler convaincant alors que les règles métier exigent des modifications simultanées de données placées de part et d’autre de la frontière.

Dans notre plateforme, séparer artificiellement la décision du dossier peut compliquer une règle qui exige leur mise à jour atomique. Avant d’introduire une coordination distribuée, il faut se demander si cette séparation correspond à une véritable autonomie métier.

### Propriété logique et isolation physique

Chaque module ou service devrait avoir un propriétaire explicite pour ses données et ses règles de modification. Partager un serveur de base de données ne signifie pas nécessairement partager la propriété des tables. À l’inverse, utiliser plusieurs bases de données n’empêche pas un couplage fort si les applications dépendent directement de leurs schémas internes.

Dans un monolithe, on peut commencer par séparer les accès et les responsabilités. Pour des services indépendants, les écritures doivent passer par le propriétaire concerné. Les [considérations sur les données dans les microservices](https://learn.microsoft.com/en-us/azure/architecture/microservices/design/data-considerations) éclairent ce lien entre autonomie et gestion décentralisée des données.

Une séparation physique devient pertinente lorsque la sécurité, la disponibilité, les performances ou le cycle de vie le demandent. Elle ne remplace pas les contrôles d’accès. Des services distincts disposant tous de droits étendus sur les mêmes données ne créent pas une isolation effective.

### Accepter les conséquences sur les lectures

Une vue transversale peut nécessiter des appels à plusieurs propriétaires ou une projection alimentée de manière asynchrone. La première option crée une dépendance de disponibilité. La seconde introduit du retard et des copies supplémentaires.

Il faut préciser la fraîcheur acceptable pour chaque usage. Un tableau de suivi peut tolérer un décalage qu’une décision réglementaire ou une vérification d’autorisation ne tolère pas.

Les copies de données élargissent aussi les obligations de protection, de conservation et de suppression. La sécurité doit couvrir les consommateurs et les traitements asynchrones, avec une autorisation adaptée aux opérations. Une passerelle d’API ne suffit pas à résoudre ces responsabilités.

## 8. Moderniser progressivement sans transformer la migration en objectif

La modernisation d’un système existant vise un résultat : réduire un délai de livraison, éliminer un risque, améliorer la disponibilité ou absorber une charge. Le nombre de services extraits constitue rarement un bon indicateur de réussite.

### Établir un point de comparaison

Avant de modifier la structure, documenter les difficultés observées : changements fréquemment bloqués, incidents récurrents, temps de traitement, coût d’exploitation et opérations manuelles. Relever aussi les parcours et règles qui fonctionnent correctement.

Ce point de comparaison permet d’évaluer le résultat. Sans elle, une migration peut être déclarée réussie parce que le nouveau système existe, même si les équipes livrent moins vite.

### Clarifier les frontières dans l’existant

Une première étape consiste souvent à modulariser sur place : regrouper les comportements qui changent ensemble, limiter les accès directs aux données et rendre les dépendances explicites.

Ce travail procure déjà un bénéfice. Il révèle aussi les séparations artificielles avant qu’elles deviennent des contrats réseau difficiles à modifier.

### Choisir une extraction représentative

Le premier candidat doit avoir une responsabilité cohérente, un bénéfice identifiable et un risque maîtrisable. Un composant entièrement périphérique peut valider les outils sans éprouver les difficultés importantes de la migration. À l’inverse, commencer par le cœur le plus couplé multiplie les inconnues.

Pour notre plateforme, les documents peuvent être un candidat si leur charge et leur rythme d’évolution posent un problème réel. Une simple exécution en arrière-plan reste une solution à comparer à l’extraction complète.

### Organiser la coexistence

Le [pattern Strangler Fig](https://learn.microsoft.com/en-us/azure/architecture/patterns/strangler-fig) permet de remplacer progressivement des capacités en dirigeant les opérations vers l’ancienne ou la nouvelle implémentation. La coexistence doit avoir un périmètre et des critères de fin.

Une [couche anticorruption](https://learn.microsoft.com/en-us/azure/architecture/patterns/anti-corruption-layer) peut traduire les contrats et les concepts historiques pour éviter qu’ils structurent entièrement le nouveau modèle. Elle ne doit pas masquer les différences métier : si l’ancien système interprète un statut autrement, cette divergence doit être résolue explicitement.

Pendant cette période, les interfaces doivent accepter les versions réellement en production. Le routage, les traitements différés et les opérations déjà commencées demandent une attention particulière.

### Préparer le transfert des données

Le déplacement des données est souvent plus délicat que celui du code. Pour chaque étape, il faut savoir quelle source fait autorité et quel composant peut écrire.

Une stratégie possible consiste à copier l’historique, rattraper les modifications intervenues pendant la copie, réconcilier les résultats, puis basculer les écritures. Selon le système, ce rattrapage peut utiliser une capture des changements ou demander une fenêtre de suspension. Le mécanisme doit être adapté aux garanties disponibles.

Écrire naïvement dans deux bases de données ne constitue pas une stratégie fiable : une écriture peut réussir et l’autre échouer. Il faut concevoir les reprises, les conflits et la vérification des invariants, y compris pour les messages déjà en circulation.

Le retour arrière exige lui aussi un scénario précis. Après des écritures dans le nouveau modèle, rétablir le routage ne restaure pas automatiquement l’ancien état. Une synchronisation inverse, une interruption contrôlée ou une correction en avant peut être nécessaire.

### Mesurer, retirer, puis décider de la suite

Après la bascule, vérifier le bénéfice attendu et supprimer les anciens chemins devenus inutiles : code, droits, synchronisations et procédures temporaires. Leur maintien prolonge le coût et l’ambiguïté de la transition.

La migration peut s’arrêter lorsque le problème initial est résolu. Elle doit aussi être réexaminée si les extractions imposent des livraisons synchronisées, multiplient les incidents ou déplacent simplement le travail vers l’exploitation. Certaines capacités peuvent rester durablement dans le monolithe. D’autres peuvent être regroupées si leurs frontières ne tiennent pas.

## 9. Construire une décision que l’on peut réévaluer

Une grille utile relie une difficulté à des preuves et à plusieurs réponses possibles. Elle évite d’attribuer une note abstraite à un style d’architecture.

| Question | Preuves à recueillir | Réponses à comparer |
|---|---|---|
| Les équipes doivent-elles livrer indépendamment ? | Changements bloqués, dépendances entre versions, fréquence des coordinations | Améliorer les frontières et la livraison commune, extraire une capacité autonome |
| Une fonction a-t-elle une charge particulière ? | Profils CPU et mémoire, pics, délais, coût des réplicas | Optimiser, augmenter la capacité, ajouter un worker, isoler un service |
| Une panne se propage-t-elle trop loin ? | Incidents, dépendances critiques, ressources partagées | Limiter les ressources, prévoir un mode dégradé, découpler, isoler |
| Le résultat peut-il arriver plus tard ? | Délai acceptable, conséquences métier, parcours utilisateur | Garder une réponse immédiate, introduire une commande différée ou des événements |
| Une règle exige-t-elle une mise à jour atomique ? | Invariants, exceptions autorisées, coûts de correction | Conserver une transaction locale, revoir la frontière, concevoir une coordination distribuée |
| La plateforme actuelle limite-t-elle le produit ? | Contraintes démontrées, charge opérationnelle, compétences | Ajuster l’hébergement, utiliser un service managé, adopter une plateforme plus contrôlable |
| Une extraction améliore-t-elle réellement la situation ? | Comparaison des délais, incidents et coûts avant et après | Poursuivre, stabiliser, arrêter, regrouper |

### Documenter les choix dans un registre de décisions

Les preuves recueillies et les arbitrages doivent être conservés dans un registre de décisions accessible à l’équipe. Une fiche de décision d’architecture, ou ADR (*Architecture Decision Record*), explique pourquoi une option a été retenue dans un contexte donné. Elle évite que les personnes qui reprennent le système aient à reconstruire les raisons du choix à partir du code.

Comme je l’explique dans mon article sur le [registre de décisions](https://codewithfrenchy.com/posts/registre-decisions/), cette pratique favorise la traçabilité, la continuité et une compréhension commune. Pour une décision structurante, consigner au minimum :

- Le contexte, le problème et les contraintes.
- Les options évaluées et les raisons du choix.
- Les avantages attendus, les coûts et les limites acceptés.
- Les hypothèses, les preuves disponibles et les critères de réévaluation.
- La date, les participants et le statut de la décision.

Le registre doit rester proportionné aux enjeux. Une fiche concise vaut mieux qu’un document exhaustif que personne ne maintient. Lorsqu’un choix est remplacé, conserver la décision précédente et la relier à la nouvelle permet de comprendre l’évolution du système. Les essais d’architecture associés doivent alors être mis à jour pour refléter les nouvelles contraintes.

Pour notre exemple, une décision pourrait être : conserver le traitement des dossiers et des décisions dans un monolithe modulaire, puis exécuter la génération documentaire de manière asynchrone. L’extraction d’un service documentaire serait réévaluée si une responsabilité d’équipe distincte, des consommateurs supplémentaires ou des contraintes de livraison indépendantes apparaissaient.

Ce raisonnement ne promet pas une architecture définitive. Il rend le choix compréhensible et sa révision possible.

## Conclusion

Une architecture se juge à sa capacité à soutenir le produit et l’organisation dans la durée. Le monolithe modulaire favorise une exploitation simple et des changements locaux, mais conserve un déploiement et des ressources partagés. Les microservices peuvent apporter une autonomie précieuse, au prix de contrats distribués, d’une gestion des données plus exigeante et d’une charge opérationnelle accrue.

L’événementiel peut découpler les traitements dans le temps, à condition de prendre en charge les délais, les reprises et la compréhension du processus métier. L’approche cloud-native peut faciliter la livraison et l’exploitation, sans imposer une topologie particulière ni rendre une plateforme complexe nécessaire.

Le rôle de l’architecte consiste à rendre ces compromis explicites : quel problème résout-on, quel coût accepte-t-on, quelles preuves attend-on et quand faut-il revoir la décision ? Partir d’une solution simple, protéger ses frontières et mesurer les résultats permet de faire évoluer le système sans confondre sophistication technique et progrès.
