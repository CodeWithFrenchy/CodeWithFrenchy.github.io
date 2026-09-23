---
title: Que devrait contenir un dépôt .NET?
date: 2024-12-31 19:00:00 -0400
categories: [outil-developpement]
tags: [dotnet]
---

> 💡 à valider une dernière fois

## Préambule

Vous clonez un projet .NET, ouvrez la solution et lancez la compilation. Il manque une version du SDK, une source NuGet privée ou une configuration connue uniquement par un collègue. Une fois ces obstacles franchis, les tests passent localement, mais échouent dans le pipeline.

Ces difficultés finissent souvent par être considérées comme normales. Pourtant, elles révèlent surtout que les conditions nécessaires pour travailler sur le projet ne sont pas suffisamment explicites.

Un dépôt bien organisé doit permettre de comprendre rapidement ce que fait l’application, comment la démarrer et comment vérifier une modification. Dans cet article, le terme « dépôt » désigne le référentiel Git qui contient le code et les fichiers nécessaires à son développement et à sa livraison.

Cette organisation relève directement de l’expérience développeur, ou DevEx. Un projet bien outillé réduit le temps passé à chercher une commande, à reconstruire une configuration ou à comprendre une erreur. Il accélère l’intégration des nouveaux membres et facilite aussi le retour sur un produit après plusieurs mois d’absence.

L’objectif est de définir un socle utile à la plupart des projets .NET, puis de l’enrichir selon les besoins. Une petite API et une plateforme composée de plusieurs services n’ont pas besoin de la même structure. Elles ont toutefois besoin de conventions compréhensibles et de commandes fiables.

## Commencer par les questions du prochain développeur

Le premier fichier à soigner est le `README.md`. Une personne qui découvre le dépôt devrait pouvoir y trouver le rôle du produit, ses prérequis, les étapes de démarrage et les commandes de validation.

Je recommande d’y documenter un parcours que l’équipe a réellement vérifié depuis un clone neuf : installation du SDK attendu, accès aux dépendances, préparation de la configuration locale, démarrage et exécution des tests. Les prérequis particuliers, comme un moteur de conteneurs ou un accès à un registre privé, doivent apparaître avant les commandes qui en dépendent.

Le reste de la documentation peut être réparti selon sa fonction :

| Fichier ou dossier | Contenu attendu |
|---|---|
| `README.md` | Rôle du projet, démarrage rapide, commandes et liens utiles. |
| `docs/` | Architecture, configuration et procédures qui dépassent le démarrage rapide. |
| Registre de décisions lié depuis le `README` | Historique des choix, contexte, options étudiées et conséquences. |
| `.gitignore` | Exclusion des fichiers générés et locaux qui ne doivent pas être versionnés. |
| `.gitattributes` | Conventions Git, notamment la normalisation des fins de ligne. |

