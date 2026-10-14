---
title: "GC.SuppressFinalize dans les tests : quand peut-on l’omettre ?"
date: 2027-12-31 19:00:00 -0400
categories: [dotnet]
tags: [csharp, tests, xunit, sealed, idisposable, bonnes-pratiques]
---

## Préambule

Lorsqu’une classe de tests implémente `IDisposable`, on retrouve parfois un appel à `GC.SuppressFinalize(this)` à la fin de sa méthode `Dispose()`. Cette ligne est familière, puisqu’elle apparaît dans de nombreux exemples du pattern de libération des ressources en .NET.

Mais sa présence mérite d’être justifiée par la conception de la classe.

Pour une classe de tests déclarée `sealed`, sans finaliseur dans sa hiérarchie, cet appel n’apporte rien. On peut l’omettre et garder une méthode `Dispose()` centrée sur le nettoyage nécessaire.

Pour comprendre pourquoi, il faut distinguer la libération explicite des ressources, la finalisation et le rôle de l’héritage.

## Dispose et finalisation répondent à des besoins différents

Une méthode `Dispose()` permet de libérer explicitement les ressources dont un objet est responsable. Son exécution dépend d’un appel effectué par le code consommateur, un mécanisme comme `using` ou un framework qui gère la durée de vie de l’objet.

Le garbage collector récupère la mémoire managée des objets devenus inaccessibles. Il ne se charge pas d’appeler automatiquement `IDisposable.Dispose()`. La [documentation sur le nettoyage des ressources non managées](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/unmanaged) explique pourquoi ces ressources demandent un mécanisme de libération explicite.

Un finaliseur constitue un mécanisme distinct. Il permet notamment de prévoir un nettoyage de secours pour une ressource non managée détenue directement par un objet. En C#, sa déclaration prend la forme `~NomDeClasse()`.

