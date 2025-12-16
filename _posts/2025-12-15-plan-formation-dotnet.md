---
title: Plan de formation - .NET Framework vers .NET moderne
date: 2025-12-15 18:00:00 -0400
categories: []
tags: [dotnet]
---

> 💡 Cet article marque le dernier de l’année 2025 : je prends maintenant un congé bien mérité pour la période des fêtes et je vous invite à en faire autant. On se retrouve le 12 janvier prochain pour le prochain article.

## Préambule

Passer de **.NET Framework** à **.NET moderne** représente une étape importante pour tout développeur souhaitant rester à jour dans l’écosystème Microsoft. Ce **plan de formation** a été conçu pour accompagner cette transition de manière progressive et concrète.

L’objectif est d’aider les développeurs et architectes à :

- Comprendre les différences fondamentales entre **.NET Framework** et **.NET (Core)** ;
- Explorer les **nouvelles versions** de .NET, leurs cycles de support (LTS / STS) et leurs améliorations ;
- Découvrir les **nouveaux outils et pratiques** (configuration moderne, DI intégrée, hosting model, etc.) ;
- Se familiariser avec les **technologies clés** de l’écosystème actuel : **ASP.NET Core**, **Entity Framework Core**, **TestContainers**, **Aspire**, **GitHub Copilot**, et plus encore.

**Qu'est-ce que .NET ?** Microsoft .NET est une plateforme de développement offrant un écosystème riche pour créer des applications de tous types (web, bureau, mobiles, services, etc.). Historiquement, on distinguait **.NET Framework** (lancé en 2002) et **.NET Core** (lancé en 2016). Aujourd'hui, le terme **« .NET »** désigne principalement la version moderne issue de .NET Core, unifiée à partir de .NET 5 en 2020. À l'inverse, **.NET Framework 4.x** est l'ancienne version « legacy », désormais figée (la 4.8 étant la dernière). 

Quelles sont les grandes différences entre les deux ?

