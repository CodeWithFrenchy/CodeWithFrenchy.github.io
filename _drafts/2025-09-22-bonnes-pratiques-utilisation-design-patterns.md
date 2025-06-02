---
title: Bonnes pratiques d’utilisation des design patterns
date: 2025-09-22 20:00:00 -0400
categories: [architecture]
tags: []
---

## Préambule

Il y a une règle non écrite dans le monde du développement logiciel : __à chaque problème, un développeur a déjà tenté de le résoudre avant toi__. Mieux encore, certains ont documenté leurs solutions sous forme de _design patterns_. Mais entre nous, les design patterns, ça peut vite devenir un sujet aussi mal compris qu’un commit sans message.

Dans les discussions techniques, on les brandit parfois comme des sorts de magie noire : « Ah, ici j’irais avec un bon vieux _Strategy Pattern_ » ou « Ce service a clairement besoin d’un _Decorator_, voyons ! ». Mais derrière ces noms pompeux se cachent des concepts simples… à condition de bien les utiliser.

Parce qu’entre le stagiaire qui colle un Singleton sur tout ce qui bouge, et le senior qui décore ses services comme un sapin de Noël 🎄, il y a un juste milieu.

👉 Dans cet article, on va donc démystifier les __design patterns les plus utiles dans un projet .NET Core__, avec des __exemples concrets__, des __mises en contexte métiers__, des __astuces d’intégration avec le conteneur IoC__, et bien sûr, un peu d’__humour pédagogique__, comme à mon habitude. Pas de prise de tête, juste du code propre, bien structuré, et des idées à réutiliser dans vos projets sans faire exploser la dette technique.

On y va ? Allez, sortez votre plus belle interface, on va pattern-er tout ça comme des pros 💪.

## Le pattern Singleton

_Diagramme UML du pattern Singleton._ Le Singleton garantit qu’une classe n’a qu’une seule instance et fournit un point d’accès global à celle-ci. En UML ci-dessus, on voit une classe avec une méthode statique `Instance` retournant toujours la même référence (stockée en variable statique), et un constructeur privé empêchant la création d’instances supplémentaires.

En termes imagés, un Singleton c’est __Highlander__ dans vos objets : « _Il ne peut en rester qu’un !_ ». 😊 Par exemple, dans une application métier, on pourrait avoir un __JournalDeLogs__ en Singleton – une unique instance centralisant tous les logs de l’application. Ainsi, chaque composant n’a pas besoin de créer son propre logger : il récupère l’instance unique et y écrit ses messages. Cela évite d’avoir 42 fichiers de log différents éparpillés un peu partout (et personne ne veut ça, sauf peut-être le stagiaire qui adore débugger la prod…).

### Exemple de code Singleton (C#)

Voici une implémentation simple (non thread-safe pour la concision) d’un Singleton en C# :

```csharp
public sealed class JournalDeLogs
{
    private static JournalDeLogs? _instance;
    private JournalDeLogs() { /_ constructeur privé _/ }

    public static JournalDeLogs Instance
    {
        get
        {
            if (_instance == null)
                _instance = new JournalDeLogs();
            return _instance;
        }
    }

    public void Ecrire(string message)
    {
        Console.WriteLine($"LOG: {message}");
    }
}
```

On définit un constructeur privé et on expose une propriété statique `Instance` qui contrôle la création : la première fois, elle instancie un objet, puis renvoie toujours la même référence unique.

__Quand l’utiliser :__ le Singleton est utile pour les ressources uniques dans l’application : un gestionnaire de configuration global, une connexion à une base de données partagée, etc. En gros, tout ce qui représente __une unique instance__ logique dans le système (ex : le « gouvernement » d’un pays est souvent donné en analogie – il n’y a qu’un seul gouvernement officiel).

__À éviter si :__ on l’utilise à tort et à travers ! 😅 Le Singleton peut vite devenir un _anti-pattern_ s’il est utilisé pour tout et n’importe quoi. Il introduit une forme de _global state_ caché qui peut compliquer les tests unitaires et le débogage (si tout le monde utilise le même objet global, bon courage pour trouver qui modifie une valeur indésirable). Il viole potentiellement le Single Responsibility Principle en gérant lui-même son instance unique. Donc, évitez-le pour des objets qui pourraient exister en plusieurs exemplaires ou qui n’ont pas besoin d’être globaux.

__Intégration avec IoC (.NET Core) :__ Plutôt que d’implémenter manuellement vos singletons, profitez du conteneur d’injection de dépendances de .NET Core. En effet, il est souvent recommandé d’__éviter les singletons statiques__ et l’état global, et de __préférer des singletons gérés par l’IoC__. Vous pouvez par exemple enregistrer un service en Singleton via `services.AddSingleton<ITachePlanifiee, TachePlanifiee>();` dans le _Startup_. Le conteneur garantira une instance unique pour toute l’application, tout en permettant d’injecter ce service proprement dans vos classes (ce qui facilite les tests en pouvant substituer un mock). En somme, laissez le framework jouer les Highlander pour vous !

__Outils utiles :__ Pas de librairie magique requise pour le pattern Singleton – la plupart du temps, le support natif du DI de .NET Core suffit. Cependant, si vous cherchez à éviter des singletons maison, considérez l’utilisation de __IOptions__ ou __IConfiguration__ de .NET pour partager des paramètres globaux plutôt que d’inventer un singleton custom.

---

## Le pattern Factory

_Diagramme UML du pattern Factory Method._ En UML ci-dessus, le pattern _Factory Method_ est illustré : la classe abstraite `Creator` déclare une méthode fabrique `factoryMethod()` que chaque sous-classe concrète redéfinit pour instancier le bon type de `Product`. Plus généralement, un __Factory__ est un objet dont la responsabilité principale est de créer d’autres objets, en cachant les détails de leur instantiation. On commande à la fabrique comme au comptoir d’un café, sans se soucier de comment le barista prépare exactement notre breuvage.

Concrètement en .NET, imaginons un système de traitement de documents métier : on pourrait avoir une factory __FabriqueDocument__ qui, selon le type demandé (`DocumentWord`, `DocumentPdf`, etc.), retourne une instance du bon classe concrète implémentant l’interface IDocument. Le code client dirait juste `var doc = FabriqueDocument.CreerDocument(typeDemandé);` et la factory s’occupe de choisir la bonne classe et de l’instancier. On évite ainsi de faire du `new` partout et de dupliquer la logique de choix du type de document. Pour l’humour : la Factory, c’est un peu le __chef cuisinier__ de votre application – vous lui dites que vous voulez un gâteau au chocolat, il décide s’il doit sortir la recette sans gluten ou la version vegan, et vous sert le bon gâteau sans que vous ayez à connaître la recette !

### Exemple de code Factory (C#)

On montre ici un exemple simplifié de _Factory Method_. Supposons un jeu qui crée des personnages de différentes classes (Guerrier, Mage) via une fabrique :

```csharp
// Produit (interface)
public interface IPersonnage { void Attaquer(); }

// Produits concrets
public class Guerrier : IPersonnage 
{ 
    public void Attaquer() => Console.WriteLine("Le guerrier donne un coup d'épée!");
}
public class Mage : IPersonnage 
{ 
    public void Attaquer() => Console.WriteLine("Le mage lance un sort!");
}

// Créateur (classe abstraite)
public abstract class FabriquePersonnage 
{
    public abstract IPersonnage CreerPersonnage();
}

// Créateurs concrets
public class FabriqueGuerrier : FabriquePersonnage 
{
    public override IPersonnage CreerPersonnage() => new Guerrier();
}
public class FabriqueMage : FabriquePersonnage 
{
    public override IPersonnage CreerPersonnage() => new Mage();
}
```

