---
title: "Propriété obligatoire en C# : pourquoi je privilégie required à default!"
date: 2022-11-30 19:00:00 -0400
categories: [developpement, bonnes-pratiques]
tags: [csharp, dotnet, required, nullable, tests-unitaires, autofixture]
---

## Préambule

Lors d’un mandat, je devais ajouter une nouvelle propriété de type `string` à une entité existante. Cette information était obligatoire selon les règles métier et le projet utilisait déjà les types référence nullables.

J’ai donc choisi de déclarer cette propriété avec le modificateur `required`, afin de rendre son initialisation obligatoire dans le code qui crée l’entité.

Pendant la revue de code, un collègue m’a proposé d’utiliser plutôt `default!`. Sa préoccupation était compréhensible. L’entité était utilisée à plusieurs endroits et l’ajout de `required` obligeait à modifier plusieurs tests unitaires.

Cette discussion soulève une question qui dépasse le choix d’un mot-clé. Comment faire évoluer un modèle tout en gardant des tests faciles à maintenir ?

Pour moi, le contrat du modèle doit rester cohérent avec les règles métier. La préparation des données de test doit ensuite faciliter le respect de ce contrat. Il faut toutefois bien comprendre ce que `required` protège réellement.

## Ce que les types référence nullables apportent

Lorsque les types référence nullables sont activés, une propriété de type `string` exprime l’intention de fournir une valeur non nulle. Une propriété de type `string?` permet explicitement l’absence de valeur.

Ces annotations alimentent l’analyse du compilateur. Elles n’ajoutent pas automatiquement de vérification à l’exécution, comme le précise la [documentation sur les types référence nullables](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/nullable-reference-types).

Une propriété non nullable peut donc tout de même contenir `null` à l’exécution si son initialisation est contournée ou mal prise en charge.

Activer cette fonctionnalité devient surtout utile lorsqu’on traite ses avertissements comme des informations sur la cohérence du code.

## required rend l’initialisation explicite

Disponible depuis C# 11, le modificateur `required` permet d’exiger l’initialisation d’une propriété lors de la création d’un objet. La [documentation de Microsoft](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/required) décrit les règles associées à ce contrat.

Prenons une entité `Dossier` dont la propriété `Reference`, de type `string`, est déclarée `required`. Supposons que son constructeur sans paramètre ne déclare pas prendre en charge les membres obligatoires.

L’expression de création doit alors fournir cette information.

```csharp
new Dossier
{
    Reference = "DOS-001"
}
```

L’expression suivante produit une erreur de compilation parce que la propriété obligatoire est absente.

```csharp
new Dossier()
```

Le compilateur signale ainsi les endroits où le code de création doit être adapté. Cela concerne le code applicatif comme le code de test.

Dans le contexte de mon changement, ce signal était utile. Le contrat de création de l’entité venait d’évoluer et les appelants devaient en tenir compte.

## default! laisse une valeur nulle

Pour un type référence comme `string`, la [valeur par défaut est `null`](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/default-values).

L’opérateur `!`, appelé *null-forgiving operator*, indique au compilateur de supprimer les avertissements de nullabilité pour l’expression concernée. Il ne transforme pas la valeur et n’ajoute aucune protection à l’exécution, comme l’explique la [documentation de cet opérateur](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/null-forgiving).

Initialiser une propriété `string` avec `default!` revient donc à lui affecter `null` tout en supprimant l’avertissement associé. Pour cette utilisation, `null!` produit le même résultat.

Si personne ne fournit ensuite une vraie valeur, la propriété reste nulle. Le problème peut apparaître plus tard, lors d’un appel de méthode, d’un traitement métier ou d’une sauvegarde.

Les deux approches expriment des intentions différentes.

| Choix | Effet principal | Limite |
| --- | --- | --- |
| `string?` | Autoriser explicitement une valeur nulle | À utiliser lorsque cette absence a un sens |
| `string` avec `required` | Imposer une initialisation au code appelant | La valeur fournie doit encore être validée |
| `string` initialisée avec `default!` | Supprimer l’avertissement d’initialisation | La propriété reste nulle sans affectation ultérieure |
| `string` initialisée avec `string.Empty` | Fournir une chaîne vide réelle | Une chaîne vide peut violer la règle métier |

Une valeur par défaut doit avoir un sens dans le modèle. Pour une référence obligatoire, remplacer `default!` par `string.Empty` ne suffit donc pas nécessairement.

## required ne valide pas les règles métier

Une propriété initialisée peut encore contenir une valeur invalide.

