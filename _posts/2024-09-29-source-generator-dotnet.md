---
title: Les Générateurs de source
date: 2024-09-29 18:00:00 -0400
categories: []
tags: [dotnet, source-generator, source-generator]
---

Les [Source Generators](https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/source-generators-overview) (Générateurs de source) sont des outils intégrés au compilateur qui génèrent automatiquement du code source lors de la compilation, permettant aux développeurs d’automatiser la création de portions de code répétitives ou complexes tout en gardant le code généré visible et modifiable. [Introduits avec .NET 5](https://devblogs.microsoft.com/dotnet/introducing-c-source-generators/#:~:text=A%20Source%20Generator%20is%20a%20.NET%20Standard%202.0%20assembly%20that#:~:text=A%20Source%20Generator%20is%20a%20.NET%20Standard%202.0%20assembly%20that), ils sont rapidement devenus un élément clé de l’écosystème .NET. Contrairement à des techniques comme la réflexion, qui génèrent du code dynamiquement à l'exécution, les Source Generators produisent du code statique, optimisé et vérifié lors de la compilation.

## La révolution sur le quotidien

L'arrivée des Source Generators a considérablement transformé la manière de travailler au quotidien, surtout pour les développeurs confrontés à du *boilerplate code* ou des scénarios nécessitant une génération de code personnalisée.

Voici comment ils révolutionnent le développement :

1. **Réduction du code récurrent** : Les générateurs de code prennent en charge la génération de classes de mapping, DTOs, modèles de validation, ou tout autre code répétitif, permettant ainsi de se concentrer sur des tâches à plus forte valeur ajoutée.
2. **Meilleures performances** : Le code étant généré à la compilation, il est plus performant que les approches reposant sur des mécanismes comme la réflexion, qui sont dynamiques et coûteux en termes de performance.
3. **Centralisation de la logique** : Les règles de génération peuvent être définies une fois et appliquées de manière uniforme dans tout le projet, améliorant la maintenabilité et réduisant les erreurs.
4. **Compatibilité AOT** : L’un des avantages cruciaux des Source Generators est qu’ils ne reposent pas sur la **réflexion**, une technique incompatible avec le mode de compilation **Ahead-Of-Time (AOT)**. AOT est utilisé pour optimiser les applications .NET destinées à des environnements contraints en ressources, comme les applications mobiles et les conteneurs. En remplaçant la réflexion par du code généré statiquement, les Source Generators rendent le code compatible avec AOT et permettent ainsi une exécution plus rapide et plus légère.

## Exemple avec Mapperly

[Mapperly](https://github.com/riok/mapperly) est un exemple parfait d'outil tirant parti des Source Generators pour automatiser les tâches de **mapping** entre objets, tout en éliminant la dépendance à la réflexion.

Imaginons que vous ayez une classe `Personne` et un DTO `PersonneDto`, et que vous souhaitiez les mapper :

```csharp
public class Personne
{
    public string Prenom { get; set; }
    public string Nom { get; set; }
    public DateTime DateNaissance { get; set; }
}

public class PersonneDto
{
    public string Prenom { get; set; }
    public string Nom { get; set; }
}
```

Avec Mapperly, il vous suffit de créer un mapper sous la forme d’une classe partielle et d’appliquer l’attribut `[Mapper]`.

```csharp
[Mapper]
public partial class PersonneMapper
{
    public partial PersonneDto MapToDto(Personne personne);
}
```

À la compilation, Mapperly génère automatiquement le code nécessaire pour mapper les propriétés correspondantes :

```csharp
public partial class PersonneMapper
{
    public partial PersonneDto MapToDto(Personne personne)
    {
        return new PersonneDto
        {
            Prenom = personne.Prenom,
            Nom = personne.Nom
        };
    }
}
```

Ici, Mapperly remplace la réflexion, souvent utilisée dans des solutions comme [AutoMapper](https://docs.automapper.org/en/stable/), par du code statique et optimisé. Ce qui, en plus de rendre le mapping plus rapide, le rend compatible avec le mode **AOT**, où la réflexion n’est pas une option possible.

## Outils à surveiller pour exploiter pleinement les Source Generators

L’écosystème .NET regorge d’outils qui exploitent les Source Generators pour simplifier et automatiser certaines tâches courantes du développement. 

Voici une sélection d’outils à surveiller qui pourraient transformer vos pratiques de développement :

1. [Mapperly](https://github.com/riok/mapperly) - **Mapperly**, comme mentionné précédemment, est un outil qui génère du code de **mapping** entre objets de manière statique. Contrairement à [AutoMapper](https://docs.automapper.org/en/stable/), qui utilise la réflexion, Mapperly repose sur des Source Generators, ce qui le rend beaucoup plus performant et compatible avec des environnements **AOT**.
2. [TUnit](https://github.com/thomhurst/TUnit) - **TUnit** est une bibliothèque de tests unitaires qui utilise les Source Generators pour générer des méthodes de test à la volée. Cela permet de réduire considérablement le code à écrire pour chaque test, en particulier pour les scénarios répétitifs où seule la donnée testée varie.
3. [CoreWCF](https://github.com/CoreWCF/CoreWCF) - **CoreWCF** est un port de **Windows Communication Foundation (WCF)** pour .NET Core. Il utilise les Source Generators pour générer automatiquement des contrats de service et des proxy, facilitant la migration des applications WCF existantes vers des technologies modernes comme **.NET 6+**.
4. [Enum.Source.Generator](https://github.com/EngRajabi/Enum.Source.Generator) - Cet outil permet de générer du code à partir des énumérations définies dans vos projets. Plutôt que d’écrire manuellement des méthodes pour obtenir des valeurs ou des descriptions associées à chaque enum, **Enum.Source.Generator** génère automatiquement ce code pour vous, réduisant ainsi les risques d’erreurs et la charge de maintenance.
5. [Dunet](https://github.com/domn1995/dunet) - **Dunet** est une bibliothèque pour faciliter la création de **types discriminés** (*discriminated unions*), une fonctionnalité très puissante que l'on retrouve dans des langages comme **F#**. Dunet génère automatiquement le code nécessaire pour gérer ces types discriminés en .NET, simplifiant grandement la gestion des cas particuliers dans les applications.

## Conclusion

Les **Source Generators** représentent une avancée majeure, permettant aux développeurs d'écrire moins de code répétitif et d'améliorer les performances globales de leurs applications. En éliminant la nécessité de la réflexion dans certains scénarios critiques, comme le mapping d'objets ou la génération de proxy, ils permettent également une meilleure compatibilité avec les environnements **AOT**. Des outils comme **Mapperly**, **TUnit**, **CoreWCF**, **Enum.Source.Generator** et **Dunet** montrent bien le potentiel transformateur de ces outils.

Restez attentif à ces outils qui permettent de tirer pleinement parti des capacités de .NET !
