---
title: Optimiser la performance des logs dans un projet .NET
date: 2026-01-26 19:00:00 -0400
categories: []
tags: [dotnet]
---

## Introduction

La journalisation est un élément central de toute application .NET : elle permet de diagnostiquer des erreurs, de suivre des traitements et d'auditer les actions. La plupart des développeurs utilisent `ILogger<T>` et les méthodes d'extension comme `LogInformation()` ou `LogError()`. Pourtant ces méthodes pratiques cachent un coût non négligeable : à chaque appel, le **message doit être analysé**, les paramètres sont **convertis en object** et les **types "valeurs" sont boxés**. Ce mécanisme crée des allocations mémoire et du travail pour le GC, surtout lorsque le niveau de journalisation ne permet pas d'émettre le message. Heureusement, depuis plusieurs versions de .NET un **pattern de journalisation à hautes performances** existe : les **templates de loggers** (`LoggerMessage`).

## Pourquoi les méthodes ILogger classiques sont lentes

Les extensions de `ILogger<T>` acceptent des paramètres sous forme d'arguments variadiques (params object?), ce qui entraîne [plusieurs problèmes](https://learn.microsoft.com/en-us/dotnet/core/extensions/high-performance-logging#:~:text=The%20LoggerMessage%20%20class%20exposes,scenarios%2C%20use%20the%20LoggerMessage%20pattern) :

- **Boxing des types valeurs** : chaque fois qu'une structure (int, bool, etc.) est passée, elle est enveloppée dans un objet, créant une allocation.
- **Analyse répétée du template** : la chaîne contenant les *placeholders* ("Process {Item} at {Time}") doit être analysée à chaque appel pour extraire les noms des champs. Même si le niveau de log n'est pas activé, **l'analyse est effectuée**.
- **Interpolation de chaînes** : lorsque l'on utilise la syntaxe `$"User {userId} logged in"`, une nouvelle chaîne est créée et tous les arguments sont convertis en chaînes avant l'appel.

Ces coûts, parfois faibles à l'unité, deviennent importants dans des scénarios à fort trafic ou pour des services très sollicités. Microsoft a introduit des solutions pour contourner ces problèmes : les méthodes pré‑générées avec `LoggerMessage`.

## Historique : LoggerMessage.Define et l'arrivée du source‑generator

### Naissance des templates de loggers

