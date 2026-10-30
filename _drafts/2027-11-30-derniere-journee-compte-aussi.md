---
title: "Dates en .NET : la dernière journée compte aussi"
date: 2027-11-30 19:00:00 -0400
categories: [dotnet, bonnes-pratiques]
tags: [dotnet, csharp, dateonly, dates, logique-metier, ef-core, tests]
---

## Préambule

Quand on te dit que ton mandat chez un employeur se termine le 27 novembre, tu comprends que le 27 novembre est ta dernière journée de travail. Cette journée fait encore partie du mandat. Tu as peut-être une dernière livraison à effectuer, quelques dossiers à transférer ou une rencontre de clôture à ton agenda.

Pourtant, dans une application, une simple comparaison peut faire disparaître cette dernière journée. Un opérateur `>` à la place de `>=`, et voilà que ton mandat est considéré comme terminé dès le matin.

La même erreur peut se produire au début. Si ton mandat commence le 2 novembre, tu t’attends à ce qu’il soit effectif le 2 novembre.

Prenons une application .NET pour voir comment traduire cette règle métier dans le modèle et dans les conditions de recherche.

## Un mandat du 2 au 27 novembre

Imaginons une application de gestion du personnel. Elle permet de consulter les mandats effectifs à une date donnée et de sélectionner les employés auxquels attribuer des tâches.

Son entité `Mandat` contient notamment les propriétés suivantes.

| Propriété | Type .NET | Valeur de notre exemple |
| --- | --- | --- |
| `Id` | `int` | `42` |
| `EmployeId` | `int` | `123` |
| `DateDebut` | `DateOnly` | `2026-11-02` |
| `DateFin` | `DateOnly?` | `2026-11-27` |

La règle convenue est explicite : `DateDebut` représente la première journée du mandat et `DateFin` représente sa dernière journée. Une date de fin absente signifie qu’aucune fin n’est définie pour le moment.

Le comportement attendu est donc le suivant.

| Date évaluée | Mandat effectif | Explication |
| --- | --- | --- |
| 1er novembre 2026 | Non | Le mandat n’a pas encore commencé |
| 2 novembre 2026 | Oui | La première journée est incluse |
| 16 novembre 2026 | Oui | La date se situe dans la période |
| 27 novembre 2026 | Oui | La dernière journée est incluse |
| 28 novembre 2026 | Non | Le mandat est terminé |

On parle ici de la validité du mandat au calendrier. L’horaire de travail et les jours ouvrables constituent des règles distinctes.

## Le bogue qui retire la dernière journée

Supposons que l’application utilise cette expression C# pour déterminer si un mandat est effectif. La variable `dateEvaluation` est de type `DateOnly`.

```csharp
mandat.DateDebut <= dateEvaluation &&
(mandat.DateFin == null || mandat.DateFin.Value > dateEvaluation)
```

Le début est correctement inclus. Le problème se trouve dans la comparaison avec `DateFin`.

Le 27 novembre, la condition vérifie si le 27 novembre est strictement après le 27 novembre. Le résultat est faux. L’application exclut donc le mandat pendant sa dernière journée de validité.

Dans notre scénario, l’employé disparaît de la liste des personnes auxquelles on peut attribuer des tâches, alors qu’il travaille encore cette journée-là.

Avec une comparaison stricte sur la date de début, le problème serait similaire. La condition `mandat.DateDebut < dateEvaluation` exclurait la première journée du mandat.

**Lorsque les propriétés désignent la première et la dernière journée de validité, les deux bornes doivent être incluses.**

## Traduire la règle avec des comparaisons inclusives

La condition correcte devient la suivante.

```csharp
mandat.DateDebut <= dateEvaluation &&
(mandat.DateFin == null || mandat.DateFin.Value >= dateEvaluation)
```

Elle exprime trois éléments.

- Le mandat a commencé au plus tard à la date évaluée.
- Une date de fin absente n’impose aucune limite supérieure.
- Lorsqu’une date de fin existe, elle doit être égale ou postérieure à la date évaluée.

Pour un mandat dont les deux dates sont définies, on obtient un intervalle fermé, noté `[DateDebut, DateFin]`. Les deux extrémités font partie de la période.

Une date de fin absente ne rend toutefois pas le mandat immédiatement effectif. Un mandat prévu pour le 2 novembre demeure à venir si on l’évalue le 1er novembre, même si `DateFin` vaut `null`.

Le même prédicat peut être utilisé dans une expression de filtrage EF Core.

```csharp
dbContext.Mandats.Where(x =>
    x.DateDebut <= dateEvaluation &&
    (x.DateFin == null || x.DateFin.Value >= dateEvaluation))
```

