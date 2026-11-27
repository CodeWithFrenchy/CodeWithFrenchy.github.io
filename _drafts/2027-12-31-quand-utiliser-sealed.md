---
title: "Utiliser sealed quand l’héritage n’est pas prévu"
date: 2027-12-31 19:00:00 -0400
categories: [dotnet, bonnes-pratiques]
tags: [csharp, sealed, heritage, conception-logicielle, tests-unitaires, idisposable, maintenabilite]
---

> ⚠️ À RELIRE + valider le format du code

## Préambule

Lorsqu’on crée une classe en C#, on pense à sa responsabilité, à ses dépendances et aux méthodes qu’elle expose. En revanche, la possibilité d’en hériter est souvent laissée au comportement par défaut du langage.

Une classe peut convenir parfaitement à son usage actuel sans avoir été conçue pour servir de classe de base. Dans le code applicatif, je privilégie donc cette règle : lorsqu’une classe n’a pas été conçue pour être héritée, je la déclare `sealed`.

Cette convention est particulièrement utile pour les records qui transportent des données et pour les classes de tests. Elle simplifie aussi la gestion des ressources lorsqu’une classe de tests implémente `IDisposable`.

## Ce que signifie réellement sealed

Appliqué à une classe, le mot-clé [`sealed`](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/sealed) empêche de créer une classe dérivée.

Prenons un validateur simple :

```csharp
public sealed class DossierValidator
{
    public bool EstNomValide(string nom) =>
        !string.IsNullOrWhiteSpace(nom);
}
```

On peut créer un `DossierValidator`, appeler ses méthodes et l’utiliser comme dépendance. Le compilateur refusera toutefois qu’un autre type en hérite.

La déclaration indique donc que cette classe fournit une implémentation concrète et ne constitue pas un point d’extension par héritage.

Il faut distinguer cette décision de la possibilité de redéfinir une méthode. En C#, une méthode ordinaire n’est pas redéfinissable avec `override`. Le [polymorphisme par héritage](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/object-oriented/polymorphism) repose notamment sur les membres `virtual` et `abstract`.

Une classe sans méthode virtuelle peut tout de même servir de classe de base. `sealed` exprime cette restriction pour le type entier.

## L’héritage fait partie du contrat d’une classe

Laisser une classe ouverte « au cas où » semble parfois plus flexible. Pourtant, concevoir une classe de base demande de répondre à plusieurs questions : quels comportements une classe dérivée peut-elle modifier? Quelles règles doit-elle préserver? Dans quel ordre les opérations doivent-elles être exécutées? Comment les ressources seront-elles libérées si plusieurs niveaux de la hiérarchie en possèdent?

Ces questions ont un coût de conception et de maintenance. Pour moi, une classe destinée à l’héritage devrait avoir des réponses claires. Un service qui applique une règle précise ou un objet qui représente une demande n’a pas nécessairement besoin de ce contrat supplémentaire.

Avec `sealed`, le lecteur connaît la limite choisie. Si un besoin d’héritage apparaît dans le code que l’équipe maîtrise, la conception pourra être réexaminée et les points d’extension ainsi que les garanties attendues seront définis avant d’ouvrir la classe.

## Les records : sealed pour les types qui ne servent pas de base

Les records conviennent bien à cette convention lorsqu’ils représentent des DTO, des commandes ou des messages autonomes.

Par exemple :

```csharp
public sealed record DossierDto(
    int Id,
    string Nom);
```

Ce type représente les données d’un dossier. Si aucun scénario métier ne prévoit de le spécialiser par héritage, je préfère l’indiquer dès sa déclaration.

`sealed` conserve les usages habituels d’un record, notamment l’égalité par valeur et les expressions `with` :

```csharp
var dossier = new DossierDto(
    42,
    "Autorisation de travaux");

var dossierRenomme = dossier with
{
    Nom = "Autorisation de rénovation"
};
```

L’expression produit une nouvelle instance sans créer de classe dérivée.

### Un record scellé n’est pas automatiquement immuable

