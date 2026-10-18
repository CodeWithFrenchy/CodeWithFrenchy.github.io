---
title: "EF Core : Any() ou Count > 0 pour vérifier l’existence d’éléments ?"
date: 2027-11-30 19:00:00 -0400
categories: []
tags: [dotnet, csharp, ef-core, linq, performance, bonnes-pratiques]
---

## Préambule

Vérifier si une collection contient au moins un élément semble être une opération anodine. Pourtant, le choix entre `Any()`, `Count()` et la propriété `Count` touche à la fois la lisibilité du code et la façon dont une requête peut être exécutée.

Dans une requête Entity Framework Core, utiliser `Any()` pour exprimer une condition d’existence est généralement un bon choix. Mais justifier ce choix uniquement par une promesse de performance serait incomplet. Les optimisations du fournisseur, la version d’EF Core et le contexte d’exécution peuvent changer le résultat.

Partons d’un exemple pour distinguer ce qui relève de l’intention, de la traduction SQL et des collections déjà en mémoire.

## Exprimer la question que l’on pose réellement

Prenons une requête qui sélectionne les documents d’un dossier ayant au moins un statut et au moins une métadonnée. Son filtre pourrait être écrit ainsi :

```csharp
.Where(document =>
    document.DossierId == dossierId &&
    document.Statuts.Count > 0 &&
    document.Metadonnees.Count > 0)
```

Le résultat attendu est clair lorsqu’on prend le temps de lire la condition. Toutefois, le nombre exact de statuts ou de métadonnées ne nous intéresse pas. Nous voulons seulement vérifier leur présence.

Le même filtre peut donc s’écrire :

```csharp
.Where(document =>
    document.DossierId == dossierId &&
    document.Statuts.Any() &&
    document.Metadonnees.Any())
```

`Any()` exprime directement la question « existe-t-il au moins un élément ? ». À la lecture de cette requête, il devient immédiatement évident que la présence d’un seul statut et d’une seule métadonnée suffit.

Ce choix reste pertinent même lorsque les deux expressions aboutissent au même SQL. Il rapproche le code de la condition métier et fournit une convention simple pour les revues de code.

## Ce que COUNT et EXISTS expriment en SQL