Ici, `FabriquePersonnage` déclare la méthode abstraite `CreerPersonnage()`. Les sous-classes `FabriqueGuerrier` et `FabriqueMage` la surchargent pour instancier le personnage approprié (`Guerrier` ou `Mage`). Le client, lui, peut utiliser `FabriquePersonnage` de manière générique sans connaître les détails internes :

```csharp
FabriquePersonnage fabrique = ObtenirFabriqueSelonChoixUtilisateur();
IPersonnage perso = fabrique.CreerPersonnage();
perso.Attaquer();
```

__Quand l’utiliser :__ le pattern Factory s’applique quand la logique de création d’objets est complexe, ou sujette à variation, et que vous voulez __encapsuler cette logique__ en un seul endroit. Cela permet de __découpler__ le code client des classes concrètes instanciées. Par exemple, si la manière de créer un objet dépend de conditions runtime (configuration, type de fichier, etc.), une factory centralise ce choix. C’est idéal _« lorsque la logique de création doit pouvoir varier sans modifier le code client »_. Autre cas : quand on a une famille d’objets à créer (ex : différentes connexions BD), la factory peut sélectionner dynamiquement le bon « produit » selon l’environnement ou les préférences.

__À éviter si :__ si la création d’objets est triviale et ne varie pas, inutile d’alourdir la conception avec un pattern Factory. Par exemple, faire une factory pour instancier une simple classe sans complexité, c’est du zèle injustifié. De plus, n’utilisez pas ce pattern si la diversité d’objets à créer n’évoluera jamais – un simple `new` ou un switch suffira. En somme, _« pas la peine de construire une usine pour cuire un seul œuf »_. 😉

__Intégration avec IoC :__ En .NET Core, le conteneur DI peut parfois __réduire le besoin de factories explicites__. En effet, la DI sert à injecter les dépendances déjà instanciées. Toutefois, pour des besoins de création _à la demande_ d’objets configurés, on peut combiner DI et factory. Par exemple, vous pouvez injecter une factory sous forme de __délégué Func\<T>__ ou d’interface dédiée. .NET Core permet d’enregistrer des méthodes factory : par exemple `services.AddTransient<Func<IPersonnage>>(...);` pour fournir une lambda de création. On trouve également des _patterns_ où la DI résout toutes les implémentations d’une interface et une factory choisit laquelle utiliser selon un paramètre. Autrement dit, la DI s’occupe des __dépendances stables__, et la Factory peut s’occuper des __dépendances dynamiques ou variables__. N’hésitez pas à utiliser les deux de concert : la DI peut injecter dans la factory les services nécessaires à la construction des objets complexes.

__Outils utiles :__ Pas de librairie spéciale nécessaire pour les factories. Cependant, si vous trouvez que vos factories font beaucoup de boulot (beaucoup de _if/else_ pour décider quel objet créer), envisagez des solutions plus déclaratives. Par exemple, utiliser un dictionnaire de types ou un peu de _réflexion_ pour enregistrer automatiquement des correspondances type -> créateur. Des bibliothèques comme __Scrutor__ peuvent scanner des assemblées, mais c’est surtout utile pour le _décorateur_ (voir plus bas) ou l’enregistrement de services. Pour les factories, une bonne veille classe statique ou service DI fait généralement l’affaire.

---

## Le pattern Strategy

_Diagramme UML du pattern Strategy._ Ce diagramme illustre le pattern Strategy : un `Contexte` utilise une interface `IStratégie` pour appeler une méthode qui varie selon l’implémentation concrète de la stratégie. Les différentes stratégies (`ConcreteStrategyA`, `ConcreteStrategyB`...) implémentent la même interface, et le _Contexte_ peut en choisir une à utiliser dynamiquement.

Le pattern Strategy (stratégie) permet de __sélectionner un algorithme à l’exécution__ plutôt que de l’implémenter en dur dans le code. Pour l’analogie humoristique : imaginez un canard espion (vive _Strategy_ dans _Duck_ tales) qui peut changer de comportement de vol ou de coin-coin selon la situation – un coup il vole avec des ailes standards, un coup avec un jetpack. En .NET, cela pourrait se traduire par une classe _Canard_ qui possède une stratégie de vol modifiable (implémentant une interface IVol). Ainsi, notre canard peut à tout moment changer sa stratégie de vol sans que la classe _Canard_ ait à connaître les détails de comment on vole avec un jetpack ou des ailes.

Un exemple concret courant : la __validation de données__. Vous pourriez avoir une classe `Validateur` qui applique une stratégie en fonction du type de données ou du contexte utilisateur. Par exemple, si ce sont des données financières, on utilise une stratégie de validation stricte A, si ce sont des données de profil utilisateur, on utilise une stratégie B plus souple. Le choix peut dépendre de paramètres à l’exécution (configuration, préférences utilisateur, etc.). Grâce au pattern Strategy, on peut encapsuler chaque algorithme de validation dans sa classe et les interchanger facilement.

### Exemple de code Strategy (C#)

Imaginons un système de paiement qui peut appliquer différentes stratégies de calcul de réduction (soldes, fidélité, etc.) :

```csharp
// Stratégie (interface)
public interface ICalculReduction 
{ 
    decimal AppliquerReduction(decimal prix); 
}

// Stratégies concrètes
public class ReductionSoldes : ICalculReduction 
{ 
    public decimal AppliquerReduction(decimal prix) => prix _ 0.5m;  // -50% soldes 
}
public class ReductionFidelite : ICalculReduction 
{ 
    public decimal AppliquerReduction(decimal prix) => prix _ 0.9m;  // -10% fidélité 
}

// Contexte
public class CalculateurPrix
{
    private ICalculReduction _strategie;
    public CalculateurPrix(ICalculReduction strat) => _strategie = strat;
    public void ChangerStrategie(ICalculReduction strat) => _strategie = strat;
    public decimal CalculerPrixFinal(decimal prixBase) 
        => _strategie.AppliquerReduction(prixBase);
}
```

Ici, `CalculateurPrix` est le contexte qui utilise une implémentation de `ICalculReduction`. On peut changer la stratégie via `ChangerStrategie()`. Si on exécute :

```csharp
var calc = new CalculateurPrix(new ReductionSoldes());
Console.WriteLine(calc.CalculerPrixFinal(100));  // Affiche 50
calc.ChangerStrategie(new ReductionFidelite());
Console.WriteLine(calc.CalculerPrixFinal(100));  // Affiche 90
```

On voit qu’on peut intervertir aisément les algorithmes de réduction à la volée.

__Quand l’utiliser :__ utilisez le pattern Strategy quand vous avez __plusieurs variantes d’un algorithme__ et que vous voulez pouvoir choisir l’implémentation appropriée dynamiquement. Cela permet d’éviter d’énormes blocs de conditions (`if/else` ou `switch`) pour chaque variante. C’est utile par exemple pour : des politiques de tarification différentes (selon client, saison…), des méthodes de tri différentes, des comportements de jeu interchangeables (IA agressive vs défensive…), etc. Le code en est plus flexible, et l’algorithme peut évoluer indépendamment du contexte qui l’emploie.