Avec SQL Server, la [prise en charge de `DateOnly` dans les propriétés d’entités est intégrée depuis EF Core 8](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-8.0/whatsnew#dateonlytimeonly-supported-on-sql-server). Ces propriétés peuvent être mappées à des colonnes SQL de type `date`.

La règle doit rester cohérente dans les recherches, les validations et les traitements planifiés. Un mandat ne devrait pas être effectif dans un écran et déjà terminé dans un autre à la même date.

## Choisir DateOnly quand le métier parle de journées

Dans notre exemple, le mandat commence et se termine à des dates du calendrier. Il n’y a aucune heure à saisir.

Le type [`DateOnly` représente précisément une date sans composante horaire](https://learn.microsoft.com/fr-fr/dotnet/standard/datetime/how-to-use-dateonly-timeonly#la-structure-dateonly). Il convient donc à ce modèle et permet de comparer directement les journées concernées.

Cela évite un piège fréquent avec `DateTime`. Imaginons une date de fin enregistrée sous la forme `2026-11-27 00:00:00`. Si on la compare à `2026-11-27 14:00:00`, dans le même référentiel horaire, le mandat paraît déjà terminé. Même une comparaison inclusive ne peut pas corriger cette différence, puisque 14 h est après minuit.

**Inclure une date de fin et inclure un instant de fin sont deux décisions différentes.**

Pour notre mandat, conserver des dates avec `DateOnly` permet de représenter directement la règle attendue. Lorsqu’on évalue le mandat « aujourd’hui », il faut néanmoins déterminer cette date dans le fuseau horaire retenu par l’organisation. `DateOnly` ne choisit pas ce fuseau à notre place.

Passer la date d’évaluation explicitement au code métier permet aussi de vérifier le comportement au 2 ou au 27 novembre sans dépendre de la date réelle d’exécution.

## Une fin exclusive peut aussi être correcte

Les dates de début et de fin ne sont pas inclusives par nature. Leur interprétation dépend du contrat métier.

Dans notre scénario, `DateFin` désigne la dernière journée du mandat. L’inclure est donc nécessaire pour respecter ce contrat.

Une application pourrait toutefois gérer une autorisation qui expire à un instant précis. Si l’expiration est fixée au 28 novembre à minuit, l’autorisation n’est plus valide à partir de cet instant. Une période avec début inclus et fin exclue convient alors, souvent notée `[debutInclus, finExclue)`.

Pour représenter la fin de la journée du 27 novembre avec des horodatages, une borne exclusive au début du 28 novembre évite d’inventer une « dernière milliseconde » comme `23:59:59.999`. Les bornes doivent être établies dans le fuseau métier, puis comparées dans un référentiel cohérent.

Le [guide Microsoft sur les types temporels](https://learn.microsoft.com/fr-fr/dotnet/standard/datetime/choosing-between-datetime) distingue notamment les dates civiles et les instants. `DateTimeOffset` permet de représenter un instant avec un décalage UTC, mais ce décalage ne remplace pas les règles d’un fuseau horaire.

Le nom de la propriété doit rendre l’intention claire. Une `DateFin` présentée comme la dernière journée et un `InstantExpiration` présenté comme le début de l’invalidité n’expriment pas la même chose.

## Tester les bornes et les périodes particulières

Un test effectué au milieu du mandat ne détectera pas l’erreur de comparaison. Le 16 novembre, une condition avec `>` ou `>=` sur la date de fin retourne le même résultat.

Les cinq dates du tableau initial constituent donc une bonne base de tests. Elles vérifient la journée précédente, le début, une date intermédiaire, la fin et la journée suivante.

Quelques cas supplémentaires méritent aussi une vérification.

- **Mandat d’une seule journée**. Si le début et la fin sont le 27 novembre, le mandat est effectif le 27 novembre uniquement.
- **Mandat sans date de fin**. Il est effectif à partir de sa date de début, y compris cette journée.
- **Mandat futur sans date de fin**. Il demeure non effectif avant sa date de début.
- **Fin antérieure au début**. Cette période doit être rejetée à la validation.

La validation doit donc autoriser une date de fin égale à la date de début. Exiger une fin strictement postérieure empêcherait de représenter un mandat d’une journée.

Enfin, si deux mandats ne doivent pas se chevaucher, une fin le 27 novembre et un nouveau début le 27 novembre partagent bien une journée. Avec cette convention inclusive, la prochaine journée sans chevauchement est le 28 novembre.

## Conclusion

Une date de début et une date de fin portent une signification métier. Lorsqu’elles représentent la première et la dernière journée d’un mandat, ces deux journées font partie de la période.

En .NET, cette intention se traduit par des valeurs `DateOnly`, une comparaison `<=` sur le début, une comparaison `>=` sur la fin et des tests ciblés sur les bornes. Une fin absente et un mandat d’une seule journée doivent également être traités explicitement.

Le choix entre une fin inclusive et une fin exclusive doit rester cohérent avec ce que les utilisateurs comprennent et avec ce que les propriétés représentent.

Parce qu’une date de fin de mandat, c’est encore une journée de mandat. **La dernière journée, tu la travailles !**
