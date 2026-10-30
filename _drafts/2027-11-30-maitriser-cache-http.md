---
title: "ASP.NET Core : maîtriser le cache HTTP du contexte utilisateur"
date: 2027-11-30 19:00:00 -0400
categories: [bonnes-pratiques, architecture]
tags: [dotnet, aspnet-core, angular, http, cache, securite]
---

## Préambule

Au démarrage d’une application Angular, un appel à l’API permet souvent de récupérer le contexte de l’utilisateur connecté. Ce contexte peut contenir ses rôles ou les fonctionnalités à afficher.

Ces informations dépendent de son identité et peuvent évoluer pendant sa session. Leur mise en cache mérite donc une décision explicite. Pour un endpoint qui expose ce contexte, désactiver le stockage dans les caches HTTP constitue un choix prudent. Il faut aussi prévoir comment l’application actualise les données déjà chargées.

## Une configuration ciblée dans ASP.NET Core

Sur l’action d’un contrôleur ASP.NET Core qui retourne le contexte utilisateur, on peut appliquer cet attribut C# :

```csharp
[ResponseCache(Location = ResponseCacheLocation.None, NoStore = true)]
```

Cette combinaison produit les en-têtes suivants :

```http
Cache-Control: no-store,no-cache
Pragma: no-cache
```

L’attribut configure les en-têtes de la réponse. La [documentation Microsoft sur le cache de réponse](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/response?view=aspnetcore-10.0) confirme ce comportement pour la combinaison de `NoStore = true` et de `Location = ResponseCacheLocation.None`.

Il est inutile d’ajouter `Duration = 0` dans cette configuration. Selon la [documentation de `NoStore`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.responsecacheattribute.nostore?view=aspnetcore-10.0), ASP.NET Core ignore la propriété `Duration` lorsque `NoStore` vaut `true`. La retirer rend donc l’intention plus lisible sans modifier le comportement.

Cette politique peut rester ciblée sur l’action concernée. Les autres endpoints peuvent conserver une stratégie de cache adaptée à leurs données.

## Comprendre les directives

Les directives HTTP suivantes, définies dans la [RFC 9111 sur le cache HTTP](https://www.rfc-editor.org/rfc/rfc9111.html#section-5.2.2), ont des effets différents :

| Directive | Effet |
| --- | --- |
| `no-store` | Interdit aux caches HTTP conformes de stocker la réponse pour la réutiliser |
| `no-cache` | Autorise le stockage, mais impose une validation auprès du serveur avant réutilisation |
| `private` | Interdit le stockage dans un cache partagé, mais autorise un cache privé comme celui du navigateur |
| `max-age=0` | Rend la réponse immédiatement périmée, sans interdire son stockage |

Pour empêcher la conservation du contexte dans les caches HTTP, la directive déterminante est donc `no-store`. `no-cache` seul ne répond pas à cet objectif.

ASP.NET Core ajoute aussi `Pragma: no-cache` avec cette configuration, dans une logique de compatibilité historique. Pour les caches modernes, `Cache-Control` porte la politique à appliquer.

## Angular conserve sa propre responsabilité

Une fois la réponse reçue, l’application peut conserver le contexte dans un service, un signal, un store ou un stockage explicite comme `localStorage`.

Les en-têtes HTTP ne réinitialisent pas cet état applicatif.

Prenons un exemple. À sa connexion, une personne est coordonnatrice pour deux régions. Angular charge ce contexte et le conserve dans un service. Un administrateur retire ensuite une région de son périmètre.

Si l’application continue de lire l’objet déjà présent dans le service, elle peut afficher l’ancien périmètre. L’attribut `ResponseCache` ne déclenche aucun nouvel appel et ne transmet aucune notification de changement.

Il faut donc définir le cycle de vie du contexte. Selon les besoins, cela peut comprendre son chargement après la connexion, son actualisation après une modification connue et sa réinitialisation à la déconnexion ou au changement de compte.

La [documentation Angular sur les requêtes HTTP](https://angular.dev/guide/http/making-requests) précise que les observables produits directement par `HttpClient` déclenchent une requête à chaque souscription. Un mécanisme applicatif qui conserve et réutilise une valeur peut toutefois éviter ce nouvel appel. C’est cette logique qu’il faut examiner lorsqu’un ancien contexte reste affiché.

## Vérifier aussi les mécanismes côté serveur

Une réponse HTTP nouvellement produite peut contenir des données provenant d’un cache applicatif. L’attribut ne vide pas `IMemoryCache` ou un cache distribué utilisé pour construire le contexte.

Il faut également distinguer `ResponseCache` du [middleware `OutputCache`](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/output?view=aspnetcore-10.0). Ce dernier utilise ses propres politiques. Par défaut, il exclut les réponses aux requêtes authentifiées, mais les politiques personnalisées doivent être examinées séparément.

La fraîcheur du contexte dépend donc aussi de la source utilisée par l’API et de ses règles d’invalidation.

## **Maintenir les autorisations dans l’API**

Le contexte transmis à Angular permet d’adapter l’interface. L’API doit néanmoins vérifier les autorisations nécessaires pour chaque opération protégée.

Si une permission est retirée du périmètre d’une personne, masquer le bouton correspondant dans l’interface ne suffit pas à protéger les données. Le serveur doit appliquer les règles d’accès pertinentes, avec une stratégie adaptée pour tenir compte des changements de droits.

La [documentation Angular sur les gardes de navigation](https://angular.dev/guide/routing/route-guards) rappelle d’ailleurs que ces contrôles côté client ne peuvent pas constituer l’unique contrôle d’accès.

## Vérifier le comportement réel

Pour valider cette configuration dans l’application, je recommande de vérifier trois situations :

1. Inspecter les en-têtes réellement reçus sur la réponse du contexte utilisateur, en passant par le proxy ou la passerelle utilisés en environnement déployé.
2. Modifier un rôle ou une permission, puis vérifier que le mécanisme d’actualisation recharge le contexte attendu.
3. Se déconnecter et ouvrir une session avec un autre compte, puis vérifier que l’ancien contexte applicatif a été réinitialisé.

Ces vérifications couvrent à la fois la politique HTTP et la gestion de l’état dans l’application.

## Conclusion

Pour un endpoint qui retourne le contexte de l’utilisateur connecté, `ResponseCache` avec `NoStore = true` et `Location = ResponseCacheLocation.None` exprime clairement la volonté d’empêcher son stockage dans les caches HTTP conformes. `Duration = 0` est superflu dans cette configuration.

La cohérence du résultat repose ensuite sur une gestion explicite du contexte dans Angular, une stratégie de fraîcheur côté serveur et des autorisations appliquées par l’API. Ces responsabilités se complètent pour que l’interface et les opérations reflètent les droits de l’utilisateur.
