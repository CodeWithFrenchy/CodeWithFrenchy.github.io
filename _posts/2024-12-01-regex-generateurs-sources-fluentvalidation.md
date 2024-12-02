---
title: Optimisation des Expressions Régulières avec les Générateurs de Sources dans .NET
date: 2024-12-01 20:00:00 -0400
categories: []
tags: [dotnet, fluentvalidation, source-generator]
---

Avec .NET, les **générateurs de sources pour les expressions régulières** (*Regex Source Generators*) permettent d'optimiser la création et l'exécution des expressions régulières en générant du code source au moment de la compilation. Ces générateurs, introduits à partir de .NET 7, utilisent l'attribut **[GeneratedRegex]** pour définir des expressions régulières de manière déclarative.  

L'objectif est d'améliorer les performances en éliminant les étapes coûteuses de compilation des expressions régulières au moment de l'exécution. Au lieu de créer une instance de `Regex` dynamique, le générateur produit un code précompilé, offrant ainsi une solution plus rapide et adaptée aux scénarios exigeants.  

> 💡 Pour plus de détails, consultez la documentation officielle : [Regular Expression Source Generators](https://learn.microsoft.com/en-us/dotnet/standard/base-types/regular-expression-source-generators).  

## FluentValidation et les Générateurs de Sources  

On peut légitimement se demander si [FluentValidation](https://docs.fluentvalidation.net/en/latest/#) utilise les générateurs de sources pour optimiser l'instruction `Matches("^[a-zA-Z0-9]*$");`. Pour explorer cette question, j'ai conçu un benchmark comparant deux implémentations de validation : l'une utilisant des expressions régulières classiques et l'autre tirant parti des générateurs de sources avec **[GeneratedRegex]**.

## Benchmark Comparatif  

Voici le code du benchmark :  

``` csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Columns;
using BenchmarkDotNet.Order;
using FluentValidation;
using System.Text.RegularExpressions;

namespace TestBenchmarkDotnet.TestRegexFluentValidation;

[MemoryDiagnoser]
[Orderer(SummaryOrderPolicy.FastestToSlowest)]
[RankColumn]
[HideColumns(Column.Error, Column.Median, Column.RatioSD, Column.StdDev)]
public partial class BenchmarkRegexFluentValidation
{
    private ValidatorSansGeneratedRegex _validatorSansGeneratedRegex;
    private ValidatorAvecGeneratedRegex _validatorAvecGeneratedRegex;
    private TestModel _instance;

    [GlobalSetup]
    public void GlobalSetup()
    {
        _validatorSansGeneratedRegex = new();
        _validatorAvecGeneratedRegex = new();
        _instance = new() { Value = "TestString123" };
    }

    [Benchmark(Baseline = true)]
    public void ValidationSansGeneratedRegex() =>
        _ = _validatorSansGeneratedRegex.Validate(_instance).IsValid;

    [Benchmark]
    public void ValidationAvecGeneratedRegex() =>
        _ = _validatorAvecGeneratedRegex.Validate(_instance).IsValid;

    public class TestModel
    {
        public required string Value { get; set; }
    }

    public class ValidatorSansGeneratedRegex : AbstractValidator<TestModel>
    {
        public ValidatorSansGeneratedRegex()
        {
            RuleFor(x => x.Value).Matches("^[a-zA-Z0-9]*$");
        }
    }

    public partial class ValidatorAvecGeneratedRegex : AbstractValidator<TestModel>
    {
        public ValidatorAvecGeneratedRegex()
        {
            RuleFor(x => x.Value).Matches(MyRegex());
        }

        [GeneratedRegex(@"^[a-zA-Z0-9]*$", RegexOptions.Compiled)]
        private static partial Regex MyRegex();
    }
}
```

### Analyse des résultats  

```txt
| Method                       | Mean     | Ratio | Rank | Gen0   | Allocated | Alloc Ratio |
|----------------------------- |---------:|------:|-----:|-------:|----------:|------------:|
| ValidationAvecGeneratedRegex | 157.3 ns |  0.67 |    1 | 0.0715 |     600 B |        1.00 |
| ValidationSansGeneratedRegex | 234.7 ns |  1.00 |    2 | 0.0715 |     600 B |        1.00 |
```  

1. **Temps d'exécution** :  
   L'utilisation des générateurs de sources réduit le temps d'exécution de la validation d'environ 33 %, passant de 234,7 ns à 157,3 ns. Cette amélioration est due à l'optimisation apportée par le code précompilé généré au moment de la compilation.

2. **Allocations mémoire** :  
   Les allocations mémoire restent identiques dans les deux cas (600 B). Cela montre que l'amélioration des performances est principalement due à la réduction des calculs nécessaires pour traiter les expressions régulières.

3. **Classement** :  
   Le benchmark classe clairement la validation avec générateurs de sources comme la méthode la plus rapide.

Ces résultats démontrent que l'intégration des générateurs de sources peut apporter des bénéfices significatifs en termes de performance, particulièrement pour des validations répétées ou critiques en production.

## Conclusion  

Les générateurs de sources pour les expressions régulières représentent une avancée importante dans l'écosystème .NET. En utilisant l'attribut **[GeneratedRegex]**, les développeurs peuvent bénéficier d'améliorations notables en termes de vitesse d'exécution, tout en conservant une syntaxe déclarative et lisible.

Bien que **FluentValidation** ne supporte pas directement les générateurs de sources dans ses règles comme `Matches`, l'approche manuelle démontrée ici peut être utilisée pour combiner les avantages des deux outils.

Les développeurs souhaitant optimiser leurs applications .NET devraient envisager d'adopter cette technique, notamment dans les scénarios où des expressions régulières complexes ou souvent utilisées jouent un rôle clé.
