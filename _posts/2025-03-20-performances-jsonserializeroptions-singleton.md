---
title: Amélioration des performances en utilisant JsonSerializerOptions en singleton
date: 2025-03-20 20:00:00 -0400
categories: []
tags: [dotnet]
---

## Introduction

Récemment, j'ai donné une formation interne où je partageais différentes astuces pour améliorer les performances d'applications en .NET. Parmi ces astuces, une technique particulièrement efficace concernait l'utilisation de `JsonSerializerOptions` en singleton.

Historiquement, la sérialisation d'objets en .NET a souvent été réalisée avec le NuGet [Newtonsoft](https://www.newtonsoft.com/json). Toutefois, depuis l'introduction de `System.Text.Json` en .NET Core 3.0, une alternative plus performante et mieux intégrée au runtime est devenue disponible. Bien que `System.Text.Json` soit généralement plus rapide que `Newtonsoft`, son efficacité peut être compromise par une mauvaise gestion des options de sérialisation (`JsonSerializerOptions`). Lorsque ces options sont recréées à chaque appel au lieu d'être réutilisées, cela peut entraîner une augmentation notable des allocations mémoire et une dégradation des performances.

Dans cet article, nous allons démontrer l'impact de l'utilisation d'une instance singleton de `JsonSerializerOptions` par rapport à la création d'options à chaque appel. Nous allons utiliser [BenchmarkDotNet](https://benchmarkdotnet.org/) pour comparer les deux méthodes et mettre en évidence les gains potentiels en matière de performances.

## Contexte

L'analyseur [CA1869](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca1869) recommande d'utiliser une instance de `JsonSerializerOptions` en singleton pour optimiser les performances. Cela est particulièrement pertinent lorsqu'une configuration personnalisée est requise, comme l'utilisation de `PropertyNamingPolicy` ou `PropertyNameCaseInsensitive`.

## Configuration de l'analyse de performances

Pour mesurer les performances, nous utilisons le code suivant qui compare deux méthodes :

- **SerializeSansSingleton()** : Crée une nouvelle instance de `JsonSerializerOptions` à chaque sérialisation.
- **SerializeAvecSingleton()** : Réutilise une instance singleton préconfigurée (`JsonSerializerOptions.Web`) pour chaque sérialisation.

Voici le code complet :

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Columns;
using BenchmarkDotNet.Order;
using System.Text.Json;

namespace TestBenchmarkDotnet.TestJsonSerializerOptions;

[MemoryDiagnoser]
[Orderer(SummaryOrderPolicy.FastestToSlowest)]
[RankColumn]
[HideColumns(Column.Error, Column.Median, Column.RatioSD, Column.StdDev)]
public class BenchmarkJsonSerializerOptions
{
    private readonly TestModel _testModel = new()
    {
        Id = 1,
        Nom = "Test",
        Description = "This is a test description",
        EstActif = true
    };

    [Params(1, 100)]
    public int Iteration { get; set; }

    [Benchmark(Baseline = true)]
    public string SerializeSansSingleton()
    {
        var json = "";
        for (var i = 0; i < Iteration; i++)
        {
            var options = new JsonSerializerOptions
            {
                PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
                PropertyNameCaseInsensitive = true
            };
            json = JsonSerializer.Serialize(_testModel, options);
        }
        return json;
    }

    [Benchmark]
    public string SerializeAvecSingleton()
    {
        var json = "";
        for (var i = 0; i < Iteration; i++)
        {
            json = JsonSerializer.Serialize(_testModel, JsonSerializerOptions.Web);
        }
        return json;
    }
}

public class TestModel
{
    public int Id { get; set; }
    public required string Nom { get; set; }
    public required string Description { get; set; }
    public bool EstActif { get; set; }
}
```

## Résultats des benchmarks

L'exécution des benchmarks a produit les résultats suivants :

```txt
| Méthode                | Itérations | Durée moyenne | Ratio | Allocations Gen0 | Mémoire allouée |
|------------------------|------------|---------------|-------|------------------|-----------------|
| SerializeAvecSingleton | 1          | 205.6 ns      | 0.24  | 0.0219           | 184 B           |
| SerializeSansSingleton | 1          | 861.3 ns      | 1.00  | 0.0429           | 390 B           |
| SerializeAvecSingleton | 100        | 21,592.5 ns   | 0.32  | 2.1973           | 18400 B         |
| SerializeSansSingleton | 100        | 67,384.9 ns   | 1.00  | 4.1504           | 39035 B         |
```

## Analyse des résultats

Les résultats montrent clairement qu'en utilisant une instance singleton de `JsonSerializerOptions` :

- La méthode est environ **4 fois plus rapide** pour une seule itération.
- Les allocations mémoire sont réduites de **plus de 50 %**, ce qui est significatif pour des traitements en masse.

L'approche utilisant `JsonSerializerOptions.Web` réutilise la configuration en singleton, ce qui permet d'éviter la création d'objets inutiles et améliore ainsi les performances de façon considérable.

## Conclusion

L'utilisation d'une instance singleton pour `JsonSerializerOptions` est une bonne pratique qui améliore non seulement les performances en termes de temps d'exécution, mais réduit également les allocations mémoire. Cela est particulièrement bénéfique dans les scénarios où les opérations de sérialisation sont répétées de manière intensive.

L'analyseur  [CA1869](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca1869) dans .NET fournit des recommandations qui méritent d'être suivies pour éviter les pièges courants en matière de performances.

En résumé, si votre application effectue des sérialisations fréquentes avec des configurations spécifiques, envisagez d'utiliser une instance singleton pour vos `JsonSerializerOptions`.

>💡 __Astuces supplémentaires__ : Pour vous assurer que l'analyseur CA1869 est actif dans votre projet, vous pouvez l'ajouter dans votre fichier `.editorconfig` comme suit `dotnet_diagnostic.CA1869.severity = warning`.
