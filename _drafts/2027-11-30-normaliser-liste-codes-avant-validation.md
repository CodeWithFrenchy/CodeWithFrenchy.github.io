---
title: "ASP.NET Core : normaliser une liste de codes avant de la valider"
date: 2027-11-30 19:00:00 -0400
categories: [dotnet, bonnes-pratiques]
tags: [csharp, aspnet-core, normalisation, validation, fluentvalidation, linq]
---

## Préambule

Lors de l’enregistrement de nouvelles données, il est fréquent de vouloir nettoyer les valeurs reçues par notre service. Dans une API ASP.NET Core, le contrôleur reçoit un DTO, puis construit une commande qui sera traitée par l’application.

Prenons la création d’un profil auquel on associe une liste de codes de types de fonction. Ces codes correspondent à un domaine de valeurs, par exemple des données de référence dont les identifiants comptent généralement quatre ou cinq caractères. On s’attend à recevoir des valeurs comme `ADMN`, `GEST` ou `LECTR`.

Dans les faits, rien ne garantit que le client transmettra exactement cette représentation. Un code peut arriver en minuscules, être entouré d’espaces ou même être remplacé par une valeur `null`. Une propriété déclarée comme une liste de chaînes non nullables ne suffit pas à garantir la qualité de son contenu.

Quelques lignes de C# permettent d’uniformiser ces valeurs. Encore faut-il bien distinguer ce que cette transformation règle et ce que la validation doit continuer à vérifier.

## Une liste de chaînes peut contenir des valeurs inattendues

Une requête peut contenir ceci :

```json
{
  "codesTypesFonction": [
    " admn ",
    "Gest",
    null,
    "   ",
    "LECTR"
  ]
}
```

