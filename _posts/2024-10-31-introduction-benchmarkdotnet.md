---
title: Introduction à BenchmarkDotNet
date: 2024-10-31 19:00:00 -0400
categories: [outil-developpement]
tags: [dotnet]
---

[BenchmarkDotNet](https://benchmarkdotnet.org/) est une bibliothèque populaire pour effectuer des benchmarks en .NET. Elle permet de mesurer précisément les performances de vos applications et d'identifier les points à optimiser.

Voici comment commencer :

1. **Installation** : Tout d'abord, installez BenchmarkDotNet via NuGet dans votre projet :

   ```bash
   dotnet add package BenchmarkDotNet
   ```

2. **Création d'un benchmark** : Créez une classe pour votre benchmark et ajoutez-y une méthode marquée avec l'attribut `[Benchmark]`.

3. **Configuration** : Vous pouvez configurer votre benchmark en utilisant des attributs comme `[Params]` pour spécifier différents paramètres de test.

## Exemple d'utilisation

Supposons que vous vouliez mesurer les performances de deux méthodes différentes pour obtenir la valeur texte d'un `enum`.

Voici comment vous pourriez procéder :

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Order;

namespace TestBenchmarkDotnet.TestEnumToString;

[MemoryDiagnoser]
[Orderer(SummaryOrderPolicy.FastestToSlowest)]
[RankColumn]
public class BenchmarkEnumToString
{
    [Params(1, 100, 1_000)]
    public int Iteration { get; set; }

    [Benchmark(Baseline = true)]
    public List<string> EnumToString()
    {
        var valeursEnum = new List<string>();

        for (var i = 0; i < Iteration; i++)
        {
            valeursEnum.Add(Joueur.ValeurEnum1.ToString());
        }

        return valeursEnum;
    }

    [Benchmark]
    public List<string> EnumNameof()
    {
        var valeursEnum = new List<string>();

        for (var i = 0; i < Iteration; i++)
        {
            valeursEnum.Add(nameof(Joueur.ValeurEnum1));
        }

        return valeursEnum;
    }

    private enum Joueur
    {
        ValeurEnum1,
        ValeurEnum2
    }
}

// ------------- au niveau du program.cs

using BenchmarkDotNet.Running;
using TestBenchmarkDotnet.TestEnumToString;

BenchmarkRunner.Run<BenchmarkEnumToString>();
```

## Démarrer l'exécution d'un benchmark

Pour exécuter un benchmark, il est essentiel d'exécuter vos benchmarks en mode `Release`.

Voici comment procéder :

1. **Exécution des benchmarks** : Vous pouvez exécuter vos benchmarks en ligne de commande.

   ```bash
   dotnet run -c Release
   ```

   > 💡 Assurez-vous d'exécuter la commande où se trouve votre fichier `.csproj`.

2. **Analyse des résultats** : Après l'exécution, BenchmarkDotNet générera un rapport détaillé dans la console avec des métriques telles que le temps d'exécution moyen, l'écart-type et l'allocation mémoire.

   Voici ce que ces métriques signifient :

   - **Temps d'exécution moyen (`Mean`)** : C'est la moyenne des temps d'exécution mesurés pour chaque itération du benchmark. Mesuré en nanosecondes (`ns`), microsecondes (`us`), ou millisecondes (`ms`).

   - **Écart-type (`StdDev`)** : Il mesure la variation ou la dispersion des temps d'exécution mesurés autour de la moyenne. Une valeur faible indique une grande stabilité dans les mesures.

   - **Allocation mémoire (`Allocated`)** : C'est la quantité totale de mémoire allouée pendant l'exécution du benchmark. Mesurée en octets (`B`), kilooctets (`KB`), ou mégaoctets (`MB`), selon l'ampleur de l'allocation.

## Résultat de l'exécution

```txt
| Method       | Iteration | Mean        | Error      | StdDev     | Median      | Ratio | RatioSD | Rank | Gen0   | Gen1   | Allocated | Alloc Ratio |
|------------- |---------- |------------:|-----------:|-----------:|------------:|------:|--------:|-----:|-------:|-------:|----------:|------------:|
| EnumNameof   | 1         |    13.71 ns |   0.117 ns |   0.109 ns |    13.69 ns |  0.59 |    0.02 |    1 | 0.0105 |      - |      88 B |        0.79 |
| EnumToString | 1         |    22.44 ns |   0.496 ns |   1.002 ns |    22.32 ns |  1.00 |    0.00 |    2 | 0.0134 |      - |     112 B |        1.00 |
|              |           |             |            |            |             |       |         |      |        |        |           |             |
| EnumNameof   | 100       |   375.49 ns |   7.477 ns |  18.621 ns |   367.17 ns |  0.35 |    0.02 |    1 | 0.2618 | 0.0010 |    2192 B |        0.48 |
| EnumToString | 100       | 1,065.24 ns |  21.091 ns |  44.488 ns | 1,064.77 ns |  1.00 |    0.00 |    2 | 0.5474 | 0.0019 |    4592 B |        1.00 |
|              |           |             |            |            |             |       |         |      |        |        |           |             |
| EnumNameof   | 1000      | 3,190.26 ns | 110.750 ns | 319.539 ns | 3,023.35 ns |  0.35 |    0.03 |    1 | 1.9836 | 0.0572 |   16600 B |        0.41 |
| EnumToString | 1000      | 9,346.52 ns | 186.882 ns | 360.058 ns | 9,284.09 ns |  1.00 |    0.00 |    2 | 4.8523 | 0.1373 |   40602 B |        1.00 |
```

On constate qu'il est plus rapide d'utiliser `nameof` et également moins coûteux en termes d'allocation mémoire.

## Points à considérer dans les benchmarks

Lorsque vous analysez les résultats de vos benchmarks avec BenchmarkDotNet, voici quelques points clés à garder à l'esprit :

- **Métriques** : Les métriques comme le temps d'exécution moyen, l'écart-type et l'allocation mémoire sont cruciales pour évaluer les performances et l'efficacité de votre code.
- **Comparaison** : Utilisez les résultats pour comparer différentes implémentations ou configurations de votre code.

## Conclusion

BenchmarkDotNet est un outil précieux pour évaluer les performances de vos applications .NET de manière précise et reproductible. En suivant ces étapes, vous pouvez commencer à utiliser efficacement BenchmarkDotNet pour optimiser vos applications.
