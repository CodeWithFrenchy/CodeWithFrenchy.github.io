---
title: Introduction aux Minimal APIs
date: 2024-10-10 20:00:00 -0400
categories: [outil-developpement, architecture]
tags: [dotnet, aspnet-core, minimal-api, api-rest, microservices, aot, tests-unitaires]
---

L’utilisation des Minimal APIs est une approche simplifiée pour créer des API HTTP rapides avec ASP.NET Core. Elles permettent de créer des points de terminaison REST entièrement fonctionnels avec un minimum de code et de configuration. Vous pouvez ignorer la génération automatique classique et éviter les contrôleurs en déclarant directement des routes et des actions d’API. 

Par exemple, le code suivant crée une API à la racine de l’application web qui retourne simplement le texte "Hello World!" : 

```csharp
var app = WebApplication.Create(args)

app.MapGet("/", () => "Hello World!");

app.Run();
```

Cela suffit pour démarrer, mais il y a bien plus à découvrir. Les API minimales offrent également la configuration et la personnalisation nécessaires pour évoluer vers plusieurs API, gérer des routes complexes, appliquer des règles d’autorisation, et contrôler le contenu des réponses.

## Comment créer un premier minimal Api

Ce [tutoriel](https://learn.microsoft.com/fr-ca/aspnet/core/tutorials/min-web-api?view=aspnetcore-8.0&tabs=visual-studio) décrit les principes fondamentaux liés à la création d’un Minimal API avec ASP.NET Core.

👨‍💻 Le code du tutoriel est disponible [ici](https://github1s.com/dotnet/AspNetCore.Docs/tree/main/aspnetcore/tutorials/min-web-api/samples/8.x). Si vous ne le saviez pas, il est possible d’ouvrir [un référentiel GitHub directement dans une version web de Visual Studio Code](https://github.com/conwnet/github1s) en ajoutant "1s" après "github" dans l'URL.

## Comment structurer ces endpoints

Que remarquez-vous à la suite de la lecture du tutoriel ? Les Minimal APIs offrent une grande flexibilité sur la manière de les structurer. On peut définir des endpoints directement dans le fichier Program.cs, contrairement à l’utilisation classique des contrôleurs. C’est très pratique pour réaliser des prototypes ou même des projets de faible envergure.

Mais que faire si notre projet devient plus complexe ? Comment gérer les essais unitaires et structurer le code pour faciliter sa lisibilité et son entretien ?

Voici quelques recommandations importantes à garder en tête :

- Pour être testable unitairement, une méthode doit être exposée et non simplement définie en tant qu'endpoint ([exemple](https://github.com/dotnet/AspNetCore.Docs.Samples/blob/main/fundamentals/minimal-apis/samples/MinApiTestsSample/UnitTests/TodoMoqTests.cs)).
- Il est possible de définir des méthodes statiques dans le Program.cs, mais il est recommandé de [déplacer les points de terminaison à l’extérieur](https://learn.microsoft.com/fr-ca/aspnet/core/fundamentals/minimal-apis/route-handlers?view=aspnetcore-8.0#endpoint-defined-outside-of-programcs) pour une meilleure organisation.
- Utilisez le type [TypedResults](https://learn.microsoft.com/fr-fr/dotnet/api/microsoft.aspnetcore.http.typedresults?view=aspnetcore-8.0) au lieu du type générique [Results](https://learn.microsoft.com/fr-fr/dotnet/api/microsoft.aspnetcore.http.results?view=aspnetcore-8.0) pour un [meilleur contrôle des réponses API](https://learn.microsoft.com/fr-ca/aspnet/core/tutorials/min-web-api?view=aspnetcore-8.0&tabs=visual-studio#use-the-typedresults-api).

Le code associé au tutoriel illustre ces pratiques à travers la structure de la classe [TodoEndpointsV2](https://github.com/dotnet/AspNetCore.Docs.Samples/blob/main/fundamentals/minimal-apis/samples/MinApiTestsSample/WebMinRouteGroup/TodoEndpointsV2.cs).

## D’autres lectures pour aller plus loin

Si vous souhaitez approfondir vos connaissances sur les Minimal APIs, voici quelques ressources que je vous recommande fortement :

- [Gestionnaires de routage](https://learn.microsoft.com/fr-ca/aspnet/core/fundamentals/minimal-apis/route-handlers?view=aspnetcore-8.0)
- [Liaison de paramètres](https://learn.microsoft.com/fr-ca/aspnet/core/fundamentals/minimal-apis/parameter-binding?view=aspnetcore-8.0)
- [Création de réponses](https://learn.microsoft.com/fr-ca/aspnet/core/fundamentals/minimal-apis/responses?view=aspnetcore-8.0)
- [Filtres](https://learn.microsoft.com/fr-ca/aspnet/core/fundamentals/minimal-apis/min-api-filters?view=aspnetcore-8.0)
- [Tests unitaires et d'intégration](https://learn.microsoft.com/fr-ca/aspnet/core/fundamentals/minimal-apis/test-min-api?view=aspnetcore-8.0)

### Quand utiliser les Minimal APIs ?

Les Minimal APIs brillent dans les cas où la simplicité, la rapidité, et la légèreté sont des priorités. Elles sont particulièrement adaptées pour :

- **Les microservices** : Leur faible complexité en fait un excellent choix pour les services isolés, où les fonctionnalités sont limitées et où les performances sont cruciales.
- **Les prototypes ou petites applications** : Si vous avez besoin de mettre en place rapidement un service sans trop de structure, les Minimal APIs offrent une solution agile.
- **Les fonctions serverless** : Leur nature légère permet de réduire les temps de démarrage et d’améliorer les performances, particulièrement pour des environnements comme Azure Functions ou AWS Lambda.
- **Compatibilité avec l’AOT (Ahead-of-Time) compilation** : Contrairement aux contrôleurs classiques, les Minimal APIs sont compatibles avec la compilation AOT, ce qui est un atout important si vous avez des exigences de performance et de déploiement rapide. Les Web APIs classiques, reposant sur la réflexion, ne peuvent pas bénéficier de cette optimisation.

Les Minimal APIs et les Web APIs classiques ont toutes deux leur utilité, en fonction des besoins du projet. Pour des applications simples ou des microservices, les Minimal APIs offrent rapidité, compatibilité AOT, et efficacité. Pour des applications plus complexes nécessitant une structure bien définie, les Web APIs classiques restent le choix idéal.

## Conclusion 

L'utilisation des API minimales dans ASP.NET Core offre une manière moderne et efficace de créer des services web légers et performants. Grâce à leur simplicité, elles permettent aux développeurs de se concentrer sur l'essentiel, en réduisant la complexité du code et en accélérant le développement. Bien qu’elles soient idéales pour des scénarios simples et des microservices, elles offrent également la flexibilité nécessaire pour évoluer vers des architectures plus complexes si besoin. En adoptant cette approche, vous bénéficiez d'une solution élégante, facile à maintenir, tout en profitant des puissantes fonctionnalités d'ASP.NET Core.

💡 Pour approfondir le débat entre Minimal APIs et Web APIs classiques, je vous recommande vivement de visionner la vidéo de Nick Chapsas, où il partage son avis éclairé sur le sujet. 

<iframe width="740" height="473" src="https://www.youtube.com/embed/bpVN_plQm5E?si=yVabG1jG36FkAPML" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Références 

- [Vue d’ensemble des API minimales](https://learn.microsoft.com/fr-ca/aspnet/core/fundamentals/minimal-apis/overview?view=aspnetcore-8.0)
- [Tutoriel : Créer une API minimale avec ASP.NET Core](https://learn.microsoft.com/fr-ca/aspnet/core/tutorials/min-web-api?view=aspnetcore-8.0&tabs=visual-studio)
- [Gestionnaires de routage dans les applications API minimales](https://learn.microsoft.com/fr-ca/aspnet/core/fundamentals/minimal-apis/route-handlers?view=aspnetcore-8.0)
- [Liaison de paramètres dans les applications API minimales](https://learn.microsoft.com/fr-ca/aspnet/core/fundamentals/minimal-apis/parameter-binding?view=aspnetcore-8.0)
- [Créer des réponses dans les applications API minimales](https://learn.microsoft.com/fr-ca/aspnet/core/fundamentals/minimal-apis/responses?view=aspnetcore-8.0)
- [Filtres dans les applications d’API minimales](https://learn.microsoft.com/fr-ca/aspnet/core/fundamentals/minimal-apis/min-api-filters?view=aspnetcore-8.0)
- [Tester les applications API minimales](https://learn.microsoft.com/fr-ca/aspnet/core/fundamentals/minimal-apis/test-min-api?view=aspnetcore-8.0)
