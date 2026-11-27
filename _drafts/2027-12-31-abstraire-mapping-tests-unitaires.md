---
title: Faut-il abstraire les mappings dans les tests unitaires
date: 2027-12-31 19:00:00 -0400
categories: [outil-developpement, bonnes-pratiques]
tags: [dotnet, csharp, tests-unitaires, automapper, mapping, mocking, nsubstitute, qualite-code]
---

> ⚠️ À RELIRE + valider le format du code

## Préambule

Dans les tests d’un service, on remplace souvent certaines dépendances par des objets simulés. C’est courant pour un accès à une base de données, un client HTTP ou un service externe. Pourtant, lorsqu’un `IMapper` figure dans le constructeur, on le simule parfois lui aussi.

Pour un mapping en mémoire, rapide et déterministe, je préfère généralement conserver l’implémentation réelle. La transformation contribue au résultat retourné par l’application. La remplacer réduit donc la partie du comportement vérifiée par le test.

Dans les systèmes existants qui utilisent AutoMapper, j’ajoute aussi un test de validation de sa configuration. Il donne rapidement l’alerte lorsqu’une évolution des modèles ou des DTO rend un mapping invalide, sans qu’il soit nécessaire de démarrer l’application ou d’exécuter un scénario de bout en bout.

Ce test peut aussi aider les agents d’IA qui modifient le code. Après un changement de modèle, de DTO ou de profil, ils peuvent l’exécuter et utiliser les erreurs pour repérer les incohérences de configuration.

Ces pratiques vérifient des choses différentes : le test de configuration contrôle les règles déclarées à AutoMapper, tandis que les tests qui exécutent les mappings vérifient les valeurs produites et leur rôle dans le comportement de l’application.

## Abstraire et simuler sont deux décisions différentes

Une abstraction et un double de test répondent à des besoins différents.

| Décision | Ce qu’elle change |
| --- | --- |
| Abstraire le mapping | La dépendance connue du code appelant, par exemple une interface propre à l’application |
| Simuler le mapping | L’implémentation exécutée pendant le test, remplacée par un comportement prédéfini |

Un service peut dépendre d’une interface et recevoir le véritable mapper dans ses tests. À l’inverse, utiliser directement `AutoMapper.IMapper` permet déjà de créer un double avec une bibliothèque de simulation comme NSubstitute.

La présence d’une interface ne suffit donc pas à justifier sa simulation. Je regarde plutôt le comportement vérifié et ce que le test perdrait en remplaçant cette dépendance.