- **Multiplateforme vs Windows uniquement :** .NET moderne est **cross-platform** : une application .NET peut s'exécuter sur Windows, Linux ou macOS avec les mêmes performances. Le développement n'est plus limité à Windows : on peut coder sous Visual Studio Code ou d'autres éditeurs sur Linux/macOS, en plus de Visual Studio sous Windows. En comparaison, .NET Framework était uniquement compatible Windows.
- **Open source et innovation :** Le nouveau .NET est open source et évolue rapidement. La communauté peut contribuer aux améliorations du runtime et des bibliothèques via GitHub. .NET Framework, lui, était en grande partie fermé et évolue très peu depuis 2019.
- **Performance et scalabilité :** .NET (Core) a été repensé pour être **beaucoup plus performant**. Le runtime optimisé et l'environnement d'exécution (CLR) font d'ASP.NET Core l'un des frameworks web les plus rapides selon TechEmpower. À l'inverse, .NET Framework est plus lourd et moins optimisé pour les scénarios modernes (microservices, conteneurs, etc.).
- **Modularité et déploiement :** .NET Core introduit une architecture modulaire (via NuGet) et permet le déploiement _auto‐contenu_ (les runtimes nécessaires peuvent être embarqués avec l'application). .NET Framework, en revanche, est installé au niveau du système et partagé par toutes les applications, ce qui rend les mises à jour délicates. De plus, .NET moderne autorise d'installer **plusieurs versions en parallèle** sur la même machine (side-by-side), chaque application peut cibler sa version de runtime sans conflit. Avec .NET Framework, une seule version 4.x est active à la fois sur Windows, limitant la flexibilité.
- **APIs et fonctionnalités disponibles :** .NET Core a rattrapé et même dépassé .NET Framework sur la plupart des fonctionnalités. Certains anciens modules (Web Forms, WCF, Workflow…) ne sont disponibles que sous .NET Framework, mais ils ont souvent des alternatives modernes (par ex. gRPC à la place de WCF). Le nouveau .NET inclut également de nouveaux frameworks (ASP.NET Core, Blazor, MAUI pour le multiplateforme, etc.) non présents dans l'ancien.

En résumé, **.NET « Core » unifié est l'avenir de l'écosystème .NET**, idéal pour tout nouveau projet, tandis que .NET Framework sert uniquement à maintenir d'anciennes applications sur Windows.

**À écouter :**

<iframe width="740" height="473" src="https://www.youtube.com/embed/X75vbT-Yv-c?si=zTln5vaEec8HLcts" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Évolution des versions .NET 6 à .NET 10

Pour chaque version majeure de .NET (de la version 6 à la version 10), vous trouverez ci-dessous au moins une vidéo explicative des nouveautés ainsi que l'article détaillé de [Stephen Toub](https://x.com/stephentoub) sur les améliorations techniques (notamment de performance) de cette version.

**.NET 6 (2021 - 2024)**

- **Vidéos :** _What's New in .NET 6 and C# 10?_ - [.NET 6 deep dive](https://www.youtube.com/watch?v=GJ_PaRNDe9E) et [What's new in C# 10](https://www.youtube.com/watch?v=dfzBMxXQUOc)
- **Article :** _Performance Improvements in .NET 6_ - [Performance Improvements in .NET 6 - .NET Blog](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-6/)

**.NET 7 (2022 - 2024)**

- **Vidéos :** _What's New in .NET 7 C# 11?_ - [.NET 7 Overview](https://www.youtube.com/watch?v=9NqthBLHBDg) et [What's New in C# 11](https://www.youtube.com/watch?v=H18CfoinPZg)
- **Article :** _Performance Improvements in .NET 7_ - [Performance Improvements in .NET 7 - .NET Blog](https://devblogs.microsoft.com/dotnet/performance_improvements_in_net_7/)

**.NET 8 (2023 - 2026)**

- **Vidéos :** _What's New in .NET 8 C# 12?_ - [What's new in .NET 8](https://www.youtube.com/watch?v=pJGDPEk45Jc) et [Every New Feature Added in C# 1](https://www.youtube.com/watch?v=Gv2uBJzBAms).
- **Article :** _Performance Improvements in .NET 8_ - [Performance Improvements in .NET 8 - .NET Blog](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-8/).

**.NET 9 (2024 - 2026)**

- **Vidéos :** _What's New in .NET 9 C# 13?_ - [What's new in .NET 9 & C# 13](https://www.youtube.com/watch?v=snPgTcxH8-s) et [What's New in .NET 9](https://www.youtube.com/watch?v=PvB5jtA-QfM).
- **Article :** _Performance Improvements in .NET 9_ - [Performance Improvements in .NET 9](https://www.youtube.com/watch?v=aLQpnpSxosg) et [Performance Improvements in .NET 9 - .NET Blog](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-9/).

**.NET 10 (2025 - 2028)**

- **Vidéo :** _What's New in .NET 10 C# 14?_ - [csproj is GONE! 'dotnet run app.cs' is Here](https://www.youtube.com/watch?v=j4tLg4bMZK4&list=PLUOequmGnXxMQy-vu2O5VUFAdhu1fE5bY)
- **Article :** _Performance Improvements in .NET 10_ - [Performance Improvements in .NET 10 - .NET Blog](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-10/)

## Le cycle de support de LTS versus STS avec .NET Core

Depuis .NET Core 3.1, Microsoft suit un cycle de publication fixe pour .NET avec une alternance entre versions **LTS (Long-Term Support)** et **STS (Standard-Term Support)**.

**LTS - Long-Term Support**

- Support officiel de **3 ans**
- Idéal pour les applications critiques ou à cycle de vie long
- Mises à jour de sécurité et corrections sans changements de rupture
- Exemples : .NET 6 (LTS), .NET 8 (LTS), .NET 10 (LTS)  

**STS - Standard-Term Support**

- Support étendu à **24 mois** depuis .NET 9 (**contre 18 mois auparavant**)
- Accès anticipé aux nouveautés du runtime et du langage
- Adapté aux projets agiles pouvant adopter un rythme de mise à jour régulier
- Exemples : .NET 7 (**18 mois**), .NET 9 (**24 mois**)

**Vidéo explicative recommandée :**

<iframe width="740" height="473" src="https://www.youtube.com/embed/3i6YASbyuHw?si=htCjY6ZrAvQvMLk5" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Différences entre les outils de l'écosystème .Net Framework à .NET Core

Le passage de .NET Framework à .NET Core s'accompagne de nombreux changements d'outils, de conventions et de choix technologiques par défaut.

Voici un aperçu des évolutions les plus notables :

**Remplacements technologiques fréquents**

| **Outil / Approche (Framework)** | **Évolution / Remplacement en .NET Core+** | **Commentaire** |
| --- | --- | --- |
| **Autofac (DI)** | DI natif intégré à ASP.NET Core | Autofac est toujours compatible, mais plus nécessaire pour la plupart des cas |
| **Web.config, app.config** | appsettings.json + configuration par injection | Plus flexible, hiérarchique, supporte IOptions |
| **System.Web** | Supprimé, remplacé par ASP.NET Core Hosting & Middleware | Architecture légère, cross-platform |
| **Global.asax** | Remplacé par `Program.cs` et Startup.cs ou minimal hosting | Démarrage centralisé simplifié |
| **IIS exclusivement** | Cross-platform (Kestrel, Nginx, Apache, etc.) | Kestrel est souvent utilisé en production |
| **csproj verbeux** | Nouveau format SDK-style (`<Project Sdk="...">`) | Plus simple, support multi-targeting |
| **MSBuild-based NuGet restore** | dotnet restore intégré au build | Plus rapide, CLI unifiée |
| **Project.json** (obsolète depuis Core 1.x) | Fusionné dans le nouveau format csproj moderne | Stabilisation du modèle de projet |

**Changements fréquents dans l'écosystème .NET Core et modernes**

Même entre les versions de .NET Core, certains outils ou comportements évoluent :

- **Swagger / Swashbuckle** : était activé par défaut dans certains templates .NET 5/.NET 6 (notamment dans les APIs), **ce n'est plus le cas en .NET 9/10**. Il faut l'ajouter et le configurer manuellement.
- **Hosting Model** : depuis .NET 6, le modèle minimal de démarrage (`Program.cs` simplifié) est devenu la norme, remplaçant la combinaison Startup.cs + `Program.cs`.
- **HTTP Logging natif** : ajouté dans .NET 6, souvent oublié.
- **Prise en charge des tests avec dotnet test** : meilleure intégration CLI, y compris pour les tests en parallèle.
- **Nouvelle manière de gérer les secrets utilisateurs** : dotnet user-secrets introduit depuis .NET Core 2, mais plus encouragé avec les secrets Azure pour les apps cloud.

**Changement de licences dans l'écosystème .NET**

Un point important à connaître : Microsoft et d'autres éditeurs ont **modifié les licences de nombreux packages dans l'écosystème .NET**, ce qui a un impact sur :

- L'utilisation commerciale de certains outils open source
- La compatibilité avec certaines entreprises ou environnements sensibles
- Les pratiques de contribution ou de fork

**Article recommandé :** <https://codewithfrenchy.com/posts/changement-de-licence-ecosysteme-dotnet/>

## Développement web avec ASP.NET Core

ASP.NET Core est la plateforme web principale de .NET moderne. Elle permet de créer des applications performantes, testables, sécurisées et facilement déployables, grâce à une architecture modulaire, multiplateforme et cloud-ready.

**Vidéo recommandée :**

<iframe width="740" height="473" src="https://www.youtube.com/embed/NbfhbDKiFpM?si=pl1vqM9-F40sgi6X" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Accès aux données avec Entity Framework Core

Entity Framework Core (EF Core) est l'ORM moderne de Microsoft pour .NET. Il remplace EF6 dans l'écosystème .NET Core et s'intègre naturellement à l'injection de dépendances, à la configuration moderne et aux API REST.

**Vidéo recommandée :**

<iframe width="740" height="473" src="https://www.youtube.com/embed/2t88FOeQ898?si=KPUxyjeQQAEvzrBC" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Migration d'une application ASP.NET MVC (Framework) vers ASP.NET Core

Migrer une application .NET Framework à .NET Core est un projet structurant. Il ne s'agit pas d'un simple "portage", mais bien d'une **reconstruction partielle** dans un écosystème différent, plus modulaire, plus performant et multiplateforme.

**Playlist recommandée :** <https://www.youtube.com/watch?v=zHgYDZK3MrA&list=PLdo4fOcmZ0oWiK8r9OkJM3MUUL7_bOT9z>

## Déploiement local et dans le cloud (Azure App Services)

Déployer une application ASP.NET Core en 2025 peut se faire très facilement, aussi bien localement qu'en environnement cloud. Comprendre les options disponibles et les outils recommandés est essentiel pour industrialiser son projet.

**Vidéo recommandée :**

<iframe width="740" height="473" src="https://www.youtube.com/embed/dLuUJcxFcxU?si=bwI73AASTf5t9RGP" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Bonus

### Structure moderne des solutions

Passer à .NET 6 jusqu'à .NET 10 implique d'adopter de nouvelles conventions de structure et d'organisation du code. Une architecture propre facilite la maintenabilité, les tests, la scalabilité et les bonnes pratiques DevOps.

**Vidéo recommandée :**

<iframe width="740" height="473" src="https://www.youtube.com/embed/QRgtcbxJlo0?si=kUiNNBljWQ6TxHWJ" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

### Roadmap

L'écosystème .NET évolue rapidement, notamment avec .NET 8/9/10, les microservices, le cloud et la montée en puissance de la performance et de la productivité. En 2025, un développeur .NET moderne devrait idéalement maîtriser plusieurs axes, à la fois techniques et méthodologiques.

**Vidéo recommandée :**

<iframe width="740" height="473" src="https://www.youtube.com/embed/4I07X_EGwTY?si=PRQZdV4lo5mkTOi-" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

### Moderniser une application .NET avec GitHub Copilot

GitHub Copilot, basé sur l'IA, peut assister les développeurs dans la **modernisation d'une application existante** en l'accélérant.

**Vidéo recommandée :**

<iframe width="740" height="473" src="https://www.youtube.com/embed/-YKguff5GY8?si=JGVjziProb6MIYJ3" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

### Faciliter les essais

Dans les versions récentes de .NET, Microsoft a introduit plusieurs outils pour **faciliter les tests automatisés** sans avoir besoin de dépendances tierces.  

Deux nouveautés très utiles sont :

- `TimeProvider` / `FakeTimeProvider`
- `ILogger<T>` / `FakeLogger<T>`

**Articles recommandés :** <https://grantwinney.com/how-to-use-timeprovider-and-faketimeprovider/> et <https://www.freecodecamp.org/news/how-to-use-fakelogger-to-make-testing-easier-in-net>

### Tester contre des conteneurs réels avec les TestContainers

**TestContainers** est une bibliothèque .NET qui permet de lancer des services Docker (bases de données, RabbitMQ, Redis, etc.) dans un environnement de test, à la volée. Elle garantit des tests **réalistes**, **isolés** et **reproductibles**.

C'est une solution idéale pour remplacer :

- Les scripts SQL manuels d'initialisation
- Les bases de données en dur sur la machine du développeur
- Les mocks peu fiables de services externes

**Article recommandé :** <https://codewithfrenchy.com/posts/introduction-testcontainers/>

### Aspire : orchestrer des applications

**Aspire** est une **stack d'orchestration** proposée par Microsoft, conçue pour faciliter le développement, la configuration, l'observation et le déploiement d'**applications cloud natives composées de plusieurs projets ou services**.

C'est une réponse directe aux défis rencontrés avec les microservices, les BFF, les APIs, les workers, etc.

**Playlist :** [Migrating From Docker Compose to Aspire (my experience) - YouTube](https://www.youtube.com/watch?v=-73fAqw8ckU&list=PLYpjLpq5ZDGtGCUub_LP7_a3PRqzWPNoK)
