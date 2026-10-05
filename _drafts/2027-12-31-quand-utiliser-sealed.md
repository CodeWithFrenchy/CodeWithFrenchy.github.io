---
title: "Utiliser sealed quand l’héritage n’est pas prévu"
date: 2027-12-31 19:00:00 -0400
categories: []
tags: [dotnet, tests, bonnes-pratiques]
---

## Préambule

Lorsqu’on crée une classe en C#, on pense généralement à sa responsabilité, à ses dépendances et aux méthodes qu’elle expose. La possibilité d’en hériter est souvent laissée au comportement par défaut du langage.

Pourtant, ce choix mérite d’être explicite. Une classe peut être parfaitement adaptée à son utilisation actuelle sans avoir été conçue pour devenir une classe de base.

Dans le code applicatif, je privilégie donc une règle simple :

**Lorsqu’une classe n’a pas été conçue pour être héritée, je la déclare `sealed`.**

Cette convention est particulièrement utile pour les records qui transportent des données et pour les classes de tests. Elle devient encore plus intéressante lorsqu’une classe de tests implémente `IDisposable`, parce qu’elle permet de garder une gestion des ressources adaptée au besoin réel.

Voyons ce que cette règle apporte et dans quels cas il faut y faire exception.

## Ce que signifie réellement sealed

Appliqué à une classe, le mot-clé [`sealed`](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/sealed) empêche de créer une classe dérivée.

Prenons un validateur volontairement simple :

```csharp
public sealed class DossierValidator
{
    public bool EstNomValide(string nom) =>
        !string.IsNullOrWhiteSpace(nom);
}
```

On peut créer un `DossierValidator`, appeler ses méthodes et l’utiliser comme dépendance. En revanche, le compilateur refusera qu’un autre type en hérite.

La déclaration communique donc une décision de conception : cette classe fournit une implémentation concrète et ne constitue pas un point d’extension par héritage.

Il faut distinguer cette décision de la possibilité de redéfinir une méthode. En C#, une méthode ordinaire n’est déjà pas redéfinissable avec `override`. Le [polymorphisme par héritage](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/object-oriented/polymorphism) repose notamment sur les membres `virtual` et `abstract`.

L’absence de méthodes virtuelles ne rend toutefois pas une classe impossible à hériter. `sealed` exprime cette restriction au niveau du type entier.

## L’héritage fait partie du contrat d’une classe

Laisser une classe ouverte « au cas où » semble parfois plus flexible. Mais une classe de base demande une réflexion supplémentaire.

Quels comportements une classe dérivée peut-elle modifier ? Quelles règles doit-elle préserver ? Dans quel ordre les opérations doivent-elles être exécutées ? Comment les ressources seront-elles libérées si plusieurs niveaux de la hiérarchie en possèdent ?

Ces questions ont un coût de conception et de maintenance. Pour moi, une classe destinée à l’héritage devrait avoir des réponses claires à ces questions.

À l’inverse, un service qui applique une règle précise ou un objet qui représente une demande n’a pas nécessairement besoin de porter ce contrat supplémentaire.

Avec `sealed`, le lecteur connaît immédiatement la limite choisie. Il n’a pas à deviner si l’héritage a été prévu, oublié ou simplement jamais utilisé.

Si un besoin d’héritage apparaît ensuite dans le code que l’équipe maîtrise, on pourra réexaminer la conception. Ce sera l’occasion de définir les points d’extension et les garanties attendues avant d’ouvrir la classe.

## Les records : sealed pour les types qui ne servent pas de base

Les records sont de bons candidats à cette convention lorsqu’ils représentent des DTO, des commandes ou des messages autonomes.

Par exemple :

```csharp
public sealed record DossierDto(
    int Id,
    string Nom);
```

Ce type représente les données d’un dossier. Si aucun scénario métier ne prévoit de le spécialiser par héritage, je préfère l’indiquer dès sa déclaration.

L’ajout de `sealed` conserve les usages habituels d’un record, notamment l’égalité par valeur et les expressions `with` :

```csharp
var dossier = new DossierDto(
    42,
    "Autorisation de travaux");

var dossierRenomme = dossier with
{
    Nom = "Autorisation de rénovation"
};
```

L’expression produit une nouvelle instance. Elle ne demande aucune classe dérivée.

### Un record scellé n’est pas automatiquement immuable