__À éviter si :__ vous n’avez qu’une seule façon de faire quelque chose (introduire une abstraction pour une seule implémentation n’a pas de valeur). De même, si le choix de l’algorithme _ne change jamais_ à l’exécution et qu’il n’y a pas besoin de substitution facile, le pattern peut être superflu. Par ailleurs, avoir trop de stratégies peut rendre le code difficile à suivre – si tous les 3 mois un développeur rajoute une nouvelle classe de stratégie sans supprimer les anciennes obsolètes, on se retrouve avec une “collection de stratégies” confuse. 😉 Comme toujours, ne complexifiez pas inutilement : pas la peine d’un Strategy pour choisir entre deux valeurs statiques (un simple booléen ferait l’affaire).

__Intégration avec IoC :__ Le pattern Strategy s’intègre bien avec l’injection de dépendances. Plutôt que de créer directement une stratégie concrète dans le code, vous pouvez __injecter__ celle dont le contexte a besoin. Par exemple, dans ASP.NET Core on peut imaginer enregistrer plusieurs implémentations d’une interface (stratégies possibles), puis injecter _la bonne_ en utilisant un qualificateur (par nom, par tag, etc.). .NET Core DI ne gère pas nativement la sélection par nom, mais vous pouvez utiliser des usines (Factories) ou des options de configuration. Une autre approche est d’injecter __IEnumerable\<ICalculReduction>__ de toutes les stratégies et de faire votre choix dans le code (par type ou par critère). Des bibliothèques IoC avancées ou des extensions peuvent aider à _taguer_ des implémentations. En somme, la DI peut fournir les stratégies disponibles et éventuellement un __Strategy Selector__ maison décide laquelle utiliser. Cela reste du fait-main, mais c’est mieux que du new caché dans un coin.

__Outils utiles :__ Il n’y a pas de package “Strategy” à installer, c’est un pattern natif au langage. Cependant, si vous avez beaucoup de stratégies et que leur gestion devient lourde, regardez du côté des techniques de __configuration__ (par exemple, stocker en config la stratégie à utiliser et charger dynamiquement la bonne classe). Pour la sélection via DI, la lib __Scrutor__ (mentionnée plus loin) peut aider à scanner et enregistrer des implémentations, même si elle n’offre pas directement un sélecteur. Pour des besoins très avancés, certains utilisent même des expressions lambda ou du code généré pour éviter les _if_ : cela sort du scope ici, mais sachez que c’est possible.

---

## Le pattern Observer

_Diagramme UML du pattern Observer._ Ce diagramme montre la structure classique _Subject/Observer_ : le sujet (ou _Observable_) maintient une liste d’observateurs et fournit des méthodes pour les attacher/détacher. Lorsqu’un événement survient ou que l’état du sujet change, il appelle la méthode de mise à jour (`update()`) de chacun des Observers inscrits.

Le pattern Observer, c’est le principe du __“pub/sub”__ (publication/souscription) : un objet (le sujet) publie des notifications auxquelles d’autres objets se sont abonnés (les observateurs). Imaginez un __compte Twitter__ : vous (Subject) tweetez un chaton mignon, et tous vos followers (Observers) reçoivent la photo dans leur fil. En C#, on peut penser aux événements .NET (basés sur `EventHandler`) qui implémentent cette logique d’abonnement/notification.

Par exemple, dans une application de bourse en temps réel, une classe _Bourse_ peut notifier plusieurs _Afficheurs_ dès qu’un cours d’action change. Les afficheurs se sont abonnés au sujet _Bourse_ pour recevoir automatiquement les updates. Le gros avantage est le __découplage__ : _Bourse_ n’a pas besoin de connaître les détails de chaque afficheur, il se contente de pousser l’info à tous. Comme disait l’autre : _« Si ça se trouve, je parle dans le vide »_, mais avec Observer vous êtes sûr qu’il y a bien quelqu’un de l’autre côté qui écoute – sinon il ne se serait pas abonné. 😅

### Exemple de code Observer (C#)

Utilisons les interfaces du .NET _framework_ pour illustrer (IObserver/T observable). Supposons un sujet _StationMeteo_ qui notifie des _Afficheurs_ :

```csharp
// Le sujet (observable)
public class StationMeteo : IObservable<int>
{
    private List<IObserver<int>> _abos = new List<IObserver<int>>();
    public IDisposable Subscribe(IObserver<int> observer) 
    {
        _abos.Add(observer);
        return new Desinscription(() => _abos.Remove(observer));
    }
    public void NouvelleTemperature(int temp)
    {
        foreach (var abo in _abos)
            abo.OnNext(temp);
    }
    // Classe interne pour gérer la désinscription
    private class Desinscription : IDisposable 
    { 
        private Action _desinscrire;
        public Desinscription(Action desinscrire) => _desinscrire = desinscrire;
        public void Dispose() => _desinscrire();
    }
}

// Un observateur concret
public class AfficheurConsole : IObserver<int>
{
    public void OnNext(int value) 
        => Console.WriteLine($"Température reçue : {value}°C");
    public void OnCompleted() { }
    public void OnError(Exception error) { }
}
```

On peut alors tester :

```csharp
var station = new StationMeteo();
var abo = station.Subscribe(new AfficheurConsole());
station.NouvelleTemperature(25);  // "Température reçue : 25°C" s'affiche
abo.Dispose(); // se désinscrit
```

Ici, `StationMeteo` notifie tous les `IObserver<int>` inscrits via `Subscribe`. Chaque observateur (ici un `AfficheurConsole`) reçoit les mises à jour via `OnNext`. On utilise le pattern _Push_ de .NET (IObservable/IObserver) mais on aurait pu coder manuellement une méthode `Notifier()` dans StationMeteo qui appelle une méthode `MiseAJour(temp)` sur chaque observateur, le principe est identique.

__Quand l’utiliser :__ le pattern Observer est tout indiqué dès qu’on a un scénario __One-to-many__ d’annonces d’événements ou de changements d’état. Typiquement : systèmes de GUI (ex: un bouton observable à des dizaines de listeners pour le clic), systèmes de notifications (messagerie, WebSocket), structures modulaires où certains modules doivent réagir aux changements d’un autre. Par exemple, comme mentionné, les GUI utilisent massivement ce pattern pour mettre à jour les éléments à l’écran dès que les données changent. Si vous avez besoin que __plusieurs parties de l’application réagissent à quelque chose qui se passe ailleurs__, sans les coupler entre elles, Observer est la bonne recette.

__À éviter si :__ vous n’avez qu’un _single_ destinataire pour les infos – dans ce cas un appel direct suffit. Aussi, le pattern Observer peut devenir un piège si mal géré : _fuites de mémoire_ si on n’enlève pas les abonnements (classique en .NET : oublier de se désinscrire d’un event et empêcher la collecte de l’observateur), difficultés de debug car l’appel est déclenché implicitement, et avalanche de notifications. Par exemple, si un sujet a 1000 observateurs mais qu’un seul est réellement concerné par un certain type de message, on aura quand même appelé les 999 autres pour rien. Donc si la distribution n’est pas sélective, ça peut être inefficace. Évitez également d’abuser d’Observer _partout_, au risque d’avoir un système d’événements incontrôlable où plus rien ne transite de façon synchrone et lisible. Comme on dit, _« trop de notifications tuent la notification »_ (sensation familière sur smartphone 😉).

