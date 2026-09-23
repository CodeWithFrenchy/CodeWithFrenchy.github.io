---
title: Stryker.NET - vos tests détectent-ils vraiment les erreurs?
date: 2024-12-31 19:00:00 -0400
categories: [outil-developpement]
tags: [dotnet, essais]
---

> 💡 à valider une dernière fois

## Préambule

Une suite de tests passe au vert. La couverture de code atteint l’objectif fixé par l’équipe. Pourtant, une modification mineure à une règle métier peut encore introduire une régression sans faire échouer un seul test.

Ce décalage mérite qu’on s’y attarde. La couverture indique quelles parties du code ont été exécutées. Elle ne dit pas si les scénarios et les assertions permettraient de repérer un comportement incorrect.

Le *mutation testing*, ou test par mutation, permet d’explorer cette limite : on introduit volontairement de petites modifications dans le code, puis on observe si les tests les détectent. Dans cet article, nous allons utiliser [Stryker.NET](https://stryker-mutator.io/docs/stryker-net/introduction/) pour appliquer cette démarche à une règle métier, améliorer ses tests xUnit et intégrer l’analyse dans Azure DevOps ou GitHub Actions.

## Une bonne couverture peut laisser passer une erreur

Prenons une règle simple : une commande est admissible à la livraison gratuite à partir de 100 $. Les montants sont exprimés en dollars. La règle utilise le type `decimal`, adapté aux montants décimaux. Les données de test utilisent ici des dollars entiers, convertis implicitement en `decimal` lors de l’appel.

```csharp
namespace Livraison.Core;

public static class PolitiqueLivraison
{
    public static bool EstAdmissibleLivraisonGratuite(decimal sousTotalEnDollars)
        => sousTotalEnDollars >= 100m;
}
```

Voici deux scénarios xUnit : une commande sous le seuil et une autre au-dessus.

```csharp
using Livraison.Core;
using Xunit;

namespace Livraison.Core.Tests;

public class PolitiqueLivraisonTests
{
    [Theory]
    [InlineData(99, false)]
    [InlineData(101, true)]
    public void EstAdmissibleLivraisonGratuite_MontantAutourDuSeuil_RetourneAdmissibiliteAttendue(
        int sousTotalEnDollars,
        bool resultatAttendu)
    {
        bool resultatObtenu = PolitiqueLivraison.EstAdmissibleLivraisonGratuite(sousTotalEnDollars);

        Assert.Equal(resultatAttendu, resultatObtenu);
    }
}
```

Ces tests exécutent la règle et vérifient les deux résultats possibles. Mais remplaçons maintenant `>=` par `>` :

```csharp
public static bool EstAdmissibleLivraisonGratuite(decimal sousTotalEnDollars)
    => sousTotalEnDollars > 100m;
```

Les deux tests passent toujours. Une commande de **99 dollars** demeure inadmissible. Une commande de **101 dollars** demeure admissible. La régression touche uniquement la commande de **100 dollars**, un cas absent de notre jeu de données.

**Le code est exécuté, mais la frontière de la règle métier n’est pas protégée.**

## Ce que Stryker.NET apporte

[Stryker.NET](https://github.com/stryker-mutator/stryker-net) automatise ce type d’expérience dans les projets .NET. Il produit des variantes du code, appelées *mutants*, puis utilise les tests existants pour vérifier si elles provoquent un échec. Il ne faut donc pas modifier soi-même les opérateurs pour chaque essai.

Dans notre exemple, la mutation de `>=` vers `>` représente une erreur plausible : exclure accidentellement la valeur limite. Si les tests restent verts, cette mutation révèle un scénario à examiner.

L’article utilise exclusivement xUnit pour les exemples. **NUnit et MSTest sont également supportés**, avec les adaptateurs et le moteur d’exécution compatibles avec votre projet.

Cette démarche complète les autres contrôles qualité. Les tests d’intégration vérifient les échanges entre composants. Les revues examinent les choix d’implémentation. L’analyse statique repère certaines catégories de problèmes. Le test par mutation apporte un regard supplémentaire sur la capacité des tests à détecter des changements de comportement.

## Démarrer avec un projet xUnit

Pour reproduire l’exemple, nous allons utiliser le SDK .NET 10. Les [prérequis de Stryker.NET](https://stryker-mutator.io/docs/stryker-net/getting-started/) indiquent actuellement un runtime .NET 10 ou plus récent pour exécuter l’outil. Cette exigence ne force pas votre application existante à cibler .NET 10 : elle doit conserver ses propres SDK et runtimes nécessaires.

Depuis un dossier vide, créez la bibliothèque, le projet de tests et la référence entre les deux :

```bash
dotnet new classlib -n Livraison.Core -o src/Livraison.Core -f net10.0
dotnet new xunit -n Livraison.Core.Tests -o tests/Livraison.Core.Tests -f net10.0
dotnet add tests/Livraison.Core.Tests/Livraison.Core.Tests.csproj reference src/Livraison.Core/Livraison.Core.csproj

dotnet tool install --global dotnet-stryker
```

Cette organisation reprend le principe du [projet de tests xUnit documenté par Microsoft](https://learn.microsoft.com/en-us/dotnet/core/testing/unit-testing-csharp-with-xunit) : les tests référencent la bibliothèque dont ils vérifient le comportement.

Remplacez `Class1.cs` par un fichier `PolitiqueLivraison.cs` contenant la règle originale avec `>=`. Remplacez `UnitTest1.cs` par `PolitiqueLivraisonTests.cs` contenant les deux scénarios précédents.

L’installation globale rend `dotnet stryker` accessible depuis les différents dépôts du compte utilisateur. Elle ne rend pas la configuration commune à tous les projets. Pour mettre l’outil à jour, utilisez `dotnet tool update --global dotnet-stryker`.

Pour une exécution reproductible, ajoutez `--version` suivi de la version validée par l’équipe à la commande d’installation, sur les postes et en CI. Les exemples ci-dessous installent la version disponible, à remplacer par cette sélection explicite dans votre pipeline. Sur un agent réutilisé, prévoyez aussi le cas où l’outil est déjà installé.

Vérifiez les tests, puis lancez Stryker depuis leur répertoire :

```bash
dotnet test tests/Livraison.Core.Tests/Livraison.Core.Tests.csproj

cd tests/Livraison.Core.Tests
dotnet stryker
```

Ce mode d’exécution depuis le projet de tests convient à notre exemple, qui référence une seule bibliothèque métier.

## Lire le rapport et améliorer le bon scénario

Commencez par le [rapport HTML de Stryker](https://stryker-mutator.io/docs/stryker-net/reporters/), généré sous `StrykerOutput` dans le répertoire d’exécution. Il permet d’examiner les modifications proposées et leur résultat directement dans le code.

Pensez à exclure les rapports générés du dépôt. Ajoutez cette règle au `.gitignore` à la racine pour couvrir les répertoires de sortie, quel que soit leur emplacement :

```gitignore
**/StrykerOutput/
```

Les rapports utiles seront conservés comme artefacts de CI plutôt que versionnés avec le code.

Les [états des mutants](https://stryker-mutator.io/docs/mutation-testing-elements/mutant-states-and-metrics/) se lisent ainsi :

| État | Signification |
|---|---|
| `Killed` | Au moins un test a échoué avec cette mutation. |
| `Survived` | Les tests exécutés n’ont pas détecté la mutation. |
| `No coverage` | Aucun test ne couvre le mutant. |
| `Timeout` | L’exécution a dépassé le délai. Le mutant est compté comme détecté. |
| `Compile error` / `Runtime error` | Le mutant n’a pas pu être évalué normalement. Il est exclu du score. |
| `Ignored` | Le mutant est ignoré et n’entre pas dans le score. |

Dans notre démonstration, les deux scénarios ne distinguent pas `>=` de `>`. Pour détecter cette mutation, ajoutez le cas exact de 100 $ à la théorie existante :

```csharp
[Theory]
[InlineData(99, false)]
[InlineData(100, true)]
[InlineData(101, true)]
public void EstAdmissibleLivraisonGratuite_MontantAutourDuSeuil_RetourneAdmissibiliteAttendue(
    int sousTotalEnDollars,
    bool resultatAttendu)
{
    bool resultatObtenu = PolitiqueLivraison.EstAdmissibleLivraisonGratuite(sousTotalEnDollars);

    Assert.Equal(resultatAttendu, resultatObtenu);
}
```

Avec la règle originale, ce scénario passe. Avec l’opérateur `>`, il échoue : 100 $ devient inadmissible, contrairement à l’exigence. Relancez Stryker pour observer l’effet de ce nouveau cas.

L’amélioration tient à une donnée de test supplémentaire. Il n’a fallu ni multiplier les mocks ni exposer un détail interne de la classe. Le test exprime maintenant une exigence métier qui était implicite.

### Interpréter le score sans en faire une cible absolue

Le score correspond à `100 × détectés / valides`, avec `détectés = Killed + Timeout` et `valides = Killed + Timeout + Survived + No coverage`.

Ce pourcentage aide à suivre une évolution, mais il ne suffit pas pour décider quoi corriger. Un mutant survivant dans un calcul de facturation mérite généralement plus d’attention qu’un changement sans effet utile sur le comportement attendu.

Il existe aussi des [mutants équivalents](https://stryker-mutator.io/docs/mutation-testing-elements/equivalent-mutants/) : leur code diffère, mais leur comportement reste identique. Aucun test comportemental ne peut alors les distinguer de l’original. Avant d’ajouter un test, identifiez l’exigence que la mutation enfreint réellement.

## Garder une configuration ciblée

Pour notre pilote, je recommande de limiter explicitement l’analyse à la règle étudiée. Placez ce fichier `stryker-config.json` dans `tests/Livraison.Core.Tests` :

```json
{
  "stryker-config": {
    "project": "Livraison.Core.csproj",
    "mutate": ["**/PolitiqueLivraison.cs"],
    "reporters": ["progress", "html", "json"],
    "thresholds": {
      "high": 80,
      "low": 60,
      "break": 0
    }
  }
}
```

Dans les [options de configuration](https://stryker-mutator.io/docs/stryker-net/configuration/), `project` désigne le nom du fichier projet, sans chemin. `mutate` restreint les fichiers analysés. Les seuils `high` et `low` déterminent la présentation du score. `break` fixe le seuil d’échec. Ici, un score faible ne bloque pas le pipeline, mais une erreur d’exécution peut toujours le faire échouer.

### Centraliser la configuration à la racine du dépôt

Dans cet exemple pédagogique, le fichier reste directement dans le projet de tests. Pour un dépôt comportant plusieurs projets, je préfère une configuration à la racine, versionnée et partagée avec la CI. L’installation globale de Stryker reste indépendante de ce choix.

L’option [`test-projects`](https://stryker-mutator.io/docs/stryker-net/configuration/#test-projects-string) permet de déclarer plusieurs projets de tests couvrant **un même projet applicatif**. Dans ce mode, lancez Stryker depuis le projet applicatif. Voici une configuration racine pour `Livraison.Core`, en supposant qu’un second projet `Livraison.Integration.Tests` existe et référence cette bibliothèque :

```json
{
  "stryker-config": {
    "test-projects": [
      "../../tests/Livraison.Core.Tests/Livraison.Core.Tests.csproj",
      "../../tests/Livraison.Integration.Tests/Livraison.Integration.Tests.csproj"
    ],
    "mutate": ["**/PolitiqueLivraison.cs"],
    "reporters": ["progress", "html", "json"],
    "thresholds": {
      "high": 80,
      "low": 60,
      "break": 0
    }
  }
}
```

Depuis la racine, exécutez :

```bash
cd src/Livraison.Core
dotnet stryker --config-file ../../stryker-config.json
```

Les chemins des projets de tests sont ici exprimés depuis le répertoire d’exécution. Le dossier `StrykerOutput` s’y trouvera également. Pour plusieurs bibliothèques à muter, prévoyez des exécutions par cible, orchestrées par un script ou une matrice CI, ou évaluez le mode solution. Une liste de projets de tests ne désigne pas automatiquement toutes les bibliothèques à muter.

### Élargir progressivement le périmètre

Après quelques analyses, élargissez le périmètre aux règles voisines : calcul de frais, admissibilité, validations ou transitions d’état. Pour une API ASP.NET Core, je commencerais par ces composants avant d’inclure la configuration de l’application ou les branchements d’injection de dépendances.

Évitez également les exclusions systématiques basées uniquement sur le nom d’une méthode. Une méthode de conversion ou de comparaison peut porter une règle importante. Le périmètre doit refléter le risque métier, et toute exclusion doit pouvoir s’expliquer.

## Intégrer l’analyse dans la CI

Pour ce petit projet, une analyse complète du périmètre configuré à chaque pull request reste un point de départ raisonnable. Le temps d’exécution peut toutefois devenir relativement long selon la taille du projet, le nombre de mutants et la durée des tests. Mesurez ce coût avant de rendre l’analyse complète obligatoire sur toutes les PR. Une analyse ciblée sur les changements et une exécution complète planifiée peuvent mieux préserver la rapidité de rétroaction.

Les pipelines ci-dessous installent Stryker globalement, exécutent les tests, lancent l’analyse et conservent les rapports, y compris lorsque l’analyse échoue.

Ils reprennent le pilote avec un seul projet de tests et son fichier de configuration local, tous deux versionnés. Pour adopter la configuration racine, ajustez le répertoire d’exécution, l’argument `--config-file` et le chemin des rapports conformément à l’exemple précédent. Le seuil demeure à zéro pendant le pilote.

### Azure DevOps

Ajoutez ce fichier `azure-pipelines.yml` à la racine du dépôt. L’exemple vise Azure DevOps et utilise les tâches [UseDotNet](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/use-dotnet-v2?view=azure-pipelines) et [PublishPipelineArtifact](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/publish-pipeline-artifact-v1?view=azure-pipelines).

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
    displayName: Installer le SDK .NET
    inputs:
      packageType: sdk
      version: 10.0.x

  - script: dotnet tool install --global dotnet-stryker
    displayName: Installer Stryker globalement

  - script: dotnet test tests/Livraison.Core.Tests/Livraison.Core.Tests.csproj
    displayName: Exécuter les tests

  - bash: mkdir -p StrykerOutput
    displayName: Préparer le dossier des rapports
    workingDirectory: tests/Livraison.Core.Tests

  - script: dotnet stryker
    displayName: Exécuter Stryker
    workingDirectory: tests/Livraison.Core.Tests

  - task: PublishPipelineArtifact@1
    displayName: Publier les rapports Stryker
    condition: succeededOrFailed()
    inputs:
      targetPath: tests/Livraison.Core.Tests/StrykerOutput
      artifact: stryker-report
```

Avec Azure Repos Git, associez ce pipeline à une [politique de validation de build sur la branche cible](https://learn.microsoft.com/en-us/azure/devops/pipelines/repos/azure-repos-git?view=azure-devops#pr-triggers) pour l’exécuter sur les pull requests. Le déclencheur `trigger` ci-dessus couvre les mises à jour de `main`.

### GitHub Actions

Ajoutez ce workflow dans `.github/workflows/mutation-testing.yml`. Il utilise les actions officielles [checkout](https://github.com/actions/checkout), [setup-dotnet](https://github.com/actions/setup-dotnet) et [upload-artifact](https://github.com/actions/upload-artifact).

```yaml
name: Mutation testing

on:
  pull_request:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  stryker:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v7

      - name: Installer le SDK .NET
        uses: actions/setup-dotnet@v6
        with:
          dotnet-version: 10.0.x

      - name: Installer Stryker globalement
        run: dotnet tool install --global dotnet-stryker

      - name: Exécuter les tests
        run: dotnet test tests/Livraison.Core.Tests/Livraison.Core.Tests.csproj

      - name: Exécuter Stryker
        working-directory: tests/Livraison.Core.Tests
        run: dotnet stryker

      - name: Publier les rapports Stryker
        if: ${{ always() }}
        uses: actions/upload-artifact@v7
        with:
          name: stryker-report
          path: tests/Livraison.Core.Tests/StrykerOutput/
          if-no-files-found: warn
```

Ces exemples conservent les rapports comme artefacts du pipeline. Ils ne nécessitent aucune clé de Stryker Dashboard. Si l’analyse échoue avant de générer un rapport, les journaux du job restent le point de départ du diagnostic.

### Adapter la fréquence quand le projet grandit

La durée mérite d’être mesurée dès le pilote. La [FAQ de Stryker](https://stryker-mutator.io/docs/General/faq/) recommande notamment de commencer par améliorer la rapidité des tests. Une suite qui initialise inutilement une application complète ou dépend de services externes rendra les répétitions plus coûteuses.

Je recommande ensuite de choisir la fréquence selon le temps de rétroaction acceptable :

| Contexte | Approche proposée |
|---|---|
| Petite bibliothèque et tests rapides | Analyse complète du périmètre sur les PR. |
| Projet plus volumineux | Analyse ciblée sur les PR et analyse complète planifiée. |
| Code historique peu testé | Pilote sur un module critique, puis extension graduelle. |

L’option [`since`](https://stryker-mutator.io/docs/stryker-net/configuration/#since-flag-committish) limite l’analyse aux changements depuis une référence Git et produit un rapport partiel. [`with-baseline`](https://stryker-mutator.io/docs/stryker-net/configuration/#with-baseline-flag-committish) réutilise les résultats antérieurs pour compléter le rapport. Ces modes sont mutuellement exclusifs. La baseline est documentée comme expérimentale.

Si vous introduisez cette optimisation, rendez disponibles l’historique Git et la référence cible sur l’agent. Conservez aussi une analyse complète périodique : comparer un score partiel à un ancien score global donnerait une lecture trompeuse.

## Adopter l’outil sans transformer le score en objectif

Le premier rapport peut montrer beaucoup de survivants. Je recommande de les examiner avec l’équipe sur un petit module, puis de retenir quelques cas qui correspondent à des risques concrets : une borne oubliée, une exception attendue, une transition interdite ou un résultat insuffisamment vérifié.

Une fois le périmètre stabilisé, un seuil d’échec peut empêcher certaines régressions du score. Par exemple, `dotnet stryker --break-at 60` fait échouer l’analyse sous 60 %. Ce chiffre est un exemple, à ajuster à partir du pilote.

Pour suivre l’utilité de la démarche, je retiendrais trois éléments : les survivants pertinents corrigés, l’évolution du score à périmètre constant et la durée d’analyse. Un score qui augmente après l’exclusion d’un dossier n’a pas la même signification qu’un score qui augmente après l’ajout de scénarios métier.

Enfin, une suite renforcée reste dépendante des exigences qu’elle exprime. Si l’équipe a mal compris la règle de livraison gratuite, Stryker ne peut pas retrouver l’intention métier à sa place. Les échanges avec le métier et les revues des scénarios conservent toute leur valeur.

## Conclusion

Stryker.NET donne une façon concrète de vérifier ce que nos tests protègent réellement. Dans l’exemple de livraison gratuite, deux scénarios semblaient suffisants. La mutation d’un opérateur a révélé l’absence du cas exact de 100 $, et une seule donnée supplémentaire a permis de protéger cette frontière.

Pour une équipe .NET, je recommande de commencer avec une bibliothèque métier, des tests rapides et un rapport que l’on prend le temps de lire. L’intégration CI et les seuils viennent ensuite, lorsque le bénéfice et le coût sont visibles.

La valeur du mutation testing se mesure surtout aux erreurs plausibles que la suite de tests sait désormais détecter. C’est ce lien avec le comportement attendu qui rend l’outil utile au quotidien.