Cette réflexion rejoint celle de Derek Comartin dans [Testing Needs a Seam, Not an Interface](https://codeopinion.com/testing-needs-a-seam-not-an-interface/). Il explique que les tests ont besoin d’un point de substitution (*seam*) adapté au comportement à contrôler. Une interface peut fournir ce point, mais tout code testable n’en a pas besoin. Dans son exemple, il contrôle les échanges HTTP tout en conservant l’implémentation qui traite la réponse.

Pour un mapping rapide et déterministe, ce raisonnement m’amène à contrôler les données d’entrée tout en gardant la transformation réelle.

## Ce que recommande Jimmy Bogard

Jimmy Bogard, le créateur d’AutoMapper, a exprimé sa position dans [une réponse sur Stack Overflow publiée en 2017](https://stackoverflow.com/questions/43817242/how-to-mock-automapper-imapper-in-controller) :

> *I would recommend not mocking AutoMapper.*

Il recommande ensuite d’utiliser l’implémentation réelle et compare la simulation du mapper à celle d’un sérialiseur JSON.

Je partage cette approche pour les mappings habituels. Quand la transformation s’exécute entièrement en mémoire et sans effet de bord, utiliser le vrai mapper permet de vérifier davantage de comportement sans préparation excessive.

Cette recommandation invite aussi à laisser les mappings participer aux tests lorsque leur résultat fait partie du contrat observé.

## Ce qu’un mapping simulé peut masquer

Supposons qu’un service lise un client, puis le transforme en DTO. Si le test configure le mapper pour retourner un DTO déjà construit, il détermine lui-même le résultat de la transformation.

Le service peut alors retourner le DTO attendu même si le profil AutoMapper a été supprimé, si une propriété provient du mauvais champ ou si la construction du DTO échoue dans l’application.

Le test peut toujours servir à vérifier une branche du service ou un autre comportement. Le mapping reste simplement hors de son périmètre.

C’est particulièrement discutable lorsque le comportement testé consiste surtout à retourner les données transformées. Dans ce cas, préparer entièrement le DTO dans le double retire une partie importante du scénario à vérifier.

Je préfère alors fournir les données sources, exécuter le véritable mapping et vérifier les valeurs retournées.

## Un test unitaire peut utiliser plusieurs vrais objets

La définition d’un test unitaire ne fait pas consensus. Martin Fowler distingue notamment les [tests solitaires et les tests sociables](https://martinfowler.com/bliki/UnitTest.html). Les premiers remplacent les collaborateurs de l’unité testée, tandis que les seconds peuvent conserver des collaborateurs réels.

Un service testé avec un mapper en mémoire peut donc relever d’une approche de tests unitaires sociables. Certaines équipes préféreront parler de test de composant. L’essentiel est d’utiliser un vocabulaire cohérent dans le projet.

Pour ma part, j’évalue si le test reste rapide, reproductible et facile à comprendre, et s’il détecte les erreurs qui comptent pour le comportement visé.

Le compromis existe : une erreur dans un profil peut faire échouer plusieurs tests de services qui l’utilisent. Des tests ciblés sur les mappings aident à localiser la cause. Cette couverture supplémentaire me paraît généralement utile quand la transformation fait partie du résultat observable.

## Commencer par valider la configuration AutoMapper

Dans un système existant, le test suivant constitue un bon point de départ. Cette forme du constructeur convient notamment aux projets utilisant AutoMapper 13 ou 14 :

```csharp
using AutoMapper;
using Xunit;

public class AutoMapperConfigurationTests
{
    [Fact]
    public void AutoMapper_Configuration_DevraitEtreValide()
    {
        var configuration = new MapperConfiguration(
            options => options.AddMaps(typeof(DependencyInjection).Assembly));

        configuration.AssertConfigurationIsValid();
    }
}
```

`DependencyInjection` représente ici un type accessible de l’application, situé dans l’assembly qui contient les profils. Il faut l’adapter à la structure du projet.

La méthode [`AddMaps`](https://docs.automapper.io/en/latest/Configuration.html#assembly-scanning-for-auto-configuration) découvre notamment les profils dans les assemblies indiqués. Si les profils sont répartis dans plusieurs assemblies, le test doit couvrir ceux qui appartiennent au périmètre voulu.

À partir d’AutoMapper 15, le [constructeur de `MapperConfiguration` demande aussi un `ILoggerFactory`](https://docs.automapper.io/en/latest/15.0-Upgrade-Guide.html#mapperconfiguration). Dans un test qui n’a pas besoin de capturer les journaux, on peut construire la configuration ainsi :

```csharp
using Microsoft.Extensions.Logging.Abstractions;

var configuration = new MapperConfiguration(
    options => options.AddMaps(typeof(DependencyInjection).Assembly),
    NullLoggerFactory.Instance);

configuration.AssertConfigurationIsValid();
```

Cette adaptation change la construction de la configuration, mais pas l’objectif du test.

### Réutiliser les réglages de l’application

Le test doit rester aligné avec la configuration de l’application. Si celle-ci applique des conventions globales, des filtres ou des réglages particuliers, charger uniquement les profils peut produire une configuration différente.

Je recommande de réutiliser le code de configuration de l’application lorsqu’il existe, plutôt que de recopier ses règles dans le projet de tests.

Un test fondé sur `AddMaps` ne vérifie pas à lui seul que le conteneur d’injection de dépendances enregistre et résout correctement tous les composants. Si des résolveurs ou des convertisseurs dépendent de services injectés, un test de composition peut compléter la validation.

## Une configuration valide ne garantit pas un résultat correct

La [validation de configuration d’AutoMapper](https://docs.automapper.io/en/latest/Configuration-validation.html) vérifie par défaut que les membres de destination sont couverts par les mappings déclarés. Elle peut notamment signaler une propriété de destination sans correspondance.

Elle ne connaît toutefois pas l’intention fonctionnelle de l’application. Par exemple, affecter le prénom à une propriété destinée au nom de famille peut rester valide si les deux propriétés sont des chaînes de caractères.

La portée de la validation dépend aussi de la configuration. Un membre déclaré avec `Ignore()` est volontairement exclu du mapping. `MemberList.None` désactive la validation des membres pour le mapping concerné. C’est aussi le comportement par défaut de `ReverseMap()`.

Je déconseille d’ajouter des exclusions uniquement pour faire passer le test. Chaque exclusion devrait correspondre à un choix assumé.

Il faut également distinguer un mapping invalide d’un mapping absent. La validation ne parcourt pas le code appelant pour découvrir tous les couples source et destination dont l’application aura besoin. Un test qui exécute une transformation attendue couvre ce risque différemment.

## Tester le résultat d’un véritable mapping

Prenons un exemple simple : l’application expose un DTO de client dont le nom complet doit présenter le prénom avant le nom de famille.

Les exemples qui suivent utilisent xUnit et la syntaxe du constructeur d’AutoMapper 15 ou ultérieur.

Voici les types de l’application :

```csharp
public sealed class Client
{
    public required int Id { get; init; }
    public required string Prenom { get; init; }
    public required string Nom { get; init; }
}

public sealed class ClientDto
{
    public required int Id { get; init; }
    public required string NomComplet { get; init; }
}
```

Le profil appartient lui aussi au code de l’application :

```csharp
using AutoMapper;

public sealed class ClientProfile : Profile
{
    public ClientProfile()
    {
        CreateMap<Client, ClientDto>()
            .ForMember(
                destination => destination.NomComplet,
                options => options.MapFrom(
                    source => source.Prenom + " " + source.Nom));
    }
}
```

Dans le projet de tests, une petite fabrique (*factory*) construit un mapper à partir de ce profil réel :

```csharp
using AutoMapper;
using Microsoft.Extensions.Logging.Abstractions;

internal static class MapperFactory
{
    public static IMapper Creer()
    {
        var configuration = new MapperConfiguration(
            options => options.AddProfile<ClientProfile>(),
            NullLoggerFactory.Instance);

        return configuration.CreateMapper();
    }
}
```

La fabrique charge le profil à tester. Elle ne redéfinit pas un autre `CreateMap` dans le projet de tests. Le test global de configuration, lui, couvre l’ensemble des profils attendus.

On peut maintenant vérifier la transformation :

```csharp
using Xunit;

public sealed class ClientMappingTests
{
    [Fact]
    public void Map_ClientAvecPrenomEtNom_DevraitProduireNomComplet()
    {
        var mapper = MapperFactory.Creer();
        var client = new Client
        {
            Id = 42,
            Prenom = "Alexis",
            Nom = "Garon-Michaud"
        };

        var resultat = mapper.Map<ClientDto>(client);

        Assert.Equal(42, resultat.Id);
        Assert.Equal("Alexis Garon-Michaud", resultat.NomComplet);
    }
}
```

Si une modification inverse le prénom et le nom dans le profil, la configuration peut rester valide. Ce test échouera sur la valeur attendue.

Les données choisies comptent aussi. Un identifiant différent de zéro et deux noms distincts évitent de masquer certaines erreurs avec des valeurs par défaut ou identiques.

Le résultat attendu est défini indépendamment du mapper. Le calculer en appelant une seconde fois la même transformation réduirait la capacité du test à détecter une erreur.

Les profils, les conventions retenues et les conversions personnalisées font partie du code de l’application. Les tests doivent sécuriser ces choix.

### Choisir les scénarios qui ont de la valeur

Je concentre les tests de comportement sur les choix propres à l’application : transformations personnalisées, valeurs de remplacement, conversions et propriétés importantes du contrat exposé.

Les valeurs absentes et les collections méritent une attention particulière lorsque leur comportement a une incidence. Par exemple, AutoMapper [transforme par défaut une collection source nulle en collection vide](https://docs.automapper.io/en/latest/Lists-and-arrays.html#handling-null-collections), avec des options pour adapter ce comportement. Le test devrait exprimer ce que l’application attend dans ce cas.

Pour les copies directes par convention, la validation de configuration couvre déjà une partie du risque. J’ajoute des assertions explicites lorsque l’importance du champ ou l’historique des erreurs le justifie.

## Garder le vrai mapper dans les tests du service

Le même principe s’applique au code qui utilise le mapping. Voici un service minimal, écrit avec un *primary constructor* de C# 12 :

```csharp
using AutoMapper;
using System.Threading.Tasks;

public interface ILecteurClient
{
    Task<Client?> Obtenir(int identifiant);
}

public sealed class ServiceClient(
    ILecteurClient lecteurClient,
    IMapper mapper)
{
    public async Task<ClientDto?> Obtenir(int identifiant)
    {
        var client = await lecteurClient.Obtenir(identifiant);

        return mapper.Map<ClientDto>(client);
    }
}
```

Dans le test, NSubstitute simule la lecture des données avec [cette syntaxe de configuration du résultat](https://nsubstitute.github.io/help/set-return-value/). Le mapping utilise le profil de l’application :

```csharp
using NSubstitute;
using System.Threading.Tasks;
using Xunit;

public sealed class ServiceClientsTests
{
    [Fact]
    public async Task Obtenir_ClientExistant_DevraitRetournerClientMappe()
    {
        var client = new Client
        {
            Id = 42,
            Prenom = "Alexis",
            Nom = "Garon-Michaud"
        };

        var lecteurClient = Substitute.For<ILecteurClient>();
        lecteurClient.Obtenir(42)
            .Returns(client);

        var sut = new ServiceClient(
            lecteurClient,
            MapperFactory.Creer());

        var resultat = await sut.Obtenir(42);

        Assert.NotNull(resultat);
        Assert.Equal(42, resultat.Id);
        Assert.Equal("Alexis Garon-Michaud", resultat.NomComplet);
    }
}
```

Ce test exécute le service et la transformation configurée, tout en contrôlant les données fournies par le lecteur. Il ne nécessite aucune base de données.

Il vérifie quelques éléments du contrat public du service. Les variantes détaillées du mapping peuvent rester dans les tests dédiés, ce qui limite les répétitions dans les classes qui l’utilisent.

Si la construction des configurations devient coûteuse dans une grande suite, les [fixtures de xUnit](https://xunit.net/docs/shared-context) permettent de partager cette préparation. La configuration AutoMapper est immuable après sa création. Les objets sources des tests doivent toutefois rester indépendants, et les éventuels résolveurs partagés doivent être compatibles avec l’exécution concurrente.

## Le cas particulier des projections et de Mapperly

Avec `ProjectTo`, le résultat dépend aussi de la [capacité du fournisseur LINQ à traduire la projection](https://docs.automapper.io/en/latest/Queryable-Extensions.html#query-provider-limitations). Un test en mémoire, ou une liste convertie avec `AsQueryable()`, ne démontre pas cette compatibilité. Je compléterais les projections importantes par un test d’intégration avec le fournisseur et le moteur de base de données de l’application.

Avec Mapperly, ses [diagnostics d’analyse](https://mapperly.riok.app/docs/configuration/analyzer-diagnostics/) peuvent signaler certaines erreurs pendant la compilation. Ces vérifications interviennent plus tôt dans le développement, mais ne démontrent pas que toutes les valeurs produites répondent au besoin.

J’ai présenté cet outil dans mon article sur [la performance de Mapperly](https://codewithfrenchy.com/posts/performance-mapperly/). Pour les tests, je garde la même préférence : exécuter le mapping généré et vérifier les comportements importants. Cette approche vaut aussi pour un mapping écrit manuellement.

## Quelle combinaison de tests choisir?

| Risque à couvrir | Vérification à privilégier |
| --- | --- |
| Configuration AutoMapper incohérente | Test global avec `AssertConfigurationIsValid()` |
| Transformation incorrecte ou mapping attendu absent | Test exécutant le vrai mapping avec des données représentatives |
| Mauvaise utilisation du mapping par un service | Test du comportement du service avec le vrai mapper |
| Enregistrement incomplet de résolveurs ou de convertisseurs | Test de composition à partir de la configuration d’injection de dépendances |
| Projection incompatible avec le fournisseur de données | Test d’intégration de la projection |

Dans un système existant, je commencerais par le test global de configuration et quelques transformations importantes. J’ajouterais ensuite des tests de régression à mesure que des erreurs concrètes sont découvertes.

Il n’est pas nécessaire de réécrire tous les tests de services d’un seul coup. À chaque modification d’un test, on peut réévaluer l’utilité de son double de mapper et voir s’il serait préférable d’utiliser le profil réel.

## Conclusion

Abstraire une dépendance et décider de la simuler dans un test sont deux choix distincts. La présence d’une interface ne devrait pas, à elle seule, déterminer le périmètre du test.

Pour les transformations simples en mémoire, je privilégie l’implémentation réelle afin d’exercer le code qui contribue au résultat de l’application.

Avec AutoMapper, je combine un test de validation de configuration et des tests ciblés sur les transformations importantes. Le premier contrôle la cohérence des règles déclarées. Les seconds vérifient que les données produites correspondent au contrat attendu.

Cette approche protège le code lorsque les modèles évoluent, sans imposer un double de test pour chaque propriété ou chaque dépendance.