__Intégration avec IoC :__ Le pattern Observer s’intègre moyennement directement dans le DI, car il repose sur un mécanisme d’abonnement dynamique. Néanmoins, votre conteneur de DI peut très bien __fournir__ les instances du sujet et des observateurs. Par exemple, vous pouvez enregistrer `StationMeteo` en singleton (une seule station de mesure) et enregistrer plusieurs `IObserver<int>`. Ensuite, au démarrage de l’appli, injecter `IEnumerable<IObserver<int>>` dans la StationMeteo et abonner automatiquement chaque observer. C’est une façon d’assembler les choses proprement via DI. .NET ne le fait pas tout seul, mais on peut écrire un petit bout de code dans `Program.cs` ou un HostedService qui boucle sur les observers injectés et appelle `station.Subscribe(observer)`.

Par ailleurs, mention spéciale : le pattern Mediator (qu’on verra plus loin) peut parfois remplacer Observer dans une archi en diffusant des notifications de manière centralisée (ex: __MediatR__ a un concept de Notifications où plusieurs _Handlers_ s’inscrivent à un message). C’est un Observer déguisé en Mediator. En résumé, IoC va surtout vous aider à __gérer le cycle de vie__ des sujets/observateurs, mais c’est à vous de brancher les événements.

__Outils utiles :__ En .NET, on a de la chance : le pattern Observer est en quelque sorte intégré via les __événements C#__ (les delegates/events) et l’interface `IObservable<T>` du Reactive Framework. Si votre cas d’utilisation est simple, les events classiques (`event EventHandler`) sont souvent la solution la plus directe – le mot clé `event` encapsule l’ajout/retrait de subscribers et l’invocation. Pour des scénarios plus poussés (composition de flux d’événements), jetez un œil à __Reactive Extensions (Rx)__ qui implémente l’interface IObservable et offre tout un arsenal d’opérateurs pour filtrer, transformer les streams d’événements. __Prism__ (pour les applis WPF) offre un EventAggregator (un Mediator spécialisé dans les events). En bref, vous n’aurez généralement pas besoin d’installer un package juste pour observer, le framework a ce qu’il faut.

---

## Le pattern Decorator

_Diagramme UML du pattern Decorator._ Le schéma montre le composant de base `Component` avec une méthode `Operation()`, un composant concret `ConcreteComponent` qui l’implémente, et un décorateur `Decorator` abstrait qui __dérive de Component tout en ayant un champ vers un Component__ (pour pouvoir l’enrober). Les décorateurs concrets (`DecoratorA`, `DecoratorB`…) ajoutent chacun un comportement avant/après de déléguer l’appel au composant encapsulé.

Le pattern Decorator permet __d’ajouter dynamiquement des fonctionnalités à un objet__, en le “décorant” avec d’autres objets du même type qui enveloppent le composant de base. Pensez à un __sapin de Noël__ 🎄 : l’arbre est votre objet de base, et chaque guirlande ou boule accrochée est un décorateur qui ajoute une feature (de la lumière, de la couleur) sans changer la nature du sapin. En .NET, on retrouve ce concept par exemple avec les flux (`Stream`) où on peut chaîner des _streams_ de décoration (un `BufferedStream` ou un `CryptoStream` qui enveloppe un `FileStream` pour lui ajouter un comportement de buffer ou de chiffrement).

Autre analogie : c’est comme un oignon 🧅 (ou un mille-feuille) – on enrobe couche par couche. Chaque couche implémente la même interface que l’oignon initial, et on peut empiler autant de couches qu’on veut. Au final, quand on appelle une méthode, l’appel traverse toutes les couches.

### Exemple de code Decorator (C#)

Supposons qu’on a une interface `IMessage` représentant un message à envoyer, avec une méthode `Envoyer()`. On veut pouvoir _décorer_ l’envoi de messages en ajoutant des fonctionnalités comme la journalisation ou le chiffrement, sans modifier la classe d’envoi de base.

```csharp
// Composant de base
public interface IMessage 
{ 
    void Envoyer(string contenu); 
}

// Composant concret
public class MessageTexte : IMessage 
{
    public void Envoyer(string contenu)
    {
        Console.WriteLine($"Envoi du message : {contenu}");
    }
}

// Decorator abstrait
public abstract class MessageDecorator : IMessage 
{
    protected readonly IMessage _inner;
    protected MessageDecorator(IMessage inner) { _inner = inner; }
    public virtual void Envoyer(string contenu) 
    {
        _inner.Envoyer(contenu);
    }
}

// Décorateur concret 1 : ajoute de la journalisation
public class MessageAvecLog : MessageDecorator 
{
    public MessageAvecLog(IMessage inner) : base(inner) { }
    public override void Envoyer(string contenu)
    {
        Console.WriteLine("[LOG] Préparation de l'envoi...");
        _inner.Envoyer(contenu);
        Console.WriteLine("[LOG] Message envoyé.");
    }
}

// Décorateur concret 2 : ajoute un "chiffrement" (simulé)
public class MessageChiffre : MessageDecorator 
{
    public MessageChiffre(IMessage inner) : base(inner) { }
    public override void Envoyer(string contenu)
    {
        var texteChiffre = Convert.ToBase64String(
                                System.Text.Encoding.UTF8.GetBytes(contenu));
        _inner.Envoyer($"(chiffré){texteChiffre}");
    }
}
```

On peut utiliser ces décorateurs ainsi :

```csharp
IMessage service = new MessageTexte();
// Ajouter une couche de log
service = new MessageAvecLog(service);
// Ajouter une couche de chiffrement
service = new MessageChiffre(service);

// Maintenant, envoyer un message passera par les deux décorateurs
service.Envoyer("Bonjour");
// Sortie console : 
// [LOG] Préparation de l'envoi...
// Envoi du message : (chiffré)Qm9uam91cg==
// [LOG] Message envoyé.
```

On a démarré avec un `MessageTexte` normal. Puis on l’a décoré avec `MessageAvecLog` (ajoute des logs) puis avec `MessageChiffre` (chiffre le contenu). Au final, l’objet `service` final est un MessageChiffre qui, quand on appelle `Envoyer`, va d’abord logger (via MessageAvecLog) puis chiffrer le texte avant d’appeler le composant de base.

__Quand l’utiliser :__ le pattern Decorator est très utile quand on veut __enrichir progressivement un objet__ sans modifier son code source, et sans exploser le nombre de sous-classes par héritage. Par exemple, pour ajouter des responsabilités comme la __mise en cache__, la __journalisation__, la __sécurisation__ (contrôle d’accès) autour d’un objet de base, les décorateurs sont parfaits. On peut empiler plusieurs décorateurs de natures différentes autour du même objet de base. Le grand avantage est la __flexibilité__ : on compose les comportements à la volée, au lieu d’avoir à coder toutes les combinaisons possibles dans des classes différentes. (Imaginez devoir créer une classe `RepositoryAvecCacheEtLogEtSecu` pour chaque combinaison – ingérable ! Autant avoir 3 décorateurs séparés et les combiner selon le besoin).