Déclarer une propriété avec le type `IReadOnlyList<string>` exprime un contrat dans lequel la collection et ses éléments sont non nullables. Cela ne constitue pas, à lui seul, une validation des données reçues. Les [types de référence nullables en C#](https://learn.microsoft.com/fr-fr/dotnet/csharp/fundamentals/null-safety/nullable-reference-types) reposent sur une analyse statique et des avertissements à la compilation.

La documentation de [System.Text.Json sur les annotations de nullabilité](https://learn.microsoft.com/fr-fr/dotnet/standard/serialization/system-text-json/nullable-annotations#limitations) précise d’ailleurs que l’option `RespectNullableAnnotations`, disponible à partir de .NET 9, ne fait pas respecter la nullabilité des éléments d’une collection. Son activation ne suffit donc pas à interdire les éléments `null` dans une collection de chaînes déclarées non nullables.

Il faut traiter séparément la présence de la collection et la validité de chaque élément.

## Normaliser les valeurs avec une projection LINQ

Lors du passage du DTO à la commande, on peut appliquer cette transformation :

```csharp
CodesTypesFonction = dto.CodesTypesFonction
    .Select(x => x?.Trim().ToUpperInvariant() ?? string.Empty)
    .ToList();
```

L’intention est de donner une représentation commune à des codes que le contrat considère équivalents.

La méthode [`Trim()`](https://learn.microsoft.com/fr-fr/dotnet/api/system.string.trim?view=net-8.0#system-string-trim) retire les caractères d’espacement au début et à la fin de chaque chaîne. Elle conserve les espaces situés à l’intérieur. Ainsi, `" admn "` devient `"admn"`, tandis que `"ad mn"` conserve son espace interne.

La méthode [`ToUpperInvariant()`](https://learn.microsoft.com/fr-fr/dotnet/api/system.string.toupperinvariant?view=net-8.0) applique ensuite les règles de casse de la culture invariante. Pour des codes techniques dont la représentation attendue est en majuscules, cela évite de faire dépendre la transformation de la culture courante du processus.

L’[opérateur conditionnel de nullité `?.`](https://learn.microsoft.com/fr-fr/dotnet/csharp/language-reference/operators/member-access-operators#null-conditional-operators--and-) empêche l’exécution de la chaîne d’appels lorsque `x` vaut `null`. Dans cette expression, ni `Trim()` ni `ToUpperInvariant()` ne seront alors exécutés. L’[opérateur `??`](https://learn.microsoft.com/fr-fr/dotnet/csharp/language-reference/operators/null-coalescing-operator) fournit ensuite `string.Empty` comme valeur de remplacement.

Cette protection reste utile à la frontière de l’application, même avec un DTO non nullable. Elle permet de transformer une valeur reçue qui ne respecte pas le contrat en une chaîne vide que le validateur pourra rejeter.

Enfin, [`ToList()`](https://learn.microsoft.com/fr-fr/dotnet/api/system.linq.enumerable.tolist?view=net-8.0) matérialise immédiatement le résultat dans une nouvelle liste. La collection du DTO n’est pas modifiée par cette projection.

Voici le résultat pour quelques entrées :

| Valeur reçue | Valeur normalisée |
| --- | --- |
| `" admn "` | `"ADMN"` |
| `"Gest"` | `"GEST"` |
| `" lectr "` | `"LECTR"` |
| `"\tadmn\t"` | `"ADMN"` |
| `null` | `""` |
| `""` | `""` |
| `"   "` | `""` |
| `"ad mn"` | `"AD MN"` |

Cette normalisation convient si les différences de casse et les espaces périphériques n’ont pas de signification métier. Elle ne devrait pas être appliquée automatiquement à toutes les chaînes de l’application.

## Déclarer un DTO non nullable et protéger la normalisation

Notre DTO déclare `CodesTypesFonction` comme une collection non nullable de chaînes non nullables. Il exprime ainsi les valeurs attendues par l’opération `CreerProfil`.

L’expression précédente protège contre les éléments `null`. Elle suppose toutefois que `dto.CodesTypesFonction` désigne une collection existante. Si une collection nulle atteint malgré tout le constructeur de la commande, on peut aussi prévoir son traitement.

Pour notre exemple de création, supposons qu’une liste nulle et une liste vide doivent produire la même erreur de validation, puisqu’au moins un code est requis. On les ramène alors à une liste vide avant de normaliser les éléments.

Voici une version simplifiée du DTO et de la commande, limitée à cette propriété :

```csharp
public sealed record CreerProfilDto(
    IReadOnlyList<string> CodesTypesFonction);

public sealed record CreerProfilCommand
{
    public CreerProfilCommand(CreerProfilDto dto)
    {
        CodesTypesFonction = (dto.CodesTypesFonction ?? [])
            .Select(x => x?.Trim().ToUpperInvariant() ?? string.Empty)
            .ToList();
    }

    public IReadOnlyList<string> CodesTypesFonction { get; }
}
```

Le DTO et la commande exposent le même type non nullable. Dans le constructeur, `?? []` protège le traitement si la collection reçue est nulle, tandis que `x?.Trim().ToUpperInvariant() ?? string.Empty` protège contre un élément nul. Après cette transformation, la commande contient une collection non nulle et des chaînes non nulles, qui peuvent toutefois être vides ou invalides.

Selon la configuration de la désérialisation et de la validation ASP.NET Core, une collection absente ou nulle peut être rejetée avant la construction de la commande. La protection présentée ici couvre le cas où cette valeur atteint quand même la commande.

Le remplacement par `[]` constitue un choix de traitement des entrées invalides. Si l’absence de la liste et une liste vide ont des significations différentes, il faut conserver cette distinction et les valider séparément. Le même raisonnement s’applique au remplacement d’un élément `null` par une chaîne vide.

## Valider les codes après leur normalisation

On parle parfois de sanitisation pour désigner ce nettoyage. Ici, il s’agit plus précisément de normaliser la représentation des codes. La validité métier reste à établir.

Après la transformation, plusieurs vérifications peuvent encore être nécessaires :

- La collection contient le nombre d’éléments attendu.
- Chaque code contient une valeur.
- La longueur et les caractères respectent le format du domaine de valeurs.
- Chaque code existe dans le référentiel et peut être utilisé dans ce contexte.
- La collection ne contient aucun doublon si ceux-ci sont interdits.

Avec FluentValidation, [`ForEach`](https://docs.fluentvalidation.net/en/latest/collections.html) permet de combiner les contrôles de la collection et ceux de ses éléments dans un seul `RuleFor`. Pour l’exemple suivant, nous imposons au moins un code, une longueur de quatre à cinq caractères et l’absence de doublons.

```csharp
using FluentValidation;

public sealed class CreerProfilCommandValidator
    : AbstractValidator<CreerProfilCommand>
{
    public CreerProfilCommandValidator()
    {
        RuleFor(x => x.CodesTypesFonction)
            .Cascade(CascadeMode.Continue)
            .NotEmpty()
            .WithMessage("Au moins un code de type de fonction est requis.")
            .Must(x => x.Distinct(StringComparer.Ordinal).Count() == x.Count)
            .WithMessage("Un même code de type de fonction ne peut pas apparaître plusieurs fois.")
            .ForEach(x => x
                .Cascade(CascadeMode.Stop)
                .NotEmpty()
                .WithMessage("Chaque code de type de fonction doit contenir une valeur.")
                .Length(4, 5)
                .WithMessage("Chaque code de type de fonction doit contenir de 4 à 5 caractères."));
    }
}
```

Le validateur [`NotEmpty`](https://docs.fluentvalidation.net/en/latest/built-in-validators.html#notempty-validator) s’applique ici à deux niveaux. Sur la collection, il impose la présence d’au moins un élément. À l’intérieur de `ForEach`, il rejette les chaînes vides ou composées uniquement d’espaces.

La condition `Must` vérifie les doublons sur la collection complète, après normalisation. Elle peut accéder à la collection sans vérifier sa nullité, puisque le constructeur de `CreerProfilCommand` garantit sa présence.

Les [modes de cascade](https://docs.fluentvalidation.net/en/latest/cascade.html) précisent comment enchaîner ces contrôles. Sur la règle principale, `CascadeMode.Continue` permet d’exécuter les contrôles des éléments même si des doublons sont présents. À l’intérieur de `ForEach`, `CascadeMode.Stop` évite d’ajouter une erreur de longueur lorsqu’une valeur est vide.

La règle de longueur est illustrative. Des codes qui comptent généralement quatre ou cinq caractères ne justifient pas automatiquement une restriction à ces deux longueurs. La règle doit refléter le contrat réel du référentiel.

Cet exemple ne vérifie pas encore l’appartenance au domaine de valeurs. Le code `ZZZZ` respecte la longueur demandée, mais peut être inconnu. De même, `AD MN` compte cinq caractères sans nécessairement correspondre à un code autorisé. Ces contrôles doivent être ajoutés à partir de la source de référence de l’application.

La validation doit être exécutée avant le traitement de la commande et l’enregistrement. Déclarer une classe de validation ne suffit pas à la faire intervenir dans le parcours d’exécution.

## Conserver les erreurs pour pouvoir les signaler

Il pourrait être tentant d’ajouter un filtre pour supprimer les chaînes vides, puis un appel à `Distinct()` pour retirer les doublons.

Dans un contrat qui interdit ces entrées, ces transformations masqueraient des erreurs du client. Une requête contenant un code valide et plusieurs valeurs invalides pourrait être traitée partiellement sans que l’appelant le réalise.

L’expression initiale conserve le nombre d’éléments et leur ordre. Une entrée `null` devient vide, mais reste présente pour que le validateur puisse la signaler.

La normalisation doit également précéder la détection des doublons. Les valeurs `"admn"` et `" ADMN "` deviennent toutes les deux `"ADMN"`. Dans l’exemple, la comparaison ordinale porte donc sur les codes déjà normalisés.

Supprimer les valeurs vides ou dédupliquer peut être un comportement acceptable lorsque le contrat le prévoit explicitement. Ce choix doit correspondre au fonctionnement attendu de l’opération.

## Placer la normalisation à une frontière claire

Le passage du DTO à la commande est un endroit approprié lorsque cette commande représente l’entrée du cas d’utilisation. Les règles de validation et le traitement travaillent alors sur la même représentation des codes.

Le parcours devient explicite : réception du DTO, normalisation, validation de la commande, puis traitement et enregistrement.

Si plusieurs points d’entrée alimentent le même cas d’utilisation, ils doivent appliquer les mêmes règles. Une API, un import et un traitement en arrière-plan ne devraient pas interpréter différemment le même code. Une fonction partagée peut être extraite lorsque cette répétition apparaît.

Pour une transformation aussi courte, l’expression LINQ reste un point de départ lisible. Le besoin de centralisation doit venir de la réutilisation et de la cohérence à maintenir.

## Conclusion

Normaliser une liste de codes demande peu de code. `Trim()` retire les espaces périphériques, `ToUpperInvariant()` harmonise la casse et `?? string.Empty` permet de traiter les éléments nuls de façon uniforme.

Un DTO déclaré avec `IReadOnlyList<string>` exprime le contrat attendu. Les protections à la frontière de l’application permettent de traiter les valeurs qui ne le respectent pas, puis la validation les rejette avec un message adapté.

La décision essentielle concerne le contrat. Quelles différences peut-on corriger sans changer le sens de la demande, et quelles entrées doivent être rejetées avec un message clair ?

En normalisant les codes avant de les valider, on simplifie les comparaisons et on repère les doublons sous leur forme commune. La validation conserve ensuite son rôle, qui est de vérifier que les valeurs reçues sont réellement acceptables pour le cas d’utilisation.