`sealed` contrôle l’héritage, pas la modification des données.

Un record peut exposer des propriétés modifiables. Même avec des propriétés `init`, les objets référencés peuvent rester modifiables, par exemple une liste. La [documentation des records](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/record) distingue ces aspects.

Il faut donc examiner séparément la possibilité d’hériter du type et celle de modifier son état.

### Certains records sont volontairement des classes de base

Tous les records ne devraient pas nécessairement être scellés. Une [hiérarchie de records](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-9.0/records.md#inheritance) peut être un choix métier explicite :

```csharp
public abstract record EvenementDossier(int IdDossier);

public sealed record DossierCree(
    int IdDossier,
    string Nom) : EvenementDossier(IdDossier);

public sealed record DossierFerme(
    int IdDossier,
    string Motif) : EvenementDossier(IdDossier);
```

Ici, `EvenementDossier` est volontairement ouvert à l’héritage. Les deux événements concrets sont scellés. D’autres records peuvent encore dériver directement de `EvenementDossier`.

Un `record struct` ne peut pas être hérité. Il n’est donc pas nécessaire de lui ajouter `sealed` :

```csharp
public readonly record struct IdentifiantDossier(int Valeur);
```

Ma convention concerne les *record classes* qui ne sont pas conçues comme types de base.

## Les classes de tests sont aussi des classes concrètes

Une classe de tests regroupe généralement des scénarios autour d’un comportement. Si elle ne sert pas de base à d’autres classes de tests, elle peut elle aussi être scellée.

Voici un exemple avec xUnit :

```csharp
using Xunit;

public sealed class DossierValidatorTests
{
    [Fact]
    public void EstNomValide_NomAvecEspaces_RetourneFaux()
    {
        var sut = new DossierValidator();

        var resultat = sut.EstNomValide("   ");

        Assert.False(resultat);
    }
}
```

Cette déclaration indique que la classe contient ses propres scénarios et ne sert pas de socle à spécialiser.

Une classe de base commune peut être justifiée. Dans ce cas, on la conçoit comme telle et on peut garder ses classes dérivées concrètes scellées.

Le partage de code ne nécessite pas toujours une hiérarchie. Une méthode utilitaire, un constructeur de données de tests ou une fixture peuvent répondre au besoin. Le choix dépend de ce qu’on souhaite partager : des opérations, des données ou une ressource avec une durée de vie particulière. `sealed` rend ce choix visible dans le code de tests.

## IDisposable : un cas où la fermeture simplifie le code

Avec xUnit, une nouvelle instance de la classe de tests est créée pour chaque test exécuté. Si la classe implémente `IDisposable`, xUnit appelle sa méthode `Dispose()` pour le nettoyage. Ce fonctionnement est décrit dans la [documentation sur le contexte des tests](https://xunit.net/docs/shared-context).

Considérons un lecteur qui normalise le nom d’un dossier à partir d’un flux de texte :

```csharp
using System.IO;

public sealed class LecteurDossier
{
    public string LireNom(TextReader source) =>
        source.ReadLine()?.Trim() ?? string.Empty;
}
```

Le lecteur reçoit une source qu’il ne possède pas. Le code qui crée cette source reste responsable de sa durée de vie.

Dans les tests, on peut utiliser un `StringReader` :

```csharp
using System;
using System.IO;
using Xunit;

public sealed class LecteurDossierTests : IDisposable
{
    private readonly StringReader _source = new("  Autorisation de travaux  ");

    [Fact]
    public void LireNom_NomEntoureDEspaces_RetourneNomSansEspaces()
    {
        var sut = new LecteurDossier();

        var nom = sut.LireNom(_source);

        Assert.Equal("Autorisation de travaux", nom);
    }

    [Fact]
    public void LireNom_SourceEpuisee_RetourneChaineVide()
    {
        var sut = new LecteurDossier();
        sut.LireNom(_source);

        var nom = sut.LireNom(_source);

        Assert.Equal(string.Empty, nom);
    }

    public void Dispose() =>
        _source.Dispose();
}
```

Chaque test dispose de sa propre instance de `_source`. Le premier test ne consomme donc pas les données du second.

La classe de tests possède la ressource et son nettoyage délègue à `StringReader.Dispose()`. Cette forme suffit ici : la classe est scellée, ne possède pas de finaliseur et ne gère pas directement de ressource native. La documentation .NET présente cette [implémentation simple de `IDisposable`](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-dispose) pour une classe scellée propriétaire d’un objet disposable.

### Pourquoi une classe ouverte demande davantage de réflexion

Si `LecteurDossierTests` restait ouverte, une classe dérivée pourrait ajouter ses propres ressources. Il faudrait alors prévoir comment elle participe au nettoyage.

Pour une classe de base qui introduit `IDisposable`, le pattern extensible habituel comporte une méthode publique `Dispose()` et une méthode `protected virtual Dispose(bool)`. Les classes dérivées complètent cette dernière et appellent l’implémentation de base.

La règle d’analyse [CA1063](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca1063) vérifie notamment ce contrat. Ce n’est pas l’interface `IDisposable` elle-même qui impose la surcharge avec un booléen. C’est le pattern utilisé pour permettre un nettoyage cohérent dans une hiérarchie.

Dans notre exemple, aucun besoin d’héritage ne justifie cette infrastructure. `sealed` permet de garder un nettoyage simple, adapté à la conception retenue.

### La responsabilité de libérer les ressources reste entière

`sealed` ne déclenche aucun nettoyage automatique. Dans cet exemple, c’est xUnit qui appelle `Dispose()`.

Le [pattern de libération des ressources](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/dispose-pattern) prévoit aussi que la méthode supporte plusieurs appels sans répéter le nettoyage ni provoquer d’erreur. Selon les ressources utilisées, on peut déléguer à leur propre implémentation ou suivre explicitement l’état de l’objet.

Il n’est pas nécessaire d’ajouter un finaliseur à cette classe. Elle ne gère pas directement de ressource native. De plus, `GC.SuppressFinalize(this)` n’apporterait rien puisqu’elle est scellée et n’a pas de finaliseur.

Si la ressource ne sert qu’à un seul test, un `using` local peut suffire. Si elle appartient à une fixture partagée, son nettoyage relève de cette fixture, à la fin de sa durée de vie. C’est la propriété de la ressource qui détermine où placer le nettoyage.

## Une classe scellée peut toujours participer au polymorphisme

Sceller une implémentation reste compatible avec l’utilisation d’une interface.

Par exemple :

```csharp
public interface IFormateurDossier
{
    string Formater(DossierDto dossier);
}

public sealed class FormateurDossier : IFormateurDossier
{
    public string Formater(DossierDto dossier) =>
        $"{dossier.Id} - {dossier.Nom}";
}
```

Un consommateur peut dépendre de `IFormateurDossier`. Une autre classe peut implémenter cette interface, et un décorateur peut enrichir son comportement en recevant une implémentation existante.

Le point d’extension est alors l’interface. `FormateurDossier` reste une implémentation concrète qu’on ne peut pas spécialiser par héritage.

Il n’est pas nécessaire de créer une interface pour chaque classe scellée. Une classe simple peut être utilisée et testée directement. L’abstraction doit répondre à un besoin de conception.

## Dans quels cas conserver une classe ouverte?

Je garderais une classe ouverte lorsqu’un besoin concret le justifie, notamment lorsqu’elle est conçue pour servir de classe de base ou lorsqu’une contrainte technique l’exige.

Par exemple, les [proxies de chargement différé d’EF Core](https://learn.microsoft.com/en-us/ef/core/querying/related-data/lazy) reposent sur des classes dont ils peuvent hériter et sur des propriétés de navigation redéfinissables. Une entité utilisée de cette manière ne peut pas être scellée.

Certains outils de doublures de tests utilisent également l’héritage pour substituer une classe concrète. Dans ce cas, `sealed` peut empêcher leur fonctionnement. La [documentation de Moq](https://github.com/devlooped/moq/wiki/Quickstart) rappelle le rôle des membres redéfinissables. On peut tester directement une classe scellée, mais pas la substituer par une classe dérivée.

Le contexte d’une bibliothèque publique demande aussi une décision spécifique. Les [Framework Design Guidelines sur la fermeture des classes](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/sealing) sont historiquement plus favorables à l’extensibilité des bibliothèques réutilisables. Leurs utilisateurs peuvent avoir des besoins d’extension que l’auteur n’avait pas anticipés.

La convention présentée ici s’applique donc au code applicatif. Ce n’est pas une règle universelle imposée par C# ou .NET.

Si une classe publique est déjà distribuée dans un package, lui ajouter `sealed` peut casser les consommateurs qui en héritent. Les [règles de compatibilité .NET](https://learn.microsoft.com/en-us/dotnet/core/compatibility/library-change-rules) identifient cette modification comme incompatible pour un type auparavant ouvert à l’héritage.

## Les performances peuvent en profiter

Le JIT peut tirer parti d’un type scellé lorsqu’il connaît suffisamment le type visé. Il peut notamment remplacer certains appels virtuels par des appels directs, puis éventuellement intégrer le corps de la méthode appelée.

Stephen Toub décrit ces possibilités dans la [proposition de l’analyseur de fermeture des types internes](https://github.com/dotnet/runtime/issues/49944).

Le résultat dépend du code, du type connu au point d’appel et des optimisations du runtime. Ajouter `sealed` ne garantit donc pas une accélération mesurable pour chaque classe ou chaque test.

Dans cette convention, le premier bénéfice attendu reste la lisibilité du contrat. Les gains éventuels de performance sont un avantage supplémentaire à mesurer lorsqu’ils comptent pour l’application.

Pour accompagner cette pratique, l’analyseur [CA1852](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca1852) peut signaler les types non accessibles hors de leur assembly qui n’ont pas de classes dérivées :

```ini
[*.cs]
dotnet_diagnostic.CA1852.severity = suggestion
```

Son périmètre est plus restreint que notre convention. Il ne fera pas appliquer automatiquement `sealed` à tous les DTO publics ou à toutes les classes de tests publiques. Par défaut, il tient aussi compte de la présence d’`InternalsVisibleTo` en évitant de signaler ces types lorsqu’un assembly expose ses membres internes à un autre assembly.

## Une convention facile à appliquer en équipe

Pour les nouveaux types, je résumerais la convention ainsi :

| Type de classe | Choix privilégié |
| --- | --- |
| Classe applicative sans besoin d’héritage | `sealed class` |
| DTO, commande ou message déclaré comme record class autonome | `sealed record` |
| Classe de tests qui ne sert pas de base | `sealed class` |
| Classe de tests avec `IDisposable`, sans besoin d’héritage | `sealed class` et nettoyage adapté aux ressources possédées |
| Classe ou record volontairement utilisé comme base | Ouvert à l’héritage, avec un contrat explicite |
| Type soumis à une contrainte de proxy par héritage | Ouvert selon les exigences de l’outil |
| `record struct` | Aucun `sealed` à ajouter |

Pour le code existant, vérifiez les usages avant de modifier une API.

En révision de code, demandez-vous si la classe a été conçue pour que d’autres classes en héritent. Si ce n’est pas le cas et qu’aucune contrainte technique ne l’exige, `sealed` exprime clairement la décision.

## Conclusion

Je privilégie `sealed` pour les classes qui ne sont pas destinées à l’héritage, en particulier les records autonomes et les classes de tests.

Pour une classe de tests qui implémente `IDisposable`, cette décision permet souvent de garder une méthode `Dispose()` courte, centrée sur les ressources possédées, sans prévoir d’extension par des classes dérivées.

L’héritage reste utile lorsqu’il fait partie de la conception. Dans les autres cas, `sealed` précise les usages pris en charge par la classe.