__À éviter si :__ l’overdose de couches n’est pas forcément bonne non plus 😅. Si vous commencez à avoir 7 couches de décorateurs autour d’un objet, le débogage devient une chasse au trésor (où est passé mon appel ?). Il faut rester raisonnable et garder une architecture claire. Si l’ordre des décorateurs importe et devient subtil, là encore ça peut être source de bugs difficiles. De plus, si un simple héritage suffit (par ex. vous voulez juste override une méthode unique une fois pour toutes), un décorateur est peut-être overkill. Le décorateur brille surtout pour __les responsabilités orthogonales multiples__ qu’on veut combiner librement. S’il s’agit juste d’étendre un comportement de manière fixe, une sous-classe ou une modification du code existant peut être plus simple.

__Intégration avec IoC (.NET Core) :__ Le décorateur s’intègre un peu difficilement _out of the box_ avec le conteneur Microsoft.Extensions.DependencyInjection, car ce dernier n’a pas de support natif pour “enrober” un service avec un autre. Heureusement, la librairie __Scrutor__ vient à la rescousse ! Scrutor offre une méthode `Decorate` qui permet de prendre un service déjà enregistré et de le remplacer par un décorateur. En gros, vous enregistrez d’abord l’implémentation de base (ex: `services.AddTransient<ITruc, Truc>();`), puis vous appelez `services.Decorate<ITruc, TrucAvecCache>();` pour que toute injection de ITruc fournisse en réalité un TrucAvecCache qui contient un Truc. Cela évite de devoir manuellement chaîner les appels. _Scrutor est un petit ajout qui “branche” ce qui manque dans l’IoC par défaut, sans passer à un conteneur plus lourd._ Comme le dit Andrew Lock, Scrutor ajoute du __Assembly scanning__ et du __service decoration__ à ASP.NET Core. C’est parfait pour implémenter proprement le pattern Decorator dans vos services.

En l’absence de Scrutor, vous pouvez toujours faire l’enrobage à la main dans le code d’initialisation : par exemple, résoudre l’instance de base depuis le provider puis la passer au constructeur du décorateur. Mais soyons honnêtes, c’est fastidieux et ça finit souvent en code spaghetti. Donc __Scrutor__ est l’outil tout trouvé pour la décoration dans .NET Core. En bonus, Scrutor permet aussi de __scanner__ automatiquement vos assemblies pour enregistrer les implémentations (plus besoin de AddTransient 50 fois), ce qui s’avère utile si vous avez beaucoup de décorateurs ou de patterns similaires à enregistrer.

__Outils utiles :__ On l’a dit : __Scrutor__! Pour l’installer : `dotnet add package Scrutor`. Ensuite, utilisez les méthodes d’extension `Scan` et `Decorate`. Par exemple, `services.Scan(scan => scan.FromAssemblyOf<Startup>().AddClasses().AsImplementedInterfaces());` pour enregistrer plein de services d’un coup, puis `services.Decorate<IMonService, MonServiceDecorateur>();` pour chaque décorateur. Scrutor se charge de la plomberie pour que, quand vous demandez `IMonService`, vous obteniez en réalité un `MonServiceDecorateur` qui lui-même contient l’implémentation d’origine. Cela rend l’utilisation des décorateurs __transparente__ pour le code appelant, ce qui est exactement le but recherché du pattern.

---

## Le pattern Chain of Responsibility

_Structure conceptuelle du pattern Chain of Responsibility._ Ce pattern organise une __chaîne d’objets handlers__ où chaque maillon de la chaîne peut traiter ou non une requête et la passer au suivant. Dans l’image ci-dessus (tirée de refactoring.guru), on voit qu’une requête entre par un bout de la chaîne et est examinée successivement par chaque Handler jusqu’à ce que l’un d’eux décide de la prendre en charge (ou qu’on arrive au bout sans preneur).

La Chain of Responsibility (Chaîne de responsabilité) est un patron comportemental qui permet de __faire transiter une requête à travers une chaîne dHandlers__. Chaque handler dans la chaîne a la possibilité soit de traiter la requête (et possiblement de stopper sa propagation), soit de la transmettre au suivant dans la chaîne. Un exemple courant dans la vie réelle : le support client 📞. Votre demande commence avec le __technicien niveau 1__. S’il ne peut pas résoudre, il passe au niveau 2, etc. Cela évite que le client sache _qui_ va résoudre son problème – il appelle un point d’entrée et sa requête navigue jusqu’à trouver le bon interlocuteur. Côté code, ça évite de faire un gros `if` avec tous les cas de figure : on structure chaque condition dans un objet séparé.

Contexte applicatif : imaginons un système d’approbation de dépenses. Une demande de dépense a un montant X. On crée une chaîne de handlers : Chef d’équipe (peut approuver jusqu’à 100€), Directeur (jusqu’à 1000€), CEO (au-delà). La requête “achat PC à 800€” va passer au Chef (100€ insuffisant, il passe au suivant), puis au Directeur (1000€ > 800€, il peut approuver donc il le fait et ne passe pas plus loin). Si la dépense était de 5000€, Chef -> Directeur -> CEO (seul CEO peut approuver). Ce pattern offre une __flexibilité__ : on peut ajouter/retirer des maillons (ex un nouveau manager intermédiaire) sans changer la logique globale, juste en reconfigurant la chaîne.

### Exemple de code Chain of Responsibility (C#)

Implémentons le cas des approbations avec une classe abstraite `Approbateur` et des sous-classes :

```csharp
public abstract class Approbateur
{
    protected Approbateur? Suivant;
    public void DefinirSuivant(Approbateur suivant) => Suivant = suivant;
    public abstract bool TraiterDemande(decimal montant);
}

public class ChefEquipe : Approbateur
{
    public override bool TraiterDemande(decimal montant)
    {
        if (montant <= 100)
        {
            Console.WriteLine("Chef d'équipe: J'approuve la dépense de " + montant);
            return true;
        }
        else if (Suivant != null)
        {
            Console.WriteLine("Chef d'équipe: Je passe au supérieur...");
            return Suivant.TraiterDemande(montant);
        }
        return false;
    }
}

public class Directeur : Approbateur
{
    public override bool TraiterDemande(decimal montant)
    {
        if (montant <= 1000)
        {
            Console.WriteLine("Directeur: J'approuve la dépense de " + montant);
            return true;
        }
        else if (Suivant != null)
        {
            Console.WriteLine("Directeur: Je passe au supérieur...");
            return Suivant.TraiterDemande(montant);
        }
        return false;
    }
}

public class CEO : Approbateur
{
    public override bool TraiterDemande(decimal montant)
    {
        Console.WriteLine("CEO: J'approuve _TOUJOURS_ la dépense de " + montant);
        return true;
    }
}
```

Configuration de la chaîne et test :

```csharp
var chef = new ChefEquipe();
var dir = new Directeur();
var ceo = new CEO();
chef.DefinirSuivant(dir);
dir.DefinirSuivant(ceo);

chef.TraiterDemande(50);    // Chef d'équipe approuve
chef.TraiterDemande(300);   // Chef passe -> Directeur approuve
chef.TraiterDemande(2000);  // Chef passe -> Directeur passe -> CEO approuve
```

On a chaîné `chef -> dir -> ceo`. Le Chef traite les petites demandes, sinon passe au Directeur, etc. Résultat:

_ 50€ : approuvé par Chef.
_ 300€ : Chef ne peut pas (passe), Directeur peut (approuve).
_ 2000€ : Chef passe, Directeur passe, CEO approuve (notre CEO approuve tout, c’est un chic type 😁).

