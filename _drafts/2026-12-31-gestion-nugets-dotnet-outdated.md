---
title: Gestion des dépendances NuGet avec dotnet-outdated
date: 2027-12-31 19:00:00 -0400
categories: [outil-developpement]
tags: [dotnet]
---

## Préambule

Ajouter un paquet NuGet à une application prend généralement quelques secondes. Assurer son suivi pendant plusieurs années demande un peu plus d'organisation. Entre les nouvelles fonctionnalités, les correctifs, les changements de comportement et les vulnérabilités découvertes après sa publication, une dépendance continue d'évoluer bien après son installation.

Lors d'un précédent mandat, j'avais préparé une page pour accompagner l'équipe dans cette gestion. Elle présentait notamment `dotnet-outdated`, les vérifications de vulnérabilités et leur intégration dans Azure DevOps. J'en reprends ici les principes avec des exemples génériques et une approche applicable à d'autres projets .NET.

L'objectif est de rendre l'entretien des dépendances plus régulier et plus prévisible. Pour y arriver, il faut savoir quelles versions nous utilisons, comprendre les mises à jour disponibles et disposer de validations suffisantes pour les adopter avec confiance.

`dotnet-outdated` constitue un bon point de départ pour ce travail. Son intérêt devient encore plus concret lorsqu'on l'intègre à une pratique d'équipe.

## Pourquoi entretenir ses dépendances régulièrement?

Une application peut fonctionner correctement tout en accumulant du retard sur ses dépendances. Ce retard devient visible lorsqu'une vulnérabilité impose une mise à jour rapide ou qu'une migration du framework exige de revoir plusieurs bibliothèques en même temps.

À ce moment, les changements se cumulent. Il faut parfois adapter des API, modifier de la configuration et valider plusieurs évolutions de comportement dans une seule intervention. La mise à jour devient alors plus difficile à estimer, à tester et à livrer.

Je préfère réserver une petite capacité récurrente à cet entretien. Des changements ciblés donnent généralement des demandes de fusion plus faciles à réviser et facilitent le diagnostic lorsqu'une régression apparaît.

