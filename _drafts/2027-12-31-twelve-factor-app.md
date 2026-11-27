---
title: Twelve-Factor App - principes, bénéfices et limites pour les applications .NET
date: 2027-12-31 19:00:00 -0400
categories: [architecture, bonnes-pratiques]
tags: [dotnet, azure, twelve-factor-app, cloud-native, devops, configuration, scalabilite, deploiement-continu]
---

> ⚠️ À RELIRE + valider le format du code

## Préambule

Une application fonctionne sur son serveur habituel, mais son déploiement sur une deuxième instance révèle parfois des dépendances inattendues. Les sessions résident en mémoire, des documents sont écrits sur le disque local et un traitement nocturne démarre deux fois. Une configuration différente exige une nouvelle compilation, tandis que les journaux disparaissent au redémarrage.

Ces problèmes touchent à la manière dont l’application est construite, configurée et exploitée.

Twelve-Factor App fournit une grille pour rendre ces responsabilités explicites. Ses principes peuvent aider les équipes à automatiser les déploiements, remplacer des instances et faire évoluer la capacité, quel que soit leur langage ou leur plateforme. Leur application demande toutefois de comprendre les compromis qu’ils impliquent : dépendances réseau, état partagé, compatibilité des versions et coût opérationnel.

Nous allons examiner les douze facteurs sous cet angle. Les exemples .NET et Azure illustrent leur application, mais la démarche peut être transposée à d’autres environnements. L’objectif est de comprendre ce qu’ils apportent et ce qu’il reste à concevoir autour d’eux.

## Comprendre la portée de la méthode