__Quand l’utiliser :__ quand vous avez __une série d’étapes de traitement conditionnel__ à appliquer à une requête, sans savoir d’avance laquelle va réellement la gérer. C’est parfait pour les systèmes de filtres ou de pipelines de traitement. Exemples : chaine de filtres de sécurité (authentification -> validation -> autorisation -> etc.), système de logging avec plusieurs cibles (console, fichier, etc.) où chaque logger essaie ou passe au suivant, moteur de règles métier où chaque règle peut traiter ou non. L’intérêt est d’éviter un couplage rigide entre l’émetteur de la requête et le gestionnaire final : le client envoie la requête dans la chaîne sans connaître qui va s’en charger. On peut facilement __réorganiser ou étendre la chaîne__ sans tout casser.

__À éviter si :__ si l’ordre ou le nombre d’étapes est trivial, une simple séquence d’appels explicites peut suffire. La CoR ajoute de la structure (peut-être inutile) si vous n’avez qu’un seul handler de toute façon ou si vous avez toujours besoin que _tous_ les handlers traitent (dans ce cas, un pattern comme Iterator ou simplement une boucle sur une collection de handlers pourrait être plus clair). Aussi, attention à ne pas créer des chaînes trop longues qui deviennent difficiles à suivre ou impactent les performances (chaîner 50 handlers, c’est potentiellement 50 appels en cascade à chaque requête). Enfin, si la requête doit __absolument__ être traitée par _quelqu’un_ et qu’aucun n’est sûr de la prendre, prévoyez un _handler par défaut_ en fin de chaîne pour éviter qu’elle se perde dans la nature.

__Intégration avec IoC :__ Ce pattern est propice à l’utilisation combinée avec l’IoC, surtout dans ASP.NET Core. D’ailleurs, __le pipeline de Middleware ASP.NET Core est une Chain of Responsibility déguisée__. Vous pouvez configurer vos handlers via DI en enregistrant par exemple tous les `Approbateur` et en les enchaînant par ordre. Par contre, le conteneur ne sait pas tout seul l’ordre ni qui est suivant de qui – c’est à vous de le définir. Une approche : taguer chaque handler avec une priorité et les injeter triés, puis parcourir la liste. Une autre : injeter le _premier_ de la chaîne comme service, et dans sa factory (via Scrutor ou la configuration) lier manuellement ses successeurs. Ce n’est pas ultra-automatique mais DI peut au moins gérer la portée de vos handlers (souvent Transient).

Notons que __MediatR__ (dont on parle juste après) offre la notion de __Pipeline Behaviors__, qui est fondamentalement une CoR qui s’exécute autour du handler principal de Mediator. Par exemple, on peut enregistrer des behaviors de logging, de validation, de timing qui vont s’enrouler autour du traitement d’une requête MediatR et se passer le relai l’un l’autre. C’est une utilisation très élégante de CoR couplée au Mediator.

__Outils utiles :__ Si votre CoR est pour le pipeline HTTP, ASP.NET Core le gère nativement via les __middlewares__ (voir section suivante). Pour vos propres chaînes, il n’y a pas de framework dédié car c’est assez simple à implémenter. Toutefois, mentionnons __Polly__ qui, bien que n’étant pas exactement CoR, permet de composer des _delegates_ pour de la résilience (retry, circuit breaker) d’une manière enchaînée. Ce n’est pas une CoR classique (car tous les delegates agissent), mais ça peut compléter un pipeline. Enfin, __MediatR Pipeline__ comme cité, si vous utilisez MediatR vous pouvez profiter de son système de comportement sans réinventer une CoR séparée.

---

## Le pattern Middleware

Le pattern Middleware est un cas spécifique (et très concret) de Chain of Responsibility, appliqué notamment au traitement des requêtes HTTP dans ASP.NET Core. Un __Middleware__ est une composante logicielle insérée dans un pipeline de requête HTTP, qui a la capacité de faire quelque chose à l’aller (en manipulant la requête) _et/ou_ au retour (en manipulant la réponse), puis de passer la main au middleware suivant dans la chaîne. Pour rester dans les images culinaires : imaginez une chaîne de montage de burgers 🍔 où chaque cuisinier ajoute un ingrédient. Le burger (la requête) passe de station en station : un middleware ajoute la salade, le suivant ajoute le fromage, etc., et à la fin on remet le burger complet au client (la réponse).

En ASP.NET Core, le pipeline global est défini dans `Startup.Configure` avec des appels du style `app.UseXyz(...)` ou `app.Run(...)`. Chaque `app.Use` inscrit un middleware dans la chaîne. Par exemple, `app.UseAuthentication()` ajoute un middleware qui vérifie les tokens JWT en début de requête et, s’il n’y a pas de token ou qu’il est invalide, il peut court-circuiter le reste du pipeline (en renvoyant directement un 401). Sinon, il passe la main au suivant. Ce comportement __« j’exécute ma logique, puis je délègue »__ est exactement celui de la CoR, comme confirmé par Microsoft : _« chaque middleware est responsable d’appeler le suivant ou de court-circuiter le pipeline »_.

### Exemple de code Middleware (C# ASP.NET Core)

Créer un middleware custom en .NET Core consiste à faire une classe avec une méthode `InvokeAsync(HttpContext, RequestDelegate next)` :

```csharp
public class LoggingMiddleware
{
    private readonly RequestDelegate _next;
    public LoggingMiddleware(RequestDelegate next) 
        => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        Console.WriteLine($"→ Traitement de {context.Request.Method} {context.Request.Path}");
        // Appel du middleware suivant dans la chaîne
        await _next(context);
        Console.WriteLine($"← Réponse {context.Response.StatusCode} pour {context.Request.Path}");
    }
}
```

Ce middleware très simple logge l’entrée de la requête (flèche →) et la sortie de la réponse (flèche ←). Pour l’enregistrer, dans `Startup.Configure` :

```csharp
app.UseMiddleware<LoggingMiddleware>();
// ... puis d'autres app.Use...
app.Run(async ctx => { await ctx.Response.WriteAsync("Hello World!"); });
```

Désormais, toute requête passera par `LoggingMiddleware` avant d’atteindre le `Run` final (qui ici envoie "Hello World"). On aurait en console :
→ Traitement de GET /hello
← Réponse 200 pour /hello

On peut naturellement avoir plusieurs middlewares custom ou intégrés. L’ordre des `app.Use` détermine l’ordre de passage. Chaque middleware décide de __continuer ou non__ le pipeline. Par exemple, un middleware d’auth pourrait renvoyer 401 sans appeler `_next` si l’utilisateur n’est pas authentifié.

__Quand l’utiliser :__ dès que vous développez une application ASP.NET Core, vous utilisez forcément le pattern Middleware, ne serait-ce qu’avec les composants intégrés (StaticFiles, Routing, Endpoints, etc.). Vous créerez des middlewares custom pour les besoins cross-cutting : logging global (comme ci-dessus), gestion d’erreurs centralisée (un middleware de _exception handling_ qui attrape toute exception non gérée plus loin et retourne un JSON d’erreur), compression de réponses, modification des en-têtes, etc. Le pattern s’applique aussi hors HTTP, par exemple vous pourriez imaginer un pipeline de traitement de messages où chaque étape est un middleware (même principe). Use Middleware quand vous voulez __chaîner des traitements génériques autour d’une opération principale__, avec possibilité de stop au milieu.