La priorité dépend toutefois du contexte. Un correctif de sécurité, une fin de support prochaine et une nouvelle fonctionnalité facultative ne justifient pas nécessairement la même urgence. Cette réflexion rejoint celle que je présente dans mon article sur la [priorisation de la dette technique](https://codewithfrenchy.com/posts/priorisation-dette-technique/).

## Ce que dotnet-outdated apporte

Le projet communautaire [dotnet-outdated](https://github.com/dotnet-outdated/dotnet-outdated) permet de repérer les paquets NuGet pour lesquels des versions plus récentes sont disponibles. Il propose aussi des filtres pour cibler l'analyse et peut appliquer des mises à jour.

La CLI .NET possède déjà des commandes pour dresser un inventaire et rechercher des versions plus récentes. L'intérêt de `dotnet-outdated` réside surtout dans le travail interactif autour des mises à jour, notamment lorsqu'on veut limiter les changements à une plage de versions ou mettre à jour une famille de paquets.

Il faut néanmoins garder trois questions distinctes.

| Question | Ce qu'il faut vérifier |
| --- | --- |
| Une version plus récente existe-t-elle? | Les versions publiées dans les sources NuGet accessibles |
| La version utilisée présente-t-elle une vulnérabilité connue? | Les avis de sécurité disponibles pour cette version |
| La mise à jour convient-elle à notre application? | La compatibilité, les changements annoncés et les résultats des essais |

Un outil d'inventaire aide à démarrer l'analyse. La décision de modifier une dépendance reste liée à son utilisation dans l'application.

## Installer l'outil

### Une installation globale pour commencer

Le paquet à installer porte le nom `dotnet-outdated-tool`. La commande utilisée ensuite est `dotnet outdated`.

```powershell
dotnet tool install --global dotnet-outdated-tool
```

Pour mettre l'outil à jour et consulter son aide :

```powershell
dotnet tool update --global dotnet-outdated-tool
dotnet outdated --help
```

Prévoyez un SDK .NET capable d'analyser votre solution ainsi que le runtime requis par la version de l'outil. Les prérequis de celle-ci sont indiqués sur sa [page NuGet](https://www.nuget.org/packages/dotnet-outdated-tool). La version du framework ciblé par votre application et celle nécessaire pour exécuter un outil sont deux éléments à vérifier séparément.

### Une installation locale pour l'équipe

Pour un usage partagé, je privilégie un [outil local .NET](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-tool-install). Son manifeste permet de conserver la version de l'outil dans le dépôt Git.

À la racine du dépôt, créez le manifeste s'il n'existe pas déjà, puis installez l'outil :

```powershell
dotnet new tool-manifest
dotnet tool install --local dotnet-outdated-tool
```

Ajoutez le fichier `.config/dotnet-tools.json` au contrôle de source. Les autres membres de l'équipe pourront ensuite restaurer la version déclarée :

```powershell
dotnet tool restore
```

Cette approche rend l'outillage plus prévisible entre les postes et le pipeline. La mise à jour de l'outil devient elle-même un changement visible dans le dépôt.

Les installations globale et locale sont deux options. Il suffit de choisir celle qui convient à votre usage.

## Analyser une solution

Depuis le répertoire contenant votre solution, lancez simplement :

```powershell
dotnet outdated
```

Pour éviter les ambiguïtés, notamment lorsqu'un répertoire contient plusieurs solutions, je préfère préciser la cible :

```powershell
dotnet outdated ./GestionDocuments.sln
```

On peut également cibler un projet :

```powershell
dotnet outdated ./src/GestionDocuments.Api/GestionDocuments.Api.csproj
```

Cette analyse seule ne met pas à jour les versions déclarées. Elle peut toutefois effectuer les opérations de restauration nécessaires à l'évaluation des projets.

### Interpréter les versions proposées

L'outil utilise des couleurs pour distinguer les changements de versions. Le tableau suivant illustre leur lecture avec des paquets et des numéros fictifs.

| Paquet fictif | Version utilisée | Version proposée | Type de changement | Couleur habituelle |
| --- | --- | --- | --- | --- |
| `Exemple.Journalisation` | `2.4.1` | `2.4.3` | Correctif | Vert |
| `Exemple.Validation` | `3.2.0` | `3.5.0` | Mineur | Jaune |
| `Exemple.ClientApi` | `1.8.2` | `2.0.0` | Majeur | Rouge |

Selon le [versionnage sémantique](https://semver.org/lang/fr/), les correctifs et les versions mineures préservent la compatibilité de l'API publique, alors qu'une version majeure peut introduire des changements incompatibles. Ces indications supposent que l'éditeur respecte cette convention. Les versions `0.x` demandent aussi davantage de prudence, puisque leur API peut encore évoluer librement.

Je considère donc la couleur comme un indicateur de l'effort d'analyse. Même un correctif peut changer un comportement sur lequel notre application s'appuyait, par exemple une validation plus stricte ou le traitement d'une valeur particulière.

La compilation et les essais restent nécessaires, quelle que soit la couleur affichée.

## Cibler les mises à jour

Une première analyse peut produire une longue liste de résultats. Les options de filtrage permettent de réduire le périmètre pour travailler par étapes.

### Rester dans une même version majeure ou mineure

```powershell
dotnet outdated ./GestionDocuments.sln --version-lock Major
dotnet outdated ./GestionDocuments.sln --version-lock Minor
```

Les noms de ces options désignent la partie de la version qui reste verrouillée.

| Option | Avec une version actuelle `4.2.1` |
| --- | --- |
| `--version-lock Major` | Recherche dans la branche `4.x` |
| `--version-lock Minor` | Recherche dans la branche `4.2.x` |

Ces filtres sont utiles pour une intervention de maintenance limitée. Je conserverais aussi une analyse périodique sans verrouillage pour repérer les migrations majeures à préparer.

### Choisir les paquets et les préversions

Pour examiner une famille de paquets internes :

```powershell
dotnet outdated ./GestionDocuments.sln --include Entreprise.
```

Pour exclure temporairement cette famille d'une analyse :

```powershell
dotnet outdated ./GestionDocuments.sln --exclude Entreprise.
```

Ces filtres recherchent une sous-chaîne dans le nom des paquets. Une exclusion devrait correspondre à une décision suivie, par exemple une migration déjà planifiée, afin d'éviter qu'une dépendance disparaisse durablement de la maintenance.

Pour limiter les propositions aux versions stables :

```powershell
dotnet outdated ./GestionDocuments.sln --pre-release Never
```

Ce choix est pratique pour un entretien courant. Les essais de préversions peuvent faire l'objet d'une intervention distincte.

### Examiner les dépendances transitives

Les bibliothèques que nous ajoutons utilisent elles-mêmes d'autres paquets. Ces dépendances transitives font aussi partie de ce que l'application consomme.

```powershell
dotnet outdated ./GestionDocuments.sln --transitive --transitive-depth 2
```

La profondeur limite l'exploration demandée à l'outil. Cet exemple ne constitue donc pas un inventaire exhaustif de toutes les dépendances transitives. Sur une grande solution, élargir l'analyse augmente aussi son coût d'exécution.

## Appliquer une mise à jour avec un périmètre maîtrisé

L'option `--upgrade` applique les mises à jour proposées. Le mode `--upgrade:Prompt` demande une confirmation pour les paquets concernés.

Voici un exemple pour réviser les correctifs disponibles dans les branches mineures actuelles :

```powershell
dotnet outdated ./GestionDocuments.sln --version-lock Minor --pre-release Never --upgrade:Prompt
```

J'exécuterais cette commande dans une branche dédiée, avec un état de travail propre. Il devient alors facile de voir les fichiers modifiés et de revenir sur une proposition.

Après la mise à jour, une première validation peut comprendre :

```powershell
dotnet restore ./GestionDocuments.sln
dotnet build ./GestionDocuments.sln --configuration Release --no-restore
dotnet test ./GestionDocuments.sln --configuration Release --no-build
git diff
```

Ces commandes supposent une solution dont les essais s'exécutent avec `dotnet test`. Elles doivent être complétées par les validations propres au projet, notamment les essais d'intégration et les scénarios fonctionnels critiques.

Avant la fusion, je vérifierais les notes de version, les adaptations requises et le comportement réellement utilisé par l'application. Pour une bibliothèque de sérialisation, les contrats JSON peuvent être plus révélateurs qu'un grand nombre de tests unitaires sans lien avec les échanges externes. Pour un accès à une base de données, les essais d'intégration prennent davantage d'importance.

Si vous utilisez la [gestion centralisée des versions NuGet](https://learn.microsoft.com/en-us/nuget/consume-packages/central-package-management), examinez aussi les modifications dans `Directory.Packages.props`. Une seule version partagée peut concerner plusieurs projets et élargir le périmètre des validations.

L'automatisation peut très bien préparer ces changements. La fusion automatique demande toutefois une politique plus précise, fondée sur la criticité des dépendances et la qualité des validations disponibles.

## Utiliser un dépôt NuGet privé

Dans un environnement organisationnel, une partie des paquets provient souvent d'un dépôt privé, par exemple Azure Artifacts. L'analyse dépend alors de deux éléments distincts : la configuration des sources et l'authentification.

### Déclarer les sources dans le dépôt Git

NuGet combine des [fichiers de configuration à plusieurs niveaux](https://learn.microsoft.com/en-us/nuget/consume-packages/configuring-nuget-behavior), dont ceux de la machine, de l'utilisateur et du dépôt. Il ne faut donc pas supposer qu'une seule configuration personnelle décrit l'environnement de toute l'équipe.

Voici un exemple de fichier `nuget.config` placé à la racine, à côté de la solution :

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org"
         value="https://api.nuget.org/v3/index.json" />
    <add key="nugetPrive"
         value="https://pkgs.dev.azure.com/mon-organisation/_packaging/nugetPrive/nuget/v3/index.json" />
  </packageSources>
</configuration>
```

Le dépôt `nugetPrive` et l'organisation `mon-organisation` sont fictifs. Remplacez l'adresse par celle fournie par votre dépôt. Un flux Azure Artifacts associé à un projet comporte aussi ce projet dans son URL.

L'élément `<clear />` évite d'hériter d'autres sources de paquets. Il faut donc déclarer ici toutes celles requises par la solution. Le fichier décrit les emplacements et ne doit pas contenir de secret.

Si votre organisation impose un dépôt interne comme point d'entrée unique, adaptez cet exemple à cette règle. Lorsque plusieurs sources sont permises, le [Package Source Mapping](https://learn.microsoft.com/en-us/nuget/consume-packages/package-source-mapping) permet de préciser lesquelles peuvent fournir certains identifiants de paquets. L'ordre des sources ne constitue pas une priorité fiable pour protéger les paquets internes.

### Authentifier le poste de développement

Pour Azure Artifacts, le [Azure Artifacts Credential Provider](https://github.com/microsoft/artifacts-credprovider) prend en charge l'obtention des informations d'authentification. Après son installation selon les instructions adaptées à votre plateforme, amorcez l'authentification depuis la solution :

```powershell
dotnet restore ./GestionDocuments.sln --interactive
dotnet outdated ./GestionDocuments.sln
```

L'authentification pourra être réutilisée tant qu'elle demeure valide. Une analyse qui échoue à joindre une source doit être corrigée avant d'interpréter ses résultats.

Si `dotnet-outdated` signale explicitement l'absence de `DOTNET_HOST_PATH`, indiquez le chemin de l'exécutable `dotnet` pour la session. En PowerShell :

```powershell
$env:DOTNET_HOST_PATH = (Get-Command dotnet).Source
```

En Bash :

```bash
export DOTNET_HOST_PATH="$(command -v dotnet)"
```

Ce réglage aide le fournisseur d'authentification à trouver l'hôte .NET. Il ne remplace ni une connexion valide ni les droits d'accès au dépôt.

## Rechercher les vulnérabilités connues

Une version récente peut contenir une vulnérabilité connue. À l'inverse, une version plus ancienne peut encore convenir à l'application. Il faut donc compléter la recherche de mises à jour par un audit.

Avec le SDK .NET 10, la [commande d'inventaire des paquets](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-package-list) s'utilise ainsi :

```powershell
dotnet restore ./GestionDocuments.sln
dotnet package list --project ./GestionDocuments.sln --vulnerable --include-transitive --no-restore
```

Avec les SDK .NET 8 et .NET 9, utilisez la syntaxe suivante après la restauration :

```powershell
dotnet list ./GestionDocuments.sln package --vulnerable --include-transitive
```

La syntaxe dépend du SDK exécuté. L'option `--include-transitive` inclut les dépendances indirectes dans ce rapport, sans la limite de profondeur de l'exemple précédent avec `dotnet-outdated`.

Un résultat sans vulnérabilité signifie qu'aucune correspondance n'a été trouvée dans les données consultées pour les paquets analysés. Il ne prouve pas l'absence de faille. Vérifiez aussi que l'analyse s'est terminée correctement et que les sources d'avis de sécurité étaient accessibles.

Lorsqu'un paquet transitif est concerné, commencez par examiner une mise à jour de la dépendance directe qui l'introduit. Ajouter une référence directe pour imposer une version différente peut être nécessaire dans certains cas, mais demande de vérifier les contraintes de compatibilité et d'en suivre la raison.

### Distinguer les sources de paquets des sources d'audit

Le serveur qui distribue un paquet ne fournit pas nécessairement les avis de sécurité associés. Avec un SDK récent, la section [`auditSources` de `nuget.config`](https://learn.microsoft.com/en-us/nuget/reference/nuget-config-file#auditsources) permet de déclarer ces sources séparément.

Le bloc suivant peut être ajouté sous `<configuration>`, à côté de `<packageSources>`, dans l'exemple précédent :

```xml
<auditSources>
  <clear />
  <add key="nuget.org"
       value="https://api.nuget.org/v3/index.json" />
</auditSources>
```

Les [versions prenant en charge les sources d'audit](https://learn.microsoft.com/en-us/nuget/concepts/auditing-packages#audit-sources) diffèrent selon l'opération. La restauration les utilise à partir du SDK .NET 9.0.100, et l'inventaire des vulnérabilités à partir du SDK .NET 9.0.300. L'exemple de pipeline ci-dessous utilise le SDK .NET 10.

Cette séparation permet de consulter les avis de nuget.org même si les téléchargements passent par un dépôt interne. Elle suppose que l'accès réseau à la source d'audit est autorisé. Les paquets développés exclusivement à l'interne demandent aussi un suivi de sécurité propre à l'organisation.

## Intégrer les vérifications dans Azure DevOps

L'analyse locale est utile pendant une intervention. Le pipeline rend les vérifications visibles pour l'équipe et conserve leurs résultats avec l'exécution.

Pour un simple rapport, les commandes natives du SDK évitent d'installer un outil supplémentaire sur l'agent. On peut aussi utiliser `dotnet-outdated` dans le pipeline en restaurant son manifeste local si ses filtres ou son rapport répondent mieux au besoin.

L'exemple suivant utilise les commandes natives, une solution `GestionDocuments.sln` et le fichier `nuget.config` présenté plus haut, complété par `auditSources`. Il suppose que le dépôt Azure Artifacts appartient à la même organisation que le pipeline.

```yaml
pool:
  vmImage: ubuntu-latest

steps:
  - checkout: self

  - task: UseDotNet@2
    displayName: 'Installer le SDK .NET'
    inputs:
      packageType: sdk
      version: '10.0.x'

  - task: NuGetAuthenticate@1
    displayName: 'Authentifier les sources NuGet'

  - script: >-
      dotnet restore ./GestionDocuments.sln
      --configfile ./nuget.config
      -p:NuGetAudit=true
      -p:NuGetAuditMode=all
      -p:NuGetAuditLevel=low
    displayName: 'Restaurer et auditer les dépendances'

  - script: >-
      dotnet package list --project ./GestionDocuments.sln
      --outdated --no-restore
    displayName: 'Afficher les mises à jour disponibles'

  - script: >-
      dotnet package list --project ./GestionDocuments.sln
      --vulnerable --include-transitive --no-restore
    displayName: 'Afficher les vulnérabilités connues'
```

La tâche [`NuGetAuthenticate@1`](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/nuget-authenticate-v1?view=azure-pipelines) configure l'authentification utilisée par les commandes suivantes. L'identité du pipeline doit disposer des permissions requises sur le dépôt. Pour une source dans une autre organisation, configurez une connexion de service NuGet et référencez-la avec `nuGetServiceConnections`.

Adaptez le SDK aux contraintes de votre solution. Les projets utilisant des charges de travail particulières, par exemple certains projets mobiles, nécessitent également les composants correspondants sur l'agent.

### Définir ce qui doit réellement bloquer le pipeline

Les rapports précédents donnent de la visibilité. **La présence d'un résultat dans les journaux ne constitue pas, à elle seule, une règle de blocage.** Le [suivi de cette demande dans NuGet](https://github.com/NuGet/Home/issues/11315) illustre notamment cette distinction pour la commande d'inventaire des vulnérabilités.

Pour appliquer une règle de sécurité, on peut s'appuyer sur [NuGet Audit pendant la restauration](https://learn.microsoft.com/en-us/nuget/concepts/auditing-packages). Une politique possible consiste à conserver tous les avis et à bloquer les vulnérabilités élevées ou critiques, ainsi que les erreurs empêchant d'obtenir les données d'audit.

Dans l'étape de restauration précédente, ajoutez l'option [MSBuild `-warnaserror`](https://learn.microsoft.com/en-us/visualstudio/msbuild/msbuild-command-line-reference) suivante :

```yaml
  - script: >-
      dotnet restore ./GestionDocuments.sln
      --configfile ./nuget.config
      -p:NuGetAudit=true
      -p:NuGetAuditMode=all
      -p:NuGetAuditLevel=low
      "-warnaserror:NU1900,NU1903,NU1904,NU1905"
    displayName: 'Valider la politique de sécurité des dépendances'
```

Cette étape remplace la restauration précédente. Les codes `NU1903` et `NU1904` correspondent aux vulnérabilités élevées et critiques. `NU1900` concerne l'obtention des données d'audit et `NU1905` une source d'audit qui ne fournit pas les informations attendues.

Les avis faibles et modérés restent des avertissements, sauf si d'autres règles du dépôt les transforment déjà en erreurs. Les suppressions d'avis ou d'avertissements existantes doivent aussi être examinées, puisqu'elles peuvent retirer des résultats avant l'application de cette règle.

Le seuil et le traitement des exceptions doivent être convenus avec l'équipe. Une exception documentée devrait avoir un responsable, une justification et une date de révision.

Pour la fraîcheur des paquets, `dotnet-outdated` propose aussi `--fail-on-updates`. Je réserverais cette option à une politique assumée de maintenance. Faire échouer toutes les demandes de fusion dès qu'une nouvelle version paraît peut créer du bruit sans tenir compte de son importance pour le projet.

## Faire vivre la pratique dans l'équipe

Une vérification dans le pipeline de demandes de fusion réagit aux changements du dépôt. Une exécution planifiée couvre un autre besoin : détecter un nouvel avis de sécurité alors que personne n'a modifié le code.

Pour commencer, je proposerais une organisation simple :

- **Une revue régulière des mises à jour** - par exemple chaque semaine, avec une personne responsable de traiter les résultats.
- **Un audit de sécurité planifié** - dont la fréquence dépend de la criticité de l'application.
- **Des demandes de fusion ciblées** - regroupant les paquets qui doivent évoluer ensemble et isolant les migrations plus importantes.
- **Une capacité de maintenance réservée** - pour que les rapports débouchent sur des changements livrés.

Les notes de version, la compatibilité des frameworks et les conditions d'utilisation font également partie de la décision. `dotnet-outdated` ne vérifie pas toutes ces dimensions. J'aborde d'ailleurs les enjeux de gouvernance des dépendances dans mon article consacré à [Polly et à l'évolution de son modèle de maintenance](https://codewithfrenchy.com/posts/polly-devient-payant/).

Le suivi devrait surtout nous permettre de répondre à des questions concrètes : quelles dépendances demandent une intervention, qui s'en occupe et quand la correction pourra-t-elle être livrée?

## Conclusion

`dotnet-outdated` facilite une tâche que l'on reporte facilement : regarder l'état des dépendances et préparer leur mise à jour. Ses filtres et son mode interactif permettent d'avancer avec un périmètre adapté à la capacité de l'équipe.

Pour en tirer pleinement profit, je le combinerais avec un audit des vulnérabilités, une configuration NuGet partagée et des validations adaptées aux comportements de l'application. Les sources privées et les règles du pipeline doivent également être explicites pour que les résultats restent fiables entre les postes et l'intégration continue.

Une pratique régulière aide à garder les changements compréhensibles et à préparer les migrations avant qu'elles deviennent urgentes. C'est cette continuité, avec des résultats pris en charge par l'équipe, qui rend la gestion des dépendances durable.
