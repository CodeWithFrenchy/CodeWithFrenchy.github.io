---
title: NuGets recommandés pour le développement .NET
date: 2027-12-31 19:00:00 -0400
categories: [outil-developpement]
tags: [dotnet]
---

## Préambule

Ajouter un paquet NuGet prend quelques secondes. Comprendre ce qu’il apporte, ce qu’il impose et ce qu’il faudra entretenir demande beaucoup plus de réflexion.

Dans un projet .NET, les dépendances finissent rapidement par structurer une partie importante de notre façon de développer, de tester, de documenter et parfois même d’architecturer l’application. Le bon outil peut faire gagner énormément de temps. À l’inverse, une dépendance ajoutée par habitude peut créer du couplage, compliquer les mises à jour ou devenir difficile à remplacer quelques années plus tard.

Les changements de licences dans l’écosystème .NET nous l’ont rappelé. **Open source un jour != Open Source pour toujours.** J’en parle dans mon article sur les [changements de licences et leurs implications](https://codewithfrenchy.com/posts/changements-licences-ecosysteme-dotnet/).

Avant une adoption ou une mise à jour importante, je regarde donc le besoin réel, la maturité du projet, son coût d’intégration, ses dépendances et les contraintes associées à la version retenue. Une bibliothèque payante peut très bien justifier son coût. Une bibliothèque gratuite peut aussi nous coûter cher en complexité ou en maintenance.

Dans cet article, je présente dix outils que je trouve particulièrement intéressants pour le développement .NET. L’objectif n’est pas de constituer une liste de paquets à installer systématiquement, mais plutôt d’identifier les contextes où chacun peut réellement apporter de la valeur et les situations où je serais plus réservé.

## La sélection en un coup d’œil

| Outil | Besoin principal |
| --- | --- |
| xUnit.net | Écrire et exécuter des essais automatisés |
| NSubstitute | Remplacer certaines dépendances dans les essais |
| AutoFixture | Réduire la préparation des données d’essai |
| Testcontainers | Vérifier les intégrations avec de vrais services conteneurisés |
| ArchUnitNET | Vérifier automatiquement des règles d’architecture |
| FluentValidation | Organiser les règles de validation |
| Mapperly | Générer des conversions entre objets |
| Scrutor | Enregistrer et décorer des services |
| Scalar | Consulter la documentation d’une API et essayer ses opérations |
| BenchmarkDotNet | Mesurer et comparer les performances |

## 1. xUnit.net pour les essais automatisés

[xUnit.net](https://github.com/xunit/xunit) fournit une base pour écrire les essais et vérifier leurs résultats. Un `Fact` décrit un scénario, tandis qu’une `Theory` permet de rejouer un même comportement avec plusieurs jeux de données.

**Ce que j’apprécie** est la possibilité de commencer simplement. Les assertions natives couvrent déjà beaucoup de besoins. Une bibliothèque d’assertions supplémentaire peut se justifier, mais je commencerais par regarder ce qui manque réellement.

**La limite** se trouve surtout dans la manière de concevoir les essais. Un framework ne nous protège pas contre les scénarios trop larges, les dépendances partagées ou les assertions qui vérifient des détails internes sans intérêt pour le comportement attendu.

Je le retiendrais pour un nouveau projet. Pour une équipe déjà bien équipée avec NUnit ou MSTest, je chercherais un bénéfice concret avant de proposer une migration.

## 2. NSubstitute pour isoler une dépendance

[NSubstitute](https://github.com/nsubstitute/NSubstitute) permet de configurer le comportement d’une dépendance et de vérifier certaines interactions. C’est utile lorsqu’un essai doit contrôler la réponse d’un service externe ou vérifier qu’une notification a été demandée.

**Son avantage** est une syntaxe assez directe. On peut exprimer ce que la dépendance retourne et les appels attendus sans que la préparation occupe toute la place.

**Son piège** est de multiplier les substituts jusqu’à vérifier uniquement que les objets s’appellent dans un ordre précis. Ces essais deviennent sensibles aux refactorisations, même lorsque le comportement reste correct.

Je privilégierais les interfaces aux classes concrètes. La [documentation précise les limites liées aux membres non virtuels](https://nsubstitute.github.io/help/creating-a-substitute/), qui peuvent exécuter le vrai code. Les analyseurs de NSubstitute aident à repérer plusieurs mauvaises utilisations.

Je l’utiliserais aux frontières utiles du comportement testé, tout en gardant de vrais objets pour les collaborations simples.

## 3. AutoFixture pour alléger la préparation des essais

[AutoFixture](https://autofixture.com/docs/get-started/introduction/) crée des objets et remplit leurs données pour réduire le code de préparation. C’est particulièrement intéressant lorsqu’un scénario nécessite un objet avec plusieurs propriétés, mais que seulement deux influencent le comportement vérifié.

**Le gain** est de rendre ces deux propriétés visibles. Pour vérifier qu’une commande dépasse un seuil de livraison gratuite, je veux voir explicitement le montant de la commande et le seuil. Les autres renseignements peuvent être générés.

La [construction personnalisée avec `Build` et `With`](https://autofixture.com/docs/fundamentals/build-dsl/) permet justement de fixer les valeurs importantes.

**La limite** est qu’une donnée générée automatiquement n’est pas nécessairement valide pour notre domaine. Les règles entre plusieurs propriétés, les références circulaires et les constructeurs particuliers peuvent demander des personnalisations.

Je préfère AutoFixture lorsque l’objectif est de réduire la préparation répétitive. Les cas limites et les valeurs qui expliquent le scénario doivent rester explicites. Si la configuration de la fixture devient plus difficile à lire qu’une construction manuelle, le gain mérite d’être réévalué.

## 4. Testcontainers pour vérifier les intégrations

[Testcontainers pour .NET](https://dotnet.testcontainers.org/) permet de démarrer des services dans des conteneurs pour les besoins des essais, puis de gérer leur cycle de vie.

**L’avantage** est de vérifier une intégration avec un vrai moteur de base de données ou un vrai service. Une requête SQL, une contrainte ou une transaction peut se comporter différemment avec un substitut en mémoire. Ces différences sont précisément ce qu’on veut détecter.

**Le coût** est opérationnel. Il faut un environnement compatible avec l’API Docker, des ressources pour les conteneurs et une configuration adaptée dans la chaîne d’intégration continue. Le démarrage des services augmente aussi la durée des essais.

Je le retiendrais pour les comportements qui dépendent réellement de l’infrastructure. L’isolation des données entre essais demeure essentielle, même lorsqu’on partage un conteneur pour accélérer leur exécution. Un conteneur local ne reproduit pas non plus toutes les particularités d’un service managé en production.

## 5. ArchUnitNET pour rendre les règles d’architecture vérifiables

[ArchUnitNET](https://github.com/TNG/ArchUnitNET) analyse les types et leurs dépendances pour vérifier des règles d’architecture. On peut, par exemple, empêcher le domaine de dépendre de l’infrastructure.

**Son intérêt** est de transformer une intention documentée en une vérification exécutée avec les autres essais. Une nouvelle dépendance interdite devient visible dans la chaîne d’intégration, au moment où elle est introduite.

**La limite** est la qualité des règles choisies. Une collection de contraintes sur les noms ou les dossiers peut devenir encombrante si elle ne protège aucune décision importante. Ces vérifications ne remplacent pas la réflexion sur les responsabilités.

J’en parle plus en détail dans mon article sur les [essais d’architecture automatisés en .NET](https://codewithfrenchy.com/posts/essais-architecture-automatises-dotnet/).

Je commencerais avec quelques frontières qui comptent réellement pour le projet, puis j’ajouterais des règles en fonction des problèmes observés.

## 6. FluentValidation pour des validations qui se complexifient

[FluentValidation](https://github.com/FluentValidation/FluentValidation) permet de définir les règles dans des validateurs dédiés. C’est pratique lorsqu’elles deviennent conditionnelles, combinent plusieurs propriétés ou doivent être réutilisées.

**Le bénéfice** est de garder ces validations regroupées et faciles à essayer. Une demande de création peut ainsi avoir des règles différentes d’une demande de modification, sans accumuler tous les cas sur le même modèle.

**La contrepartie** est une abstraction et une dépendance supplémentaires. Pour quelques contraintes simples, les mécanismes déjà disponibles dans .NET peuvent suffire.

Les [règles asynchrones](https://docs.fluentvalidation.net/en/latest/async.html) demandent aussi une intégration adaptée. Un validateur qui en contient doit être appelé avec `ValidateAsync`. L’ancien mécanisme de validation automatique MVC, synchrone, ne convient pas à ce scénario.

Je le retiendrais lorsque les règles d’entrée gagnent en complexité. Les invariants métier doivent aussi être protégés là où le domaine est modifié, même si une validation préalable a déjà eu lieu.

## 7. Mapperly quand les conversions deviennent répétitives

[Mapperly](https://github.com/riok/mapperly) utilise un générateur de code pour produire les conversions entre objets à la compilation. Le code généré peut être inspecté, et plusieurs problèmes de correspondance sont signalés pendant la compilation.

**L’avantage** apparaît lorsqu’un projet contient beaucoup de conversions mécaniques entre des modèles et des DTO. Cela réduit une partie du travail répétitif sans introduire un moteur de mapping fondé sur la réflexion à l’exécution.

**La limite** est que les conventions restent à comprendre et à entretenir. Dès qu’une conversion contient une décision métier, des valeurs par défaut importantes ou des règles conditionnelles, sa lecture mérite une attention particulière.

Pour quelques conversions, une méthode écrite à la main peut être très claire. Je considérerais Mapperly lorsque le volume de mapping justifie l’outil.

Avec EF Core, je regarderais aussi la projection effectuée dans la requête. Charger des entités complètes pour ensuite conserver trois propriétés dans un DTO peut coûter davantage que la conversion elle-même.

## 8. Scrutor pour l’enregistrement des services et les décorateurs

[Scrutor](https://github.com/khellang/Scrutor) ajoute deux possibilités au conteneur d’injection de dépendances de Microsoft. Il peut parcourir des assemblies pour enregistrer des services selon des conventions, et il facilite l’application de décorateurs sur des services déjà enregistrés.

**Le premier gain** concerne les modules qui suivent des conventions stables. Au lieu de répéter de nombreux enregistrements similaires, on décrit les types concernés, les interfaces à exposer et leur durée de vie.

**Le second gain** est particulièrement intéressant pour ajouter un comportement autour d’un service. Un décorateur peut, par exemple, mesurer la durée d’une opération tout en déléguant son exécution au service existant.

**La limite** est la visibilité. Avec des filtres trop larges, il devient plus difficile de comprendre ce qui est enregistré. Une convention mal définie peut aussi sélectionner des classes inattendues ou appliquer une durée de vie inadaptée.

Je limiterais le parcours à des assemblies et à des familles de services bien identifiées. Pour quelques enregistrements, les appels explicites restent faciles à lire. Les décorateurs peuvent, à eux seuls, justifier Scrutor.

## 9. Scalar comme alternative à Swagger UI

[Scalar](https://scalar.com/products/api-references/integrations/aspnetcore/integration), avec le paquet `Scalar.AspNetCore`, fournit une interface pour lire la documentation d’une API et essayer ses opérations.

La distinction utile est celle-ci. **Scalar peut remplacer Swagger UI comme interface de documentation.** Le document OpenAPI doit toujours être produit. On peut le générer avec `Microsoft.AspNetCore.OpenApi`, Swashbuckle ou NSwag, puis le présenter avec Scalar.

**Ce qui m’intéresse** est l’expérience de consultation, la navigation entre les opérations et les exemples de requêtes. Pour une API destinée à d’autres équipes, une documentation agréable à parcourir peut réduire les échanges nécessaires pour commencer à l’utiliser.

**Sa limite** reste la qualité du document fourni. Des schémas incomplets, des réponses mal décrites ou une authentification absente de la définition OpenAPI donneront une documentation incomplète.

Je le considérerais pour un nouveau projet ou pour améliorer une documentation existante. Une équipe satisfaite de Swagger UI peut évaluer le gain avant de migrer. L’intégration est aussi présentée dans la [documentation ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/openapi/using-openapi-documents?view=aspnetcore-10.0).

## 10. BenchmarkDotNet pour mesurer avant de choisir

[BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet) aide à comparer les performances de morceaux de code dans un cadre de mesure structuré. Il prend en charge les répétitions, l’échauffement et la présentation des résultats. Le diagnostic mémoire permet aussi d’observer les allocations.

**Son avantage** est de donner des éléments concrets pour départager deux implémentations. Une conversion, un traitement de collection ou une opération de sérialisation peut alors être évalué avec les données qui nous intéressent.

**Sa limite** est celle du scénario mesuré. Un résultat obtenu sur une petite collection en mémoire ne décrit pas automatiquement les performances d’une API complète. La version de .NET, le matériel et la distribution des données peuvent modifier les conclusions.

Je l’utiliserais pour répondre à une question précise, en suivant les [bonnes pratiques du projet](https://benchmarkdotnet.org/articles/guides/good-practices.html), notamment une exécution en mode Release sans débogueur. Les mesures en conditions représentatives et les essais de charge restent nécessaires pour évaluer le comportement global.

### Et Dapper face à EF Core ?

Dapper mérite une évaluation contextuelle. EF Core peut déjà être très performant, et un changement d’outil ne garantit pas une amélioration perceptible.

Dans une application très orientée vers la lecture, avec des exigences de performance élevées, comparer les deux sur les requêtes critiques peut être pertinent. Je regarderais d’abord les projections, les index, le nombre d’allers-retours et le volume de données récupérées.

La [documentation de performance d’EF Core](https://learn.microsoft.com/en-us/ef/core/performance/) rappelle que les écarts observés dans des microbenchmarks peuvent devenir négligeables lorsque le travail de la base et la latence réseau dominent.

Un gain mesuré peut justifier Dapper pour certains accès. Il faut alors le mettre en balance avec le SQL à entretenir et la complexité d’un éventuel usage conjoint avec EF Core. Je garderais donc ce choix lié au contexte du projet.

## Un médiateur doit aussi répondre à un besoin

La réflexion s’applique également à MediatR et aux bibliothèques qui reprennent le même rôle. Séparer les commandes et les lectures peut se faire avec des services ou des gestionnaires appelés directement.

Un médiateur devient intéressant lorsque la centralisation de la distribution et des comportements transversaux apporte un bénéfice clair. Une implémentation maison très limitée peut convenir à un besoin précis, mais elle peut aussi grossir jusqu’à devenir un framework qu’il faudra entretenir.

Avant de chercher un remplacement à une bibliothèque, je vérifierais donc quelles fonctionnalités le projet utilise réellement.

## Conclusion

Il n’existe pas de liste universelle de paquets NuGet qui conviendrait à tous les projets .NET. Un outil devient intéressant lorsqu’il répond à un problème réel et qu’il améliore suffisamment l’expérience de développement, la qualité ou la maintenabilité pour justifier la dépendance supplémentaire.

Dans cette sélection, xUnit.net, NSubstitute, AutoFixture et Testcontainers couvrent des besoins complémentaires autour des essais. ArchUnitNET aide à rendre certaines décisions d’architecture vérifiables. FluentValidation, Mapperly et Scrutor répondent à des besoins plus ciblés dans le code applicatif. Scalar améliore l’expérience autour de la documentation OpenAPI, tandis que BenchmarkDotNet permet de prendre des décisions de performance à partir de mesures plutôt que d’intuitions.

Je préfère donc voir ces outils comme une boîte à outils plutôt que comme une checklist. Certains deviendront presque incontournables dans un contexte donné, alors que d’autres ne seront jamais nécessaires. Le bon choix reste celui qui simplifie réellement le projet sans créer plus de complexité qu’il n’en retire.

