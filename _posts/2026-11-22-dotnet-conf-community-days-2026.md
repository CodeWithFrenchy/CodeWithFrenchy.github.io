---
title: Ce que les .NET Conf Community Days 2026 nous apprennent
date: 2026-11-22 18:00:00 -0400
categories: [dotnet, developpement]
tags: [dotnet-conf, dotnet-11, csharp-15, azure, architecture-logicielle, tests, intelligence-artificielle]
---

Après les annonces des équipes produit, les .NET Conf Community Days donnent la parole aux développeurs qui appliquent ces technologies dans leurs projets. Les journées des 12 et 13 novembre 2026 ont réuni des présentations sur l’architecture, l’observabilité, les tests, l’infrastructure et l’intelligence artificielle. Le [programme officiel](https://www.dotnetconf.net/agenda) et les [rediffusions des conférences](https://www.youtube.com/@dotnet) permettent de retrouver les sujets dans leur contexte.

Les notes de plusieurs présentations font ressortir des thèmes communs. Les nouveautés de .NET 11 et de C# 15 intéressent les équipes, mais elles s’inscrivent dans des enjeux plus larges : faire évoluer des applications existantes, produire une télémétrie exploitable, limiter la complexité opérationnelle et garder la maîtrise du code généré avec l’aide de l’IA.

## L’IA accélère le travail, mais le contexte reste déterminant

Plusieurs conférences abordent l’IA sous des angles complémentaires. Dans [Developer Workflows in AI Age: Local or Cloud](https://www.youtube.com/watch?v=k5NgrQDCdKE), Sam Basu présente les choix qui structurent un flux de développement assisté : modèle local ou infonuagique, environnement de travail, instructions, compétences spécialisées, outils MCP et délégation à des sous-agents.

La démonstration insiste sur l’importance de fournir aux agents le contexte propre au projet et à l’équipe. Des fichiers d’instructions concis, des compétences qui décrivent une manière de travailler et des outils qui donnent accès aux ressources pertinentes aident à orienter les résultats. L’agent peut aussi déléguer une tâche bien délimitée à un autre agent. Le développeur conserve toutefois la responsabilité de vérifier le résultat et de décider ce qui entre dans le produit.

Cette question du contrôle revient dans [Building a multi agent solution using Aspire](https://www.youtube.com/watch?v=NSWn3c2IFB0), qui décrit un prototype de tests d’intrusion piloté par plusieurs agents. Les agents organisent les étapes et analysent les résultats, tandis que des outils déterministes exécutent les actions sensibles. La présentation évoque notamment la limitation des cibles autorisées, le contrôle des coûts, la journalisation et un mécanisme d’arrêt. Ces choix illustrent une règle utile pour tout système agentique : limiter les actions possibles et rendre leur exécution vérifiable.

[.NET AI Architecture and DevOps](https://www.youtube.com/watch?v=k_DUJR6Zhrc) élargit la discussion aux applications qui intègrent des modèles de langage. Il faut alors choisir où situer le modèle dans l’architecture et prévoir la gestion des changements dans les flux de développement : contrôle de version, compilation, tests et déploiement. L’IA ne se limite donc pas à un outil d’édition de code. Elle peut devenir une dépendance de l’application, ce qui impose de traiter son intégration comme une décision d’architecture et d’exploitation.

Les trois présentations convergent sur un point : donner davantage d’autonomie à un agent ne dispense pas de définir les limites, les responsabilités et les contrôles. Les agents peuvent contribuer à la réalisation, mais l’équipe doit pouvoir comprendre et valider les résultats.

## Moderniser sans réécrire d’un seul coup

La migration d’une application existante vers une nouvelle interface revient dans [Lessons from Blazor WASM in Production](https://www.youtube.com/watch?v=uUJztTsg-SY). L’expérience présentée porte sur une interface WebForms complexe remplacée progressivement par Blazor WebAssembly. L’équipe a séparé les responsabilités entre interface, services et composants communs, puis maintenu temporairement certaines pages WebForms dans une application Blazor.

Cette étape de transition s’appuyait notamment sur YARP comme passerelle, sur une architecture d’authentification partagée et sur des messages JavaScript entre les pages existantes et les composants Blazor. La migration a aussi demandé de repenser la gestion de l’état et la cadence des mises à jour de l’interface.

La présentation rappelle que les performances dépendent du profil réel de l’application. La compilation AOT a apporté des gains dans les scénarios montrés, particulièrement pour les traitements intensifs dans le navigateur. Elle a aussi augmenté le temps de publication et les artefacts produits. L’équipe a dû tenir compte des environnements clients où certaines instructions SIMD n’étaient pas disponibles. Pour une application de grande taille, ces coûts influencent les pipelines et la stratégie de livraison.

Une autre présentation, [Miniservices not Microservices in Azure](https://www.youtube.com/watch?v=9UbzM5WbZkk), pose une question d’échelle. Une petite ou moyenne application n’a pas nécessairement besoin d’un environnement Kubernetes ni d’une architecture composée de nombreux microservices. Les services gérés d’Azure, comme App Service, Azure Functions, les bases de données et Service Bus, peuvent fournir une architecture plus simple à exploiter.

Le terme « miniservices » exprime ici une recherche de juste mesure : découper les responsabilités lorsque cela aide l’équipe, sans ajouter une infrastructure distribuée dont les coûts d’exploitation dépasseraient les bénéfices. La taille d’un système devrait guider le niveau de complexité qu’on lui impose.

## L’infrastructure comme produit partagé

Deux conférences abordent la conception et le partage de l’infrastructure.

Dans [Infrastructure as (Actual) Code with C# 15 and .NET 11](https://www.youtube.com/watch?v=gOtjSXdWxAI), l’intervenant présente Pulumi et son SDK .NET pour définir des ressources Azure en C#. Les équipes peuvent exprimer leur infrastructure dans un langage qu’elles connaissent déjà, utiliser les types et le compilateur, puis déployer par les outils de Pulumi. La [documentation officielle de Pulumi](https://www.pulumi.com/docs/iac/languages-sdks/dotnet/) décrit les langages et les SDK pris en charge.

La présentation relie cette approche à l’usage de l’IA pour produire du code d’infrastructure. Une instruction dans un fichier de consignes peut guider un agent, mais le système de types et les validations du compilateur permettent d’intercepter certaines erreurs avant le déploiement. Cela ne remplace pas la revue des changements, la gestion des secrets ou les politiques d’organisation. C’est une couche de contrôle supplémentaire dans le processus.

[ IaC Forged in Code: Building Resilient Infrastructure with Modules](https://www.youtube.com/watch?v=okZ_ybEqXPY) se concentre sur les modules partagés avec Bicep et Terraform. Une équipe plateforme peut y centraliser des conventions de nommage, des étiquettes, des politiques et des configurations communes. La version et la distribution des modules deviennent alors des décisions d’architecture : les équipes consommatrices doivent savoir ce qui change et quand adopter une nouvelle version.

Ces deux approches répondent à des besoins différents. Décrire les ressources en C# rapproche l’infrastructure du code applicatif. Les modules Bicep ou Terraform facilitent la réutilisation des patrons d’infrastructure entre équipes. Dans les deux cas, la valeur dépend de la qualité des interfaces, du versionnement et de la gouvernance autour des composants partagés.

## Observabilité commune aux applications .NET et polyglottes

[ASP.NET Core 11 loves OpenTelemetry](https://www.youtube.com/watch?v=3mAhtBjBW88) présente l’intérêt des conventions sémantiques d’OpenTelemetry pour les métriques et les traces HTTP. ASP.NET Core 11 émet désormais des données qui suivent les parties requises des conventions OpenTelemetry pour les serveurs HTTP, comme le précisent la [documentation Microsoft sur les métriques ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/metrics/http) et la [spécification OpenTelemetry pour HTTP](https://opentelemetry.io/docs/specs/semconv/http/).

La conférence montre un parcours réparti entre plusieurs services écrits en .NET, Java, Node.js, Python et Go. Lorsque les services produisent une télémétrie conforme à des conventions communes, les équipes peuvent réutiliser des requêtes, des tableaux de bord et des alertes sur plusieurs langages. Les traces rendent aussi visibles les appels qui composent une opération, ce qui facilite l’analyse d’une requête lente ou en échec.

La présentation distingue la télémétrie des audits métier. Les traces décrivent le chemin technique d’une requête. Un audit métier doit conserver les étapes et les décisions nécessaires pour comprendre ce qui est arrivé à une transaction. Les deux données répondent à des questions différentes et ne devraient pas être confondues.

L’interopérabilité ne vient pas uniquement de l’installation d’un SDK. Les équipes doivent choisir les signaux, les attributs, les conventions et les règles de validation qu’elles veulent maintenir. La conférence présente également Weaver, un outil du projet OpenTelemetry qui peut vérifier si les applications émettent les données attendues.

## C# 15, .NET 11 et la modélisation du domaine

Avec la sortie de [.NET 11](https://devblogs.microsoft.com/dotnet/announcing-dotnet-11/) le 10 novembre 2026, plusieurs sessions ont exploré les nouveautés du langage et de la plateforme. C# 15 est la version par défaut pour les projets ciblant .NET 11. Les types union permettent de représenter une valeur qui peut prendre l’une de plusieurs formes déclarées, tandis que les hiérarchies fermées rendent possible une vérification plus exhaustive de certains cas par le compilateur. La [documentation de C# 15](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-15) décrit ces fonctionnalités.

[Meeple-Driven Design: Modeling Board Game Domains with C# 15 & .NET 11](https://www.youtube.com/watch?v=l58PiFWCxA4) utilise des règles de jeux de société pour illustrer leur intérêt. Un tour peut, par exemple, correspondre à une action parmi plusieurs possibilités. Exprimer explicitement ces formes aide à éviter les valeurs textuelles imprécises, les objets remplis de propriétés nulles et les conversions risquées.

La session aborde aussi la sérialisation de ces états avec System.Text.Json et l’indexation de types complexes avec EF Core 11. Ces exemples montrent comment les choix de modélisation peuvent se poursuivre jusqu’aux échanges JSON et à la persistance. Une meilleure représentation dans le code ne suffit toutefois pas à trancher les questions de compatibilité ou de forme du schéma : ces aspects restent à valider selon les contrats de l’application.

La nouveauté la plus utile n’est pas toujours celle qui réduit le nombre de lignes de code. Les unions et les hiérarchies fermées peuvent rendre les états possibles plus visibles et aider le compilateur à repérer les cas oubliés. Leur pertinence dépend de la stabilité du modèle et de la manière dont les autres systèmes consomment ces données.

## Des stratégies de test qui vérifient des risques différents

[When Unit Tests Are Not Enough: Property-based Testing and Snapshot Tests](https://www.youtube.com/watch?v=Yd_fI485naI) compare trois façons de vérifier un comportement : les tests basés sur des exemples, les tests de propriétés et les tests instantanés.

Un test basé sur un exemple vérifie un scénario précis. Un test de propriétés exprime une règle qui doit tenir pour une variété d’entrées générées. Dans la démonstration, une propriété vérifie qu’un mélange de cartes conserve les éléments et leur multiplicité, même si leur ordre change. FsCheck génère des entrées et peut réduire un cas en échec à un exemple plus petit, ce qui aide à comprendre la violation.

Un test instantané conserve une représentation approuvée d’une sortie, comme un document sérialisé, puis signale les différences lors des exécutions suivantes. Le développeur doit examiner ces changements : l’instantané rend l’évolution visible, mais ne détermine pas si elle est correcte. Si la compatibilité avec d’anciens fichiers est requise, il faut aussi tester directement leur lecture.

La conférence donne ainsi un critère de choix plus concret que le nombre de tests ou le taux de couverture : quel risque le test doit-il détecter? Avec xUnit, FsCheck et Verify, les équipes peuvent combiner des scénarios lisibles, des invariants généraux et des comparaisons de sorties lorsque ces approches correspondent aux risques du système.

## Architecture propre, tranches verticales et modules

[Clean Architecture AND MORE with ASP.NET Core](https://www.youtube.com/watch?v=HK3IwBezg0w) revient sur trois notions souvent combinées : Clean Architecture, tranches verticales et modularité.

Clean Architecture aide à limiter le couplage entre le domaine et les détails d’infrastructure. Les tranches verticales regroupent le code par fonctionnalité afin de faciliter les changements livrés ensemble. Mais aucune organisation de dossiers ne garantit, à elle seule, que les modules respectent leurs responsabilités. Une application peut encore devenir difficile à faire évoluer si ses frontières sont seulement conventionnelles.

La présentation recommande de renforcer ces frontières avec une architecture modulaire encapsulée. Chaque module devrait contrôler ses dépendances et la manière dont les autres parties de l’application accèdent à ses capacités. Cette idée complète les choix de découpage : l’organisation du code doit refléter des règles vérifiables, pas seulement une préférence de classement.

## Les sujets à retenir

Les Community Days 2026 donnent moins une recette unique qu’un ensemble de pistes à confronter aux systèmes existants. Plusieurs idées se dégagent :

- L’IA devient plus utile lorsque les agents disposent du contexte pertinent, d’outils contrôlés et de validations adaptées.
- La modernisation progressive peut réduire le risque de migration, à condition de planifier les frontières temporaires et leur retrait.
- L’infrastructure partagée demande des interfaces et un versionnement soignés, qu’elle soit définie en C# ou sous forme de modules.
- Des conventions d’observabilité communes facilitent l’exploitation des systèmes polyglottes.
- Les nouveautés de C# 15 peuvent rendre certains modèles plus explicites, mais leur adoption doit tenir compte des contrats externes et des données persistées.
- Les tests sont plus efficaces lorsqu’ils expriment clairement le risque qu’ils couvrent.

Ces thèmes complètent les annonces produit par des retours d’expérience et des démonstrations. Pour une équipe, le meilleur point de départ est souvent un changement ciblé : une migration qui peut être découpée, une convention de télémétrie à uniformiser ou un invariant métier que les tests actuels n’expriment pas encore. Les conférences permettent ensuite de comparer ces pistes aux contraintes concrètes du projet.
