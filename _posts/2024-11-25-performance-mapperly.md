---
title: La performance de Mapperly
date: 2024-11-25 17:00:00 -0400
categories: [outil-developpement]
tags: [dotnet]
---

La semaine dernière, j’ai eu la chance de participer à Microsoft Ignite à Chicago. Cet événement riche en annonces et en échanges m’a inspiré plusieurs sujets que je partagerai sur mon blog dans les jours à venir !

Dans un [article précédent](https://codewithfrenchy.com/posts/source-generator-dotnet/), j'ai mentionné les avantages de l'utilisation des Source Generators. J'avais également présenté un exemple avec **Mapperly**, un outil qui utilise les Source Generators pour effectuer le mapping entre deux entités, évitant ainsi la réflexion. 

Curieux de comparer les performances entre [AutoMapper](https://docs.automapper.org/en/stable/), un outil fréquemment utilisé dans mes mandats, et **Mapperly**, j'ai décidé de réaliser quelques benchmarks !

## Mise en place de l'environnement

J'ai commencé par créer un projet .NET 9 et j'ai ajouté la structure de base pour utiliser **BenchmarkDotNet**. 

J'ai défini une classe `Personne` et un DTO `PersonneDto` :

```csharp
public class Personne
{
    public required string Prenom { get; set; }
    public required string Nom { get; set; }
    public required int Age { get; set; }
    public required string Adresse { get; set; }
    public required string Ville { get; set; }
    public required string Province { get; set; }
}

public class PersonneDto
{
    public required string Prenom { get; set; }
    public required string Nom { get; set; }
    public required string Adresse { get; set; }
    public required string Ville { get; set; }
    public required string Province { get; set; }
}
```

Pour Mapperly, j'ai créé le mapper suivant :

```csharp
[Mapper]
public partial class PersonneMapper
{
    public partial PersonneDto PersonneToPersonneDto(Personne personne);
}
```

## Configuration des benchmarks

J'ai configuré mes benchmarks pour itérer sur 1, 100 et 1 000 fois :

```csharp
[MemoryDiagnoser]
[Orderer(SummaryOrderPolicy.FastestToSlowest)]
[RankColumn]
[HideColumns(Column.Error, Column.Median, Column.RatioSD, Column.StdDev)]
public class BenchmarkMapperlyAutomapper
{
    private IMapper _automapper;
    private PersonneMapper _mapperly;
    private Personne _personne;

    [GlobalSetup]
    public void GlobalSetup()
    {
        _automapper = new MapperConfiguration(cfg => cfg.CreateMap<Personne, PersonneDto>()).CreateMapper();
        _mapperly = new PersonneMapper();
        _personne = new Fixture().Create<Personne>();
    }

    [Params(1, 100, 1_000)]
    public int Iteration { get; set; }

    [Benchmark(Baseline = true)]
    public List<PersonneDto> AvecAutomapper()
    {
        var personnes = new List<PersonneDto>();
        for (var i = 0; i < Iteration; i++)
        {
            var personneDto = _automapper.Map<PersonneDto>(_personne);
            personnes.Add(personneDto);
        }
        return personnes;
    }

    [Benchmark]
    public List<PersonneDto> AvecMapperly()
    {
        var personnes = new List<PersonneDto>();
        for (var i = 0; i < Iteration; i++)
        {
            var personneDto = _mapperly.PersonneToPersonneDto(_personne);
            personnes.Add(personneDto);
        }
        return personnes;
    }
}
```

## Résultats des benchmarks

Voici les résultats obtenus :

```txt
| Method         | Iteration | Mean         | Ratio | Rank | Gen0   | Gen1   | Allocated | Alloc Ratio |
|--------------- |---------- |-------------:|------:|-----:|-------:|-------:|----------:|------------:|
| AvecMapperly   | 1         |     25.61 ns |  0.32 |    1 | 0.0172 |      - |     144 B |        1.00 |
| AvecAutomapper | 1         |     80.79 ns |  1.00 |    2 | 0.0172 |      - |     144 B |        1.00 |
|                |           |              |       |      |        |        |           |             |
| AvecMapperly   | 100       |  1,506.71 ns |  0.22 |    1 | 0.9308 | 0.0229 |    7792 B |        1.00 |
| AvecAutomapper | 100       |  6,938.45 ns |  1.00 |    2 | 0.9308 | 0.0153 |    7792 B |        1.00 |
|                |           |              |       |      |        |        |           |             |
| AvecMapperly   | 1000      | 14,079.26 ns |  0.20 |    1 | 8.6670 | 1.7242 |   72600 B |        1.00 |
| AvecAutomapper | 1000      | 70,748.49 ns |  1.01 |    2 | 8.6670 | 1.7090 |   72600 B |        1.00 |
```

## Conclusion

Les résultats montrent clairement que **Mapperly** est beaucoup plus performant que **AutoMapper**. L'utilisation des Source Generators plutôt que de la réflexion porte ses fruits ! 

La grande question reste : comment l'outil se comporte-t-il avec des structures plus complexes ? Restez à l'affut pour un prochain article !

Pour en savoir plus sur Mapperly, consultez la [documentation officielle](https://mapperly.riok.app/docs/intro/).