__À éviter si :__ ne détournez pas le concept pour tout et n’importe quoi en dehors du contexte pipeline. Un middleware n’est pas censé contenir la logique métier finale (ex: ne programmez pas votre fonction métier complète dans un middleware alors qu’un contrôleur ou un service métier serait plus clair). Il faut garder le pipeline pour les préoccupations _transverses_. Aussi, faites attention à l’ordre d’enregistrement : un middleware mal positionné peut provoquer des effets de bord (ex: mettre le middleware d’authentification _après_ celui qui gère les endpoints, ça ne marchera pas, car on aura déjà décidé de la réponse). En gros, __évitez de complexifier__ le pipeline plus que nécessaire et documentez-en le fonctionnement si vous ajoutez beaucoup de custom, pour que vos collègues ne se perdent pas.

__Intégration avec IoC :__ Bonne nouvelle, ASP.NET Core _est_ IoC-friendly par design. Vos middlewares peuvent avoir des dépendances et elles seront injectées via le constructeur ! Dans l’exemple `LoggingMiddleware`, on n’en a pas, mais on aurait pu demander `ILoggingService` en paramètre du constructeur, et ASP.NET l’injecterait si enregistré. Le `app.UseMiddleware<T>` se charge d’obtenir une instance du middleware T depuis le conteneur, donc tout ce que vous avez `AddSingleton` ou `AddScoped` est disponible. À noter que par défaut les middlewares sont construits _une fois_ et réutilisés (donc leurs dépendances Scoped deviennent effectively Singletons au sens où c’est la même instance pour chaque requête – sauf si vous inscrivez le middleware via `UseMiddleware` sans passer par DI mais c’est rare). Du coup, faites attention aux middlewares qui devraient être Scoped (il y a des techniques, mais sortons du scope). En général, IoC et Middleware s’aiment bien : vous configurez services et pipeline, et tout se recolle tout seul.

__Outils utiles :__ Vous allez principalement utiliser ce que fournit Microsoft : les méthodes `app.Use...` standard (il y en a une flopée dans les packages Microsoft, par ex `UseAuthentication`, `UseCors`, `UseResponseCompression`, etc.). Ces _middlewares préconstruits_ couvrent la majorité des besoins. Pour des besoins spécifiques, vous écrirez les vôtres comme montré. Vous pouvez éventuellement utiliser le concept d’__IMiddleware__ (une interface qui, si implémentée par votre classe et enregistrée en DI, permet un registre plus propre via app.UseMiddleware sans type générique). Ce n’est qu’un détail. Il n’y a pas de library tierce dominante pour “gérer les middlewares” car le framework fait déjà tout. Néanmoins, rappelez-vous que _Middleware = Chain of Responsibility_ – donc tout outil ou pattern utile pour CoR (comme analyser le pipeline d’appels, mesurer la perf de chaque maillon, etc.) s’applique. Par exemple, __AppMetrics__ ou __Serilog__ peuvent être branchés pour logguer le temps passé dans chaque middleware. Enfin, un conseil outil : utilisez le __Middleware Diagnostics__ (app.UseDeveloperExceptionPage en dev) pour bien voir ce qui se passe en cas d’erreur non gérée, c’est un middleware lui aussi.

---

## Le pattern Mediator

Last but not least, le pattern Mediator ! Si on reste dans les analogies de bureau, imaginez un open-space sans mediator : chaque développeur crie ses questions aux 50 autres, c’est le chaos 🙉. Introduisez un médiateur (par ex. un chef d’équipe ou un channel Slack), et hop : chacun envoie ses messages au médiateur, qui les relaie aux concernés. En conception logicielle, le __Mediator__ est un objet qui encapsule les communications entre plusieurs composants, afin qu’ils ne se parlent pas directement les uns aux autres. __Il réduit les dépendances en évitant le “tout le monde parle à tout le monde”__.

Concrètement en .NET, on trouve souvent ce pattern via la bibliothèque __MediatR__ (de Jimmy Bogard). MediatR implémente un médiateur en-procès : au lieu d’appeler directement des services, un code envoie une __requête__ ou __commande__ à MediatR (par ex. `mediator.Send(new CreerUtilisateurCommand(...))`). Le médiateur reçoit la commande et la transfère au _handler_ approprié (ici une classe `CreerUtilisateurHandler` qui implémente `IRequestHandler<CreerUtilisateurCommand>`). Le caller n’a pas besoin de connaître quel handler va traiter la requête. Résultat : le contrôleur n’a plus de dépendance directe sur le service qui fait réellement l’action, il ne dépend que du Mediator. __Le Mediator centralise et distribue la communication__.

Un autre exemple simple : un chatroom. Chaque participant (colleague) envoie son message au ChatRoom (mediator), qui se charge de l’acheminer à tous les autres participants. Sans chatroom, chaque participant devrait connaître les références de tous les autres pour leur envoyer le message -> combinatoire ingérable quand on a beaucoup d’objets. Le médiateur sert de __hub__ central.

### Exemple de code Mediator (C# style MediatR simplifié)

Sans partir sur MediatR directement, montrons un mini-médiateur de chat :

```csharp
// Colleague (collaborateur) abstrait
public abstract class Participant
{
    protected IChatRoom? _chatRoom;
    public string Name { get; }
    public Participant(string name) { Name = name; }
    internal void SetChatRoom(IChatRoom chatRoom) => _chatRoom = chatRoom;
    public void Envoyer(string destinataire, string message)
    {
        _chatRoom?.EnvoyerMessage(Name, destinataire, message);
    }
    public abstract void Recevoir(string expediteur, string message);
}

// Mediator interface
public interface IChatRoom 
{
    void Inscrire(Participant participant);
    void EnvoyerMessage(string expediteur, string destinataire, string message);
}

// Mediator concret
public class ChatRoom : IChatRoom
{
    private readonly Dictionary<string, Participant> _participants = new();
    public void Inscrire(Participant participant)
    {
        if(!_participants.ContainsKey(participant.Name))
        {
            _participants[participant.Name] = participant;
            participant.SetChatRoom(this);
        }
    }
    public void EnvoyerMessage(string expediteur, string destinataire, string message)
    {
        if(_participants.TryGetValue(destinataire, out var participant))
        {
            participant.Recevoir(expediteur, message);
        }
    }
}

// Colleague concret
public class UtilisateurChat : Participant
{
    public UtilisateurChat(string name) : base(name) {}
    public override void Recevoir(string expediteur, string message)
    {
        Console.WriteLine($"{Name} reçoit de {expediteur}: {message}");
    }
}
```

Testons la chatroom :

```csharp
var chat = new ChatRoom();
var alice = new UtilisateurChat("Alice");
var bob   = new UtilisateurChat("Bob");
chat.Inscrire(alice);
chat.Inscrire(bob);

alice.Envoyer("Bob", "Salut Bob, ça va ?");
bob.Envoyer("Alice", "Hello Alice, bien et toi ?");
```

Ici, la classe `ChatRoom` sert de médiateur : Alice ne connaît pas Bob directement. Quand Alice veut envoyer un message, elle appelle `_chatRoom.EnvoyerMessage(...)` et c’est la ChatRoom qui trouve Bob et lui transmet. On a ainsi découplé les participants (ils dépendent de IChatRoom, pas l’un de l’autre). Sortie attendue :
Bob reçoit de Alice: Salut Bob, ça va ?
Alice reçoit de Bob: Hello Alice, bien et toi ?