Dès .NET Core 2.1, l'équipe ASP.NET Core a ajouté la classe **LoggerMessage**. Cette classe permet de **définir des délégués** de journalisation qui sont mis en cache. Chaque délégué associe un niveau de log, un `EventId` et un modèle de message. À l'exécution, on n'a plus besoin d'analyser le modèle ni de boxer les valeurs : on invoque simplement la méthode compilée. [Microsoft explique que ces délégués](https://learn.microsoft.com/en-us/dotnet/core/extensions/high-performance-logging#:~:text=The%20LoggerMessage%20%20class%20exposes,scenarios%2C%20use%20the%20LoggerMessage%20pattern) « créent des événements de log avec moins d'allocations par rapport aux méthodes d'extension ». 

Le pattern consiste à :

- **Déclarer un champ `Action<ILogger, T1, T2, Exception>`** à l'aide de `LoggerMessage.Define()`. On précise le niveau de log, l'ID de l'événement et le modèle de message.
- **Créer une méthode d'extension** qui appelle ce délégué en passant les valeurs typées.

Exemple :

```csharp
private static readonly Action<ILogger, string, int, Exception?> _logProcessingItem =
    LoggerMessage.Define<string, int>(
        LogLevel.Information,
        new EventId(1001, nameof(LogProcessingItem)),
        "Traitement de l’élément {Item} en {Milliseconds} ms");

public static void LogProcessingItem(this ILogger logger, string item, int milliseconds)
{
    _logProcessingItem(logger, item, milliseconds, null);
}
```

À partir de .NET 6, Microsoft a simplifié cette approche grâce au **source‑generator**. En décorant une **méthode partielle** avec l'attribut `LoggerMessageAttribute`, le compilateur [génère automatiquement le délégué et la méthode d'extension](https://learn.microsoft.com/en-us/dotnet/core/extensions/logger-message-generator#:~:text=.NET%206%20introduces%20the%20,conjunction%20with%20%20119%20functionality). Cette approche évite d'écrire du code répétitif. Le guide officiel précise que les méthodes doivent être partial, retourner void et ne pas être génériques.

Exemple :

```csharp
public static partial class Logs
{
    [LoggerMessage(
        EventId = 1002,
        Level = LogLevel.Warning,
        Message = "Erreur lors du traitement de l’utilisateur {UserId}")]
    public static partial void LogUserProcessingError(
        ILogger logger,
        string userId,
        Exception ex);
}

// Appel :
Logs.LogUserProcessingError(_logger, userId, exception);
```

Le compilateur génère une implémentation qui appelle un délégué caché et offre ainsi les mêmes bénéfices de performance qu'avec `LoggerMessage.Define`.

### Quand ces techniques sont‑elles apparues ?

- **.NET Core 2.1** (2018) : apparition de `LoggerMessage.Define`. L'API permet déjà de mettre en cache la structure du message et de réduire les allocations.
- **.NET 6** (novembre 2021) : introduction de **LoggerMessageAttribute**, un générateur de code qui réduit la verbosité du modèle. Cette version marque aussi l'arrivée de la règle d'analyse **CA1848** (Utilisez les délégués LoggerMessage) activable en niveau d'analyse _recommended_.
- **.NET 7 / 8 et 9** : la source‑génération s'améliore et les analyzers incitent toujours plus à adopter ces patterns. L'analyseur **CA1848** n'est cependant pas activé par défaut ; il faut choisir un niveau d'analyse pour l'activer.

## Bénéfices mesurés : benchmarks

Pour savoir si l'effort vaut la peine, plusieurs développeurs ont mesuré les différences de performance. L'article _Don't box your logs_ de Daniel Genezini compare trois techniques : l'appel direct à `ILogger.LogInformation()`, l'utilisation d'un délégué avec `LoggerMessage.Define()` et l'utilisation de `LoggerMessageAttribute`. 

Voici un résumé des résultats :

| Méthode | Temps moyen par appel | Allocations | Description |
| --- | --- | --- | --- |
| **Extension ILogger** | ~43-155 ns | **Allocations (~56 B)** | Les paramètres sont convertis en object, la chaîne est analysée et les types valeurs sont boxés. |
| **LoggerMessage.Define** | ~12-20 ns | **Aucune allocation** | Le délégué est généré au démarrage et mis en cache ; les paramètres restent typés. |
| **LoggerMessageAttribute** | **~9 ns** | **Aucune allocation** | Le compilateur génère le code de log au build ; la méthode est aussi courte que possible. |

On constate que les templates de loggers multiplient les performances par **4 à 10** et **suppriment complètement les allocations**, ce qui réduit la pression sur le GC. Dans des micro‑services où des milliers de logs sont produits par seconde, ces gains sont précieux.

## Exemples d'utilisation

### 1. Enregistrer le traitement d'une commande

Imaginons un service qui traite des commandes et veut journaliser le début et la fin de chaque traitement. Avec **LoggerMessageAttribute**, on crée une classe statique de logs :

```csharp
public static partial class CommandLogs
{
    [LoggerMessage(
        EventId = 2001,
        Level = LogLevel.Information,
        Message = "Début du traitement de la commande {OrderId}")]
    public static partial void StartProcessing(
        ILogger logger,
        string orderId);

    [LoggerMessage(
        EventId = 2002,
        Level = LogLevel.Information,
        Message = "Fin du traitement de la commande {OrderId} en {ElapsedMs} ms")]
    public static partial void EndProcessing(
        ILogger logger,
        string orderId,
        double elapsedMs);
}

// Dans votre service :
var sw = Stopwatch.StartNew();
CommandLogs.StartProcessing(_logger, orderId);
// exécution …
CommandLogs.EndProcessing(_logger, orderId, sw.Elapsed.TotalMilliseconds);
```

Les méthodes générées sont statiques et ne créent aucune allocation lorsque le niveau Information est désactivé.

## Enforçant la règle CA1848 dans .editorconfig

Pour encourager l'équipe à adopter les templates de loggers, il est utile d'ajouter la règle **CA1848** dans un fichier .editorconfig. Ce fichier permet de configurer la sévérité des analyzers pour un projet ou une solution.

Voici un exemple :

```txt
# Applique à tous les fichiers C#
[*.cs]

# Utiliser un niveau d’analyse afin d’activer CA1848 et d’autres règles
# AnalyseLevel peut être défini sur `latest` ou `latest-recommended` selon la version de .NET
dotnet_analyzer_diagnostic.severity = warning

# Spécifie que la règle CA1848 est considérée comme une erreur
# Les développeurs devront corriger les appels `ILogger` classiques
dotnet_diagnostic.CA1848.severity = error
```

La documentation sur la configuration des analyzers précise que le préfixe `dotnet_diagnostic.<rule ID>.severity` permet de définir la sévérité d'une règle individuelle. Les valeurs possibles sont error (fait échouer la compilation), warning, suggestion, silent, none ou default. Ce paramétrage s'applique à l'ensemble du projet, mais on peut ajuster la portée en ajoutant plusieurs blocs `*.cs`, `MyApp/Program.cs` ou `*/Tests` selon les besoins.

## Conseils pour adopter les templates de loggers

- **Centralisez vos messages** : créez une classe statique par fonctionnalité ou domaine (OrderLogs, UserLogs…), ce qui facilite la recherche et la cohérence des messages.
- **Utilisez des EventId uniques** : associez chaque événement à un identifiant qui permettra de filtrer et de corréler les logs. Les générateurs acceptent la propriété EventId et un nom.
- **Soyez parcimonieux avec les arguments** : limitez le nombre d'arguments et privilégiez des types simples ou des objets qui se sérialisent bien. Évitez de passer des entités entières ; préférez des identifiants ou des DTO.
- **Gardez la compatibilité** : pour les bibliothèques NuGet ciblant plusieurs versions de .NET, fournissez une implémentation conditionnelle avec `LoggerMessage.Define` et, si possible, une variante source‑générée via `#if`.
- **Mettez en place des tests et des benchmarks** : utilisez [BenchmarkDotNet](https://benchmarkdotnet.org/) pour vérifier que votre adoption n'impacte pas la performance globale. Les mesures présentées plus haut montrent un gain substantiel, mais chaque application est unique.

## Conclusion

L'apparition des **templates de loggers** marque une évolution importante pour la journalisation en .NET. Contrairement aux méthodes d'extension `ILogger<T>`, ces templates **éliminent les allocations liées au boxing**, **cachent l'analyse du modèle** et **améliorent sensiblement les temps d'exécution**. La première approche, disponible depuis .NET Core 2.1 avec `LoggerMessage.Define`, offre déjà un gain appréciable, tandis que le **source‑generator** introduit avec .NET 6 simplifie l'usage et apporte un boost supplémentaire. Des benchmarks réalisés par la communauté montrent un **temps divisé par dix** entre l'appel classique et l'appel généré, et surtout aucune allocation mémoire supplémentaire.

En adoptant ces patterns, les développeurs .NET améliorent les performances de leurs services sans sacrifier la lisibilité du code. L'ajout de la règle **CA1848** dans votre `.editorconfig` aidera votre équipe à détecter et corriger les appels non optimisés. Enfin, centraliser les messages dans des classes dédiées et utiliser des **EventId** cohérents facilite l'analyse des logs et renforce la maintenabilité des applications.