Je recommande de mettre en place un [registre de décisions](https://codewithfrenchy.com/posts/registre-decisions/) pour conserver les raisons des choix structurants. Il peut vivre dans le wiki utilisé par l’équipe, avec un lien direct depuis le `README`. Chaque entrée précise le contexte, les options étudiées, la décision, ses conséquences, sa date et les personnes impliquées. Cela aide les nouveaux membres à comprendre les contraintes du projet et évite de reprendre les mêmes débats sans connaître les arbitrages précédents.

Le `README` joue aussi un rôle de relais. Il peut orienter vers un Wiki Azure DevOps pour l’architecture détaillée, les procédures d’exploitation, les règles de contribution et le registre de décisions. Gardez le démarrage rapide au plus près du code, puis indiquez où se trouve la source de référence pour le reste. Copier la même procédure dans plusieurs outils multiplie les risques de divergence. Vérifiez également que les nouveaux membres disposent des droits nécessaires pour consulter les liens.

Pour évaluer ce parcours d’intégration, demandez à une personne qui découvre le projet de démarrer l’application et de lancer les tests à partir des seules instructions. Chaque étape qui exige une explication orale signale une amélioration possible du dépôt ou du wiki.

La documentation doit aussi rester proche de la réalité. Lorsqu’une PR change la procédure de démarrage, elle devrait modifier le `README` dans le même mouvement.

## Une structure qui correspond au produit

Je partirais d’une organisation courte, que l’on peut expliquer sans présenter toute l’architecture de l’entreprise :

| Emplacement | Responsabilité |
|---|---|
| `src/` | Projets applicatifs et bibliothèques. |
| `tests/` | Projets de tests et ressources nécessaires à leur exécution. |
| `docs/` | Documentation technique durable. |
| `scripts/` | Scripts de développement, de validation ou de livraison, si nécessaire. |
| `infra/` | Infrastructure déclarative, lorsque le dépôt en est responsable. |
| Racine | Solution, conventions communes et points d’entrée. |

Dans la suite de l’article, les exemples utilisent une solution `Product.slnx` à la racine et des projets sous `src/` et `tests/`. Adaptez ce nom à votre dépôt. Une solution `.sln` existante convient également.

Cette organisation n’impose pas un découpage en projets `Domain`, `Application` et `Infrastructure`. Le nombre de projets devrait refléter des responsabilités et des dépendances utiles. Ajouter des couches uniquement pour reproduire une arborescence de référence complique les petits produits.

Le choix entre monorepo et plusieurs dépôts vient ensuite. Je privilégierais un même dépôt lorsque les composants évoluent fréquemment ensemble. Des dépôts distincts peuvent mieux convenir à des responsabilités, des droits d’accès ou des cycles de maintenance séparés. Dans les deux cas, les conventions communes peuvent être distribuées et entretenues à l’échelle de l’organisation.

## Aspire : faire du démarrage local un parcours partagé

Pour une application .NET qui combine une API, une interface web et des dépendances comme une base de données ou un cache, je considère désormais [Aspire](https://aspire.dev/get-started/what-is-aspire/) comme un incontournable du développement local. Il apporte un point d’entrée commun pour décrire, démarrer et observer les composants.

L’AppHost décrit les ressources et leurs relations. Le tableau de bord rassemble leur état et la télémétrie disponible. Au lieu de transmettre une série de commandes et de ports à configurer, l’équipe entretient un modèle exécutable dans le dépôt.

Dans notre exemple, `src/Product.AppHost/` peut porter cette orchestration. Le `README` indique les prérequis et la commande de démarrage validée. Le dossier `scripts/` accueille les opérations complémentaires, comme l’initialisation des données de démonstration ou la remise à zéro de l’environnement.

### Un gain concret pour l’intégration des nouveaux membres

Une personne qui arrive doit pouvoir lancer un parcours fonctionnel, retrouver ses journaux et comprendre quelle dépendance est indisponible. Aspire réduit les manipulations de démarrage, mais l’équipe doit encore prévoir les accès, la configuration et les données nécessaires à ce parcours.

Ce socle demande de l’entretien. Les conteneurs exigent un moteur compatible et des ressources sur le poste. Les dépendances externes peuvent nécessiter une connectivité particulière. L’environnement local ne reproduit pas toutes les contraintes de production. Pour une bibliothèque isolée, un AppHost apporte moins de valeur et peut être superflu.

### Un environnement observable pour les agents IA

L’intérêt augmente avec les agents IA. Lire le code ne leur indique pas si un service démarre correctement ni pourquoi une requête échoue. Les [outils Aspire pour les agents de développement](https://aspire.dev/get-started/ai-coding-agents/) donnent accès à l’état des ressources, aux journaux et aux traces. La configuration peut s’appuyer sur les outils en ligne de commande et les skills d’Aspire, avec un serveur MCP optionnel selon l’intégration retenue.

Un agent peut ainsi rapprocher une erreur du composant concerné, proposer une correction et examiner une nouvelle exécution. Cela enrichit le diagnostic, sans remplacer les tests ni la revue. Un contrôle de santé réussi ne prouve pas que le parcours métier est corrigé.

Documentez les outils autorisés et les étapes de configuration pour éviter que chaque personne improvise sa propre intégration. Les journaux et les données locales accessibles à l’agent doivent respecter les règles de confidentialité du projet. Les conventions générales destinées aux assistants IA seront abordées plus loin, puis approfondies dans un article dédié.

## Rendre l’outillage prévisible

### Centraliser les propriétés communes de compilation

Les fichiers [`Directory.Build.props` et `Directory.Build.targets`](https://learn.microsoft.com/en-us/visualstudio/msbuild/customize-by-directory?view=vs-2022) permettent de partager la configuration MSBuild entre plusieurs projets. Le premier convient aux propriétés communes. Le second devient utile pour des ajustements ou des cibles évalués plus tard dans le processus de build.

Pour une solution moderne, ce `Directory.Build.props` constitue un point de départ :

```xml
<Project>
  <PropertyGroup>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <AnalysisLevel>10.0-recommended</AnalysisLevel>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>
  </PropertyGroup>
</Project>
```

Cet exemple suppose une équipe prête à corriger les avertissements. Dans un dépôt historique, je procéderais par étapes, avec des exceptions ciblées et suivies. Activer toutes les contraintes d’un coup, puis masquer globalement les diagnostics pour retrouver un build vert, apporte peu de valeur.

Évitez également de créer un `Directory.Build.targets` vide par convention. Ces fichiers doivent porter une responsabilité réelle. Attention enfin aux configurations imbriquées : MSBuild recherche le fichier applicable en remontant les dossiers. Il ne fusionne pas automatiquement tous les fichiers de même nom rencontrés.

### Garder les outils locaux sous contrôle

Lorsqu’un projet utilise des outils CLI complémentaires, versionnez leur [manifeste d’outils locaux](https://learn.microsoft.com/en-us/dotnet/core/tools/local-tools-how-to-use) `.config/dotnet-tools.json` et documentez `dotnet tool restore`. Cela permet à l’équipe et à la CI de restaurer les versions retenues sans dépendre d’installations globales propres à chaque machine.

## Faire respecter les conventions de code

Un `.editorconfig` partagé évite que chaque éditeur reformate le code différemment. Les conventions s’appliquent dès l’ouverture du projet, ce qui réduit les corrections de style en revue et les réglages à expliquer aux nouveaux membres. Je détaille ces bénéfices dans mon article sur les [avantages d’un fichier `.editorconfig` en .NET](https://codewithfrenchy.com/posts/avantages-editorconfig/).

Commencez avec quelques règles stables. Je dis ça alors que mon propre `.editorconfig` dépasse les 1 450 lignes… Disons que j’ai pris le temps de développer le sujet ! L’idée reste d’ajouter des règles comprises par l’équipe, au rythme des besoins :

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
indent_style = space
indent_size = 4

[*.{json,yml,yaml}]
indent_size = 2

[*.cs]
dotnet_diagnostic.CA1822.severity = warning
```

La [configuration des analyseurs .NET](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/configuration-options) permet de choisir les règles et leur sévérité. La cohérence de mise en forme et la détection de problèmes de code ont des rôles complémentaires : toutes les préférences de l’éditeur ne constituent pas automatiquement un contrôle bloquant au build.

Pour vérifier la mise en forme sans modifier les fichiers, les exemples de CI utiliseront [`dotnet format whitespace`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-format) avec `--verify-no-changes`. La compilation exécutera les analyseurs configurés.

Une plateforme d’analyse centralisée peut compléter ce socle si l’équipe a besoin de tendances ou de politiques transversales. Elle n’est pas un préalable pour commencer à appliquer des règles utiles dans le dépôt.

## Maîtriser la restauration des dépendances

Trois mécanismes remplissent des responsabilités différentes :

| Mécanisme | Question traitée |
|---|---|
| `Directory.Packages.props` | Quelles versions de dépendances directes voulons-nous partager? |
| `NuGet.Config` | À quelles sources la restauration peut-elle accéder? |
| `packages.lock.json` | Quel graphe de dépendances a été résolu pour le projet? |

### Partager les versions lorsque plusieurs projets les utilisent

La [gestion centralisée des packages NuGet](https://learn.microsoft.com/en-us/nuget/consume-packages/central-package-management) permet de définir les versions dans `Directory.Packages.props`. Les projets conservent leurs `PackageReference`, sans répéter la version.

Je la recommande dès que plusieurs projets partagent des dépendances. Elle rend les écarts visibles et facilite les mises à jour coordonnées. Pour une bibliothèque isolée avec très peu de dépendances, son intérêt est plus limité.

La centralisation ne remplace toutefois pas la sélection des packages. Une bibliothèque d’assertions, de mocks ou de mapping devrait répondre à un besoin, avec un examen de sa maintenance et de ses conditions d’utilisation. Le dépôt de référence ne devrait pas ajouter ces dépendances à tous les projets par habitude.

### Déclarer les sources explicitement

Un `NuGet.Config` à la racine peut limiter la restauration aux sources attendues. Cet exemple comprend un registre public et un registre privé à remplacer par celui de l’organisation :

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org"
         value="https://api.nuget.org/v3/index.json" />
    <add key="company"
         value="https://pkgs.dev.azure.com/organization/project/_packaging/company/nuget/v3/index.json" />
  </packageSources>
  <packageSourceMapping>
    <packageSource key="company">
      <package pattern="Company.*" />
    </packageSource>
    <packageSource key="nuget.org">
      <package pattern="*" />
    </packageSource>
  </packageSourceMapping>
</configuration>
```

Le [package source mapping](https://learn.microsoft.com/en-us/nuget/consume-packages/package-source-mapping) donne priorité au motif le plus précis : ici, les packages `Company.*` sont associés à la source privée. Le motif `*` couvre les autres dépendances, y compris transitives. Ce mécanisme s’applique à la résolution depuis les sources. Des packages déjà présents dans le cache global peuvent être réutilisés sans nouvelle recherche.

Si le dépôt utilise uniquement NuGet.org, retirez la source privée et le mapping inutile. Les identifiants d’accès doivent être fournis par le mécanisme d’authentification du poste ou du pipeline, jamais inscrits dans ce fichier.

### Verrouiller le graphe restauré en CI

Avec `RestorePackagesWithLockFile`, une restauration génère les [fichiers `packages.lock.json`](https://learn.microsoft.com/en-us/nuget/consume-packages/package-references-in-project-files#locking-dependencies). Versionnez-les, puis utilisez `dotnet restore --locked-mode` en CI. Une modification de dépendance doit alors inclure la mise à jour correspondante du verrouillage.

Cette pratique est particulièrement utile pour les applications. Pour une bibliothèque distribuée, le fichier de verrouillage ne dicte pas le graphe final de l’application consommatrice. Il faut donc distinguer la reproductibilité du dépôt de la résolution chez ses utilisateurs.

## Une CI qui exécute les mêmes validations que l’équipe

Une PR devrait déclencher les validations rapides nécessaires pour vérifier le changement : restauration, compilation, tests et contrôles de conventions. La CI doit aussi conserver les résultats utiles au diagnostic.

Les deux exemples suivants utilisent la solution `Product.slnx`, un SDK .NET 10 et des fichiers `packages.lock.json` déjà générés et versionnés. Les projets de tests doivent être inclus dans la solution. Les commandes de test présentées utilisent VSTest. Adaptez les arguments si votre dépôt utilise Microsoft.Testing.Platform.

La sélection `10.0.x` installe un correctif disponible de cette famille de SDK. Documentez la version prise en charge sur les postes et alignez sa mise à jour avec celle de la CI. Une version exacte peut être retenue si le projet exige un outillage strictement identique.

Ces pipelines supposent des packages accessibles sans authentification supplémentaire. Pour un registre privé, ajoutez l’authentification avant la restauration. Si vous utilisez un manifeste d’outils locaux, ajoutez également `dotnet tool restore`.

### GitHub Actions

Le fichier `.github/workflows/ci.yml` s’appuie sur les actions officielles [checkout](https://github.com/actions/checkout), [setup-dotnet](https://github.com/actions/setup-dotnet) et [upload-artifact](https://github.com/actions/upload-artifact) :

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      - uses: actions/setup-dotnet@v6
        with:
          dotnet-version: '10.0.x'

      - name: Restaurer les dépendances
        run: dotnet restore Product.slnx --locked-mode

      - name: Vérifier la mise en forme
        run: dotnet format whitespace Product.slnx --no-restore --verify-no-changes

      - name: Compiler
        run: dotnet build Product.slnx -c Release --no-restore

      - name: Tester
        run: dotnet test Product.slnx -c Release --no-build --logger trx --results-directory TestResults

      - name: Conserver les résultats
        if: ${{ always() }}
        uses: actions/upload-artifact@v7
        with:
          name: test-results
          path: TestResults/
          if-no-files-found: warn
```

### Azure DevOps

Le fichier `azure-pipelines.yml` utilise [UseDotNet](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/use-dotnet-v2?view=azure-pipelines) pour installer le SDK indiqué dans le pipeline et [PublishTestResults](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/publish-test-results-v2?view=azure-pipelines) pour exposer les résultats :

```yaml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: ubuntu-latest

steps:
  - checkout: self

  - task: UseDotNet@2
    inputs:
      packageType: sdk
      version: '10.0.x'

  - script: dotnet restore Product.slnx --locked-mode
    displayName: Restaurer les dépendances

  - script: dotnet format whitespace Product.slnx --no-restore --verify-no-changes
    displayName: Vérifier la mise en forme

  - script: dotnet build Product.slnx -c Release --no-restore
    displayName: Compiler

  - script: dotnet test Product.slnx -c Release --no-build --logger trx --results-directory TestResults
    displayName: Tester

  - task: PublishTestResults@2
    condition: succeededOrFailed()
    inputs:
      testResultsFormat: VSTest
      testResultsFiles: '**/*.trx'
      searchFolder: $(Build.SourcesDirectory)/TestResults
      failTaskOnFailedTests: true
```

Dans Azure Repos Git, l’exécution sur les PR se configure au moyen d’une [politique de validation de build](https://learn.microsoft.com/en-us/azure/devops/pipelines/repos/azure-repos-git?view=azure-devops#pr-triggers) sur la branche cible. Le déclencheur YAML ci-dessus couvre les mises à jour de `main`.

### Relier les contrôles à la fusion et à la livraison

Versionner un pipeline ne suffit pas à empêcher une fusion qui échoue aux validations. Configurez également les contrôles requis et les revues sur la plateforme Git. Le dépôt documente ces règles. La plateforme les applique.

Pour la livraison, je recommande de produire un artefact identifié, associé au commit validé, puis de promouvoir cet artefact entre environnements. Les approbations et les accès de déploiement dépendent du contexte d’exploitation. Une PR devrait pouvoir valider le code sans recevoir des secrets de production.

## Prévoir les tests, la sécurité et l’exploitation

### Des tests organisés selon les risques

Le socle comprend des tests exécutables et des instructions claires pour les lancer. Une API peut commencer avec un projet de tests unitaires et un projet d’intégration. Les tests HTTP peuvent faire partie de ce dernier : cinq projets distincts ne sont pas nécessaires par principe.

Je réserverais les tests de bout en bout aux parcours importants et les tests de charge aux objectifs de performance mesurables. Leur fréquence doit préserver une rétroaction rapide sur les changements courants.

Documentez aussi les dépendances des tests : données initiales, base de données temporaire, ressources éphémères et nettoyage. Un test qui passe seulement après l’intervention d’un collègue n’est pas encore une validation fiable pour l’équipe.

### Des essais d’architecture pour protéger les décisions

Les tests fonctionnels ne détectent pas nécessairement une dépendance interdite entre modules. Des essais d’architecture peuvent vérifier que le cœur métier ne référence pas l’infrastructure, qu’un module utilise les contrats prévus ou que certains types restent dans la couche choisie.

Mon article sur les [essais automatisés d’architecture dans un projet .NET](https://codewithfrenchy.com/posts/essais-architecture-automatises-dotnet/) présente cette approche avec ArchUnitNET et xUnit. MSTest et NUnit disposent également d’intégrations. Ces essais peuvent être regroupés sous `tests/` et exécutés dans les validations de PR.

Le bénéfice DevEx est immédiat : une nouvelle personne reçoit un retour explicite sur une règle du projet, sans devoir connaître tout son historique. Le message du test devrait expliquer la contrainte et, lorsque c’est utile, renvoyer à l’entrée correspondante du registre de décisions.

Il faut toutefois choisir les règles avec discernement. Ces essais vérifient des propriétés de structure, pas la pertinence du découpage métier ni tous les accès effectués à l’exécution. Des contraintes trop liées aux détails d’implémentation rigidifient les refactorisations. Faites évoluer les tests avec les décisions qu’ils protègent.

### Une sécurité visible dans les fichiers et les pratiques

L’[audit NuGet](https://learn.microsoft.com/en-us/nuget/concepts/auditing-packages) permet de signaler les vulnérabilités connues à la restauration. La politique doit préciser quels diagnostics bloquent la CI et comment une exception est suivie. Avec le traitement des avertissements comme erreurs présenté plus haut, les avertissements d’audit peuvent eux aussi bloquer la restauration. Ne les neutralisez pas globalement sans décision explicite.

Ajoutez la détection de secrets et un processus de mise à jour des dépendances selon les capacités de votre plateforme. Une alerte n’a d’utilité durable que si quelqu’un en assure le traitement.

Pour le développement, le [Secret Manager .NET](https://learn.microsoft.com/en-us/aspnet/core/security/app-secrets?view=aspnetcore-10.0) garde les secrets hors du dépôt, mais ne les chiffre pas. Pour les environnements déployés, utilisez le mécanisme de secrets de la plateforme, par exemple Azure Key Vault, avec les autorisations appropriées. Le dépôt doit décrire les paramètres attendus et leur provenance, sans contenir leurs valeurs sensibles.

### Préparer le diagnostic de l’application

Pour une application déployée, prévoyez une configuration de journalisation, des vérifications de santé et les indications nécessaires pour retrouver les signaux en exploitation. Les [primitives .NET et OpenTelemetry](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/observability-with-otel) permettent de construire les logs, métriques et traces utiles au diagnostic.

Le dépôt peut contenir cette instrumentation, les conventions de corrélation et des liens vers les tableaux de bord. L’important est de pouvoir relier un problème observé à une version et à un comportement de l’application. Une bibliothèque seule n’aura pas les mêmes besoins qu’un service exposé en production.

## Ajouter les éléments qui répondent à un besoin réel

Une fois le socle en place, certains ajouts deviennent pertinents :

| Ajout | Quand il apporte de la valeur |
|---|---|
| Infrastructure déclarative | Lorsque l’équipe est responsable des ressources du produit. |
| AppHost Aspire et diagnostic local partagé | Lorsque l’application comporte plusieurs composants et dépendances à démarrer et à observer. |
| Templates de projets | Lorsque les mêmes conventions doivent être reproduites fréquemment. |
| Workflows partagés | Lorsque plusieurs dépôts répètent les mêmes validations. |
| Versionnement et publication NuGet | Lorsque des bibliothèques sont distribuées à d’autres équipes. |
| Licence et consignes communautaires | Lorsque le mode de diffusion et de contribution le demande. |

Ces ajouts ont eux aussi un coût de maintenance. Un template accélère la création d’un projet, mais il ne met pas automatiquement à jour les projets déjà générés. Une bibliothèque commune facilite la réutilisation, mais crée une dépendance à faire évoluer. Prévoyez un responsable et un mécanisme de mise à jour pour les conventions que vous partagez.

### Une place pour les conventions destinées aux assistants IA

Un dépôt peut aussi préciser aux assistants IA les commandes de validation, les limites architecturales et les conventions de contribution. Les [instructions de dépôt pour GitHub Copilot](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/add-custom-instructions/add-repository-instructions), notamment `.github/copilot-instructions.md`, sont un exemple de mécanisme prévu à cette fin. D’autres assistants reconnaissent des fichiers comme `AGENTS.md`. Leur prise en charge dépend de l’outil utilisé.

Je recommande de garder ces instructions courtes et alignées sur la documentation existante. Elles peuvent orienter le travail, mais les compilations, tests et revues restent les contrôles effectifs. Un article à venir couvrira plus en détail l’organisation et l’entretien de ces conventions IA dans un dépôt .NET.

## Construire une checklist adaptée au projet

La checklist du dépôt doit refléter le produit, ses risques et son mode de travail. Une bibliothèque NuGet, une API interne et une plateforme distribuée ne demandent pas les mêmes vérifications. Utilisez les points ci-dessous comme point de départ, puis retirez les éléments sans objet et ajoutez les exigences propres au projet.

Conservez cette checklist dans le dépôt ou le wiki, avec un lien depuis le `README`. Distinguez ce qui est vérifié une fois à l’arrivée d’un développeur, ce qui doit être contrôlé à chaque PR et ce qui relève d’une livraison. Automatisez les contrôles répétitifs pour réserver la lecture humaine aux questions de conception.

Par exemple, un projet avec SQL Server peut exiger de vérifier le démarrage local de la base de données, les migrations et le jeu de données de démonstration. Un traitement asynchrone peut ajouter la reprise d’un message en erreur. Chaque élément devrait décrire un résultat observable, et les exceptions devraient avoir un responsable et une date de revue.

### Le parcours d’intégration

- [ ] Une personne peut suivre le `README` depuis un clone neuf sans étapes implicites.
- [ ] Les accès au registre NuGet, au wiki et aux ressources de développement sont expliqués.
- [ ] Le démarrage local et un premier parcours fonctionnel ont été vérifiés.
- [ ] L’AppHost Aspire, lorsqu’il est utilisé, démarre les composants attendus et permet de les diagnostiquer.
- [ ] Les données de démonstration et la remise à zéro de l’environnement sont documentées.
- [ ] Les outils IA autorisés et leur accès au diagnostic local sont explicités, lorsqu’ils sont utilisés.



### Le socle à vérifier

- [ ] Le `README` permet de démarrer depuis un clone neuf.
- [ ] La structure et les responsabilités des projets sont compréhensibles.
- [ ] Le registre de décisions est accessible et les choix structurants sont documentés.
- [ ] Le SDK attendu et sa politique de mise à jour sont explicites.
- [ ] Les conventions de code et de compilation sont versionnées et vérifiées.
- [ ] Les sources NuGet, leur authentification et la politique de verrouillage sont documentées.
- [ ] Les tests et leurs prérequis peuvent être exécutés par l’équipe et la CI.
- [ ] Les validations requises protègent la fusion sur la branche principale.
- [ ] Les résultats de tests restent accessibles après l’exécution.
- [ ] Les secrets sont externes au dépôt et les alertes de dépendances ont un responsable.

### Les ajouts selon le contexte

- [ ] Centralisation des versions NuGet entre plusieurs projets.
- [ ] Tests d’intégration, de bout en bout ou de charge adaptés aux risques.
- [ ] Essais d’architecture alignés sur les frontières et les décisions du projet.
- [ ] Artefacts identifiés, déploiement et procédures de retour arrière.
- [ ] Infrastructure déclarative, instrumentation et documentation d’exploitation.
- [ ] Templates ou workflows partagés avec une stratégie de mise à jour.
- [ ] Instructions IA cohérentes avec les règles de contribution.

## Conclusion

Un dépôt .NET utile améliore l’expérience développeur en rendant les conditions de travail explicites. Il indique comment démarrer, sélectionne l’outillage attendu, organise les dépendances et permet de vérifier une modification avec des commandes connues de tous.

Je commencerais par ce parcours complet, du clone à la première modification validée. Un `README` qui oriente vers les bonnes sources, un environnement local partagé avec Aspire lorsque le produit s’y prête et des essais d’architecture rendent ce parcours plus accessible aux nouveaux membres. Les templates, l’infrastructure déclarative et les conventions transversales peuvent ensuite être ajoutés là où ils résolvent une difficulté récurrente.

La checklist propre au projet permet de maintenir ce parcours au fil des évolutions. La qualité du dépôt se voit dans le quotidien : moins de manipulations à expliquer, moins d’écarts entre les postes et la CI, et moins de dépendance à la mémoire de quelques personnes. C’est cette continuité qui permet à l’équipe de faire évoluer le produit avec confiance.