__Quand l’utiliser :__ quand vous sentez que __les interactions en “graphe complet” deviennent ingérables__. Par exemple dans une GUI, au lieu que chaque composant manipule les autres (bouton qui appelle directement une méthode du volet de droite, etc.), on introduit un médiateur (souvent appelé _Controller_ ou _Presenter_) qui coordonne tout. En architecture, le Mediator est prisé dans le pattern __CQRS__ pour décorréler l’émission de commandes/queries de leur traitement (via MediatR par ex). Dès que vous voyez beaucoup de classes qui se référencent mutuellement dans tous les sens, il est peut-être temps d’insérer un objet médiateur pour orchestrer les échanges. Ça __réduit le couplage__, facilite la maintenance et les modifications (ajouter un nouveau participant ne nécessite pas de mettre à jour tous les autres, juste l’enregistrer auprès du médiateur).

__À éviter si :__ si vos interactions sont simples et bien structurées déjà. Un mediator introduit de l’indirection, ce qui peut parfois compliquer la compréhension (on ne sait pas forcément qui va réagir quand on envoie un message, on doit chercher dans le médiateur). Aussi, il peut devenir un _god object_ si on lui confie trop de responsabilités. Si votre médiateur commence à avoir 5000 lignes et à connaître toutes les subtilités métier, c’est que vous lui en avez trop donné 😅. Ne l’utilisez pas juste pour « faire du pattern » – s’il n’y a que deux classes qui communiquent, pas besoin d’un tiers arbitre, qu’elles se parlent directement c’est très bien. Le mediator est puissant pour __beaucoup d’objets interconnectés__, sinon il est superflu.

__Intégration avec IoC :__ En .NET Core, l’intégration typique est d’enregistrer le Mediator (par ex. MediatR) et tous ses handlers dans le conteneur. Avec MediatR, c’est facile : un appel à `services.AddMediatR(typeof(Startup));` va scanner les classes implémentant IRequestHandler/IRequest etc. et les inscrire. Ensuite vous injectez `IMediator` (l’interface de MediatR) où vous en avez besoin (ex: dans vos contrôleurs MVC). __MediatR utilise DI__ intensément – chaque Handler peut avoir ses propres dépendances injectées, que MediatR résoudra via le conteneur lorsqu’il exécutera le handler. Bref, IoC + Mediator = ❤️. Si vous faites votre propre médiateur, vous pouvez aussi l’inscrire en singleton, et potentiellement inscrire chaque “colleague” ou handler séparément, puis soit les lier au mediator via code, soit faire comme MediatR et découvrir les handlers via DI. Par exemple, un mediator maison pourrait avoir une méthode d’enregistrement où on lui passe les instances d’handlers (injetées via IEnumerable).

La combinaison Mediator + DI + éventuellement __Pipeline Behaviors__ (via MediatR) donne un système hyper flexible : les composants ne se connaissent pas, tout passe par le mediator, et on peut facilement injecter des comportements transverses (log, validation) autour des handlers. En somme, intégrez bien MediatR dans votre Startup (ou un autre lib similaire), et vous récolterez les fruits d’un code modulaire.

__Outils utiles :__ __MediatR__ est _LE_ package à connaître pour implémenter le pattern Mediator dans les applis .NET modernes. Il gère les requêtes synchrones (Send) et asynchrones, les commandes, les requêtes (qui renvoient un résultat) et même les notifications (plusieurs handlers possibles, style Observer). D’autres existent (par ex. __Mediator__ de Fabio Maulo, plus vieux, ou des implémentations custom), mais MediatR est largement adopté, fiable et simple. Pour l’utiliser efficacement, combinez-le avec un conteneur IoC configuré – ce qui est le cas par défaut en ASP.NET Core. Mentionnons aussi __MassTransit__ ou __NServiceBus__ si vous passez au médiateur distribué (messages via bus), mais là on sort du cadre local. En résumé : Mediator pattern + MediatR = communication en étoile plutôt qu’en toile d’araignée, pour un code plus propre et maintenable.

---

## Conclusion

Pour conclure, les design patterns sont d’excellents outils… à condition de les utiliser avec discernement. En contexte .NET Core, nous avons la chance d’avoir un conteneur d’injection de dépendances et tout un écosystème qui nous simplifie l’implémentation de beaucoup de ces patterns (plutôt que de tout recoder à la main).

En résumé, retenez ces bonnes pratiques :

_ __Singleton :__ gardez-le pour les ressources vraiment uniques, évitez le global state non maîtrisé, et laissez idéalement le DI en créer des instances singleton pour vous.
_ __Factory :__ encapsulez la logique de création complexe, mais inutile de sortir l’artillerie lourde si un `new` suffit. Combinez avec DI en injectant des factories ou en utilisant Func/Delegates pour plus de flexibilité.
_ __Strategy :__ profitez-en pour rendre vos algos interchangeables sans gros _if/else_. Idéal avec DI pour injecter l’algo voulu au bon endroit. Mais ne créez pas des stratégies “pour le sport” si vous n’en avez qu’une seule possible.
_ __Observer :__ super pour les événements et mises à jour multiples. Attention aux abonnements fantômes qui traînent et aux cascades infinies. Les events .NET ou RX sont vos amis, et songez à Mediator/MediatR pour certaines notifications multi-handlers.
_ __Decorator :__ très utile pour les fonctionnalités transverses (log, cache, sécurité). Implementation facilitée par Scrutor dans .NET Core. Ne multipliez pas trop les couches de déco, sinon gare à l’oignon qui fait pleurer en debug.
_ __Chain of Responsibility :__ excellent pour enchaîner des traitements conditionnels (validations, filtres, etc.). On le voit incarné dans les middlewares. Structurez vos handlers proprement et n’hésitez pas à utiliser ce pattern pour le pipeline de traitement métier quand adapté.
_ __Middleware :__ un mot chic pour la CoR HTTP d’ASP.NET Core. Utilisez-le pour tous les aspects globaux de vos requêtes web. Gardez en tête l’ordre, et déléguez toujours au suivant sauf motif valable de couper. Profitez de l’injection de dépendances dans vos middlewares pour les garder propres.
_ __Mediator :__ la pièce maîtresse pour desserrer le couplage dans une appli modulaires/couchee. Avec MediatR, c’est presque plug-and-play pour dispatcher commandes, requêtes et événements sans que les émetteurs et receveurs se connaissent directement. Surveillez juste que votre Mediator ne devienne pas fourre-tout.

En appliquant ces patterns avec jugement, vous rendez votre code plus lisible, évolutif et testable. Et comme toujours, __choisissez le bon pattern pour le bon problème__ : un pattern n’est pas une fin en soi, mais un moyen d’arriver à une conception plus élégante. Maintenant, armé de ces bonnes pratiques et astuces .NET Core, à vous de coder… avec _frenchy_ (et design patterns) style ! 😉

__Sources :__ Bon nombre des concepts abordés proviennent de la documentation et de ressources reconnues sur les design patterns et .NET Core, notamment Refactoring.Guru, les tutoriels Microsoft et les blogs techniques d’experts comme Andrew Lock, CodeMaze et codewithmukesh. N’hésitez pas à consulter ces références pour approfondir chaque pattern ou découvrir des implémentations plus avancées.