La méthode [The Twelve-Factor App](https://12factor.net/) s’appuie sur l’expérience d’applications livrées comme services, notamment sur Heroku. Elle traite de leur construction, de leur portabilité et de leur exploitation. Elle ne définit ni les frontières métier ni les règles de cohérence des données.

Un monolithe modulaire peut appliquer ces principes. À l’inverse, un ensemble de microservices peut les respecter très mal. Le nombre de services et la présence de Kubernetes ne suffisent donc pas à évaluer l’architecture.

Prenons l’exemple d’un produit de gestion des commandes avec une API ASP.NET Core, un worker de traitement et une base de données SQL. Les documents sont stockés dans Azure Blob Storage et les traitements différés passent par un système de messagerie. Ces composants illustrent les principes, mais ne sont pas nécessaires pour les appliquer.

Il faut aussi distinguer les exigences du texte original, les mécanismes proposés par .NET et les adaptations choisies par l’équipe. Cette distinction évite de présenter chaque fonctionnalité actuelle du cloud comme un treizième principe implicite.

## 1. Codebase : une origine identifiable pour chaque application

Le facteur [Codebase](https://12factor.net/codebase) relie une application à une base de code versionnée, à partir de laquelle plusieurs déploiements peuvent être produits. Le développement, la préproduction et la production peuvent exécuter des versions différentes, mais celles-ci partagent une origine identifiable.

Ce principe vise à éviter la dérive entre plusieurs copies du même produit. Si une correction est appliquée directement sur un serveur ou maintenue dans une variante propre à un environnement, il devient difficile de savoir ce qui est réellement exécuté.

Pour notre API .NET, chaque déploiement devrait pouvoir être relié à un commit et à un artefact. Les différences entre environnements devraient être portées par la configuration plutôt que par une copie particulière du code.

Le texte original est plus strict que l’affirmation « monorepo ou multirepo, peu importe ». Il établit une correspondance entre application et base de code et recommande d’extraire le code partagé en dépendances. Un monorepo contenant plusieurs applications est une adaptation moderne qui doit préserver leurs frontières et leur traçabilité.

Un historique commun aide à retrouver l’origine d’une version et à comprendre les changements qui l’ont produite. Cette traçabilité dépend du lien établi entre les déploiements et le code versionné. Elle ne rend pas les livraisons indépendantes pour autant. Dans un dépôt partagé, les dépendances entre projets, les pipelines et la stratégie de versionnement doivent aussi soutenir le degré d’autonomie recherché.

## 2. Dependencies : déclarer ce dont l’application dépend réellement

Le facteur [Dependencies](https://12factor.net/dependencies) demande de déclarer les dépendances et de les isoler de celles présentes implicitement sur la machine. Il ne concerne pas uniquement les bibliothèques du langage.

Dans .NET, les `PackageReference`, le SDK sélectionné et les fichiers de verrouillage contribuent à rendre la construction prévisible. Une API peut aussi dépendre de bibliothèques natives, de certificats, de données de fuseaux horaires ou d’un exécutable de conversion de documents.

Une génération de PDF qui fonctionne uniquement parce qu’un outil a été installé manuellement sur un serveur révèle une dépendance non maîtrisée. Un conteneur minimal peut faire apparaître ce problème dès le premier démarrage.

L’image de conteneur ou le processus de provisionnement doit donc décrire les composants nécessaires à l’exécution. Un inventaire NuGet ne couvre pas toute la chaîne.

Une déclaration complète aide l’équipe à préparer un nouvel environnement sans reconstituer l’histoire du serveur précédent. Elle doit toutefois évoluer avec le produit. Figer une dépendance rend la construction plus prévisible, mais peut aussi prolonger l’utilisation d’une version vulnérable si les mises à jour sont négligées.

La déclaration des dépendances ne remplace ni leur audit ni une politique de mise à jour. Elle fournit les éléments nécessaires pour les gérer.

## 3. Config : séparer les paramètres de déploiement du code

Le facteur [Config](https://12factor.net/config) vise les valeurs qui changent entre déploiements, comme les adresses de services, les identifiants et les paramètres d’exploitation. Le texte original recommande les variables d’environnement, gérées indépendamment les unes des autres.

Dans ASP.NET Core, le [système de configuration](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-10.0) assemble plusieurs fournisseurs. Avec les valeurs par défaut de `WebApplication.CreateBuilder`, les variables d’environnement remplacent les valeurs correspondantes des fichiers `appsettings`, et les arguments de commande ont une priorité supérieure. Une clé comme `ConnectionStrings__Orders` représente une configuration hiérarchique. Un fournisseur ajouté ensuite peut encore modifier l’ordre effectif.

Un `appsettings.json` versionné peut contenir des valeurs par défaut non sensibles et communes au produit. Les secrets et les paramètres propres à un déploiement devraient être fournis séparément. Ajouter un coffre externe est une adaptation actuelle du principe, plutôt qu’une lecture littérale des seules variables d’environnement.

Sur Azure, [Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/general/basic-concepts) et [App Configuration](https://learn.microsoft.com/en-us/azure/azure-app-configuration/overview) répondent à des besoins différents. Le premier protège les secrets. Le second centralise les paramètres et les fonctionnalités. Une [identité managée](https://learn.microsoft.com/en-us/azure/app-service/overview-managed-identity) peut éviter de distribuer certains secrets d’accès, à condition d’accorder les autorisations appropriées à la ressource.

Séparer la configuration permet de réutiliser le même artefact dans plusieurs environnements et de réduire les différences liées à la compilation. En contrepartie, une partie du comportement dépend de paramètres gérés hors du code. Il faut pouvoir les valider, les sécuriser et retrouver leur révision. Une variable d’environnement ne protège pas un secret comme le ferait un coffre, et la rotation d’un secret ne garantit pas que tous les processus reliront immédiatement sa nouvelle valeur.

Je recommande de valider les paramètres indispensables au démarrage, par exemple avec les [options typées et `ValidateOnStart`](https://learn.microsoft.com/en-us/dotnet/core/extensions/options). Pour une configuration dynamique, il faut aussi décider comment les changements seront rafraîchis et éviter qu’une opération utilise un mélange de valeurs incompatibles.

## 4. Backing services : rattacher les ressources par un contrat explicite

Le facteur [Backing services](https://12factor.net/backing-services) traite les bases de données, caches, systèmes de messagerie et autres services consommés comme des ressources attachées. L’application reçoit les informations nécessaires pour s’y connecter au lieu de dépendre d’un emplacement implicite.

Pour notre API, on peut ainsi rattacher une base de données de test ou une nouvelle instance de stockage sans modifier le code, à condition que le contrat attendu reste compatible.

Cette condition est essentielle. Remplacer SQL Server par Azure SQL n’équivaut pas à changer de moteur. Même lorsqu’un protocole ou une abstraction est commun, les fonctionnalités, les transactions, les quotas et les comportements en cas de panne peuvent différer.

Une interface `IDistributedCache` simplifie les opérations de cache. Elle ne garantit pas que deux implémentations offrent les mêmes propriétés de concurrence et ne constitue pas, à elle seule, un mécanisme de verrouillage distribué.

Ce découplage permet de faire évoluer une ressource sans redéployer toute la logique applicative, tant que son contrat reste compatible. L’application dépend toutefois toujours de la disponibilité, de la latence et des garanties de cette ressource. Il faut évaluer ces propriétés au-delà de la compatibilité de l’interface.

Externaliser un service ne dispense pas de fixer des délais, de gérer la saturation et de décider quelles opérations peuvent être reprises. Une reprise automatique d’un appel de paiement, par exemple, exige une protection contre la duplication de l’effet métier.

## 5. Build, release, run : séparer la fabrication de l’exécution

Le facteur [Build, release, run](https://12factor.net/build-release-run) distingue la fabrication d’un artefact, son association à une configuration de déploiement et son exécution. Chaque release doit être identifiable.

Dans une chaîne .NET, la CI restaure les dépendances, valide le code et produit un package ou une image. Le déploiement associe ensuite cet artefact à une configuration et à des ressources. Je recommande de promouvoir l’artefact déjà validé plutôt que de le reconstruire pour chaque environnement.

| Élément | Ce qu’il faut pouvoir identifier |
|---|---|
| Code source | Commit à l’origine de la construction. |
| Artefact | Package ou image exacts, idéalement avec un identifiant immuable. |
| Configuration | Révision des paramètres et références de secrets, sans exposer leurs valeurs. |
| Déploiement | Environnement, date et release effectivement activée. |

Un tag d’image réutilisable ne suffit pas à identifier les octets exécutés. De même, une configuration modifiée directement dans le portail peut changer le comportement sans modification du code. Ces changements doivent rester traçables.

La séparation facilite la comparaison des environnements et l’identification de l’artefact d’une ancienne release. Elle ne garantit pas qu’un retour arrière soit possible. Une ancienne image peut ne plus comprendre les données ou le schéma de la base après une migration. La compatibilité des données et de la configuration doit donc être prise en compte dans la stratégie de livraison.

Ce principe s’applique également au déploiement d’un package sur App Service. Le conteneur est un moyen de matérialiser l’artefact, pas une obligation architecturale.

## 6. Processes : rendre une instance remplaçable

Le facteur [Processes](https://12factor.net/processes) demande que les processus ne dépendent pas d’un état durable local pour assurer la continuité du service. L’application conserve bien des données, mais celles qui doivent survivre sont placées dans les ressources appropriées.

La question utile est : si cette instance disparaît maintenant, quelles informations deviennent irrécupérables?

Dans notre produit, les commandes résident dans la base de données, les documents dans un stockage durable et les demandes de traitement dans un mécanisme rejouable. Un cache local reste possible si sa perte ne modifie pas la justesse des opérations. Les fichiers temporaires sont aussi acceptables lorsque le travail peut être recommencé.

Les sessions en mémoire et les fichiers locaux partagés implicitement entre requêtes compliquent cette propriété. En ASP.NET Core, il faut également examiner les [clés Data Protection](https://learn.microsoft.com/en-us/aspnet/core/security/data-protection/configuration/overview?view=aspnetcore-10.0) utilisées pour protéger certains cookies et jetons. Les instances qui doivent les interpréter ont besoin d’une configuration compatible et d’une persistance adaptée.

Lorsque la continuité ne dépend plus d’une instance précise, il devient plus simple de la remplacer ou de répartir les requêtes. Cette approche entraîne souvent davantage d’échanges réseau et de charge sur le stockage partagé. Le choix entre état local et état externalisé dépend de la durabilité attendue, du coût d’accès et des conséquences d’une perte.

Enregistrer un `BackgroundService` dans l’application web ne le rend ni durable ni unique. Chaque instance peut démarrer son propre traitement. Un worker déployé séparément possède un cycle de vie indépendant, mais demande encore une file durable, de l’idempotence ou une coordination selon le besoin.

## 7. Port binding : expliciter l’interface réseau du processus

Le facteur [Port binding](https://12factor.net/port-binding) décrit une application qui expose son service par un port, sans dépendre d’un serveur applicatif configuré manuellement pour héberger son code.

Kestrel fournit cette capacité aux services ASP.NET Core autonomes. En conteneur, l’application doit écouter sur une interface et un port compatibles avec le routage de la plateforme. Les images ASP.NET Core ont adopté [8080 comme port par défaut à partir de .NET 8](https://learn.microsoft.com/en-us/dotnet/core/compatibility/containers/8.0/aspnet-port), avec notamment `ASPNETCORE_HTTP_PORTS` pour les configurations compatibles.

Il faut distinguer le port d’écoute du processus, le routage interne et le point d’entrée public. Déclarer un port dans une image ne configure pas automatiquement l’ingress ni l’application elle-même.

Une interface réseau explicite facilite le raccordement au routage de la plateforme et réduit la dépendance à une configuration d’hébergement implicite. Elle ne prend pas en charge toute la sécurité des échanges. TLS, les proxys de confiance, les délais et les restrictions d’accès doivent être configurés selon l’exposition du service.

Un worker qui consomme des messages n’a pas besoin d’une API publique artificielle pour correspondre au modèle. Les capacités exposées dépendent du rôle du processus.

## 8. Concurrency : faire évoluer la capacité par type de processus

Le facteur [Concurrency](https://12factor.net/concurrency) propose de faire évoluer la capacité en multipliant les processus selon leur rôle. Il n’interdit ni les threads ni l’optimisation de la capacité d’une instance.

Pour notre produit, la charge HTTP et le traitement des commandes peuvent évoluer différemment. Ajouter des workers lorsque la file grossit évite d’augmenter inutilement le nombre d’instances de l’API. À l’inverse, un pic de consultation ne nécessite pas forcément plus de consommateurs.

Dans .NET, les opérations asynchrones permettent de mieux utiliser les ressources pendant les attentes d’entrée-sortie. Elles n’augmentent pas la capacité d’une base de données déjà saturée.

Supposons que chaque instance autorise vingt opérations concurrentes vers SQL. Passer de trois à quinze instances peut multiplier les connexions et la contention en aval. Une application capable de démarrer partout peut donc surcharger plus rapidement une dépendance commune.

Faire évoluer séparément les types de processus permet d’ajuster la capacité aux besoins et d’éviter de multiplier les composants peu sollicités. Si le facteur limitant se trouve ailleurs, par exemple dans la base de données ou une partition très sollicitée, ajouter des instances peut augmenter le coût et la contention sans améliorer le débit utile.

Je recommande de définir des limites explicites et de mesurer la capacité de bout en bout. L’autoscaling doit correspondre au débit que les ressources peuvent réellement absorber, pas seulement à une métrique locale du processus.

## 9. Disposability : préparer le démarrage, l’arrêt et le crash

Le facteur [Disposability](https://12factor.net/disposability) associe démarrage rapide et arrêt propre. Une instance doit pouvoir rejoindre ou quitter le service sans procédure manuelle fragile.

Le [Generic Host .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/generic-host) fournit les mécanismes de cycle de vie et d’arrêt. L’application doit les utiliser pour arrêter la prise de nouveaux travaux, respecter les demandes d’annulation et terminer ou rendre rejouables les opérations en cours dans le délai disponible.

L’arrêt gracieux n’est toutefois pas garanti. Un arrêt forcé ou une perte de machine peut empêcher l’exécution du nettoyage. La justesse du traitement ne doit donc pas dépendre uniquement de `StopAsync`.

### Distinguer les signaux de santé

Les [health checks ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks?view=aspnetcore-10.0) fournissent un mécanisme dont la signification doit être définie :

| Signal | Question posée |
|---|---|
| Startup | L’initialisation a-t-elle eu le temps de se terminer? |
| Liveness | L’instance fonctionne-t-elle assez bien pour éviter un redémarrage? |
| Readiness | Peut-elle recevoir le trafic attendu? |

Deux routes qui renvoient toujours un succès ne prouvent pas deux garanties distinctes. À l’inverse, inclure systématiquement SQL dans la liveness peut provoquer le redémarrage de toutes les instances pendant une panne de base de données. La readiness doit tenir compte des fonctions indispensables et des possibilités de service dégradé.

Des démarrages et des arrêts bien maîtrisés limitent les perturbations lors des déploiements et du remplacement d’instances. Le résultat dépend aussi de l’alignement avec la plateforme, notamment pour le routage et les délais d’arrêt. Les caches froids, les connexions longues et les travaux interrompus doivent être testés dans des conditions représentatives. Un arrêt propre observé localement ne suffit pas à établir cette garantie en production.

## 10. Dev/prod parity : réduire les écarts qui changent le comportement

Le facteur [Dev/prod parity](https://12factor.net/dev-prod-parity) couvre les écarts d’outillage, de personnes et de temps entre développement et production. Il vise davantage que le simple démarrage d’un conteneur en local.

Pour une application qui utilise SQL Server, remplacer tous les tests d’intégration par un stockage en mémoire peut masquer les contraintes relationnelles, les transactions et la traduction des requêtes. De même, un système de messagerie simulé ne reproduit pas nécessairement les redélivraisons ou les limites de taille des messages.

Je recommande de rapprocher les environnements sur les caractéristiques qui influencent la justesse : moteur et version des dépendances, schéma, contrats, configuration et mécanismes d’authentification. Certaines différences d’identité ou de réseau exigent une validation dans un environnement Azure représentatif.

Réduire les écarts importants entre les environnements aide à détecter plus tôt les défauts qui apparaîtraient autrement au déploiement. Reproduire intégralement la production sur chaque poste resterait coûteux, et certaines contraintes de réseau ou d’identité ne peuvent pas y être reproduites fidèlement. Il faut donc décider où valider chaque propriété importante plutôt que viser une identité complète.

La parité dépend des risques. Un développeur n’a pas besoin du volume de données de production, mais les tests de performance doivent utiliser une distribution et une charge représentatives. Les différences restantes doivent être connues, et non découvertes pendant un incident. Les secrets et les données sensibles de production n’ont pas à être copiés pour obtenir cette représentativité.

## 11. Logs : confier la collecte à l’environnement d’exécution

Le facteur [Logs](https://12factor.net/logs) considère les journaux comme un flux d’événements. L’application émet les informations. L’environnement les collecte, les conserve et les rend consultables. Le texte original privilégie la sortie standard et ne prévoit pas la gestion de fichiers de logs par l’application.

Dans .NET, `ILogger` permet d’émettre des événements avec des propriétés structurées. La conservation de cette structure dépend ensuite des fournisseurs et de leur configuration. Dans un environnement conteneurisé, un collecteur peut récupérer la sortie console sans que chaque instance gère ses propres fichiers durables.

Les [métriques et traces avec OpenTelemetry](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/observability-with-otel) complètent cette approche. Elles constituent un enrichissement moderne, car le facteur original ne définit pas un système complet d’observabilité. L’export direct vers un collecteur est aussi une adaptation dont il faut examiner le comportement en cas de panne.

Une collecte et une conservation correctement configurées rendent les journaux consultables après la disparition d’une instance et permettent de les rapprocher de ceux des autres composants. Ce diagnostic dépend de la qualité de l’instrumentation et de la fiabilité de la chaîne de collecte. Il faut aussi maîtriser les volumes, la cardinalité, la rétention et les données sensibles pour que l’observabilité reste exploitable et financièrement soutenable.

La saturation du collecteur ne devrait pas bloquer indéfiniment une requête métier. Il faut connaître les limites des tampons et les possibilités de perte. Un journal technique soumis à cette politique ne remplace pas automatiquement une piste d’audit dont la conservation doit être garantie.

## 12. Admin processes : encadrer les opérations ponctuelles

Le facteur [Admin processes](https://12factor.net/admin-processes) demande que les tâches administratives ponctuelles utilisent le code de la release et un environnement cohérent avec l’application. Il vise notamment les migrations et les interventions sur les données.

Une migration exécutée depuis le poste d’un développeur avec une version différente du produit est difficile à reproduire et à auditer. Un job contrôlé, associé à la release, identifie le programme exécuté, ses paramètres et son résultat.

Avec EF Core, les [scripts de migration et les bundles](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/applying) peuvent soutenir cette démarche. Le choix dépend des exigences de revue et de déploiement de l’organisation.

L’accès aux mêmes ressources ne signifie pas que l’application et le job doivent disposer des mêmes privilèges. Une API qui traite des commandes n’a pas nécessairement besoin du droit de modifier le schéma SQL.

Associer les interventions à une release facilite leur reproduction et leur traçabilité. Cela aide aussi l’équipe à préparer une reprise. Une opération versionnée n’est toutefois pas automatiquement rejouable sans risque. Ses effets sur les données, les exécutions concurrentes et les reprises après interruption doivent être conçus selon l’intervention.

Une migration longue ou destructive ne devient pas sûre parce qu’elle est exécutée dans un job. Lors d’un déploiement progressif, anciennes et nouvelles instances peuvent coexister. Faire évoluer le schéma par ajouts compatibles, migrer les données puis retirer l’ancien modèle réduit ce risque. Les opérations destructrices exigent toujours une stratégie de sauvegarde et de récupération adaptée.

## Ce que ces principes apportent ensemble

Les facteurs se renforcent mutuellement. Un processus remplaçable facilite la mise à l’échelle. Une configuration externe permet de promouvoir le même artefact. Des journaux centralisés rendent les instances éphémères observables.

Chaque déplacement de responsabilité crée aussi une dépendance à exploiter :

| Choix | Capacité obtenue | Responsabilité supplémentaire |
|---|---|---|
| Externaliser l’état durable | Remplacer les instances sans perdre cet état. | Disponibilité, sauvegarde et capacité du stockage. |
| Injecter la configuration | Déployer sans recompiler pour chaque environnement. | Validation, traçabilité et rafraîchissement des paramètres. |
| Multiplier les processus | Ajuster la capacité par rôle. | Concurrence, saturation en aval et coût. |
| Promouvoir un artefact identifié | Réutiliser la construction validée. | Compatibilité des données et de la configuration. |
| Centraliser les signaux | Diagnostiquer un système réparti. | Collecte, rétention, confidentialité et budget. |

La méthode clarifie certaines opérations en rendant leurs conditions explicites. Elle ne supprime pas la complexité des ressources auxquelles l’application confie son état et ses services.

## Les limites à garder en tête

Twelve-Factor ne garantit pas la cohérence d’une opération entre plusieurs services. Une commande enregistrée suivie d’un message perdu reste un problème de double écriture, même si les deux processus respectent les douze facteurs.

La méthode ne définit pas non plus une stratégie complète de sécurité, de reprise après sinistre ou de gouvernance. L’identité, les autorisations, l’isolation réseau, les sauvegardes et les objectifs de récupération doivent être conçus explicitement.

La portabilité d’un processus ne signifie pas que les fournisseurs sont interchangeables. Une application peut être facile à redéployer tout en utilisant volontairement des capacités spécifiques d’Azure SQL ou de Service Bus. Cet engagement peut être raisonnable s’il est compris et documenté.

Certains workloads ont aussi des contraintes particulières, comme une forte localité des données, des connexions persistantes ou un état de calcul difficile à déplacer. On peut appliquer plusieurs principes sans prétendre respecter entièrement le modèle d’un processus sans état local durable.

## Une adoption progressive dans une équipe .NET

Je commencerais par les difficultés observables du produit :

- Identifier la version exécutée en reliant le code, l’artefact, la configuration et le déploiement.
- Supprimer les dépendances implicites en repartant d’un environnement neuf et en vérifiant le démarrage.
- Tester la perte d’une instance et repérer les données perdues, les sessions invalidées et les traitements interrompus.
- Tester plusieurs instances pour rechercher les tâches dupliquées, la contention et les limites des ressources partagées.
- Vérifier le remplacement en cours de charge, notamment le routage, l’arrêt et la reprise des traitements.
- Encadrer les interventions sur les données en associant les migrations et les réparations à une procédure versionnée.

Cette démarche peut être menée sur une application App Service existante. Elle n’exige pas de commencer par une migration vers un orchestrateur. Le choix d’hébergement dépend des besoins de contrôle, de coût et d’exploitation du produit.

## Conclusion

Les douze facteurs posent des questions concrètes sur le fonctionnement d’une application : d’où vient-elle, comment reçoit-elle sa configuration, où conserve-t-elle son état et que se passe-t-il lorsqu’une instance disparaît?

Dans .NET et Azure, les mécanismes nécessaires sont largement disponibles. Le travail d’architecture consiste à les relier à des garanties vérifiables et à maîtriser les coûts qu’ils entraînent : échanges réseau, concurrence, compatibilité et exploitation des ressources externes.

J’utiliserais Twelve-Factor comme grille de revue. Une application bien conçue peut expliquer ses choix et ses exceptions, avec des preuves de fonctionnement en déploiement, en charge et en situation de panne.