Ces expressions satisfont l’obligation d’initialisation, même si les valeurs ne conviennent pas à une référence de dossier.

```csharp
new Dossier { Reference = "" }
```

```csharp
new Dossier { Reference = "   " }
```

Il est même possible de fournir explicitement `null` avec un avertissement de nullabilité, ou de supprimer cet avertissement avec `null!`.

**`required` protège contre l’oubli d’initialisation. La validation protège contre les valeurs interdites.**

Si la propriété doit aussi rester inchangée après sa création, l’accesseur [`init`](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/init) peut limiter les affectations depuis le code appelant. Il ne valide pas davantage le contenu de la chaîne.

Il faut donc distinguer l’obligation de fournir une valeur, la validité de cette valeur et les règles qui encadrent ses modifications.

## Aller plus loin dans la protection du domaine

Pour une entité qui porte des invariants importants, je privilégie un constructeur ou une méthode de création qui reçoit les données obligatoires et vérifie leur validité.

Par exemple, une API de création pourrait prendre cette forme.

```csharp
Dossier.Creer(reference: "DOS-001")
```

La méthode aurait la responsabilité de refuser les valeurs interdites avant de retourner une instance utilisable. Selon les conventions du projet, elle pourrait lever une exception ou retourner un résultat d’échec.

Pour une validation simple par exception, [`ArgumentException.ThrowIfNullOrWhiteSpace`](https://learn.microsoft.com/en-us/dotnet/api/system.argumentexception.throwifnullorwhitespace) permet de refuser une chaîne nulle, vide ou composée uniquement d’espaces. Les règles de format et de longueur doivent être ajoutées selon le domaine.

Cette approche doit aussi encadrer les modifications ultérieures. Un constructeur qui valide la référence perd une partie de son intérêt si un accesseur public permet ensuite de lui affecter n’importe quelle valeur.

Dans un modèle ainsi encapsulé, `required` peut devenir inutile pour les propriétés déjà prises en charge par le constructeur. Le choix dépend de l’API de création retenue et du niveau de protection recherché.

## Des tests affectés révèlent aussi un coût de construction

L’argument de mon collègue mérite d’être pris au sérieux. Une évolution qui oblige à retoucher de nombreux tests peut coûter cher et rendre la maintenance frustrante.

Cependant, lorsqu’une nouvelle donnée devient obligatoire, les tests qui construisent l’entité doivent disposer d’une valeur valide. Même un test qui vérifie un autre comportement a besoin d’un objet cohérent avec le modèle.

Le problème devient surtout visible lorsque chaque test recrée manuellement une entité complète. La connaissance de sa construction se retrouve alors dispersée dans plusieurs fichiers.

Cela ne prouve pas que tous ces tests sont mal conçus. C’est toutefois une occasion de revoir la façon dont leurs données sont préparées.

Je privilégie une fabrique de test ou un *test data builder* qui fournit une entité valide par défaut. Chaque test précise ensuite les valeurs importantes pour son scénario.

Une fabrique pourrait, par exemple, exposer une méthode comme celle-ci.

```csharp
DossiersDeTest.CreerValide()
```

Un scénario particulier pourrait fournir une référence explicite.

```csharp
DossiersDeTest.CreerValide(reference: "DOS-002")
```

Lorsqu’une propriété obligatoire s’ajoute, la construction commune est mise à jour à cet endroit. Les tests dont le comportement dépend de cette propriété doivent toujours être revus individuellement.

La réutilisation porte sur la logique de création. Chaque appel doit produire une nouvelle entité afin de préserver l’isolation des tests, particulièrement lorsque les objets sont mutables.

Les données qui déterminent le résultat attendu doivent aussi rester visibles dans le test. Une fabrique trop implicite peut rendre un scénario aussi difficile à comprendre qu’une longue préparation répétée.

## Configurer AutoFixture pour créer des entités valides

AutoFixture peut réduire cette préparation, mais il ne connaît pas automatiquement toutes les règles métier de l’application. Une chaîne générée peut être non nulle tout en ayant un format, une longueur ou une signification incorrects.

Sa [documentation sur les personnalisations](https://autofixture.com/docs/fundamentals/customizations/) présente notamment `Register` et `Customize` pour contrôler la création des objets.

Dans une configuration de test, on peut enregistrer une fabrique explicite. Pour notre entité simplifiée, l’expression d’enregistrement serait la suivante.

```csharp
fixture.Register(() => new Dossier
{
    Reference = "DOS-001"
})
```

Les appels à `fixture.Create<Dossier>()` utilisent alors cette fabrique. Elle crée une nouvelle instance à chaque appel et conserve un avantage utile : l’expression `new Dossier` reste soumise aux vérifications de `required` lors de la compilation.

Cette configuration peut être regroupée dans une implémentation de `ICustomization` réutilisée par les tests concernés.

Pour un modèle construit au moyen d’une méthode métier, la fabrique peut appeler directement cette méthode.

```csharp
fixture.Register(() => Dossier.Creer(reference: "DOS-001"))
```

Ces deux exemples correspondent à deux conceptions différentes de l’entité. La configuration choisie doit suivre son véritable mécanisme de création.

La référence fixe simplifie ici l’illustration. Si plusieurs dossiers doivent avoir des références distinctes, la fabrique doit produire des valeurs adaptées à cette contrainte. Une donnée valide individuellement ne garantit pas la cohérence de tout un scénario.

## Vérifier les garanties à la bonne place

Les tests du domaine devraient couvrir les valeurs que la règle métier accepte ou refuse.

Pour une référence obligatoire, cela peut inclure les cas suivants.

- Une valeur valide permet la création du dossier.
- Une valeur nulle est refusée.
- Une chaîne vide ou composée uniquement d’espaces est refusée.
- Une référence dont le format est invalide est refusée.
- Une modification ultérieure respecte les mêmes règles lorsque cette modification est autorisée.

Les autres tests peuvent utiliser la fabrique commune pour obtenir un point de départ valide.

Cela permet de concentrer les tests de validation sur les invariants et de garder les tests de comportement centrés sur leur scénario.

## Les cas où une suppression de nullabilité peut se justifier

L’opérateur `!` garde des usages légitimes lorsque l’initialisation ou la validité est garantie par un mécanisme que l’analyse statique ne peut pas reconnaître.

La [documentation EF Core sur les types référence nullables](https://learn.microsoft.com/en-us/ef/core/miscellaneous/nullable-reference-types) présente notamment des situations liées aux propriétés de navigation et à leur chargement.

Dans ce contexte, il faut pouvoir expliquer quel composant fournit la valeur et à quel moment son accès devient valide. Une navigation déclarée non nullable peut tout de même poser problème si elle n’a pas été chargée.

Cette justification repose sur le fonctionnement réel du système. Le nombre de tests à modifier ne constitue pas, à lui seul, une garantie d’initialisation.

## Faire évoluer aussi la persistance et les contrats d’échange

Ajouter une propriété obligatoire demande également de regarder les données existantes et les autres chemins de création.

Avec les types référence nullables activés, EF Core configure par convention une propriété `string` comme obligatoire et une propriété `string?` comme facultative. La [documentation sur les propriétés d’entité](https://learn.microsoft.com/en-us/ef/core/modeling/entity-properties#required-and-optional-properties) décrit ce comportement.

Le mot-clé C# `required` et la contrainte `NOT NULL` en base de données remplissent donc des rôles distincts. Une contrainte de stockage ne valide pas à elle seule le contenu métier d’une chaîne.

Pour une table déjà alimentée, je vérifie comment les anciennes lignes recevront une valeur valide. Selon le contexte, une migration progressive peut être nécessaire avant d’imposer la contrainte définitive. Une chaîne vide ajoutée automatiquement pour satisfaire la migration mérite la même attention qu’une valeur par défaut dans le code.

Il faut aussi vérifier les contrats exposés. Par exemple, [System.Text.Json peut exiger la présence des propriétés marquées `required`](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/required-properties). Ajouter ce modificateur à un type désérialisé peut donc affecter les messages ou les clients existants.

La présence d’une propriété dans le JSON et l’acceptation de `null` restent deux questions distinctes, comme l’explique la [documentation sur le respect des annotations de nullabilité](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/nullable-annotations).

Ces impacts doivent être traités dans la stratégie d’évolution du modèle, au même titre que les tests.

## Conclusion

Dans la situation rencontrée, je conserve ma préférence pour `required`. La propriété devait être fournie et le compilateur permettait de rendre cette obligation visible aux endroits concernés.

Remplacer cette exigence par `default!` uniquement pour éviter de modifier des tests aurait laissé une possibilité d’initialisation incomplète dans le modèle.

La réponse durable consiste à faire évoluer ensemble le contrat de création, la validation métier et les outils de préparation des tests. Des fabriques adaptées ou une configuration AutoFixture cohérente réduisent le travail répétitif tout en fournissant des entités valides.

`required` constitue une protection utile contre les oublis. L’intégrité du domaine repose ensuite sur l’ensemble des règles qui encadrent la création, la modification et la persistance de l’entité.
