---
title: "Applications C# basées sur des fichiers : créer des outils internes faciles à partager"
date: 2026-11-17 19:00:00 -0500
categories: [dotnet, outillage]
tags: [dotnet, dotnet11, csharp, file-based-apps, nuget, outils-internes, devops]
---

>  ⚠️ IMAGE

## Préambule

Nous avons souvent besoin d’un petit outil pour vérifier un export, diagnostiquer un fichier d’importation ou transformer des données avant leur traitement. Pour une équipe qui travaille déjà en C#, réutiliser le langage, ses bibliothèques et ses outils réduit le temps nécessaire pour produire quelque chose d’utile.

Les applications C# basées sur des fichiers permettent de commencer sans créer de fichier `.csproj`. Avec l’évolution de l’outillage, elles peuvent aussi inclure plusieurs fichiers, déclarer leurs dépendances NuGet et suivre les conventions de formatage du dépôt.

Dans la session [C# File-based apps turned up to 11! de .NET Conf 2026](https://www.youtube.com/watch?v=Yvn7liNioto), Damian Edwards montre comment ce format dépasse le programme de quelques lignes. L’intérêt pour les outils internes est de conserver un démarrage simple tout en structurant progressivement le code.

Après avoir présenté plusieurs nouveautés dans [mon article sur .NET 11](https://codewithfrenchy.com/posts/dotnet11/), je propose ici un exemple concret : un validateur de catalogue CSV qui produit un rapport JSON. Nous examinerons sa structure, ses dépendances, son utilisation dans un pipeline et les signes qui justifient une conversion vers un projet classique.

## Du C# compilé avec un projet implicite

Une application basée sur des fichiers reste une application C# compilée. Le SDK construit la configuration nécessaire à partir du fichier d’entrée et de ses directives. Le code utilise les API habituelles de .NET et les bibliothèques de l’écosystème.

Le format est apparu avec .NET 10. La directive `#:include` est également disponible à partir du SDK 10.0.300. Cet article utilise le SDK .NET 11 pour présenter l’ensemble du parcours, y compris le formatage et les améliorations de la ligne de commande. La [documentation du format](https://learn.microsoft.com/en-us/dotnet/core/sdk/file-based-apps) précise les disponibilités.

Le gain immédiat vient du nombre réduit de fichiers de configuration à créer. Pour un utilitaire interne, cela facilite la lecture du point d’entrée, la préparation d’un exemple et le partage d’une première version.

La compilation, la restauration des packages et la sélection du SDK existent toujours. Il faut en tenir compte lorsque l’outil quitte le poste de son auteur.

## Construire un diagnostic de catalogue CSV

Notre outil reçoit un fichier avec trois colonnes : `Reference`, `Nom` et `Prix`. Il vérifie quelques règles explicites avant qu’un autre traitement importe ces données dans une application ou une base de données.

Pour cet exemple :

- La référence est obligatoire et unique après retrait des espaces aux extrémités, avec une comparaison ordinale sans distinction de casse
- Le nom est obligatoire
- Le prix doit être un nombre décimal supérieur ou égal à zéro, avec un point comme séparateur décimal
- L’en-tête et le nombre de colonnes doivent respecter le format attendu

Le programme produit un diagnostic sans modifier le fichier d’entrée. Les règles de normalisation servent uniquement à la comparaison des références. Une correction automatique du fichier constituerait une fonctionnalité distincte, avec son propre contrat.

Voici un fichier volontairement problématique :

```csv
Reference,Nom,Prix
ABC-001,"Café, format 250 g",12.50
" abc-001 ",Café de reprise,12.50
ABC-003,,9.90
ABC-004,Thé,-2.00
```

Nous attendons un doublon au deuxième enregistrement, un nom absent au troisième et un prix négatif au quatrième. La virgule dans le nom du premier produit fait partie de sa valeur, grâce aux guillemets.

Le [code complet de l’exemple est disponible dans cette archive]({{ '/assets/downloads/diagnostic-catalogue-filebased-dotnet11.zip' | relative_url }}). Elle contient les sources C#, les fichiers d’entrée, un rapport attendu et un programme de vérification. Les extraits du billet se concentrent sur la configuration et les commandes qui rendent ce format intéressant.

## Répartir les responsabilités entre plusieurs fichiers

Je conserverais un seul point d’entrée, puis trois fichiers pour les responsabilités qui évoluent indépendamment.

| Fichier | Responsabilité |
| --- | --- |
| `diagnostiquer-import.cs` | Valider les arguments, orchestrer le diagnostic et retourner le code de sortie |
| `LectureCsv.cs` | Lire les enregistrements CSV et vérifier leur structure |
| `AnalyseCatalogue.cs` | Appliquer les règles du catalogue et construire la liste d’anomalies |
| `SortieJson.cs` | Écrire le rapport dans un format stable |

Le début du fichier d’entrée déclare la cible et les inclusions :

```text
#:property TargetFramework=net11.0
#:property PublishAot=false
#:property AssemblyName=DiagnosticCatalogue
#:include LectureCsv.cs
#:include AnalyseCatalogue.cs
#:include SortieJson.cs
```

Les fichiers inclus contribuent à la même compilation. Ils contiennent des types et des méthodes, tandis que les instructions de premier niveau restent dans le fichier d’entrée. Déposer un fichier C# supplémentaire dans le dossier ne suffit pas à l’intégrer automatiquement.

Le fichier `AnalyseCatalogue.cs` travaille sur des enregistrements représentant les valeurs lues. Il utilise un `HashSet<string>` pour détecter les références déjà rencontrées. L’analyse du prix utilise `decimal.TryParse` avec une culture invariante et des styles numériques explicites. Le comportement reste ainsi indépendant de la configuration régionale du poste.

Cette séparation permet de modifier la lecture ou la présentation du rapport sans réécrire les règles. Elle facilite aussi l’extraction ultérieure de ces règles vers une bibliothèque testée avec xUnit.

### Inclure du code ou référencer une application

Les possibilités de réutilisation ne se limitent pas à `#:include`. La session montre aussi `#:ref`, qui référence une autre application basée sur des fichiers, et `#:project`, qui référence un projet classique.

La distinction est utile : inclure un fichier ajoute son code à la compilation courante. Référencer une application ou une bibliothèque établit une dépendance vers ses API accessibles. Référencer une autre application ne lance pas son point d’entrée.

Pour notre outil, les inclusions suffisent. Si les règles du catalogue sont déjà maintenues dans une bibliothèque partagée, `#:project` peut éviter de les recopier.

## Déclarer une dépendance là où elle est utilisée

La lecture d’un CSV doit prendre en compte les guillemets, les séparateurs contenus dans une valeur et les champs multilignes. Je confie cette responsabilité à [CsvHelper](https://www.nuget.org/packages/CsvHelper/34.0.0).

Le fichier `LectureCsv.cs` commence par cette directive :

```text
#:package CsvHelper@34.0.0
```

Le traitement des directives dans les fichiers inclus permet au SDK de prendre en compte cette dépendance. Le fichier qui utilise la bibliothèque déclare donc aussi le package nécessaire.

L’exemple utilise l’API de CsvHelper 34. Les [options du lecteur](https://joshclose.github.io/CsvHelper/api/csvreaderoptions.html) sont configurées pour imposer la virgule comme séparateur et interrompre la lecture si les guillemets ou le nombre de champs sont incorrects. Le format attendu est explicite, ce qui convient à un diagnostic préalable à une importation.

Je distingue une erreur de structure d’une anomalie métier. Un prix négatif peut être analysé et signalé dans le rapport. Un enregistrement dont les colonnes sont mal délimitées empêche de savoir quelles valeurs sont réellement évaluées. Dans ce cas, le programme interrompt le diagnostic sans produire de rapport métier partiel.

La version déclarée dans la directive facilite la revue des mises à jour. Utiliser une version flottante comme `@*` pourrait faire évoluer les dépendances résolues lors d’une restauration ultérieure. Il faut aussi considérer les dépendances transitives et les sources NuGet utilisées.

### Retrouver les commandes de gestion des packages

Dans .NET 11, les commandes présentées pendant la conférence acceptent le fichier d’entrée :

```shell
dotnet package list --file diagnostiquer-import.cs --include-transitive
dotnet package list --file diagnostiquer-import.cs --outdated
dotnet package list --file diagnostiquer-import.cs --vulnerable --include-transitive
```

La [documentation de `dotnet package list`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-package-list) détaille notamment les options pour les versions disponibles et les vulnérabilités connues. Une mise à jour de package doit ensuite passer par les mêmes vérifications de comportement que toute autre modification de l’outil.

Les [améliorations du SDK .NET 11](https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-11/sdk) permettent aussi d’ajouter des métadonnées aux directives concernées, par exemple pour contrôler les références qui transitent vers un consommateur. Ces possibilités deviennent utiles lorsque plusieurs outils commencent à partager des composants.

## Exécuter l’outil et définir son contrat de sortie

Depuis le dossier de l’exemple, on lance le diagnostic ainsi :

```shell
dotnet run --file diagnostiquer-import.cs -- exemples/catalogue-anomalies.csv
```

L’option `--file` indique explicitement le point d’entrée. Le séparateur `--` marque les arguments destinés à notre programme. Cette forme évite notamment une ambiguïté lorsqu’un fichier projet se trouve dans le dossier courant.

Le programme utilise trois codes de sortie :

| Code | Signification |
| --- | --- |
| `0` | Diagnostic terminé sans anomalie |
| `1` | Diagnostic terminé avec des anomalies métier |
| `2` | Arguments invalides, erreur de lecture ou structure non conforme |

Pour le fichier présenté plus haut, voici le rapport attendu, et non une sortie mesurée dans l’environnement de rédaction :

```json
{
  "schemaVersion": 1,
  "enregistrements": 4,
  "anomalies": [
    {
      "enregistrement": 2,
      "code": "REFERENCE_DUPLIQUEE"
    },
    {
      "enregistrement": 3,
      "code": "NOM_ABSENT"
    },
    {
      "enregistrement": 4,
      "code": "PRIX_NEGATIF"
    }
  ]
}
```

Les numéros désignent les enregistrements après l’en-tête. Ils ne correspondent pas nécessairement aux lignes physiques, puisqu’une valeur CSV peut occuper plusieurs lignes.

Le champ `schemaVersion` permet de faire évoluer le rapport en tenant compte de ses consommateurs. Les codes d’anomalie offrent aussi une interface plus stable qu’un message destiné à être lu par une personne.

### Séparer la construction du diagnostic dans un pipeline

L’application réserve stdout au JSON et stderr aux erreurs bloquantes. L’exécution par `dotnet run` ajoute toutefois une étape de construction dont les messages doivent être considérés séparément.

Pour un pipeline, je construirais d’abord l’outil, puis j’exécuterais son assembly :

```shell
dotnet build diagnostiquer-import.cs --configuration Release --output ./artifacts/bin
dotnet ./artifacts/bin/DiagnosticCatalogue.dll ./exemples/catalogue-anomalies.csv > rapport.json
```

La deuxième commande suppose que la première a réussi. Le pipeline doit arrêter le traitement si la construction échoue et conserver le code de sortie du diagnostic.

Cette séparation évite de confondre une erreur de compilation avec une anomalie dans le catalogue. Elle permet également de réutiliser le même artefact pour plusieurs fichiers à vérifier.

## Automatiser le formatage et les vérifications

Le support de `dotnet format` pour les applications basées sur des fichiers est documenté dans les [notes du SDK .NET 11 RC 1](https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/rc1/sdk.md). On peut appliquer les conventions au point d’entrée et aux sources de son application :

```shell
dotnet format diagnostiquer-import.cs
```

Un fichier `.editorconfig` partagé rend les attentes visibles :

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true

[*.cs]
indent_style = space
indent_size = 4
csharp_new_line_before_open_brace = all
dotnet_sort_system_directives_first = true
```

Dans un pipeline, l’option [`--verify-no-changes` de `dotnet format`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-format) vérifie le formatage sans réécrire les fichiers :

```shell
dotnet format diagnostiquer-import.cs --verify-no-changes
```

L’archive contient également `verifier.cs`, un second point d’entrée qui construit l’outil dans un dossier temporaire et contrôle son comportement comme le ferait un consommateur externe :

```shell
dotnet run --file verifier.cs
```

Ses onze scénarios couvrent notamment les doublons après normalisation, le prix nul accepté, le prix négatif refusé, une virgule dans un nom, un champ multiligne et les fichiers mal structurés. Les vérifications portent sur les codes de sortie et les anomalies exactes du JSON. Elles contrôlent aussi que le fichier d’entrée demeure inchangé.

Ce vérificateur possède son propre point d’entrée et se formate séparément avec `dotnet format verifier.cs`. Il ne fait pas partie des inclusions de l’outil de diagnostic.

Le SDK .NET 11 n’était pas disponible dans l’environnement de rédaction. La compilation, le formatage et ces scénarios restent à exécuter sur une machine équipée. Les fichiers fournis ne constituent donc pas un résultat de validation C# déjà obtenu.

## Partager un outil avec son contexte d’exécution

Une fois plusieurs fichiers et une dépendance ajoutés, partager uniquement le point d’entrée ne suffit plus. Le dossier de l’outil devient l’unité à versionner et à distribuer aux développeurs.

Je conserverais les sources incluses, la configuration, les exemples et une courte procédure d’utilisation dans le même dépôt. Le fichier [`global.json`](https://learn.microsoft.com/en-us/dotnet/core/tools/global-json) fixe la politique de sélection du SDK :

```json
{
  "sdk": {
    "version": "11.0.100",
    "rollForward": "latestPatch",
    "allowPrerelease": false
  }
}
```

Ici, les correctifs installés de la même bande de SDK sont acceptés. Ce fichier n’installe pas le SDK. Il faut également lancer les commandes depuis le bon dossier, puisque la recherche de `global.json` dépend du contexte d’exécution.

Les configurations héritées comptent aussi. Un `Directory.Build.props`, un `Directory.Build.targets`, un `Directory.Packages.props` ou un `nuget.config` dans un dossier parent peut modifier la construction. Déplacer l’outil dans un autre dépôt peut donc changer son comportement de compilation ou de restauration.

Le `root = true` de `.editorconfig` concerne les conventions de l’éditeur. Il n’isole pas l’application des configurations MSBuild et NuGet.

### Distribuer une version compilée

Pour une personne qui utilise seulement le diagnostic, une distribution compilée évite d’exiger un SDK et une restauration NuGet à chaque nouvelle installation.

L’exemple désactive explicitement Native AOT avec `PublishAot=false`. Les applications basées sur des fichiers l’activent normalement pour la publication, mais ce choix demande de vérifier la compatibilité des dépendances et la chaîne de compilation native. Nous retenons ici une publication managée pour garder le parcours de l’exemple simple.

Voici une publication autonome pour Windows x64 :

```shell
dotnet publish diagnostiquer-import.cs --configuration Release --runtime win-x64 --self-contained true --output ./artifacts/win-x64
```

Il faut distribuer le contenu complet du dossier produit. Une publication autonome embarque le runtime nécessaire, mais ne signifie pas automatiquement qu’un seul fichier sera généré. Une autre plateforme exige un artefact adapté.

Le gain se situe alors dans l’expérience de l’utilisateur : une version identifiée de l’outil, un contrat d’entrée documenté et un résultat exploitable, sans préparation d’un environnement de développement.

## Comprendre les limites avant d’agrandir l’outil

### Le cache réduit certains coûts de lancement

La démonstration de .NET Conf montre que le SDK peut réutiliser des sorties de compilation lorsque les entrées pertinentes n’ont pas changé. Il faut donc distinguer le premier lancement, une reconstruction et l’exécution d’un artefact déjà disponible lorsqu’on mesure la durée du diagnostic.

Les inclusions explicites de notre exemple facilitent aussi la lecture du périmètre. Les motifs comme `**/*.cs` sont pris en charge, mais désactivent actuellement le cache de construction des applications basées sur des fichiers. C’est un compromis à considérer pour des outils lancés fréquemment.

### Plusieurs fichiers ne constituent pas à eux seuls un problème

Notre lecteur parcourt les enregistrements progressivement. Le `HashSet` des références et la liste des anomalies restent toutefois en mémoire. Sur des volumes importants, cette consommation augmente avec le nombre de références distinctes et d’anomalies.

Il faudra alors choisir une stratégie adaptée, par exemple plafonner le rapport, traiter les données par lots ou externaliser certaines vérifications. La présence ou l’absence d’un fichier projet ne résout pas cette question d’architecture.

Pour décider du format, je regarderais plutôt les responsabilités et le mode de maintenance :

| Situation | Choix généralement adapté |
| --- | --- |
| Diagnostic ciblé, quelques fichiers et dépendances limitées | Application basée sur des fichiers |
| Outil distribué sous forme compilée, avec un périmètre stable | Les deux formats peuvent convenir |
| Plusieurs équipes consomment les mêmes règles métier | Bibliothèque partagée avec API et tests identifiés |
| Nombreuses configurations, cibles ou personnalisations MSBuild | Projet classique plus lisible à maintenir |
| Utilitaire devenu un produit interne avec versions, tests et plusieurs exécutables | Solution structurée autour de projets classiques |

Les tests et la distribution sont possibles avec ce format. Le passage à un projet devient intéressant lorsque l’organisation explicite des composants réduit davantage la complexité que l’absence de `.csproj` ne simplifie le démarrage.

### Convertir sans réécrire la logique

La commande [`dotnet project convert`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-project-convert) permet d’amorcer cette transition :

```shell
dotnet project convert diagnostiquer-import.cs --dry-run
dotnet project convert diagnostiquer-import.cs --output ./projet-diagnostic
```

La première commande permet d’examiner les changements prévus. Après conversion, il faut relire le projet généré, ses références et le traitement des fichiers inclus, puis reprendre les vérifications de comportement.

Je profiterais de cette étape pour placer les règles réutilisables dans une bibliothèque, ajouter des tests xUnit ciblés et intégrer la construction à la solution de l’équipe. Le point d’entrée console peut conserver le même contrat pour ses utilisateurs.

## Conclusion

Les applications C# basées sur des fichiers offrent un bon point de départ pour les outils internes dont le besoin est précis. Les inclusions, les dépendances NuGet et le formatage automatisé permettent de structurer le code tout en conservant un parcours de lancement court.

Pour qu’un outil soit facile à partager, il faut aussi rendre explicites ses entrées, ses codes de sortie, ses dépendances et ses conditions d’exécution. Dans notre exemple, le rapport JSON et les vérifications de comportement comptent autant que la simplicité du lancement.

Je commencerais avec ce format pour un diagnostic ciblé, puis je le ferais évoluer selon ses usages réels. Lorsque les règles, les consommateurs et la maintenance prennent de l’ampleur, la conversion vers un projet classique permet de conserver le travail utile tout en lui donnant une structure adaptée.