Dans une requête relationnelle, une condition d’existence peut être représentée par `EXISTS`. La [documentation SQL Server sur EXISTS](https://learn.microsoft.com/en-us/sql/t-sql/language-elements/exists-transact-sql) précise que cette expression renvoie vrai lorsque sa sous-requête retourne au moins une ligne.

Voici une comparaison conceptuelle, limitée au test sur les statuts. On suppose que chaque statut référence son document par `DocumentId`.

Une forme basée sur un décompte pourrait être :

```sql
SELECT d.Id
FROM Document AS d
WHERE d.DossierId = @DossierId
  AND (
      SELECT COUNT(*)
      FROM Statut AS s
      WHERE s.DocumentId = d.Id
  ) > 0
```

Une forme basée sur l’existence serait :

```sql
SELECT d.Id
FROM Document AS d
WHERE d.DossierId = @DossierId
  AND EXISTS (
      SELECT 1
      FROM Statut AS s
      WHERE s.DocumentId = d.Id
  )
```

La première forme exprime un décompte suivi d’une comparaison. La seconde demande seulement de déterminer si une correspondance existe. Le moteur peut alors utiliser un plan qui interrompt la recherche dès qu’une correspondance suffit à établir l’existence.

Cela ne garantit pas que `EXISTS` sera toujours plus rapide. L’optimiseur choisit le plan d’exécution, et l’absence de correspondance peut nécessiter de parcourir toutes les données pertinentes. Les index et le volume de données restent déterminants, comme le rappelle le [guide Microsoft sur les requêtes efficaces avec EF Core](https://learn.microsoft.com/en-us/ef/core/performance/efficient-querying).

Ces exemples illustrent les deux formes SQL. Ils ne représentent pas une mesure de performance ni le SQL exact de toutes les versions d’EF Core.

## EF Core 9 a changé une partie de l’équation

La distinction historique entre les deux traductions doit être nuancée.

Dans la [documentation des nouveautés d’EF Core 9](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-9.0/whatsnew#queries-using-count--0-are-optimized), Microsoft présente précisément une condition utilisant `Count > 0` sur une collection de navigation. L’exemple était traduit avec `COUNT` sous EF Core 8. EF Core 9 le transforme en `EXISTS`.

Il serait donc incorrect d’affirmer que `Count > 0` produit systématiquement un décompte SQL. Cette optimisation existe pour certains patrons reconnus, et le comportement exact dépend de la version, du fournisseur et de la forme de la requête.

Dans notre filtre, je privilégierais tout de même `Any()` pour exprimer directement l’existence. Si les deux variantes produisent le même SQL, le bénéfice concerne surtout la lisibilité et la cohérence des pratiques. Un gain de performance doit être vérifié séparément.

## Count, Count() et les collections en mémoire

La recommandation précédente concerne une expression traduite par EF Core. Elle ne doit pas devenir une règle de remplacement automatique dans tout le code C#.

Pour une `List<T>` déjà en mémoire, cette expression est parfaitement appropriée :

```csharp
listeDocuments.Count > 0
```

La propriété `Count` expose un nombre déjà connu. Elle ne parcourt pas la liste pour le recalculer. La [documentation de List<T>.Count](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1.count) indique un accès en temps constant, soit `O(1)`.

La méthode LINQ `Count()` mérite une autre distinction. Elle peut parcourir une séquence pour en déterminer la taille, mais ce parcours n’est pas systématique. La [documentation de Enumerable.Count](https://learn.microsoft.com/en-us/dotnet/api/system.linq.enumerable.count) indique notamment qu’elle utilise le nombre fourni par une source qui implémente `ICollection<T>`.

Pour une séquence dont la taille n’est pas directement disponible, `Any()` évite de demander un décompte complet lorsqu’on souhaite seulement savoir si elle est vide. Avec un prédicat, il exprime aussi directement la recherche d’une correspondance :

```csharp
sequenceDocuments.Any(document => document.EstActif)
```

La [documentation de Enumerable.Any](https://learn.microsoft.com/en-us/dotnet/api/system.linq.enumerable.any) précise que l’énumération s’arrête dès que le résultat peut être déterminé.

Enfin, une navigation utilisée dans un `Where` traduit par EF Core n’est pas évaluée comme une simple liste locale. Le fournisseur analyse cette expression pour construire la requête, conformément au [fonctionnement des requêtes EF Core](https://learn.microsoft.com/en-us/ef/core/querying/how-query-works). Le contexte d’exécution compte donc autant que la syntaxe.

## Utiliser AnyAsync lorsque le résultat attendu est un booléen

Si l’on veut uniquement savoir si un dossier contient un document répondant aux critères, on peut exécuter directement un test d’existence :

```csharp
await contexte.Documents.AnyAsync(
    document =>
        document.DossierId == dossierId &&
        document.Statuts.Any() &&
        document.Metadonnees.Any(),
    cancellationToken)
```

La méthode [AnyAsync](https://learn.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.entityframeworkqueryableextensions.anyasync) retourne un booléen et accepte un jeton d’annulation. Il n’est pas nécessaire de charger une liste de documents pour ensuite vérifier si elle contient des éléments.

Les appels internes à `Any()` font ici partie de l’expression traduite. Ils ne déclenchent pas des requêtes séparées pour chaque document. C’est l’appel externe à `AnyAsync()` qui exécute la requête, selon le principe décrit dans la [documentation sur les opérateurs asynchrones d’EF Core](https://learn.microsoft.com/en-us/ef/core/querying/async#async-linq-operators).

Attention également à cette autre forme :

```csharp
(await contexte.Documents.CountAsync(
    document => document.DossierId == dossierId,
    cancellationToken)) > 0
```

Ici, le décompte est exécuté avant la comparaison. Le test `> 0` se produit en C# après le `await`. L’optimisation d’une comparaison située à l’intérieur d’une expression de requête ne s’applique donc pas de la même manière. La règle d’analyse [CA1828](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca1828) recommande justement `AnyAsync()` lorsque seul un test d’existence est nécessaire.

## Vérifier le SQL avant d’annoncer un gain

Pour examiner la traduction de notre filtre, on peut utiliser cette expression :

```csharp
contexte.Documents
    .Where(document =>
        document.DossierId == dossierId &&
        document.Statuts.Any() &&
        document.Metadonnees.Any())
    .ToQueryString()
```

La méthode [ToQueryString](https://learn.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.entityframeworkqueryableextensions.toquerystring) fournit une représentation de la requête destinée au débogage. On peut comparer les variantes avec `Any()` et `Count > 0` pour le fournisseur et la version utilisés.

Pour une opération terminale comme `AnyAsync()`, les journaux de commandes permettent d’observer le SQL réellement exécuté. Si la performance motive le changement, il faut ensuite examiner le plan d’exécution et mesurer sur des données représentatives. La [documentation sur le diagnostic de performance d’EF Core](https://learn.microsoft.com/en-us/ef/core/performance/performance-diagnosis) décrit cette démarche.

Deux écritures C# différentes ne constituent pas, à elles seules, la preuve de deux coûts d’exécution différents.

## Une convention qui tient compte du contexte

Voici la convention que je retiendrais dans une solution .NET :

| Contexte | Choix à privilégier |
| --- | --- |
| Condition d’existence dans une requête EF Core | `Any()` |
| Requête EF Core dont le résultat attendu est uniquement un booléen | `AnyAsync()` |
| Nombre exact nécessaire en base de données | `CountAsync()` |
| Liste déjà en mémoire | Propriété `Count` |
| Tableau déjà en mémoire | Propriété `Length` |
| Collection exposant une propriété adaptée | Propriété `IsEmpty`, selon le type |
| Séquence à parcourir pour vérifier une existence | `Any()` |

Les analyseurs .NET reflètent ces distinctions. La règle [CA1827](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca1827) vise les appels à `Count()` ou `LongCount()` utilisés pour tester l’existence. À l’inverse, [CA1860](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca1860) recommande les propriétés `Count`, `Length` ou `IsEmpty` lorsqu’elles sont disponibles sur une collection en mémoire.

Ces règles se complètent. Elles invitent à choisir l’opération adaptée à la source et au résultat recherché.

## Conclusion

Dans une requête EF Core, `Any()` reste un choix naturel pour exprimer une condition d’existence. Il rend l’intention explicite et évite de demander conceptuellement un nombre dont on n’a pas besoin.

Les optimisations introduites dans EF Core 9 empêchent toutefois de promettre un gain systématique face à `Count > 0`. Lorsque les traductions sont équivalentes, la valeur du changement tient principalement à la lisibilité et à la cohérence du code.

Pour les collections déjà en mémoire, les propriétés `Count`, `Length` et `IsEmpty` conservent toute leur pertinence. Une bonne convention distingue ces contextes et s’appuie sur des mesures lorsqu’elle avance un bénéfice de performance.
