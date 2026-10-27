---
title: "MediatR : IMediator ou ISender dans un contrôleur ASP.NET Core ?"
date: 2026-10-26 19:00:00 -0400
categories: [dotnet, architecture]
tags: [dotnet, aspnetcore, mediatr, solid, injection-dependances]
---

## Préambule

La journée avait été longue. C’était la fin de journée et il me restait à envoyer une requête, une *query*, avec MediatR depuis un contrôleur ASP.NET Core. Une tâche assez habituelle, jusqu’à ce que mon cerveau décide de passer en mode économie d’énergie sans me demander mon avis.

Petite crampe au cerveau. Quelle interface fallait-il déjà injecter ?

J’ouvre un autre contrôleur du projet pour me rafraîchir la mémoire et je tombe sur `IMediator`. Le réflexe aurait été de reprendre la même chose et de terminer ma journée. Après tout, si c’est déjà dans le projet, ça doit être le bon choix, non ?

Puis je m’arrête. Pourquoi ce contrôleur utilise-t-il `IMediator` alors qu’il fait uniquement des appels à `Send()` ? En vérifiant les interfaces, je retrouve celle que je cherchais, `ISender`. Et je réalise que c’est aussi celle qu’on aurait dû privilégier dans l’autre contrôleur.

Ma crampe au cerveau venait de se transformer en petite revue de code improvisée. Comme quoi, même un cerveau à 3 % de batterie peut encore trouver quelque chose.

**La leçon du jour : toujours revalider ce qu’on lit et ne rien prendre pour acquis.** Un exemple existant mérite autant notre jugement que le code qu’on est en train d’écrire.

## Trois interfaces, des capacités différentes

MediatR propose trois interfaces qui permettent d’exprimer les besoins de la classe qui les utilise.

| Interface | Capacités exposées | Besoin auquel elle répond |
| --- | --- | --- |
| `ISender` | `Send()` et `CreateStream()` | Envoyer des commandes ou des requêtes, y compris des requêtes retournant un flux |
| `IPublisher` | `Publish()` | Publier des notifications |
| `IMediator` | Les capacités de `ISender` et de `IPublisher` | Envoyer des requêtes et publier des notifications |