Son exécution est non déterministe. On ne choisit pas le moment où il sera exécuté, et certaines circonstances peuvent empêcher son exécution. La [documentation de `Object.Finalize`](https://learn.microsoft.com/en-us/dotnet/api/system.object.finalize) précise ces limites.

Implémenter `IDisposable` ne crée donc pas automatiquement un finaliseur.

## Ce que fait réellement GC.SuppressFinalize

L’appel à `GC.SuppressFinalize(this)` demande au runtime de ne pas exécuter le finaliseur de l’instance courante.

Dans une classe finalisable, il intervient normalement après le nettoyage explicite. Une fois les ressources concernées libérées, le finaliseur n’a plus à effectuer ce travail.

La [documentation de `GC.SuppressFinalize`](https://learn.microsoft.com/en-us/dotnet/api/system.gc.suppressfinalize) précise que l’appel n’a aucun effet si l’objet ne possède pas de finaliseur.

Cet appel ne déclenche pas une collecte de mémoire, ne remplace pas `Dispose()` et ne libère pas les objets stockés dans les champs de la classe.

Le paramètre `this` est essentiel pour comprendre sa portée. Dans une classe de tests, il désigne l’instance de cette classe. Il ne désigne pas le contexte de données, le flux ou la connexion qu’elle utilise.

Il faut donc examiner la finalisation de l’objet courant pour décider si cet appel a une utilité.

## Pourquoi sealed simplifie la décision

Le mot-clé [`sealed`](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/sealed) empêche de créer une classe dérivée.

Pour une classe de tests autonome, cela permet de connaître tout le comportement de nettoyage à considérer. Aucune sous-classe ne pourra ajouter un finaliseur ou des ressources supplémentaires à prendre en charge.

La règle applicable est alors précise :

**Une classe scellée, sans finaliseur déclaré dans sa classe ni dans ses classes de base, n’a pas besoin d’appeler `GC.SuppressFinalize(this)`.**

Il faut toutefois vérifier la hiérarchie. Une classe scellée peut déclarer son propre finaliseur ou dériver d’une classe qui en déclare un. La [spécification C# sur les finaliseurs](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/classes#1513-finalizers) précise que la finalisation d’une instance fait intervenir les finaliseurs de sa chaîne d’héritage.

`sealed` ferme la possibilité d’ajouter des classes dérivées. Il ne supprime aucun comportement de finalisation déjà présent.

## Le cas courant d’une classe de tests xUnit

Prenons une classe `LecteurDossierTests`, déclarée `sealed`, qui implémente `IDisposable`. Elle ne possède pas de classe de base personnalisée et ne déclare aucun finaliseur.

Chaque instance crée un `StringReader` dans un champ `_source`, puis l’utilise pour tester la lecture et la normalisation du nom d’un dossier.

Avec xUnit, chaque test exécuté reçoit une nouvelle instance de la classe de tests. Le framework appelle ensuite `Dispose()` pour effectuer son nettoyage. Ce fonctionnement est décrit dans la [documentation sur le contexte des tests](https://xunit.net/docs/shared-context).

Dans cet exemple, les deux appels envisagés n’ont pas la même utilité :

| Appel dans Dispose | Rôle | Décision pour cet exemple |
| --- | --- | --- |
| `_source.Dispose()` | Effectuer le nettoyage du lecteur possédé par la classe | À conserver |
| `GC.SuppressFinalize(this)` | Empêcher la finalisation de l’instance de tests | À omettre, puisqu’aucun finaliseur n’est présent |

La méthode `Dispose()` peut donc se limiter à l’appel `_source.Dispose()`.

La [documentation .NET sur l’implémentation de Dispose](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-dispose#cascade-dispose-calls) présente justement une classe scellée qui délègue son nettoyage à un objet disposable qu’elle possède, sans ajouter `GC.SuppressFinalize`.

Ce raisonnement s’applique aussi à une classe de tests qui possède son propre `DbContext`. Le nettoyage du contexte reste nécessaire, sans rendre la classe de tests elle-même finalisable.

## Pourquoi l’appel reste pertinent dans une classe ouverte

Une classe conçue pour être héritée doit tenir compte des besoins de ses futures classes dérivées.

Même si la classe de base ne déclare aucun finaliseur, une classe dérivée pourrait en ajouter un. L’appel à `GC.SuppressFinalize(this)` dans l’implémentation de base s’applique alors à l’instance réelle, qui peut être celle du type dérivé.

La règle d’analyse [CA1816](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca1816) explique cette raison de conserver l’appel dans une classe non scellée, même sans finaliseur propre. Elle propose également de sceller le type lorsqu’il n’est pas destiné à l’héritage.

Pour une classe de base qui introduit `IDisposable`, le pattern extensible habituel repose sur une méthode publique `Dispose()` et une méthode `protected virtual void Dispose(bool disposing)`. Cette dernière permet aux classes dérivées de participer au nettoyage, comme le décrit la règle [CA1063](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca1063).

Le choix de sceller la classe doit donc refléter son rôle. Je privilégie cette déclaration pour une classe de tests qui regroupe ses propres scénarios et n’a pas été conçue comme point d’extension.

## Les ressources possédées gardent leur propre responsabilité

Une classe peut utiliser un objet qui encapsule des ressources natives sans gérer elle-même leur finalisation.

Par exemple, lorsqu’une ressource native est encapsulée dans un `SafeHandle`, le code propriétaire appelle sa méthode `Dispose()`. Le mécanisme de finalisation de secours relève du `SafeHandle`. Microsoft recommande cette approche dans sa [documentation sur les ressources non managées](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/unmanaged).

La présence d’un objet finalisable dans un champ ne justifie donc pas, à elle seule, l’ajout d’un finaliseur à la classe qui le contient.

La propriété de la ressource demeure aussi déterminante. Une ressource locale à un seul test peut être encadrée par `using`. Lorsqu’elle appartient à une fixture partagée, son nettoyage relève de cette fixture, selon la durée de vie décrite par [xUnit](https://xunit.net/docs/shared-context#class-fixtures).

## Une convention simple pour les révisions de code

Pour décider quoi conserver, je pars de la conception de la classe :

| Situation | Traitement de GC.SuppressFinalize |
| --- | --- |
| Classe scellée, sans finaliseur dans sa hiérarchie | Omettre l’appel |
| Classe de base non scellée qui introduit IDisposable | Conserver l’appel dans le pattern extensible |
| Classe avec un finaliseur qui protège ses ressources natives | Supprimer la finalisation après le nettoyage explicite réussi |
| Classe dérivée d’une base disposable | Respecter le contrat de nettoyage de la base, même si la classe dérivée est scellée |

Pour ce dernier cas, on complète généralement `Dispose(bool)` et on appelle l’implémentation de base, selon le [pattern pour une classe dérivée](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-dispose#implement-the-dispose-pattern-for-a-derived-class).

La méthode `Dispose()` doit aussi supporter plusieurs appels sans refaire un nettoyage dommageable ni provoquer d’erreur. Cette propriété, appelée idempotence, fait partie des [recommandations du pattern Dispose](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/dispose-pattern#basic-dispose-pattern). Selon les ressources utilisées, leur propre implémentation suffit ou un suivi explicite de l’état est nécessaire.

Dans le cas courant des tests scellés sans finaliseur, le bénéfice recherché est surtout la lisibilité. Retirer l’appel rend le nettoyage plus facile à comprendre, sans prétendre apporter un gain de performance mesurable à la suite de tests.

## Conclusion

`GC.SuppressFinalize(this)` a une responsabilité précise : empêcher la finalisation de l’instance courante lorsque celle-ci n’est plus nécessaire.

Pour une classe de tests scellée, sans finaliseur dans sa hiérarchie, cet appel est superflu. On peut conserver une méthode `Dispose()` simple, qui libère les ressources possédées par la classe.

La présence d’un finaliseur ou d’un contrat d’héritage demande une analyse différente. C’est cette conception qui doit guider l’implémentation du nettoyage.

Dans mes classes de tests, je privilégie donc `sealed` lorsque l’héritage n’est pas prévu, puis une implémentation de `Dispose()` proportionnée aux ressources à libérer.
