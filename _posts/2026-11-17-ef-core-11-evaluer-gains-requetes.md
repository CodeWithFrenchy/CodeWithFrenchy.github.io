---
title: "EF Core 11 : évaluer les gains sur les requêtes de son application"
date: 2026-11-17 19:00:00 -0400
categories: [dotnet, ef-core]
tags: [dotnet, dotnet11, ef-core, sql-server, linq, json, performance, migrations]
---

## Préambule

Une mise à jour d’Entity Framework Core peut améliorer une requête sans modifier son expression LINQ. Une jointure disparaît, une fonction SQL remplace une sous-requête ou un index devient plus simple à déclarer dans le modèle.

Ces changements sont intéressants, mais le SQL le plus court n’est pas nécessairement celui qui coûte le moins cher. SQL Server peut déjà éliminer une jointure pendant l’optimisation. À l’inverse, une nouvelle traduction peut rendre un index exploitable et réduire sensiblement le travail effectué.

La session [EF Core 11: Vector Search, Smarter LINQ, and a Friendlier dotnet ef de .NET Conf 2026](https://www.youtube.com/watch?v=dgd37KXqcEQ) présente plusieurs de ces évolutions.

Après avoir couvert différentes nouveautés de [.NET 11](https://codewithfrenchy.com/posts/dotnet11/), je propose ici une démarche pour évaluer leur effet sur les requêtes d’une application. L'article se concentre sur SQL Server, les jointures, JSON, les types complexes et les outils de migration. La recherche vectorielle et les autres fournisseurs méritent des analyses distinctes.

Les exemples utilisent une application de gestion de commandes. Les extraits SQL illustrent les formes à comparer, sans constituer des captures de mesures exécutées pour cet article. Les gains devront être établis sur les données, les paramètres et les versions réellement utilisés par l’application.

## Commencer par une comparaison qui isole les changements

EF Core 11 nécessite .NET 11. Pour comparer EF Core 10 et EF Core 11, je préparerais deux variantes de la même application exécutées sur .NET 11, avec des dépendances compatibles et des versions explicitement fixées.

Le premier passage conserverait le même modèle, le même schéma, les mêmes index et le même niveau de compatibilité SQL. Les changements de modèle et d’index viendraient ensuite.

| Comparaison | Ce qui change | Ce qui reste identique |
| --- | --- | --- |
| Traduction EF Core | Version EF Core 10 ou 11 | Runtime, moteur SQL, compatibilité, données, modèle et index |
| Traduction JSON récente | Version EF Core 10 ou 11 sur SQL Server 2025 | Niveau 170, type physique des colonnes et schéma identiques |
| Effet d’un index | Présence de l’index étudié | Version EF Core, requête, données et configuration SQL |

Si le passage à SQL Server 2025 fait aussi partie du projet, il faut mesurer cette transition séparément. Un résultat obtenu après avoir changé le moteur, le fournisseur, le stockage JSON et les index décrit le gain de l’ensemble de ces changements.

Je sélectionnerais quelques requêtes importantes à partir de leur fréquence et de leur coût cumulé. Une amélioration modeste sur une requête appelée des milliers de fois peut avoir plus de valeur qu’une forte accélération sur un écran rarement utilisé.

## Vérifier les trois niveaux de compatibilité SQL Server

Avant les mesures, il faut distinguer la version du moteur, le niveau de compatibilité de la base de données et celui configuré dans EF Core.

Les [changements de compatibilité d’EF Core 11](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-11.0/breaking-changes#sql-server-compatibility-level-now-defaults-to-160) indiquent que `UseSqlServer` passe du niveau implicite 150 au niveau 160, associé à SQL Server 2022.

EF peut alors produire du SQL utilisant des fonctions plus récentes, notamment `LEAST` et `GREATEST`. Une application qui fonctionne avec SQL Server 2019 peut rencontrer des erreurs si elle conserve ce nouveau réglage implicite.

Cette requête permet de relever la version du moteur et le niveau de compatibilité de la base de données courante.

```sql
SELECT
    CAST(SERVERPROPERTY('ProductVersion') AS nvarchar(128)) AS VersionMoteur,
    [name] AS NomBaseDeDonnees,
    [compatibility_level] AS NiveauCompatibilite
FROM sys.databases
WHERE [name] = DB_NAME()
```

Si l’environnement visé reste au niveau 150, l’expression de configuration doit le préciser.

```csharp
optionsBuilder.UseSqlServer(
    chaineConnexion,
    x => x.UseCompatibilityLevel(150))
```

Pour les fonctionnalités étudiées sur SQL Server 2025, le niveau ciblé par EF est 170. Le [fournisseur SQL Server](https://learn.microsoft.com/en-us/ef/core/providers/sql-server/) précise que ce réglage influence la génération SQL, mais ne modifie pas la configuration de la base de données.

Configurer `UseCompatibilityLevel(170)` ne met donc pas le serveur à niveau et n’exécute pas un changement de compatibilité SQL à votre place.

Je conserverais ces trois valeurs avec chaque résultat de test. La mention « testé avec SQL Server » ne permet pas de reproduire une mesure.

## Capturer les commandes réellement exécutées

La méthode [`ToQueryString`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.entityframeworkqueryableextensions.toquerystring?view=efcore-11.0) permet d’inspecter rapidement une requête.

```csharp
requete.ToQueryString()
```

Elle sert au diagnostic. Sa sortie ne remplace pas la capture des commandes réellement exécutées. Les opérateurs terminaux peuvent modifier la requête, et une requête fractionnée peut produire plusieurs commandes.

Pour la comparaison, je conserverais le texte de chaque commande, les paramètres, leurs types et leurs tailles. Remplacer un paramètre par une constante dans SSMS peut changer la compilation du plan et fausser l’analyse.

La [démarche de diagnostic d’EF Core](https://learn.microsoft.com/en-us/ef/core/performance/performance-diagnosis) décrit la journalisation des commandes et l’utilisation de `TagWith` pour les relier aux expressions LINQ.

Un identifiant stable peut être ajouté à la requête étudiée.

```csharp
contexte.Commandes.TagWith("Evaluation.CommandesAvecLignes")
```

Je garderais ce marqueur identique entre les variantes. Les journaux détaillés servent à comprendre le comportement, puis leur coût doit être retiré ou contrôlé pendant les mesures chronométrées.

Il faut aussi distinguer le temps journalisé pour une commande du temps nécessaire à la lecture complète des résultats, à leur matérialisation et à leur sérialisation dans une réponse HTTP.

## Jointures éliminées : observer la requête concernée

Un cas documenté dans les [nouveautés d’EF Core 11](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-11.0/whatsnew#better-sql-for-to-one-joins) concerne les navigations de référence utilisées avec des requêtes fractionnées.

Supposons qu’une commande possède un client obligatoire et plusieurs lignes. L’expression LINQ suivante charge cet ensemble sur un intervalle de dates.

```csharp
contexte.Commandes
    .AsNoTracking()
    .Where(x =>
        x.DateCreation >= debut &&
        x.DateCreation < fin)
    .Include(x => x.Client)
    .Include(x => x.Lignes)
    .AsSplitQuery()
```

Le chargement du client appartient à la requête principale. La commande qui récupère les lignes n’a pas nécessairement besoin de joindre à nouveau la table des clients.

Voici une forme simplifiée de cette seconde commande avant l’amélioration.

```sql
SELECT [l].[Id], [l].[CommandeId], [l].[Quantite], [c].[Id], [c0].[Id]
FROM [Commandes] AS [c]
INNER JOIN [Clients] AS [c0] ON [c].[ClientId] = [c0].[Id]
INNER JOIN [LignesCommande] AS [l] ON [c].[Id] = [l].[CommandeId]
WHERE [c].[DateCreation] >= @debut
  AND [c].[DateCreation] < @fin
ORDER BY [c].[Id], [c0].[Id]
```

La forme simplifiée après élimination de la jointure devient la suivante.

```sql
SELECT [l].[Id], [l].[CommandeId], [l].[Quantite], [c].[Id]
FROM [Commandes] AS [c]
INNER JOIN [LignesCommande] AS [l] ON [c].[Id] = [l].[CommandeId]
WHERE [c].[DateCreation] >= @debut
  AND [c].[DateCreation] < @fin
ORDER BY [c].[Id]
```

Le changement réduit ici le SQL de la commande qui charge la collection. Il ne supprime pas le chargement du client et ne transforme pas l’ensemble en un seul aller-retour.

Les compromis des [requêtes fractionnées](https://learn.microsoft.com/en-us/ef/core/querying/single-split-queries) restent présents, notamment les communications supplémentaires et la cohérence entre plusieurs lectures lorsque les données changent.

### Une jointure absente du SQL n’est pas toujours un gain d’exécution

SQL Server pouvait déjà éliminer une jointure dans son plan si les contraintes lui permettaient de démontrer qu’elle était inutile. Les deux textes SQL peuvent donc aboutir à des plans très proches.

Je vérifierais si un accès à la table des clients disparaît réellement, si les lectures logiques diminuent et si les opérations de tri changent. Si le plan était déjà équivalent, il faut le constater sans transformer la simplification du texte en gain de performance chiffré.

Une jointure qui filtre les lignes, modifie leur multiplicité ou participe à une règle d’accès doit conserver cette sémantique. L’absence de colonnes projetées depuis une table ne suffit pas à conclure qu’elle est inutile.

### Le modèle doit décrire les garanties réellement présentes

Certaines applications importent des données avant que toutes les relations soient établies. Une propriété `ClientId` non nullable ne garantit pas, à elle seule, qu’un client correspondant existe.

L’API [`IsConstrained(false)`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.metadata.builders.referencecollectionbuilder.isconstrained?view=efcore-11.0) permet de décrire une relation sans cette garantie. Les requêtes la traitent alors comme optionnelle, même si la clé étrangère n’est pas nullable.

Ce réglage doit refléter le contrat des données. Il ne devrait pas être modifié simplement pour obtenir un SQL plus court. Les cas avec données absentes et les filtres globaux font partie des validations de résultat.

## JSON : distinguer la nouvelle traduction du nouvel index

### Rechercher dans une collection stockée en JSON

Supposons que les commandes possèdent une collection `Etiquettes` stockée dans une colonne JSON. L’expression à évaluer reste simple.

```csharp
contexte.Commandes
    .Where(x => x.Etiquettes.Contains("prioritaire"))
    .Select(x => x.Id)
```

Une traduction avec `OPENJSON` développe les valeurs de la collection pour effectuer le filtre.

```sql
SELECT [c].[Id]
FROM [Commandes] AS [c]
WHERE N'prioritaire' IN (
    SELECT [e].[value]
    FROM OPENJSON([c].[Etiquettes])
        WITH ([value] nvarchar(max) '$') AS [e]
)
```

Avec EF Core 11 et une cible SQL Server 2025 configurée au niveau 170, cette recherche peut utiliser `JSON_CONTAINS`.

Le prédicat peut s’exprimer ainsi en SQL, avec un chemin explicite vers les éléments du tableau situé à la racine.

```sql
SELECT [c].[Id]
FROM [Commandes] AS [c]
WHERE JSON_CONTAINS([c].[Etiquettes], N'prioritaire', '$[*]') = 1
```

Cet extrait illustre la fonction visée, pas une capture exacte du fournisseur. La [documentation de `JSON_CONTAINS`](https://learn.microsoft.com/en-us/sql/t-sql/functions/json-contains-transact-sql?view=sql-server-ver17) comporte encore une différence entre sa syntaxe générale et ses limitations concernant le caractère obligatoire du chemin. Certains exemples EF omettent ce chemin. La commande produite doit donc être vérifiée sur la version précise du moteur et du fournisseur utilisés.

Cette optimisation concerne une collection stockée dans une colonne JSON. Elle ne décrit pas automatiquement le comportement d’une collection d’identifiants fournie par l’application dans une expression comme `identifiants.Contains(x.Id)`.

La nullabilité compte aussi. EF conserve la traduction avec `OPENJSON` lorsqu’il ne peut pas garantir les conditions nécessaires à la nouvelle traduction concernant les valeurs nulles.

Le gain potentiel vient du travail évité pour développer le document et, lorsqu’un index adapté peut être utilisé, de la réduction des données parcourues. Le plan réel doit confirmer ces effets.

### Tester la présence d’un chemin JSON

Le fournisseur expose aussi `EF.Functions.JsonContains` et `EF.Functions.JsonPathExists`, documentés dans les [correspondances entre fonctions .NET et SQL Server](https://learn.microsoft.com/en-us/ef/core/providers/sql-server/functions#json-functions).

`JSON_PATH_EXISTS`, disponible depuis SQL Server 2022, répond à une question différente de la comparaison d’une valeur.

```sql
SELECT [c].[Id]
FROM [Commandes] AS [c]
WHERE JSON_PATH_EXISTS([c].[Livraison], '$.CodePostal') = 1
```

Le chemin peut exister alors que sa valeur est `null`. Les tests devraient distinguer un chemin absent, une valeur JSON nulle, une chaîne vide et une valeur renseignée. Ce sont des cas métier différents, même si l’interface les présente parfois de la même façon.

## Types complexes : choisir le stockage selon les accès

Les [types complexes](https://learn.microsoft.com/en-us/ef/core/modeling/complex-types) représentent des valeurs structurées sans identité indépendante. Une adresse est un exemple classique. Elle fait partie de son entité et n’est pas interrogée comme une entité autonome.

Ils existaient avant EF Core 11. Cette version étend notamment leur utilisation aux hiérarchies TPT et TPC, y compris avec des colonnes JSON, et permet de définir des clés ou des index à partir de propriétés scalaires imbriquées.

### Indexer une propriété stockée dans des colonnes relationnelles

Si `Adresse` est déjà configurée comme propriété complexe de `Client`, l’expression suivante déclare un index sur son code postal.

```csharp
modelBuilder.Entity<Client>()
    .HasIndex(x => x.Adresse.CodePostal)
```

L’intérêt est de décrire l’index à partir du modèle .NET. Pour un stockage relationnel, il porte sur la colonne correspondante.

Je vérifierais que cet index répond aux filtres et aux tris réellement utilisés. Un index sur le code postal peut être utile à certaines recherches, mais il ne remplace pas automatiquement un index composé qui correspond à un parcours métier plus précis.

### Indexer un chemin dans une colonne JSON native

Pour une propriété complexe `Livraison`, voici deux expressions de configuration à intégrer au modèle lorsque le moteur supporte le type SQL natif `json` et les index JSON.

```csharp
modelBuilder.Entity<Commande>()
    .ComplexProperty(
        x => x.Livraison,
        x => x.ToJson().HasColumnType("json"))
```

```csharp
modelBuilder.Entity<Commande>()
    .HasIndex("Livraison.Ville")
```

Le [support des index JSON dans le fournisseur SQL Server](https://learn.microsoft.com/en-us/ef/core/providers/sql-server/#json-indexes) permet de générer une définition de ce genre.

```sql
CREATE JSON INDEX [IX_Commandes_Livraison_Ville]
ON [Commandes]([Livraison]) FOR (N'$.Ville')
```

Ce n’est pas un index relationnel ordinaire sur une colonne `nvarchar(max)`. Le type physique, les fonctionnalités du moteur et les exigences de [`CREATE JSON INDEX`](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-json-index-transact-sql?view=sql-server-ver17) doivent être respectés, notamment la présence d’une clé de clustering admissible.

La documentation précise aussi qu’un seul index JSON peut être créé par colonne JSON et que sa création s’effectue hors ligne. Il faut regrouper les chemins utiles et prévoir l’effet de cette opération sur une table volumineuse.

Pour mesurer son intérêt, je comparerais la même requête avec et sans cet index. Le coût des écritures, la taille occupée et le temps de création font partie du résultat.

### TPT et TPC deviennent plus flexibles, sans devenir gratuits

Le support des types complexes dans ces hiérarchies réduit certaines contraintes de modélisation. Il ne supprime pas les jointures inhérentes à TPT ni les combinaisons de tables nécessaires à certaines requêtes polymorphes en TPC.

Je choisirais entre colonnes relationnelles et JSON selon les filtres, les mises à jour et les contraintes à appliquer. Regrouper des données dans un document peut simplifier un agrégat, tout en compliquant certains accès transversaux.

Le changement de mapping doit être évalué comme une migration de données et de schéma. Il ne devrait pas être mélangé à la première comparaison des traductions EF Core.

## Comparer les plans d’exécution réels

Le [plan d’exécution réel de SQL Server](https://learn.microsoft.com/en-us/sql/relational-databases/performance/display-an-actual-execution-plan?view=sql-server-ver17) montre les opérateurs utilisés et les informations recueillies pendant l’exécution. Il complète le plan estimé, sans transformer les pourcentages de coût affichés en temps mesuré.

Dans SSMS, on peut activer le plan réel avant d’exécuter la commande capturée. `SET STATISTICS XML ON` permet aussi de récupérer sa représentation XML.

Voici un exemple de batch pour la seconde requête de chargement des lignes. Les noms de tables et les paramètres correspondent au modèle illustratif de cet article.

```sql
SET STATISTICS IO ON
SET STATISTICS TIME ON
SET STATISTICS XML ON

EXEC sys.sp_executesql
    N'SELECT [l].[Id], [l].[CommandeId], [l].[Quantite], [c].[Id]
      FROM [Commandes] AS [c]
      INNER JOIN [LignesCommande] AS [l] ON [c].[Id] = [l].[CommandeId]
      WHERE [c].[DateCreation] >= @debut
        AND [c].[DateCreation] < @fin
      ORDER BY [c].[Id]',
    N'@debut datetime2(7), @fin datetime2(7)',
    @debut = '2026-01-01T00:00:00',
    @fin = '2026-02-01T00:00:00'

SET STATISTICS XML OFF
SET STATISTICS TIME OFF
SET STATISTICS IO OFF
```

Pour l’application réelle, il faut reprendre la commande et la déclaration de paramètres effectivement utilisées. Même avec le même SQL, des options de session différentes entre SSMS et l’application peuvent modifier le plan.

[`STATISTICS IO`](https://learn.microsoft.com/en-us/sql/t-sql/statements/set-statistics-io-transact-sql?view=sql-server-ver17) fournit les lectures physiques et logiques. [`STATISTICS TIME`](https://learn.microsoft.com/en-us/sql/t-sql/statements/set-statistics-time-transact-sql?view=sql-server-ver17) complète l’analyse avec les temps de compilation et d’exécution.

Je conserverais le SQL, les paramètres, les fichiers `.sqlplan` et les messages de statistiques pour chaque variante.

| Point du plan | Ce qu’il faut comparer |
| --- | --- |
| Accès aux tables | Table réellement consultée ou éliminée par l’optimiseur |
| Estimations | Écart entre le nombre de lignes estimé et observé |
| Index | Index choisi et nombre de lignes lues avant filtrage |
| Jointures et tris | Opérateurs conservés, clés utilisées et volume traité |
| Mémoire | Mémoire accordée et éventuels débordements vers `tempdb` |
| Conversions | Conversions implicites pouvant limiter l’utilisation d’un index |

Pour les requêtes fractionnées, l’unité de comparaison est le chargement complet. La disparition d’une jointure dans la seconde commande ne permet pas d’ignorer le coût de la première.

## Utiliser un protocole de mesure reproductible

Je préparerais une copie représentative des données avec les mêmes index et statistiques pour chaque variante. Les paramètres devraient couvrir plusieurs distributions, par exemple une petite période, un mois chargé et une période sans résultat.

Pour un premier protocole, on peut effectuer cinq exécutions d’échauffement, puis cent mesures par jeu de paramètres. Les variantes doivent être alternées pour limiter l’effet d’une charge de serveur qui évolue dans le temps. Ce nombre constitue un point de départ à adapter au coût des requêtes et à la dispersion observée.

Les coûts du premier démarrage sont mesurés séparément. Une requête exécutée une seule fois avec un cache froid ne décrit pas le comportement habituel d’une API déjà active.

La collecte détaillée des plans peut être réalisée dans une passe de diagnostic distincte. La passe chronométrée conserve une instrumentation comparable entre les variantes, sans journalisation détaillée supplémentaire d’un seul côté.

| Mesure | Utilité |
| --- | --- |
| Résultat métier | Même ensemble de résultats, même multiplicité et ordre lorsqu’il est garanti |
| Nombre de commandes | Détecter un changement d’allers-retours |
| Lectures logiques et CPU SQL | Quantifier le travail effectué par le moteur |
| Durée complète | Inclure la lecture et la matérialisation des résultats |
| Médiane et p95 | Observer le comportement courant et les exécutions plus lentes |
| Allocations .NET | Mesurer le coût côté application |
| Coût des écritures | Vérifier les contreparties d’un nouvel index |

Un outil comme BenchmarkDotNet peut aider à mesurer les allocations et le travail côté .NET. Il ne remplace pas l’analyse des plans et du serveur distant. Les conditions de connexion, le suivi des entités et la durée de vie du `DbContext` doivent rester comparables.

Les tests avec SQLite ou le fournisseur InMemory ne valident pas ces traductions SQL Server. Le jeu de tests doit exécuter les requêtes contre le moteur concerné.

Après le déploiement, [Query Store](https://learn.microsoft.com/en-us/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store?view=sql-server-ver17) aide à suivre l’historique des plans et les statistiques d’exécution. Ses statistiques sont agrégées par intervalle. Elles ne constituent pas une capture de plan réel détaillé pour chaque appel.

Si le SQL change, son identification peut aussi changer. Il faut relier les résultats au scénario applicatif, plutôt que supposer qu’un ancien identifiant de requête sera conservé.

## `dotnet ef` : réduire les réglages répétés dans le dépôt

### Partager les options courantes

EF Core 11 introduit le fichier [`.config/dotnet-ef.json`](https://learn.microsoft.com/en-us/ef/core/cli/dotnet#configuration-file). Il permet de définir les options répétées dans une solution où les migrations et le démarrage se trouvent dans des projets différents.

```json
{
  "project": "src/Boutique.Donnees",
  "startupProject": "src/Boutique.Api",
  "context": "BoutiqueDbContext",
  "framework": "net11.0",
  "configuration": "Release"
}
```

Les chemins sont résolus par rapport au répertoire parent de `.config`. L’outil remonte depuis le répertoire courant et utilise le premier fichier trouvé. Une option fournie explicitement sur la ligne de commande conserve la priorité.

On peut ensuite générer une migration et son script sans répéter les projets.

```bash
dotnet ef migrations add IndexLivraisonVille
dotnet ef migrations script --idempotent --output artifacts/migration.sql
```

Le répertoire `artifacts` doit être préparé pour accueillir le script. Le fichier de configuration ne devrait pas contenir les secrets de connexion.

Le gain pour l’équipe est surtout la cohérence. Un développeur, un agent et le pipeline peuvent utiliser les mêmes options de projet. La version de l’outil et les paquets EF Core doivent aussi être fixés explicitement dans le dépôt.

### Garder la génération et l’application des migrations examinables

La session présente également le support des applications C# basées sur des fichiers. Cette possibilité est utile pour préparer un petit scénario de reproduction, mais les mesures doivent conserver la même configuration de modèle que l’application étudiée.

Les nouveautés comprennent aussi l’option `--add` de `dotnet ef database update`, qui génère puis applique une migration dans la même opération. Pour une évolution de stockage JSON ou un nouvel index, je conserverais les étapes distinctes afin d’examiner le SQL avant application.

Dans la console du gestionnaire de packages de Visual Studio, EF Core 11 ajoute `-NoBuild` à `Add-Migration` et `Update-Database`. L’option `--no-build` existait déjà dans la CLI. Dans les deux cas, les assemblies doivent correspondre aux sources courantes.

Les [changements incompatibles](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-11.0/breaking-changes) méritent enfin une lecture ciblée. Le fournisseur SQL Server passe à Microsoft.Data.SqlClient 7.0, et `Migrate` ou `MigrateAsync` lève désormais une exception par défaut lorsqu’aucune migration n’est trouvée. Ces changements peuvent affecter un pipeline indépendamment de la qualité des requêtes.

## Conclusion

EF Core 11 offre plusieurs pistes de gain, notamment la réduction de jointures inutiles, des traductions JSON plus directes et une meilleure intégration des index au modèle.

Pour les évaluer, je commencerais par quelques requêtes importantes, avec une configuration SQL explicite et des données représentatives. Le SQL montre ce qu’EF demande au moteur. Le plan réel et les mesures montrent le travail effectivement réalisé.

Les types complexes et les améliorations de `dotnet ef` facilitent aussi la modélisation et le travail d’équipe. Leur valeur se confirme lorsque les changements restent reproductibles, les migrations examinables et les résultats métier inchangés.

Une mise à jour réussie ne se résume donc pas à constater que les requêtes fonctionnent encore. Elle permet d’identifier où les ressources sont économisées, où le comportement reste équivalent et où une nouvelle conception ou un index apporte un gain supplémentaire.
