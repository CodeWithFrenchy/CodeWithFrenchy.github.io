---
title: Validation asynchrone avec DataAnnotations dans .NET 11
date: 2027-12-31 19:00:00 -0400
categories: [dotnet]
tags: [dotnet11, aspnet-core]
---

## Préambule

Les DataAnnotations permettent d'exprimer simplement des règles de validation sur nos modèles. Une propriété obligatoire, une longueur maximale ou un format de courriel se décrivent avec quelques attributs bien connus.

La situation se complique lorsqu'une règle doit consulter une base de données ou appeler un service externe. Comment vérifier qu'une adresse courriel est disponible lorsque la méthode de validation ne permet pas d'utiliser `await`?

.NET 11 apporte une réponse avec la validation asynchrone dans `System.ComponentModel.DataAnnotations`. Cet ajout rend les attributs plus utiles pour les validations qui dépendent de ressources externes. Reste à déterminer où cette approche convient et si elle justifie de revoir nos solutions existantes.

## Pourquoi cet ajout est intéressant

Historiquement, `ValidationAttribute.IsValid` et `IValidatableObject.Validate` reposent sur des contrats synchrones. Pour une vérification en mémoire, cette approche suffit largement.

Pour une requête SQL ou HTTP, certaines implémentations finissent toutefois par bloquer une opération asynchrone avec `.Result` ou `.GetAwaiter().GetResult()`. Les [recommandations de performance d'ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/best-practices?view=aspnetcore-10.0#avoid-blocking-calls) expliquent pourquoi ce blocage peut épuiser les threads disponibles et dégrader les temps de réponse sous charge. Ajouter `Task.Run` autour de l'appel ne règle pas ce problème.

Il était déjà possible d'effectuer une validation asynchrone dans un service applicatif ou avec une bibliothèque spécialisée. La nouveauté est de pouvoir l'exprimer directement dans les DataAnnotations et de l'exécuter avec des API adaptées.

**Le principal bénéfice est de ne plus immobiliser un thread pendant l'attente d'une opération d'entrée-sortie.** La requête SQL ne devient pas automatiquement plus rapide, mais le serveur peut mieux utiliser ses ressources pendant cette attente.

## Ce que .NET 11 ajoute

La [documentation des nouveautés des bibliothèques .NET 11](https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-11/libraries#asynchronous-validation-with-dataannotations) présente trois points d'entrée complémentaires.

| Ajout | Utilisation |
| --- | --- |
| `AsyncValidationAttribute` | Créer un attribut dont la règle peut attendre une opération asynchrone |
| `IAsyncValidatableObject` | Définir une validation à l'échelle d'un objet, notamment lorsqu'elle concerne plusieurs propriétés |
| Les méthodes asynchrones de `Validator` | Déclencher explicitement une validation avec `TryValidateObjectAsync`, `ValidateObjectAsync` ou leurs variantes pour une propriété ou une valeur |

Les attributs habituels comme `[Required]` et `[EmailAddress]` continuent de servir. Un même modèle peut combiner des règles synchrones et asynchrones.

Le [contrat intégré au runtime](https://github.com/dotnet/runtime/commit/be1d5629e1a191e84a330f45c49f37890a7da451) impose aussi un choix explicite pour les appels synchrones. Un attribut dérivé de `AsyncValidationAttribute` doit implémenter `IsValidAsync`, mais également la surcharge protégée de `IsValid`. De même, `IAsyncValidatableObject` hérite de `IValidatableObject` et nécessite les deux méthodes de validation.

Lorsque la règle exige une opération asynchrone, lever une exception depuis son point d'entrée synchrone permet de signaler une mauvaise utilisation. Retourner un succès sans exécuter la règle ferait passer une donnée pour valide sans l'avoir vérifiée.

## Un exemple concret avec une adresse courriel

Prenons une demande de création de compte. Le courriel doit être renseigné, avoir un format acceptable et ne pas être déjà utilisé.

```csharp
using System.ComponentModel.DataAnnotations;

public sealed class CreerCompte
{
    [Required(ErrorMessage = "L'adresse courriel est obligatoire.")]
    [EmailAddress(ErrorMessage = "Le format du courriel est invalide.")]
    [CourrielDisponible]
    public string Courriel { get; init; } = string.Empty;
}
```

La consultation du registre des utilisateurs est confiée à un service. Son implémentation pourrait utiliser EF Core ou un client HTTP.

```csharp
using System.ComponentModel.DataAnnotations;
using Microsoft.Extensions.DependencyInjection;

public interface IRegistreUtilisateur
{
    Task<bool> EstCourrielExistant(
        string courriel,
        CancellationToken cancellationToken);
}

[AttributeUsage(AttributeTargets.Property)]
public sealed class CourrielDisponibleAttribute : AsyncValidationAttribute
{
    public CourrielDisponibleAttribute()
        : base("L'adresse courriel est déjà utilisée.")
    {
    }

    protected override ValidationResult? IsValid(
        object? value,
        ValidationContext validationContext)
    {
        throw new InvalidOperationException(
            "Cette règle exige une validation asynchrone.");
    }

    protected override async Task<ValidationResult?> IsValidAsync(
        object? value,
        ValidationContext validationContext,
        CancellationToken cancellationToken)
    {
        var courriel = (string?)value;

        if (string.IsNullOrWhiteSpace(courriel))
        {
            return ValidationResult.Success;
        }

        var registre = validationContext.GetRequiredService<IRegistreUtilisateur>();

        var estCourrielExistant = await registre.EstCourrielExistant(
            courriel,
            cancellationToken);

        if (!estCourrielExistant)
        {
            return ValidationResult.Success;
        }

        return new ValidationResult(
            FormatErrorMessage(validationContext.DisplayName),
            validationContext.MemberName is { } membre ? [membre] : null);
    }
}
```

L'attribut laisse la présence du courriel à `[Required]`. Il récupère son service dans le `ValidationContext`, transmet le jeton d'annulation et associe l'erreur à la propriété concernée.

Pour un appel explicite, le fragment suivant s'exécute dans une méthode asynchrone. `requete` est l'objet à valider, `services` représente le fournisseur de services de la portée courante et `cancellationToken` provient de l'opération appelante.

```csharp
var erreurs = new List<ValidationResult>();
var contexte = new ValidationContext(requete, services, items: null);

var estValide = await Validator.TryValidateObjectAsync(
    requete,
    contexte,
    erreurs,
    validateAllProperties: true,
    cancellationToken: cancellationToken);
```

Une implémentation de `IRegistreUtilisateur` doit être enregistrée dans le conteneur. Le paramètre `validateAllProperties: true` est également essentiel pour demander l'évaluation de tous les attributs de propriété, au-delà des seuls attributs `[Required]`.

`estValide` indique le résultat et `erreurs` contient les échecs de validation. Une panne du service appelé reste une défaillance technique susceptible de lever une exception, même avec une méthode nommée `TryValidateObjectAsync`.

## Quelle intégration dans ASP.NET Core?

L'ajout des API dans le runtime et leur utilisation automatique par un framework sont deux étapes distinctes.

Les [nouveautés d'ASP.NET Core 11](https://learn.microsoft.com/en-us/aspnet/core/release-notes/aspnetcore-11?view=aspnetcore-11.0#async-validation-for-minimal-apis) confirment la prise en charge asynchrone dans les Minimal APIs. La documentation décrit également son utilisation dans les formulaires Blazor.

| Contexte | Comportement documenté pour .NET 11 |
| --- | --- |
| Appel explicite à `Validator` | Utiliser les nouvelles méthodes asynchrones |
| Minimal APIs | Activer la validation avec `builder.Services.AddValidation()`. Le framework attend le résultat des vérifications asynchrones avant d'exécuter le code du point de terminaison. Si la validation échoue, ce code n'est pas exécuté |
| Formulaires Blazor | `DataAnnotationsValidator` prend en charge les règles asynchrones. La soumission par `EditForm` attend leur résultat |
| Contrôleurs MVC et Razor Pages | Leur validation automatique ne bénéficie pas de cette nouvelle intégration |

La [documentation de la validation ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/validation?view=aspnetcore-11.0) précise cette exclusion de MVC et Razor Pages. La [demande de prise en charge pour MVC](https://github.com/dotnet/aspnetcore/issues/31905) demeure suivie séparément.

**Passer un projet avec contrôleurs à .NET 11 ne suffit donc pas à rendre sa validation automatique asynchrone.** Ajouter l'attribut de notre exemple au modèle d'une action MVC pourrait atteindre son `IsValid` synchrone et déclencher l'exception. Dans ce contexte, je conserverais une validation asynchrone explicitement orchestrée dans la couche applicative.

## Les limites à garder en tête

### Une vérification préalable ne garantit pas l'unicité

Deux demandes concurrentes peuvent vérifier le même courriel, constater qu'il est disponible et tenter de créer un compte.

Dans cet exemple, je conserverais un [index unique dans la base de données](https://learn.microsoft.com/en-us/ef/core/modeling/indexes#index-uniqueness), cohérent avec la normalisation du courriel, ainsi qu'une gestion du conflit à l'enregistrement. La validation préalable améliore le retour à l'utilisateur. La contrainte protège les données au moment de l'écriture.

### Les règles peuvent s'exécuter en parallèle

L'[implémentation asynchrone de `Validator`](https://github.com/dotnet/runtime/pull/128656) prévoit l'exécution concurrente de certaines validations. Il faut donc éviter de supposer que tous les attributs seront exécutés séquentiellement.

Ce point compte particulièrement avec EF Core. La [documentation de `DbContext`](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/#avoiding-dbcontext-threading-issues) interdit plusieurs opérations simultanées sur une même instance.

Si plusieurs règles consultent la base de données, un `DbContext` partagé dans la portée de la requête peut poser problème. Selon le besoin, je regrouperais les vérifications dans un service applicatif ou j'utiliserais des contextes distincts créés avec `IDbContextFactory<TContext>`. Cette seconde approche demande aussi de maîtriser le nombre de requêtes lancées.

### Chaque consultation externe a un coût

Un attribut de validation peut maintenant déclencher un appel réseau. Une propriété qui paraît anodine peut donc ajouter de la latence et une dépendance à la disponibilité d'un autre service.

Je limiterais ces règles à des consultations ciblées, sans effet de bord, avec un délai maximal adapté. Une indisponibilité technique doit rester distincte d'une donnée invalide. Pour un formulaire validé fréquemment ou une collection volumineuse, regrouper les consultations peut être préférable à un appel par propriété.

## Faut-il adopter cette fonctionnalité à la sortie de .NET 11?

**Je la considérerais en priorité pour un projet qui utilise déjà DataAnnotations et qui a quelques règles asynchrones simples à ajouter.** Elle peut éviter d'introduire une bibliothèque uniquement pour ce besoin, à condition que le mécanisme qui déclenche la validation prenne en charge l'asynchronisme.

Pour une application existante, je commencerais par un cas limité. Je vérifierais l'exécution réelle de la règle, la remontée des erreurs, l'annulation et le comportement lorsque la dépendance est indisponible.

Je ne remplacerais toutefois pas une solution bien établie simplement parce que .NET propose maintenant cette possibilité. [FluentValidation prend déjà en charge les règles asynchrones](https://docs.fluentvalidation.net/en/latest/async.html), notamment avec `MustAsync` et `CustomAsync`.

Lorsque les règles deviennent nombreuses, dépendent du scénario ou nécessitent plusieurs services, je préfère généralement des validateurs dédiés avec des dépendances explicites. Le recours à `ValidationContext.GetRequiredService` dans un attribut reste moins visible qu'une injection par constructeur. Les opérations métier transactionnelles, comme réserver une quantité ou autoriser un changement d'état, demandent également une orchestration qui dépasse la validation d'un modèle.

## Conclusion

La validation asynchrone avec DataAnnotations comble une limite concrète de .NET. Elle permet d'attendre une consultation externe tout en conservant une manière familière de déclarer des règles sur les modèles.

À la sortie de .NET 11, ce sera une option pertinente à évaluer pour les besoins simples et les intégrations compatibles. Mon choix dépendra surtout de la lisibilité des règles, de leurs dépendances et de leur coût d'exécution. Une nouveauté utile mérite d'être adoptée lorsqu'elle simplifie le code et répond à un besoin réel.
