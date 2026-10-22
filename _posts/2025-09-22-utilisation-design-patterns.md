---
title: Les design patterns - les bons Legos pour vos applications .NET
date: 2025-09-22 10:00:00 -0400
categories: [architecture, bonnes-pratiques]
tags: [dotnet, csharp, design-patterns, solid, singleton, decorator, injection-dependances, maintenabilite]
---

## Introduction : Ne réinventez pas la roue, assemblez les Legos

En conception logicielle, un __design pattern__ (ou patron de conception) est une solution éprouvée à un problème de conception récurrent. Ce sont en quelque sorte nos __briques de Lego__ à nous, développeurs : plutôt que de tailler chaque pièce dans le bois, on pioche dans la boîte le composant adapté au bon moment. Bien comprendre ces patterns permet d’assembler des systèmes plus robustes et évolutifs, un peu comme construire un château de Lego solide sans recoller les briques au chewing-gum. 😉

En d’autres termes, les design patterns nous évitent de __réinventer la roue__ sur chaque projet. Ils définissent des structures de classes ou d’objets qui ont fait leurs preuves. Par exemple, *le Singleton s’assure de l’existence d’un seul objet de son genre et fournit un point d’accès global à celui-ci*. Utiliser un pattern approprié au bon contexte, c’est comme choisir la bonne pièce Lego standard plutôt que de mouler la vôtre. Cela accélère le développement et améliore la maintenabilité, tout en respectant les principes [SOLID](https://en.wikipedia.org/wiki/SOLID).

Avant de plonger dans sept patterns importants pour les applications de gestion en .NET Core, mentionnons une excellente ressource : la [série](https://www.youtube.com/watch?v=v9ejT8FO-7I&list=PLrhzvIcii6GNjpARdnO4ueTUAVR9eMBpc) de vidéos de __Christopher Okhravi__ sur les design patterns (disponible sur YouTube). Elle reste encore aujourd’hui une référence pédagogique (et humoristique) sur le sujet. Vous verrez qu’avec un peu de pratique, les design patterns deviendront vos alliés du quotidien pour structurer proprement un code métier (facturation, inventaire, etc.) sans transformer votre code en plat de spaghetti. 🍝

Nous allons explorer les patterns suivants, avec pour chacun une explication, un exemple concret, un extrait de code C#, les cas d’usage à favoriser (et à éviter), ainsi que des astuces d’intégration en .NET (notamment via l’IoC container et autres outils comme __Scrutor__ pour le pattern Decorator).

En piste !

## Singleton : Un seul pour les gouverner tous, un seul pour les instancier,

*Le pattern Singleton garantit qu’une classe n’a qu’une seule instance accessible globalement.*

Le Singleton est le Highlander des patterns : *« There can be only one! »* 😅 Son objectif est de restreindre l’instanciation d’une classe à un seul objet, partagé et accessible partout. Dans une application de gestion, on utilise souvent ce pattern pour des objets qui doivent être uniques : par exemple un __gestionnaire de configuration global__, une __cache en mémoire__ ou encore un __logger__ central. Cela évite d’avoir plusieurs copies incohérentes de ces ressources critiques disséminées un peu partout.

### Exemple concret

Imaginons un système de facturation qui doit accéder aux paramètres de configuration (taux de TVQ, devise, etc.) depuis n’importe quel module. On pourrait créer une classe `ConfigurationManager` en Singleton. Ainsi, que ce soit le module de génération de facture ou le module d’inventaire, tous récupèrent la même instance unique du `ConfigurationManager` avec les paramètres chargés au lancement de l’application.

```csharp
public sealed class ConfigurationManager
{
    private static readonly ConfigurationManager _instance = new();
    public static ConfigurationManager Instance => _instance;

    // Données de configuration (exemple)
    public decimal TauxTVQ { get; private set; }
    public string DeviseParDefaut { get; private set; }

    // Constructeur privé pour empêcher les instanciations externes
    private ConfigurationManager()
    {
        // Simulation de chargement depuis une source de vérité (fichier, base, ou grimoire fiscal)
        TauxTVQ = 0.09975m; // TVQ actuelle au Québec
        DeviseParDefaut = "CAD"; // Vive le dollar canadien!
    }

    // Méthode d'accès optionnelle
    public static ConfigurationManager GetInstance() => Instance;
}
```

Dans le code ci-dessus, `ConfigurationManager.Instance` renverra toujours la même instance, initialisée une seule fois. On déclare le constructeur en `private` pour empêcher de faire `new ConfigurationManager()`. Seule la propriété statique `Instance` permet d’obtenir l’objet unique.

__Quand l’utiliser 🟢 :__ Lorsqu’une seule instance d’un service doit exister et être partagée partout. Exemples classiques : configuration globale, journalisation (logging), connexion unique à une base de données, cache applicatif. Cela permet de centraliser un état ou un accès et d’éviter les duplications.

__À éviter 🔴 :__ N’abusez pas du Singleton ! Un Singleton mal placé peut devenir un *god object* omniprésent qui rend le code difficile à tester (dépendances globales cachées) et moins modulable. Si plusieurs instances seraient acceptables ou que l’objet a un cycle de vie limité, préférez des instances normales gérées par injection de dépendances. En .NET, beaucoup considèrent d’ailleurs le Singleton comme un *antipattern* si utilisé à tort et à travers.

__Intégration .NET 👩‍💻 :__ Plutôt que d’implémenter le Singleton "à la main" comme ci-dessus, on profite souvent du conteneur d’inversion de contrôle (IoC) de .NET. En enregistrant une classe en *singleton* dans le container, le framework garantit lui-même l’unicité de l’instance. Par exemple, dans le `Program.cs` :

```csharp
services.AddSingleton<IConfigurationManager, ConfigurationManager>();
```

Ici le container va créer une seule instance de `ConfigurationManager` pour toute l’application. C’est thread-safe et plus simple à tester (on peut éventuellement substituer l’implémentation via l’interface). __Astuce:__ Si votre Singleton est lourd à instancier et pas toujours utilisé, envisagez le pattern du Lazy Singleton (instanciation différée).

## Factory : Fabrique d’objets, évitez le __new__

*Le pattern Factory (ici la Factory Method) crée des objets selon le type demandé sans exposer la logique de création.*

Le pattern __Factory__ est un patron de création qui repose sur le principe suivant : _« Ne dites pas `new`, dites `Factory`! »_. Plutôt que d’instancier directement des classes dans votre code (ce qui le couplerait à des implémentations concrètes), vous déléguez cette tâche à une __fabrique__. La Factory centralise la logique de création et peut décider quel sous-type concret retourner en fonction du contexte. C’est un peu comme une usine qui, sur base d’une commande, sort le bon produit de la chaîne sans que l’acheteur ne sache exactement comment il a été fabriqué.

### Exemple concret

Dans un système de gestion documentaire (factures, bons de commande, devis, etc.), on peut avoir une interface commune `IDocument` et plusieurs implémentations (`Facture`, `Devis`, `BonCommande`…). Au lieu d’éparpiller du code `new Facture()` partout, on définit une factory qui va produire le bon type de document selon un paramètre. Par exemple, une méthode `DocumentFactory.CreateDocument(TypeDocument type)` qui retourne un `IDocument` :

```csharp
// Type énuméré pour les différents documents possibles
public enum TypeDocument
{ 
   Facture,
   Devis,
   BonCommande
}

public interface IDocument 
{ 
    string Numero { get; }
    DateTime DateEmission { get; }
    // ... autres membres communs
}

public class Facture : IDocument { /* ... */ }
public class Devis : IDocument { /* ... */ }
public class BonCommande : IDocument { /* ... */ }

public static class DocumentFactory 
{
    public static IDocument CreateDocument(TypeDocument type)
    {
        return type switch 
        {
            TypeDocument.Facture => new Facture(),
            TypeDocument.Devis => new Devis(),
            TypeDocument.BonCommande => new BonCommande(),
            _ => throw new ArgumentException("Type de document inconnu")
        };
    }
}
```

Ici, le `DocumentFactory` expose une méthode statique de création. Le client appelle par exemple `IDocument doc = DocumentFactory.CreateDocument(TypeDocument.Facture);` pour obtenir une facture prête à l’emploi, sans savoir *quelle classe concrète* est instanciée. Si demain on ajoute un nouveau type de document (par exemple un avoir), il suffit d’étendre la factory sans impacter le code client.

__Quand l’utiliser 🟢 :__ Quand la logique d’instanciation est *complexe* ou *conditionnelle*. Par exemple, si la création d’un objet dépend de paramètres (contexte utilisateur, configuration) ou nécessite de décider parmi plusieurs types dérivés. Les Factories améliorent la lisibilité (on donne un nom explicite au processus de création) et centralisent en un seul endroit le code qui fait des `new`. Très utile pour :

- __Factory Method__ : laisser des sous-classes décider de l’instanciation (patron utilisé dans les frameworks, par exemple pour créer des contrôles UI spécifiques).
- __Abstract Factory__ : groupe de factories pour créer des familles d’objets liés (exemple une AbstractFactory pour GUI qui peut produire *soit* des boutons et fenêtres version "Windows", *soit* version "macOS").

__À éviter 🔴 :__ Si la création de l’objet est triviale et ne nécessite pas de logique conditionnelle, une factory ajoute une complexité inutile. Inutile de sur-ingénierie : on ne va pas faire une factory pour instancier un simple DTO par exemple. De plus, trop de factories peuvent rendre le code abstrait à l’excès (on finit par avoir des usines qui fabriquent d’autres usines…). Comme toujours, dosez le pattern où il apporte une réelle valeur.

__Intégration .NET 👩‍💻 :__ Les IoC containers réduisent parfois le besoin explicite de factory, car ils peuvent eux-mêmes choisir quelle implémentation injecter selon la configuration. Par exemple, on peut enregistrer plusieurs implémentations d’une interface et en sélectionner une par nom ou via une factory lambda dans le container. Néanmoins, le pattern Factory reste utile pour créer des objets __métiers__ complexes.

En .NET Core, vous pouvez aussi injecter une *factory* sous forme de fonction. Ex: `Func<TypeDocument, IDocument>` que l’IoC pourrait construire. Une autre approche est d’utiliser la factory en tant que service : vous créez une classe `DocumentFactory` (non statique) enregistrée en singleton, et qui a peut-être besoin d’autres services (elle peut les recevoir via injection dans son constructeur) pour fabriquer les objets. Cette technique vous permet par exemple de choisir l’implémentation en fonction de données à l'exécution (on le verra avec le patron Strategy juste après).

## Strategy : des algorithmes interchangeables, choisis à la volée

*Le pattern Strategy définit des algorithmes interchangeables. Le contexte délègue à une stratégie concrète (`ConcreteStrategyA ou ConcreteStrategyB`) choisie dynamiquement.*

Le pattern **Strategy** permet de définir une famille d’algorithmes, de les encapsuler chacun dans une classe distincte, et de les rendre **interchangeables à la volée** dans le contexte où ils sont utilisés. En clair, on sépare le *quoi* (l’algorithme à exécuter) du *quand/comment* il est utilisé.

L’un des grands bénéfices de ce pattern est qu’il **respecte le principe d’ouverture/fermeture** (*Open/Closed Principle*) : votre code est **ouvert à l’extension**, mais **fermé à la modification**. Autrement dit, si un jour un nouveau mode de calcul doit être ajouté (par exemple une livraison par drone 🛸), pas besoin de toucher au calculateur existant : on ajoute une nouvelle stratégie, et c’est tout.

C’est une alternative élégante et maintenable aux chaînes de `if/else` ou de `switch` sur des `enums`, qui violent souvent ce principe (car chaque ajout nécessite de modifier le bloc conditionnel existant). Avec le patron Strategy, on **étend** le comportement en ajoutant une nouvelle classe, plutôt qu’en modifiant du code existant — ce qui réduit les risques de régression.



### Exemple concret

Dans une application de gestion de commerce en ligne, prenons le calcul des __frais de livraison__. Selon le mode d’expédition choisi par le client, le calcul du coût diffère (transporteur standard, express, retrait en magasin gratuit, etc.). Sans pattern, on aurait peut-être dans la classe Commande un code semblable à :

```csharp
decimal frais = mode switch
{
    "Standard" => CalculerStandard(),
    "Express" => CalculerExpress(),
    "Magasin" => 0,
    _ => throw new NotImplementedException()
};
...
```

Avec le patron Strategy, on va créer une interface `IFraisLivraisonStrategy` avec une méthode `CalculerFrais(Commande commande)`. Puis on implémente une classe concrète par mode : `StandardStrategy`, `ExpressStrategy`, `RetraitMagasinStrategy`, etc., chacune encapsulant son calcul. Le contexte (par exemple la classe `CalculateurLivraison`) utilise une référence à `IFraisLivraisonStrategy`.

On peut pousser plus loin en combinant avec une Factory Method pour choisir la stratégie en fonction de la commande :

```csharp
// Stratégie de calcul des frais de livraison
public interface IFraisLivraisonStrategy 
{
    decimal CalculerFrais(Commande cmd);
}

// Implémentations concrètes
public class LivraisonStandard : IFraisLivraisonStrategy 
{
    public decimal CalculerFrais(Commande cmd) => 5m; // Forfait simple
}

public class LivraisonExpress : IFraisLivraisonStrategy 
{
    public decimal CalculerFrais(Commande cmd) => cmd.Poids * 1.5m;
}

public class RetraitMagasin : IFraisLivraisonStrategy 
{
    public decimal CalculerFrais(Commande cmd) => 0m;
}

// Contexte qui utilise une stratégie
public class CalculateurLivraison 
{
    private IFraisLivraisonStrategy _strategie;

    // On devrait injecter la stratégie via le constructeur !
    public void ChoisirStrategie(IFraisLivraisonStrategy strat) => _strategie = strat;

    public decimal CalculerFrais(Commande cmd) => 
        return _strategie?.CalculerFrais(cmd) ??
           throw new InvalidOperationException("Stratégie non définie.");
}
```

Et quelque part dans le code appelant (par exemple lors de la finalisation de la commande) :

```csharp
var calculateur = new CalculateurLivraison();

IFraisLivraisonStrategy strategie = modeChoisi switch 
{
    "Standard" => new LivraisonStandard(),
    "Express" => new LivraisonExpress(),
    "Magasin" => new RetraitMagasin(),
    _ => throw new InvalidOperationException("Mode inconnu")
};

calculateur.ChoisirStrategie(strategie);
decimal frais = calculateur.CalculerFrais(commande);
```

Ici, on choisit la stratégie dynamiquement en fonction d’un paramètre `modeChoisi`. Le calculateur de livraison n’a pas besoin de connaître les détails de chaque mode, il délègue à la stratégie. Si un jour on ajoute un mode Drone ou à dos de vache 🐮, on crée une nouvelle classe implémentant `IFraisLivraisonStrategy` et on ajuste la sélection (idéalement via une factory au lieu d’un switch inline).

__Quand l’utiliser 🟢 :__ Quand vous avez plusieurs variantes algorithmiques interchangeables selon un critère (par exemple, différents *règles de calcul*, *politiques*, *stratégies de tarification*, etc.). Le Strategy apporte de la flexibilité : on peut même changer la stratégie en cours d’exécution si besoin. C’est aussi un bon moyen d’adhérer au principe *Open/Closed* – ajouter une nouvelle stratégie n’impacte pas les existantes ni le contexte.

__À éviter 🔴 :__ Si vos variantes d’algorithmes sont très simples ou qu’il n’y en a qu’une ou deux peu susceptibles d’évoluer, le pattern peut être overkill. Aussi, n’introduisez pas une stratégie juste pour éviter un `if` unique — le remède serait pire que le mal. Enfin, veillez à ce que les stratégies partagent bien la même interface commune et puissent réellement varier indépendamment du reste : si elles finissent par dépendre fortement du contexte externe, le gain de découplage diminue.

__Intégration .NET 👩‍💻 :__ On peut tirer parti de l’IoC container pour gérer les stratégies. Par exemple, vous pourriez __injecter toutes les implémentations__ d’une interface et sélectionner la bonne à l’exécution (via un dictionnaire de stratégies, ou en taguant chaque implémentation d’un `attribut` \ `[Strategy("Nom")])`. Une autre approche consiste à enregistrer une Factory (comme vue plus haut) dans l’IoC qui retourne la stratégie voulue. .NET Core facilite même cela avec les *Named Options* ou en combinant avec le pattern __Policy__. Bref, le patron Strategy se marie bien avec l’injection de dépendances pour éviter de faire de `new` manuellement : on peut demander au container de résoudre la stratégie dont on a besoin, ce qui simplifie le remplacement (ex: injection d’une fausse stratégie en tests unitaires).

## Decorator : Ajouter des fonctionnalités à la volée (comme une cache sur vos services)

*Le pattern Decorator ajoute dynamiquement des comportements à un objet en l’enveloppant dans un "décorateur" qui implémente la même interface.*

Le pattern __Decorator__ (ou __Wrapper__) est un patron structurel qui permet d’__ajouter dynamiquement des responsabilités__ à un objet, sans modifier son code source. Imaginez un cadeau qu’on emballe et re-emballe : le contenu reste le même, mais chaque couche de papier ajoute une fonctionnalité (un message, un ruban, etc.). En informatique, un décorateur est un objet qui *implémente la même interface* que l’objet qu’il décore, et qui contient une référence vers lui-même. Il peut ainsi intercepter les appels, faire quelque chose en plus, puis déléguer à l’objet réel.

### Exemple concret

Considérons un microservice d’inventaire qui expose un service `IProduitService` avec une méthode `ObtenirProduit(int id)` retournant les détails d’un produit. Les appels à ce service peuvent être coûteux (imaginons qu’il interroge une base de données distante). On souhaite __mettre en cache__ les résultats pour améliorer les performances. Sans changer le code du service existant, on peut créer un décorateur de cache.

```csharp
public interface IProduitService
{
    ValueTask<Produit> ObtenirProduit(int id);
}

// Implémentation principale (accès base de données par exemple)
public class ProduitService : IProduitService
{
    public async ValueTask<Produit> ObtenirProduit(int id)
    {
        Console.WriteLine($"Accès BDD pour le produit {id}");
        await Task.Delay(50); // Simule un accès I/O
        return new Produit { Id = id, Nom = $"Produit {id}" };
    }
}

// Decorator de cache
public class ProduitServiceCacheDecorator : IProduitService
{
    private readonly IProduitService _produitService;
    // Devrait provenir de l'IoC
    private readonly MemoryCache _cache = new(new MemoryCacheOptions());

    public ProduitServiceCacheDecorator(IProduitService produitService)
    {
        _produitService = produitService;
    }

    public ValueTask<Produit> ObtenirProduit(int id)
    {
        if (_cache.TryGetValue(id, out Produit produit))
        {
            Console.WriteLine($"Cache hit pour le produit {id}");
            return new ValueTask<Produit>(produit); // disponible de manière synchrone
        }

        return ObtenirEtCacher(id);
    }

    private async ValueTask<Produit> ObtenirEtCacher(int id)
    {
        var resultat = await _produitService.GetProduit(id);
        _cache.Set(id, resultat, TimeSpan.FromMinutes(5));
        return resultat;
    }
}
```

> 💡 **Pourquoi `ValueTask` ici ?** Parce que dans le cas où la cache répond, il est inutile de créer une `Task`, `ValueTask` permet de **réduire la pression sur le GC** pour les cas de réponse rapide. Mais attention : si le service sous-jacent est très souvent asynchrone, `Task<T>` reste le bon choix !

Ici, `ProduitServiceCacheDecorator` __décore__ un `IProduitService` concret (\_produitService). Il ajoute la fonctionnalité de caching autour de l’appel réel. Pour le consommateur, c’est transparent : il utilise un `IProduitService` sans savoir si c’est le cache ou le service de base. On peut composer plusieurs décorateurs les uns avec les autres si besoin (log, sécurité, etc.), chaque décorateur enveloppant le précédent.

__Quand l’utiliser 🟢 :__ Quand vous voulez __enrichir__ ou __modifier dynamiquement__ le comportement d’un objet *sans* altérer son code. C’est idéal pour les __fonctionnalités transversales__ (caching, logging, contrôle d’accès, mesure de performance…). Le Decorator est plus souple que l’héritage car on peut empiler plusieurs décorateurs et en activer/désactiver certains à la configuration.

__À éviter 🔴 :__ Si la hiérarchie de décorateurs devient trop complexe ou que vous en avez un très grand nombre, ça peut devenir difficile à suivre en debug (effet *oignon*, on se perd dans les couches). Si l’objet de base a déjà une bonne extension via des mécanismes plus simples, inutile de rajouter en plus des décorateurs. Enfin, n’utilisez pas un décorateur juste pour factoriser du code commun entre deux classes : dans ce cas, une abstraction ou un [mixin](https://en.wikipedia.org/wiki/Mixin) serait plus approprié.

__Intégration .NET 👩‍💻 :__ .NET Core facilite la vie avec l’IoC container pour chaîner les décorateurs sans douleur, grâce à des librairies comme __Scrutor__. Scrutor fournit une méthode d’extension `Decorate<,>()` qui enregistre un décorateur pour un service existant. Par exemple, pour notre cas ci-dessus :

```csharp
services.AddScoped<IProduitService, ProduitService>();
services.Decorate<IProduitService, ProduitServiceCacheDecorator>();
```

Comme le décrit Andrew Lock, *"Scrutor cherche tout service enregistré en `IProduitService` (ici `ProduitService`) et le remplace par `ProduitServiceCacheDecorator` qui prendra en paramètre de constructeur l’ancien service"*. En un appel, on a intercalé le cache entre l’application et le service réel. Scrutor gère l’ordre (il faut ajouter le décorateur __après__ le service de base) et la résolution des dépendances supplémentaires du décorateur.

Sans Scrutor, on peut toujours enregistrer manuellement un décorateur : par exemple, on enregistre `ProduitService` puis on enregistre `IProduitService` pointant vers un `provider => new ProduitServiceCacheDecorator(new ProduitService(...))`. Mais avouons que Scrutor fait ça proprement, de façon déclarative.

## Middleware : le pipeline HTTP dans ASP.NET Core

*Pipeline de middlewares ASP.NET Core : chaque middleware (1, 2, 3) traite la requête puis appelle le suivant, formant une chaîne autour de la requête/réponse.*

Le terme __Middleware__ désigne un composant logiciel intermédiaire qui s’insère dans une chaîne de traitement. En ASP.NET Core, *le pipeline de requête HTTP est une illustration concrète du pattern Chain of Responsibility* appliqué aux requêtes et réponses HTTP. Chaque middleware a la possibilité d’agir sur la requête entrante, de décider de la court-circuiter (renvoyer directement une réponse) ou de faire quelque chose avant/après de passer la main au middleware suivant.

En fait, un middleware ASP.NET Core est une implémentation spécialisée du pattern Decorator/Chain, dédiée aux requêtes HTTP. Vous en utilisez à chaque fois que vous faites `app.UseXyz(...)` dans le `Program.cs` : authentification, logging, routing, etc., sont des middlewares standards.

### Exemple concret

Supposons que l’on veuille journaliser le temps de traitement de certaines requêtes pour une API de gestion d’inventaire. On peut écrire un middleware custom `LoggingMiddleware` qui, pour chaque requête, enregistre l’heure de début, appelle le composant suivant, puis enregistre l’heure de fin et calcule la durée.

```csharp
public class LoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly TimeProvider _timeProvider;

    public LoggingMiddleware(RequestDelegate next,
       TimeProvider timeProvider)
    {
        _next = next;
        _timeProvider = timeProvider;
    }

    public async Task Invoke(HttpContext context)
    {
        var debut = _timeProvider.GetUtcNow();
        await _next(context);

        var duration = _timeProvider.GetUtcNow() - debut;
        Console.WriteLine($"Requête {context.Request.Path} traitée en {duration.TotalMilliseconds} ms");
    }
}
```

Dans le `Program.cs`, on l’enregistre dans le pipeline :

```csharp
var app = builder.Build();
app.UseMiddleware<LoggingMiddleware>();
```

Chaque requête HTTP va passer par ce middleware, puis continuer. On pourrait chaîner d’autres middlewares avant/après. Par exemple, on pourrait ajouter un middleware de cache en amont qui vérifie si la réponse n’est pas déjà en cache (et ne pas appeler `_next` du tout s’il trouve quelque chose), ou un middleware d’authentification qui vérifie le token JWT puis appelle `_next` si OK, ou renvoie 401 immédiatement si non autorisé.

__Quand l’utiliser 🟢 :__ Les middlewares sont *incontournables* en développement ASP.NET Core pour tout ce qui est __cross-cutting__ au niveau des requêtes HTTP. C’est littéralement la façon de faire officielle pour filtrer/travailler sur les requêtes et réponses. Dans un contexte plus large, on peut assimiler n’importe quelle suite de traitements séquentiels modulaires à ce concept de middleware. Donc utilisez ce pattern/pipeline dès que vous avez un traitement en plusieurs étapes où chaque étape peut décider de stopper ou de modifier le flux.

__À éviter 🔴 :__ En dehors du contexte web, évitez d’abuser des pipelines si un simple appel direct suffit. Inutile de sur-architecturer une simple séquence d’appels en pipeline ultra-générique si elle ne sera jamais modifiée. Pour ASP.NET, faites attention à l’ordre des middlewares : une mauvaise ordre (par exemple placer l’authentification après le routing alors qu’on en a besoin avant) peut causer des bugs subtils. Ce n’est pas tant le pattern en lui-même qu’il faut éviter, mais plutôt sa mauvaise configuration.

__Intégration .NET 👩‍💻 :__ Pour ASP.NET Core, l’intégration est native (méthodes `UseMiddleware<T>()`, etc.). Vous pouvez profiter de l’injection de dépendances *dans* vos middlewares en définissant un constructeur qui prend les services voulus (le framework les injectera). Ainsi votre middleware peut très bien être un décorateur qui utilise un service de cache ou un repo injecté du container. Enfin, notez que d’autres librairies adoptent ce pattern de pipeline configurable : citons __MediatR__ (avec ses Pipeline Behaviors qui agissent autour des requêtes MediatR), ou __Message delegating handlers__ pour HttpClient. Comprendre le fonctionnement des middlewares vous aidera donc dans de nombreux recoins du framework .NET.

## Mediator : un __hub__ pour la communication, grâce à MediatR

Le pattern __Mediator__ définit un objet intermédiaire qui __centralise les communications__ entre plusieurs composants au lieu qu’ils interagissent directement. Imaginez une tour de contrôle d’aéroport : les avions (composants) ne se parlent pas entre eux en direct, ils parlent tous à la tour (médiateur) qui coordonne tout. Cela réduit le couplage : chaque objet a juste à connaître le médiateur, pas les détails des autres.

En .NET, on utilise fréquemment une librairie nommée __MediatR__ (de Jimmy Bogard) pour implémenter ce pattern. MediatR permet d’envoyer des __requêtes/commandes__ et de les faire traiter par un ou plusieurs __handlers__ enregistrés, sans que l’émetteur et le récepteur se connaissent.

### Exemple concret

Dans une application de gestion d’inventaire, imaginons qu’on veuille décoller la logique métier des contrôleurs API. On peut définir une commande du domaine, par exemple `AjouterProduitCommand` (avec les infos du produit), et un handler associé `AjouterProduitHandler` qui contient la logique pour ajouter le produit (vérifier stock, enregistrer en base de données, etc.). Le contrôleur va juste *envoyer* la commande via le médiateur, qui lui se charge de trouver et exécuter le handler adéquat.

Avec MediatR, cela se traduit par :

```csharp
// Définition d'une commande (implémente IRequest<T> de MediatR)
public record AjouterProduitCommand(string Reference, int Quantite) : IRequest<ResultatProduit>;

// Implémentation du handler pour cette commande
public class AjouterProduitHandler : IRequestHandler<AjouterProduitCommand, ResultatProduit>
{
    private readonly IProduitRepository _produitRepository;

    public AjouterProduitHandler(IProduitRepository produitRepository)
    {
        _produitRepository = produitRepository;
    } 

    public async Task<ResultatProduit> Handle(AjouterProduitCommand command,
       CancellationToken cancellationToken)
    {
        Produit prod = new Produit
        { 
            Reference = command.Reference,
            Quantite = command.Quantite
        };

        await _produitRepository.Ajouter(prod, cancellationToken);

        return new ResultatProduit(prod.Id, "Produit ajouté avec succès");
    }
}
```

Dans le contrôleur (ou n’importe quel endroit du code) qui a besoin d’ajouter un produit, on ferait :

```csharp
// _mediator est injecté via IMediator de MediatR
var resultat = await _mediator.Send(new AjouterProduitCommand("REF123", 50));
Console.WriteLine(resultat.Message);
```

Ici, `_mediator.Send(...)` va en coulisses trouver le `AjouterProduitHandler` (grâce au container IoC et MediatR), exécuter sa méthode `Handle`, et renvoyer le résultat. Le contrôleur n’a aucune référence directe sur le handler ou le repository, il passe par le médiateur.

__Quand l’utiliser 🟢 :__ Lorsque vous voulez __découpler__ fortement les composants qui interagissent. Le Mediator est roi dans les architectures __CQRS__ / __Médiation__ : il permet d’envoyer des *Commandes* et *Query* sans lier le code d’envoi et le code de traitement. Utile aussi pour implémenter un système de __notifications/événements internes__ : par exemple, plusieurs parties de l’application écoutent un événement via Mediator (avec MediatR ce sont les `INotification` et leurs handlers multiples). En somme, dès que votre application commence à ressembler à un enchevêtrement de signaux entre modules, introduire un médiateur peut apporter de la lisibilité et une architecture en étoile plutôt qu’en plat de nouilles.

__À éviter 🔴 :__ Si le trafic via le médiateur devient trop centralisé, le Mediator peut lui-même devenir un goulot d’étranglement ou un *god object* déguisé (ex: un seul mediator gigantesque qui sait trop de choses). Il faut l’utiliser pour ce qu’il fait bien : la __découplage de l’envoi et du traitement__. Si deux composants sont naturellement faits pour interagir directement (faible couplage, usage local), inutile de forcer le passage par un médiateur. Par ailleurs, un excès de médiation peut compliquer le suivi du flux d’appel (on perd un peu la trace de "qui appelle qui" car tout passe par le hub central). Comme toujours, c’est un équilibre.

__Intégration .NET 👩‍💻 :__ L’intégration de MediatR en .NET Core est très simple : on installe le package, puis dans `Program.cs` on fait `services.AddMediatR(cfg => cfg.RegisterServicesFromAssemblyContaining<Program>());` (ou équivalent) pour enregistrer tous les handlers du projet. MediatR utilise le container IoC pour résoudre les handlers. On peut configurer des comportements additionnels (les fameux Pipeline Behaviors mentionnés plus haut, qui permettent d’implémenter des cross-cutting concerns autour des requêtes Mediator, comme la validation, le logging, etc. – c’est en fait un Chain of Responsibility autour du Mediator!).

Avec MediatR, on peut envoyer des commandes synchrones ou asynchrones, des requêtes qui attendent une réponse, ou des notifications sans réponse (pub/sub interne). C’est un outil formidable pour structurer du code métier dans les applications de gestion, en appliquant les principes __CQRS__ (séparer écriture/lecture) et __Clean Architecture__. À noter : plusieurs implémentations du pattern Mediator existent dans l’écosystème .NET. Il est possible d’en coder une version minimaliste maison, ou d’utiliser des bibliothèques comme `MediatR`, qui a longtemps été une référence. **Toutefois, attention : MediatR qui sera prochainement sous [licence commerciale](https://www.jimmybogard.com/automapper-and-mediatr-going-commercial/)**, ce qui **limite son usage dans les projets professionnels**.

## Builder : construction pas-à-pas et syntaxe fluide

*Le Builder pattern sépare la construction d’un objet complexe de sa représentation. Ici, le `Director` utilise un `Builder` (fluent) pour assembler un `Product` étape par étape.*

Dernier pattern mais non des moindres : __Builder__. Si je vous dis *« constructeur avec 12 paramètres »*, vous me dites 🤮. En effet, quand un objet possède beaucoup de propriétés optionnelles, le passage de paramètres devient illisible et sujet à erreur. Le Builder vient à la rescousse en fournissant une __interface de construction progressive (souvent fluide)__. On crée l’objet en plusieurs étapes, via des méthodes dédiées, plutôt qu’un seul gros constructeur. C’est un peu le mode d’emploi Ikea : on assemble pièce par pièce, et à la fin on obtient le meuble.

### Exemple concret

Dans une application de gestion, imaginons un module de génération de __rapport PDF__ complexe (par exemple un rapport d’activité mensuel avec plusieurs sections, en-tête, pied de page, etc.). Plutôt que d’avoir une méthode géante qui prend 20 arguments pour tout configurer, on va utiliser un `ReportBuilder` qui va fournir des méthodes pour ajouter les différentes parties du rapport de manière lisible.

```csharp
public class Report 
{
    public string Titre { get; set; }
    public List<string> Sections { get; set; } = [];
    public string PiedDePage { get; set; }
    // ... éventuellement d'autres propriétés complexes (graphique, tableau, etc.)
}

public class ReportBuilder
{
    private readonly Report _report = new Report();

    public ReportBuilder Titre(string titre)
    {
        _report.Titre = titre;
        return this; // on retourne le builder pour chaîner
    }

    public ReportBuilder AjouterSection(string contenuSection)
    {
        _report.Sections.Add(contenuSection);
        return this;
    }

    public ReportBuilder PiedDePage(string textePied)
    {
        _report.PiedDePage = textePied;
        return this;
    }

    public Report Build()
    {
        return _report;
    }
}
```

Utilisation en syntaxe fluide (fluent interface) :

```csharp
Report rapport = new ReportBuilder()
    .Titre("Rapport d’activité - Mars 2025")
    .AjouterSection("Chiffre d'affaires : 1M €")
    .AjouterSection("Nouveaux clients : 50")
    .PiedDePage("Confidentiel - interne")
    .Build();
```

On lit quasiment du français dans le code ! On a progressivement construit l’objet `rapport` sans se soucier de l’ordre interne des initialisations ni d’oublier un paramètre requis. Le Builder encapsule la logique d’assemblage (ici triviale, mais elle pourrait être plus complexe avec par exemple calcul de totaux en interne avant `Build`).

__Quand l’utiliser 🟢 :__ Quand la création d’un objet est __complexe__, c’est-à-dire :

* comporte de __nombreux paramètres__ (optionnels ou obligatoires) rendant un constructeur classique impraticable ou ambigu,
* nécessite des __étapes__ multiples (par exemple certaines parties doivent être construites avant d’autres),
* ou quand on veut offrir à l’API utilisateur une __syntaxe fluide__ très lisible pour configurer un objet.

Les cas d’utilisation concrets abondent en applications de gestion : construction d’un rapport comme vu, configuration dynamique d’un objet de paramétrage, montage d’une requête SQL/ElasticSearch complexe via un QueryBuilder, etc. Le Builder pattern permet aussi d’isoler la logique de création en un point unique (respect du single responsibility).

__À éviter 🔴 :__ Si l’objet à créer est simple (quelques paramètres), un constructeur ou les __object initializers__ de C# (init { Prop1 = ..., Prop2 = ... }) suffisent amplement. Le Builder, s’il est mal conçu, peut également laisser l’objet dans un état partiel incohérent tant que `Build()` n’a pas été appelé – attention donc à garantir une utilisation correcte (parfois on marque le constructeur de l’objet cible internal pour forcer le passage par le builder). Évitez aussi le builder juste pour faire du *fluent* sur des opérations statiques ou non liées à un objet complexe — là c’est le nom qui peut prêter à confusion : *Builder* sert vraiment à construire un objet.

__Intégration .NET 👩‍💻 :__ La *syntaxe fluide* (fluent syntax) est très répandue dans l’écosystème .NET moderne, souvent inspirée du Builder pattern. Par exemple, les *Options* de configuration se construisent via des `optionsBuilder.AddX()` en chaîne, les requêtes LINQ en chaîne, l’API Fluent Validation (`RuleFor(x => x.Name).NotEmpty().WithMessage("...")`), etc. Sans être toujours de purs "Builders", ces syntaxes fluides améliorent la lisibilité et l’enchaînement d’appels.

En interne, .NET utilise aussi le Builder pattern pour la construction de gros objets comme l’__`HostBuilder` / `WebApplicationBuilder`__ (qui configureront étape par étape votre application).

Côté IoC, un Builder est souvent créé via une factory ou fourni par un framework. On ne l’enregistre généralement pas dans le container (on crée le builder quand on en a besoin, puis on jette). Toutefois, rien n’empêche d’injecter un builder pré-configuré si cela a du sens dans votre design.

En résumé, le Builder est un peu l’inverse du Factory : on l’utilise pour composer __petit à petit__ un objet complexe, là où la Factory crée d’emblée un objet souvent simple ou retourne une implémentation. Avec le Builder pattern, on prend le temps d’assembler et grâce au *fluent interface*, le code appelant est clair et expressif.

## Conclusion : Choisir le bon pattern au bon moment

Nous avons fait un tour d’horizon de plusieurs design patterns clés en .NET Core. Chaque pattern est un __outil__ dans votre boîte à outils : il a un usage privilégié, des avantages, des inconvénients. L’art de l’architecture logicielle consiste à choisir *le bon Lego au bon moment*. Il n’y a pas de solution universelle : parfois un simple `if` vaut mieux qu’un Strategy surdimensionné, parfois un Mediator apporte une structure bienvenue là où le couplage devenait ingérable.

Un conseil : __entraînez-vous__ à reconnaître dans votre code ou dans les frameworks que vous utilisez quels patterns sont à l’œuvre. Vous verrez que ASP.NET Core, Entity Framework, etc., sont truffés de ces concepts (Singletons pour les services, Factory methods pour les DbContext, Decorators dans les pipeline, etc.). Comprendre les design patterns vous permettra non seulement de mieux utiliser les API .NET, mais aussi de concevoir vos propres composants de manière élégante et maintenable.

Enfin, pour aller plus loin, je le redis, la [série](https://www.youtube.com/watch?v=v9ejT8FO-7I&list=PLrhzvIcii6GNjpARdnO4ueTUAVR9eMBpc) de __Christopher Okhravi__ sur YouTube est un excellent complément visuel et pédagogique, avec une touche d’humour qui, je l’espère, aura fait écho à la lecture de cet article. 😉

En maîtrisant ces patterns, vous éviterez de réinventer la roue carrée et vous construirez des applications évolutives *brique par brique*. Alors à vos Legos, prêts, codez ! 🚀