Le [contrat de `ISender`](https://github.com/LuckyPennySoftware/MediatR/blob/main/src/MediatR/ISender.cs) expose l’envoi d’une requête vers un handler, ainsi que la création d’un flux de résultats avec `CreateStream()`. Il convient aussi bien aux commandes qu’aux requêtes de lecture.

De son côté, [`IPublisher`](https://github.com/LuckyPennySoftware/MediatR/blob/main/src/MediatR/IPublisher.cs) expose la publication d’une notification, qui peut être prise en charge par zéro, un ou plusieurs handlers.

Quant à `IMediator`, sa [déclaration dans le code source](https://github.com/LuckyPennySoftware/MediatR/blob/main/src/MediatR/IMediator.cs) montre directement la relation entre les trois interfaces.

```csharp
public interface IMediator : ISender, IPublisher
{
}
```

Injecter `IMediator` donne donc accès à l’ensemble de ces capacités. La question devient alors très concrète : lesquelles le contrôleur utilise-t-il réellement ?

## Un contrôleur qui envoie des requêtes a besoin de ISender

Prenons un contrôleur qui reçoit une requête HTTP, construit une requête applicative comme `ObtenirOrganisation`, la transmet avec `Send()` et traduit le résultat en réponse HTTP.

Dans ce scénario, **`ISender` exprime précisément le besoin du contrôleur**. Il peut transmettre la demande à l’application sans exposer la publication de notifications dans sa dépendance.

En ne montrant que la déclaration du contrôleur et son constructeur principal, l’ajustement se résume à ceci.

```diff
-public sealed class OrganisationsController(IMediator mediator) : ControllerBase
+public sealed class OrganisationsController(ISender sender) : ControllerBase
```

Les actions qui appellent déjà `Send()` peuvent conserver leur logique. Le contrôleur dépend simplement d’une interface plus ciblée.

## Une application concrète du principe de ségrégation des interfaces

Le principe de ségrégation des interfaces, le « I » de SOLID, nous invite à éviter qu’une classe dépende de capacités dont elle n’a pas besoin.

Dans notre exemple, le contrôleur utilise l’envoi de requêtes. Injecter `ISender` rend cette intention visible dès le constructeur.

Ce choix apporte trois bénéfices simples.

- La dépendance décrit plus fidèlement le rôle du contrôleur.
- Le contrat accessible au contrôleur contient moins de capacités inutilisées.
- L’ajout éventuel d’une publication de notifications demande de revoir explicitement cette dépendance.

Le code qui utilise `IMediator` peut très bien fonctionner. **Le choix de `ISender` améliore surtout la précision du contrat et la lisibilité de l’intention.** Le compilateur vérifie que les types sont compatibles, mais il nous laisse encore le plaisir de réfléchir à nos responsabilités.

## Les handlers et le pipeline restent les mêmes

Avec la configuration standard de MediatR, remplacer `IMediator` par `ISender` pour continuer à appeler `Send()` conserve le même mécanisme de traitement.

Le [code d’enregistrement des services](https://github.com/LuckyPennySoftware/MediatR/blob/main/src/MediatR/Registration/ServiceRegistrar.cs) montre que la résolution de `ISender` s’appuie sur celle de `IMediator`. On en déduit que ce changement d’interface, à configuration standard inchangée, conserve l’implémentation utilisée pour envoyer la requête.

Le [code qui construit le pipeline des requêtes](https://github.com/LuckyPennySoftware/MediatR/blob/main/src/MediatR/Wrappers/RequestHandlerWrapper.cs) compose ensuite les comportements enregistrés autour du handler concerné.

Les mécanismes déjà configurés dans ce pipeline continuent donc de s’appliquer, selon les requêtes auxquelles ils sont destinés.

- La validation des requêtes.
- La journalisation.
- Les contrôles d’autorisation applicatifs.
- La gestion des transactions.

Ces mécanismes dépendent des comportements que votre application a enregistrés. Injecter l’une ou l’autre interface ne les ajoute pas automatiquement.

Si votre projet utilise des enregistrements ou des décorateurs personnalisés propres à une interface, vérifiez leur configuration avant le remplacement. La garantie d’équivalence décrite ici repose sur l’enregistrement standard.

## Quand IPublisher ou IMediator restent pertinents

Une classe qui publie uniquement des notifications peut dépendre de `IPublisher`.

Une classe qui doit à la fois envoyer des requêtes et publier des notifications peut dépendre de `IMediator`. Son contrat correspond alors aux deux capacités réellement utilisées.

Le choix se fait selon les besoins actuels de la classe. Dans le cas de mon contrôleur, ces besoins se limitaient à `Send()`, ce qui rendait `ISender` approprié.

Cette réflexion s’applique aussi aux autres classes de l’application. Le constructeur est un bon endroit pour rendre leurs dépendances et leurs intentions explicites.

## Conclusion

Pour un contrôleur ASP.NET Core qui utilise uniquement `Send()`, **privilégier `ISender` permet d’exprimer clairement sa responsabilité**. `IPublisher` convient à la publication de notifications et `IMediator` reste pertinent lorsqu’une classe a besoin des deux capacités.

Avec la configuration standard de MediatR, ce choix conserve les handlers et le pipeline existants. Le bénéfice se situe dans la précision de la dépendance et la compréhension du code.

Mais le rappel le plus utile de cette fin de journée dépasse MediatR. Le code déjà présent dans un projet mérite lui aussi d’être relu à la lumière du besoin actuel. Avant de reprendre un exemple, quelques instants de vérification peuvent éviter de reproduire une décision qui mérite d’être ajustée.

Cette fois-ci, ma crampe au cerveau aura au moins servi à quelque chose. Je vais quand même éviter d’en faire une stratégie de revue de code.