`sealed` contrôle l’héritage. Il ne contrôle pas la modification des données.

Un record peut exposer des propriétés modifiables. Même avec des propriétés `init`, les objets référencés peuvent rester modifiables, par exemple une liste. La [documentation des records](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/record) distingue bien ces aspects.

Il faut donc examiner séparément la possibilité d’hériter du type et la possibilité de modifier son état.

### Certains records sont volontairement des classes de base

La règle « tous les records doivent être scellés » serait trop absolue.

Une [hiérarchie de records](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-9.0/records.md#inheritance) peut être un choix métier explicite :

```csharp
public abstract record EvenementDossier(int IdDossier);

public sealed record DossierCree(
    int IdDossier,
    string Nom) : EvenementDossier(IdDossier);

public sealed record DossierFerme(
    int IdDossier,
    string Motif) : EvenementDossier(IdDossier);
```

Ici, `EvenementDossier` est volontairement ouvert à l’héritage. Les deux événements concrets sont scellés.

Cela ne ferme pas l’ensemble de la hiérarchie : d’autres records peuvent encore dériver directement de `EvenementDossier`.

Enfin, un `record struct` ne peut pas être hérité. On ne lui ajoute donc pas `sealed` :

```csharp
public readonly record struct IdentifiantDossier(int Valeur);
```

Ma convention concerne ainsi les **record classes qui ne sont pas conçus comme types de base**.

## Les classes de tests sont aussi des classes concrètes

Une classe de tests regroupe généralement des scénarios portant sur un comportement donné. Si elle ne sert pas de base à d’autres classes de tests, elle peut elle aussi être scellée.

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

La déclaration indique que cette classe contient ses propres scénarios et qu’elle n’est pas un socle de tests à spécialiser.

Une classe de base commune peut bien sûr être justifiée. Dans ce cas, on la conçoit comme telle et on garde éventuellement ses classes dérivées concrètes scellées.

Le partage de code ne nécessite d’ailleurs pas toujours une hiérarchie. Une méthode utilitaire, un constructeur de données de tests ou une fixture peuvent répondre au besoin. Le choix dépend de ce qu’on souhaite partager : quelques opérations, des données ou une ressource avec une durée de vie particulière.

L’intérêt de `sealed` est de rendre ce choix visible, y compris dans le code de tests.

## IDisposable : un cas où la fermeture simplifie le code

C’est probablement le cas où cette convention apporte le bénéfice le plus immédiat.

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

La classe de tests possède la ressource, et son nettoyage délègue à `StringReader.Dispose()`. Cette forme suffit ici : la classe est scellée, ne possède pas de finaliseur et ne gère pas directement de ressource native. La documentation .NET présente cette [implémentation simple de `IDisposable`](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-dispose) pour une classe scellée propriétaire d’un objet disposable.

### Pourquoi une classe ouverte demande davantage de réflexion

Si `LecteurDossierTests` restait ouverte, une classe dérivée pourrait ajouter ses propres ressources. Il faudrait alors prévoir comment elle participe au nettoyage.

Pour une classe de base qui introduit `IDisposable`, le pattern extensible habituel comporte une méthode publique `Dispose()` et une méthode `protected virtual Dispose(bool)`. Les classes dérivées complètent cette dernière et appellent l’implémentation de base.

La règle d’analyse [CA1063](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca1063) vérifie notamment ce contrat. Ce n’est pas l’interface `IDisposable` elle-même qui impose la surcharge avec un booléen. C’est le pattern utilisé pour permettre un nettoyage cohérent dans une hiérarchie.

Dans notre exemple, aucun besoin d’héritage ne justifie cette infrastructure. Ajouter `sealed` permet de conserver le nettoyage simple qui correspond à la conception retenue.

### La responsabilité de libérer les ressources reste entière

`sealed` ne déclenche aucun nettoyage automatique. Dans cet exemple, c’est xUnit qui appelle `Dispose()`.

Le [pattern de libération des ressources](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/dispose-pattern) prévoit aussi que la méthode supporte plusieurs appels sans refaire le nettoyage ni provoquer d’erreur. Selon les ressources utilisées, on peut déléguer à leur propre implémentation ou devoir suivre explicitement l’état de l’objet.

Il n’est pas nécessaire d’ajouter un finaliseur à cette classe. Elle ne gère pas directement de ressource native, et un appel à `GC.SuppressFinalize(this)` n’apporterait rien ici, puisqu’elle est scellée et n’a pas de finaliseur.

Enfin, si la ressource ne sert qu’à un seul test, un `using` local peut suffire. Si elle appartient à une fixture partagée, son nettoyage relève de cette fixture, à la fin de sa durée de vie. La propriété de la ressource détermine où placer le nettoyage.

## Une classe scellée peut toujours participer au polymorphisme

Fermer une implémentation à l’héritage reste compatible avec l’utilisation d’une interface.

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

Le point d’extension est alors le contrat de l’interface. `FormateurDossier` reste une implémentation concrète dont on interdit la spécialisation par héritage.

Il n’est toutefois pas nécessaire de créer une interface pour chaque classe scellée. Une classe simple peut être utilisée et testée directement. L’abstraction doit répondre à un besoin de conception.

## Dans quels cas conserver une classe ouverte ?

Je garderais une classe ouverte lorsqu’un besoin concret le justifie.

Cela comprend les classes de base conçues pour être spécialisées, mais aussi certaines contraintes techniques.

Par exemple, les [proxies de chargement différé d’EF Core](https://learn.microsoft.com/en-us/ef/core/querying/related-data/lazy) reposent sur des classes dont ils peuvent hériter et sur des propriétés de navigation redéfinissables. Une entité utilisée de cette manière ne peut pas être scellée.

Certains outils de doublures de tests utilisent également l’héritage pour substituer une classe concrète. Dans ce cas, fermer la classe peut empêcher leur fonctionnement. La [documentation de Moq](https://github.com/devlooped/moq/wiki/Quickstart) rappelle notamment le rôle des membres redéfinissables. Tester une classe scellée directement reste possible, mais la substituer par une classe dérivée ne l’est pas.

Le contexte d’une bibliothèque publique mérite aussi une décision spécifique. Les [Framework Design Guidelines sur la fermeture des classes](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/sealing) adoptent historiquement une position plus favorable à l’extensibilité des bibliothèques réutilisables. Elles rappellent que leurs utilisateurs peuvent avoir des besoins d’extension non anticipés.

La convention présentée ici est donc un choix pour le code applicatif, pas une règle universelle imposée par C# ou par .NET.

Et si une classe publique est déjà distribuée dans un package, lui ajouter `sealed` peut casser les consommateurs qui en héritent. Les [règles de compatibilité .NET](https://learn.microsoft.com/en-us/dotnet/core/compatibility/library-change-rules) identifient bien cette modification comme un changement incompatible pour un type auparavant ouvert à l’héritage.

## Les performances peuvent en profiter

Le JIT peut tirer parti du fait qu’un type est scellé. Lorsqu’il connaît suffisamment le type visé, il peut notamment remplacer certains appels virtuels par des appels directs, puis éventuellement intégrer le corps de la méthode appelée.

Stephen Toub décrit ces possibilités dans la [proposition de l’analyseur de fermeture des types internes](https://github.com/dotnet/runtime/issues/49944).

Le résultat dépend toutefois du code, du type connu au point d’appel et des optimisations du runtime. Ajouter `sealed` ne garantit donc pas une accélération mesurable de chaque classe ou de chaque test.

Dans cette convention, le bénéfice attendu en premier reste la lisibilité du contrat. Les gains éventuels de performance sont un avantage supplémentaire à mesurer lorsqu’ils comptent pour l’application.

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

Pour le code existant, la même réflexion s’applique, en vérifiant les usages avant de modifier une API.

En révision de code, la question devient simple : **avons-nous conçu cette classe pour que d’autres classes en héritent ?**

Si la réponse est non et qu’aucune contrainte technique ne l’exige, `sealed` exprime clairement la décision.

## Conclusion

Je privilégie `sealed` sur les classes qui ne sont pas destinées à l’héritage, en particulier sur les records autonomes et les classes de tests.

Pour une classe de tests qui implémente `IDisposable`, cette intention a une conséquence pratique : on peut souvent conserver une méthode `Dispose()` courte, centrée sur les ressources possédées, sans préparer une extension par des classes dérivées.

L’héritage reste disponible là où il fait partie de la conception. Ailleurs, un mot-clé suffit à préciser les usages que la classe entend prendre en charge.
