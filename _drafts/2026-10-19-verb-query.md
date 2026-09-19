---
title: "HTTP QUERY : une méthode pour les recherches complexes dans nos API"
date: 2026-10-19 19:00:00 -0400
categories: [architecture]
tags: [http, aspnet-core, dotnet, openapi]
---

## Introduction

Une recherche commence souvent par quelques paramètres dans une URL. Une catégorie, un statut, une limite de résultats. Puis les besoins évoluent : plusieurs groupes de filtres, des exclusions, des plages de valeurs et des critères combinés.

À ce stade, beaucoup d’équipes finissent par créer un endpoint `POST /search` qui reçoit un objet JSON. Le traitement ne modifie pourtant aucune donnée métier. On utilise surtout POST parce qu’il permet de transmettre facilement un corps de requête.

La méthode HTTP `QUERY` répond à ce besoin : exprimer une recherche avec un corps de requête tout en annonçant une opération sûre et idempotente.

Le sujet a attiré mon attention à la lecture d’un [article de Start Debugging](https://startdebugging.net/2026/05/aspnetcore-11-http-query-method-openapi/) sur sa prise en charge dans les documents OpenAPI générés par ASP.NET Core 11. Pour comprendre l’intérêt de cette nouveauté, il faut distinguer le contrat HTTP, le code de l’API et les outils qui l’entourent.

> Au 19 septembre 2026, QUERY est défini dans la [RFC 10008](https://datatracker.ietf.org/doc/rfc10008/), publiée en juin avec le statut « Proposed Standard ». La nouveauté ASP.NET Core abordée ici a été annoncée avec .NET 11 Preview 4.

## Pourquoi GET et POST ne couvrent pas tout à fait le même besoin

Pour une recherche simple, GET reste naturel :

```http
GET /products?category=books&maxPrice=50 HTTP/1.1
Host: api.example.com
```

L’URL est lisible et facile à partager. Je ne remplacerais pas ce type d’endpoint uniquement pour adopter une nouvelle méthode.

La situation change lorsqu’un écran de recherche permet de combiner des groupes de conditions. On peut toujours inventer une convention d’encodage dans l’URL, mais il faut ensuite la documenter, la parser et la maintenir. Un objet JSON devient parfois plus simple à manipuler.

Pourquoi ne pas simplement envoyer ce JSON avec GET? La [sémantique HTTP de GET](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.3.1) n’attribue pas de signification générale à son corps de requête. Un accord entre un client et un serveur ne garantit pas que les intermédiaires le traiteront correctement.

POST permet de transmettre le contenu au serveur pour traitement. Une recherche avec POST peut parfaitement être en lecture seule dans notre application. Cependant, [la méthode POST](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.3.3) ne porte pas, à elle seule, cette garantie pour un client générique.

**QUERY rend cette intention explicite dans le protocole.**

## Ce que signifie réellement QUERY

La [définition de QUERY](https://datatracker.ietf.org/doc/rfc10008/) prévoit une opération sûre et idempotente dont le contenu décrit la recherche. Elle n’impose ni JSON ni un langage de filtres particulier. Le serveur définit les formats qu’il accepte; le client doit fournir un `Content-Type` cohérent avec son contenu.

Les termes « sûr » et « idempotent » ont ici leur [sens HTTP](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.2) :

- **Sûr** : le client ne demande pas de modification de l’état métier de la ressource. La journalisation reste possible.
- **Idempotent** : répéter la requête produit le même effet attendu sur le serveur qu’une seule exécution.

Cela ne garantit pas des réponses identiques. Deux recherches peuvent retourner des résultats différents si les données ont changé entre les appels.

Pour choisir une méthode, je retiendrais les repères suivants :

| Besoin | Choix à privilégier |
| --- | --- |
| Lire une ressource ou filtrer une liste avec quelques paramètres | GET |
| Effectuer une recherche structurée en lecture seule | QUERY, si la chaîne technique le prend en charge |
| Conserver une recherche avec corps pour des clients incompatibles avec QUERY | POST comme solution de compatibilité |
| Créer une ressource ou déclencher une action métier | Une méthode adaptée à la modification, souvent POST |

Un endpoint qui recherche des dossiers puis les marque comme « traités » ne devrait donc pas être exposé comme une simple QUERY. Le nom de la méthode engage le comportement de l’application.

## Ce qu’ASP.NET Core 11 ajoute exactement

Les [notes de version d’ASP.NET Core 11 Preview 4](https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/preview4/aspnetcore.md#http-query-in-generated-openapi-documents) précisent que le routage acceptait déjà des méthodes arbitraires avec `MapMethods`. L’ajout concerne la reconnaissance de QUERY dans la génération OpenAPI.

Il n’est donc pas nécessaire d’attendre cette version pour déclarer une chaîne `"QUERY"` dans le routage. En revanche, accepter la requête à l’exécution et la décrire correctement dans le contrat OpenAPI sont deux capacités distinctes.

La [spécification OpenAPI 3.2](https://spec.openapis.org/oas/v3.2.0.html#path-item-object) possède un champ `query` dans le Path Item Object, au même niveau que `get` et `post`. Attention au vocabulaire : ce champ décrit la méthode HTTP. Il ne désigne pas les paramètres de l’URL, représentés ailleurs avec `in: query`.

Pour les documents OpenAPI 3.0 et 3.1, le [changement intégré à ASP.NET Core](https://github.com/dotnet/aspnetcore/pull/65714) place l’opération dans l’extension `x-oai-additionalOperations`. L’information reste présente, mais un générateur de client doit comprendre cette extension pour l’exploiter.

Ce point compte lors d’une migration : obtenir un document valide ne prouve pas que le portail d’API, l’importateur et les SDK générés sauront tous utiliser l’opération.

## Un exemple avec une Minimal API

Prenons un petit catalogue de produits. L’exemple reste volontairement simple pour montrer le routage et la liaison du corps JSON. Dans une application réelle, l’intérêt d’un tel contrat augmente avec la complexité des critères.

Le code suivant vise un projet ASP.NET Core .NET 11 avec une version compatible du paquet `Microsoft.AspNetCore.OpenApi`, incluant les changements de Preview 4. La [configuration OpenAPI 3.2](https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/preview4/aspnetcore.md#http-query-in-generated-openapi-documents) se fait dans `AddOpenApi`.

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.OpenApi;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOpenApi(options =>
{
    options.OpenApiVersion = OpenApiSpecVersion.OpenApi3_2;
});

var app = builder.Build();

Product[] products =
[
    new(1, "Architecture logicielle", "books", 45m),
    new(2, "Programmation C#", "books", 65m),
    new(3, "Clavier", "accessories", 90m)
];

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}

app.MapMethods("/products/search", ["QUERY"],
    ([FromBody] ProductSearch criteria) =>
    {
        if (criteria.Limit is < 1 or > 100 || criteria.MaxPrice is < 0)
        {
            return Results.BadRequest(
                "Limit doit être compris entre 1 et 100; MaxPrice doit être positif ou nul.");
        }

        var matches = products
            .Where(p => criteria.Category is null ||
                string.Equals(p.Category, criteria.Category,
                    StringComparison.OrdinalIgnoreCase))
            .Where(p => criteria.MaxPrice is null ||
                p.Price <= criteria.MaxPrice.Value)
            .OrderBy(p => p.Id)
            .Take(criteria.Limit)
            .ToArray();

        return Results.Ok(matches);
    })
    .WithName("SearchProducts")
    .Produces<Product[]>()
    .Produces<string>(StatusCodes.Status400BadRequest);

app.Run();

public sealed record ProductSearch(
    string? Category,
    decimal? MaxPrice,
    int Limit = 20);

public sealed record Product(
    int Id,
    string Name,
    string Category,
    decimal Price);
```

L’attribut `[FromBody]` rend explicite l’origine des critères. Le traitement filtre les données sans les modifier et limite la taille du résultat.

Depuis un fichier `.http`, on peut préparer cet appel en adaptant le port à celui de l’application :

```http
@baseUrl = http://localhost:5000

QUERY {{baseUrl}}/products/search
Content-Type: application/json
Accept: application/json

{
  "category": "books",
  "maxPrice": 50,
  "limit": 20
}
```

Avec les données de l’exemple, le résultat attendu contient uniquement le livre « Architecture logicielle ». **Ce code est illustratif et n’a pas été compilé ni exécuté pour cet article.**

## Le cache : une possibilité à implémenter correctement

La [RFC 10008](https://datatracker.ietf.org/doc/rfc10008/) autorise la mise en cache des réponses QUERY. Elle impose aussi que la clé tienne compte du corps de la requête et des métadonnées associées.

Deux appels vers `/products/search` peuvent demander des résultats entièrement différents. Une clé fondée uniquement sur l’URL serait donc incorrecte.

Il faut également distinguer les règles du protocole des capacités d’un middleware. Par exemple, la [politique par défaut de l’Output Caching d’ASP.NET Core 10](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/output?view=aspnetcore-10.0#default-output-caching-policy) limite la mise en cache aux méthodes GET et HEAD, avec d’autres conditions. La nouveauté OpenAPI de .NET 11 ne constitue pas une annonce de prise en charge automatique de QUERY par ce cache.

Dans une application multitenant, je vérifierais aussi que la stratégie de cache isole correctement les résultats selon le locataire et les droits de l’utilisateur. Je testerais au minimum deux corps différents sur la même URL et deux identités ayant des accès différents.

**Je n’adopterais pas QUERY en supposant obtenir immédiatement de meilleures performances.** La méthode exprime un contrat; le bénéfice dépend de l’implémentation.

## Ce qu’il faut valider dans un environnement d’entreprise

Le premier essai local démontre que l’application accepte la méthode. Avant une mise en service, je vérifierais le parcours complet de l’appel :

- **Passerelles et sécurité réseau** : la méthode traverse-t-elle le proxy inverse, la passerelle d’API et le WAF avec son corps intact?
- **Outillage OpenAPI** : le document est-il importé correctement et l’opération apparaît-elle dans les clients générés?
- **Résilience** : les règles de retry reconnaissent-elles QUERY et limitent-elles les répétitions pour une recherche coûteuse?
- **Observabilité** : la méthode est-elle identifiable dans les journaux, les métriques et les traces?

Dans Azure, cela signifie tester le trajet réellement utilisé, par exemple à travers API Management et Application Gateway si ces composants font partie de l’architecture. Il faut relever les versions, configurations et restrictions observées, plutôt que déduire une compatibilité générale du seul comportement de Kestrel.

Pour une application web sur une autre origine, QUERY entraîne aussi un contrôle CORS préalable. La [spécification Fetch](https://fetch.spec.whatwg.org/#methods) réserve les méthodes CORS dites « safelisted » à GET, HEAD et POST. La politique CORS doit donc autoriser QUERY et les en-têtes envoyés. Un POST avec `application/json` nécessite lui aussi généralement ce contrôle préalable.

Enfin, déplacer les critères dans le corps ne les rend pas confidentiels par nature. Je vérifierais les captures de corps dans les journaux et l’APM, ainsi que les limites de taille et de complexité de la recherche. Une opération en lecture seule peut tout de même être coûteuse à exécuter.

## Faut-il remplacer nos endpoints POST de recherche?

Je commencerais par les nouveaux usages internes dont l’équipe contrôle les clients et l’infrastructure. Une recherche avancée constitue un bon candidat si l’outillage est compatible et si le contrat apporte une clarification utile.

Pour une API publique déjà consommée, je conserverais le endpoint existant tant que les clients en ont besoin. Il est possible de proposer QUERY en parallèle de POST, en faisant appeler le même traitement applicatif par les deux routes. Cela évite de maintenir deux implémentations des filtres et de la pagination.

Cette transition demande aussi de documenter les comportements : formats acceptés, règles de validation, limites, erreurs et stabilité de la pagination. Le choix d’une méthode plus précise ne remplace pas ce travail de conception.

Je garderais GET pour les recherches simples. Introduire un corps de requête dans un scénario déjà bien servi par une URL courte apporte peu de valeur et rend le partage direct de la recherche moins immédiat.

## Conclusion

HTTP QUERY répond à un besoin concret, transmettre des critères structurés tout en déclarant clairement une opération de consultation. Pour les API de recherche avancée, c’est une évolution cohérente.

L’ajout dans ASP.NET Core 11 facilite sa représentation dans OpenAPI. Son adoption reste toutefois une décision qui concerne l’ensemble de la chaîne technique, du client jusqu’aux caches et aux passerelles.

Pour ma part, je l’évaluerais sur une nouvelle recherche interne avant d’envisager une migration plus large. Si les outils suivent, le contrat devient plus expressif. Si la compatibilité impose trop de contournements, un POST de recherche bien documenté reste un choix raisonnable.
