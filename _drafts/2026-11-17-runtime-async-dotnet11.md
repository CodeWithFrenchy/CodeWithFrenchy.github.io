---
title: "Runtime Async en .NET 11 : comprendre et mesurer les gains"
date: 2026-11-17 19:00:00 -0400
categories: [dotnet, performance]
tags: [dotnet, dotnet11, csharp, async-await, runtime-async, benchmarkdotnet, aspnet-core, diagnostic]
---

## Préambule

Dans une API .NET, une requête traverse souvent plusieurs méthodes asynchrones avant d’atteindre un service distant ou une base de données. Un point de terminaison appelle un service applicatif, qui appelle un Repository, qui attend lui-même une opération d’entrée-sortie. Cette décomposition rend le code plus facile à comprendre, mais l’exécution de chaque niveau possède aussi un coût.

J’avais présenté Runtime Async dans mon [tour d’horizon de .NET 11](https://codewithfrenchy.com/posts/dotnet11/). La session [.NET Runtime vNext: Memory Safety and Runtime Async](https://www.youtube.com/watch?v=7udirOvQqLk), à la .NET Conf 2026, permet d’aller plus loin sur son fonctionnement et ses effets possibles.

L’objectif est de réduire le coût de l’asynchronisme sans changer la façon dont nous écrivons `async` et `await`. Les gains peuvent concerner le temps CPU, les allocations, la propagation des exceptions et la lisibilité du diagnostic.

Encore faut-il déterminer où ces gains apparaissent dans notre application. Cet article présente des mesures publiées, puis une démarche pour comparer les deux modèles avec BenchmarkDotNet et une API représentative. Les chiffres cités proviennent des sources indiquées. Le protocole d’API proposé n’a pas été exécuté pour cet article.

## Ce qui change dans l’exécution de nos méthodes

### La transformation classique par le compilateur

Traditionnellement, le compilateur C# transforme une méthode `async` en machine à états. Le code généré conserve les informations nécessaires pour reprendre l’exécution après une attente, coordonne les continuations et transmet le résultat ou l’exception au consommateur.

Ce mécanisme permet d’écrire un enchaînement lisible tout en libérant le thread lorsqu’une opération doit réellement attendre. Il explique aussi la présence de méthodes comme `MoveNext` et de types générés dans certains outils de diagnostic.

Cette transformation a déjà reçu de nombreuses optimisations. Une méthode asynchrone n’alloue donc pas systématiquement un nouvel objet pour chaque `await`. Les tâches déjà terminées, certains résultats mis en cache et les usages adaptés de `ValueTask` peuvent déjà limiter les coûts.

Le défi demeure lorsque plusieurs méthodes asynchrones se composent. Chaque frontière compilée peut introduire de la coordination et des objets intermédiaires, même lorsque toute la chaîne termine son travail immédiatement.

### Une transformation connue du runtime

Avec Runtime Async, le compilateur émet une représentation que le runtime sait interpréter comme une méthode asynchrone. Le JIT dispose alors de davantage d’information pour organiser sa suspension, sa reprise et ses interactions avec les méthodes appelées.

La [spécification de Runtime Async](https://github.com/dotnet/runtime/blob/main/docs/design/specs/runtime-async.md) décrit notamment le marquage `MethodImplOptions.Async`. La démonstration de la conférence montre aussi `AsyncHelpers.Await`, reconnu par le JIT. Ces éléments appartiennent au mécanisme de compilation. Leur présence dans le code généré ne signifie pas qu’il faut les ajouter manuellement aux méthodes applicatives.

Pour une chaîne d’appels directement attendus, le runtime peut éviter de matérialiser certaines tâches qui serviraient seulement à transmettre un résultat au niveau suivant. L’état nécessaire à une véritable suspension existe toujours, mais sa représentation et son parcours peuvent être optimisés.

La promesse observable reste celle de `Task`, `Task<T>` ou des types pris en charge. Une tâche réellement exposée à un consommateur doit conserver son comportement. Runtime Async n’autorise pas à ignorer les règles d’utilisation de `ValueTask`, l’annulation ou les effets de `ConfigureAwait`.

## Activer Runtime Async en .NET 11

L’[annonce officielle de .NET 11](https://devblogs.microsoft.com/dotnet/announcing-dotnet-11/) précise que Runtime Async reste en préversion et que le code applicatif l’active explicitement.

Dans un projet compilé avec le SDK .NET 11 et ciblant `net11.0`, l’activation prend cette forme.

```xml
<PropertyGroup>
  <TargetFramework>net11.0</TargetFramework>
  <Features>runtime-async=on</Features>
</PropertyGroup>
```

Cette activation ne demande pas, à elle seule, `EnablePreviewFeatures` ou `LangVersion=preview`. Le [suivi officiel de la fonctionnalité](https://github.com/dotnet/runtime/issues/109632) distingue les conditions de .NET 11 de celles des premières expérimentations en .NET 10. Il indique également que Mono n’est pas pris en charge.

Pour notre comparaison, nous retiendrons une application serveur exécutée avec CoreCLR et le JIT. Native AOT et ReadyToRun méritent des campagnes distinctes si ces modes font partie du déploiement réel.

Les anciennes variables `DOTNET_RuntimeAsync` et `UNSUPPORTED_RuntimeAsync` ont été retirées, comme le précise la [documentation du runtime .NET 11](https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-11/runtime). L’expérience doit porter sur la compilation des projets concernés.

Si une propriété `Features` existe déjà, il faut y intégrer l’option en conservant les autres fonctionnalités activées. Les exemples de cet article supposent qu’aucune autre valeur n’y est définie.

### Ce que l’activation couvre réellement

Une partie du framework .NET 11 est déjà compilée avec Runtime Async. Les [notes de version ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/release-notes/aspnetcore-11) précisent que les bibliothèques distribuées uniquement dans son framework partagé l’utilisent également. Certaines bibliothèques aussi distribuées en paquets NuGet sont exclues, notamment pour préserver la compatibilité avec WebAssembly.

Activer la fonctionnalité dans un projet ne recompile pas les paquets NuGet qu’il consomme. Il faut aussi examiner les projets de bibliothèques internes qui contiennent les méthodes à mesurer.

La comparaison pertinente est donc la suivante : même .NET 11, mêmes dépendances et même code, avec deux modes de compilation pour les projets applicatifs ciblés. Comparer uniquement .NET 10 à .NET 11 mélangerait Runtime Async avec toutes les autres évolutions du runtime et des bibliothèques.

## Allocations : des économies possibles, selon le parcours

### Lorsque l’opération termine immédiatement

Une méthode déclarée asynchrone peut terminer sans suspension. C’est fréquent lorsqu’un résultat est déjà en mémoire ou qu’une lecture peut être satisfaite à partir d’un tampon.

La conférence illustrait ce cas avec un parcours de fichiers. Certaines opérations revenaient rapidement avec les informations nécessaires, même si elles se trouvaient dans une chaîne conçue pour supporter des entrées-sorties asynchrones.

Runtime Async peut alors réduire le coût des niveaux intermédiaires. Dans les scénarios présentés, les allocations associées à certaines chaînes disparaissaient, même lorsque leur profondeur augmentait.

Pour une API sollicitée fréquemment, économiser des objets à chaque requête peut réduire le débit d’allocation et la pression sur le garbage collector. Le gain devient surtout intéressant lorsque ce parcours est fréquent et représente une part significative du travail effectué.

### Lorsque l’exécution doit réellement attendre

Une lecture réseau qui n’a pas encore de résultat exige une suspension. Le runtime doit conserver suffisamment d’état pour reprendre la méthode plus tard. Des allocations peuvent donc subsister.

L’objet métier, le contenu JSON, les buffers et les objets créés par les dépendances ne disparaissent pas non plus. Une mesure à zéro octet sur une petite chaîne asynchrone ne signifie pas qu’un point de terminaison HTTP complet n’alloue plus.

Runtime Async ne réduit pas directement le temps nécessaire au serveur SQL pour exécuter une requête, ni la latence du réseau. Il peut réduire une partie du travail réalisé autour de cette attente.

## Exceptions : réduire le coût de leur propagation

Dans le modèle classique, une exception peut être interceptée par le code généré d’un niveau, enregistrée dans une tâche, puis relancée lors de l’attente au niveau supérieur. Cette mécanique se répète le long de la chaîne.

Runtime Async permet d’éviter une partie de ce travail intermédiaire. La session montrait des améliorations sur le temps de propagation et les allocations, particulièrement lorsque plusieurs méthodes asynchrones étaient traversées.

Le bénéfice potentiel est intéressant lors d’une défaillance de dépendance. Une vague d’exceptions peut ajouter du travail CPU et des allocations au moment où le service fonctionne déjà en mode dégradé.

Cela ne transforme pas les exceptions en mécanisme de contrôle gratuit. Les reprises, les délais d’attente, la journalisation et le volume d’erreurs restent à maîtriser. Une campagne de mesure doit donc isoler le coût de propagation, puis vérifier le comportement complet avec la journalisation habituelle.

## Ce que montrent les benchmarks publiés

Dans [Performance Improvements in .NET 11](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-11/), Stephen Toub compare les deux modèles avec BenchmarkDotNet. Voici quatre résultats publiés le 15 septembre 2026, pendant le développement de .NET 11.

| Scénario publié | Modèle classique | Runtime Async | Allocations classiques | Allocations Runtime Async |
| --- | ---: | ---: | ---: | ---: |
| Chaîne de deux méthodes, terminaison synchrone | 21,221 ns | 6,151 ns | 144 octets | 0 octet |
| Chaîne de deux méthodes, suspension avec `Task.Yield` | 254,139 ns | 116,927 ns | 248 octets | 168 octets |
| Exception, profondeur paramétrée à 1, avec suspension | 6,308 µs | 8,211 µs | 1,68 Kio | 1,42 Kio |
| Exception, profondeur paramétrée à 10, avec suspension | 24,923 µs | 6,122 µs | 15,63 Kio | 2,88 Kio |

Le premier cas élimine les allocations mesurées. Le troisième alloue moins, mais prend davantage de temps. Les exemples liés utilisent le même runtime .NET 11 et une sélection expérimentale du mode de compilation par méthode.

Ces résultats dépendent de l’environnement mesuré. Ils ne prédisent pas le débit d’une API.

## Construire une comparaison utile avec BenchmarkDotNet

Je commencerais par un microbenchmark qui sépare les principales situations au lieu de les mélanger dans une moyenne.

| Dimension | Cas à inclure | Ce que l’on cherche à observer |
| --- | --- | --- |
| Terminaison | Résultat disponible immédiatement | Coût des frontières asynchrones sans suspension |
| Suspension | Une attente qui suspend réellement | Coût de conservation et de reprise de l’état |
| Profondeur | Par exemple 1, 5 et 20 niveaux | Effet de la composition des méthodes |
| Résultat | Succès et exception dans des mesures séparées | Coût du parcours normal et du parcours d’erreur |

Une tâche déjà terminée contenant un entier comme 42 permet d’exercer le premier cas. Le choix de la valeur et la création de cette tâche doivent rester identiques entre les variantes. On évite ainsi de confondre la préparation des données avec le travail mesuré.

Pour le deuxième cas, `Task.Yield` permet de forcer une suspension, mais mesure aussi l’ordonnancement de la continuation. Ce scénario synthétique ne remplace pas une véritable opération réseau. Un `Task.Delay` long pourrait, de son côté, masquer presque complètement le coût que l’on souhaite isoler.

La méthode de benchmark doit attendre l’opération complète et observer son résultat. Une mesure d’exception doit intercepter le type attendu à la même frontière dans les deux variantes. Sinon, elle mesure autre chose ou interrompt simplement le benchmark.

### Faire varier la compilation, avec le même environnement

Pour ces essais, je remplacerais l’activation fixe par une activation conditionnelle dans les projets concernés. Ce bloc se place dans leur fichier de projet, après la définition du framework cible.

```xml
<PropertyGroup Condition="'$(ActiverRuntimeAsync)' == 'true'">
  <Features>runtime-async=on</Features>
</PropertyGroup>
```

`ActiverRuntimeAsync` est ici une propriété choisie pour notre expérience. Elle ne représente pas une nouvelle option officielle du SDK. Quand elle vaut `false`, aucune autre configuration ne doit activer `runtime-async` dans les projets comparés.

BenchmarkDotNet [génère et compile ses propres applications de mesure](https://benchmarkdotnet.org/articles/configs/toolchains.html). Une propriété transmise seulement au lancement initial avec `dotnet run` ne suffit donc pas à démontrer que les projets mesurés ont reçu la bonne configuration.

Les arguments doivent être transmis aux jobs, par exemple avec [WithMsBuildArguments](https://benchmarkdotnet.org/api/BenchmarkDotNet.Jobs.JobExtensions.html). Voici les deux expressions de configuration à intégrer au programme de benchmarks.

```csharp
Job.Default
    .WithId("CompilerAsync")
    .WithMsBuildArguments("/p:ActiverRuntimeAsync=false")
```

```csharp
Job.Default
    .WithId("RuntimeAsync")
    .WithMsBuildArguments("/p:ActiverRuntimeAsync=true")
```

Le projet de benchmarks cible `net11.0` dans les deux cas, avec une version de BenchmarkDotNet compatible et fixée dans le projet. Je conserve les journaux de compilation et je vérifie, sur une méthode éligible, que les deux assemblages utilisent effectivement des transformations différentes.

Les résultats doivent aussi indiquer les versions exactes du SDK, du runtime et de BenchmarkDotNet, le système d’exploitation, l’architecture et le processeur. Le mode de GC et les paramètres de compilation progressive doivent rester identiques.

### Lire le temps et les allocations ensemble

Les [bonnes pratiques de BenchmarkDotNet](https://benchmarkdotnet.org/articles/guides/good-practices.html) recommandent notamment une compilation Release, une exécution sans débogueur et l’observation du résultat pour éviter l’élimination du travail mesuré.

Dans un projet utilisant `BenchmarkSwitcher`, une commande de lancement peut prendre cette forme.

```bash
dotnet run -c Release --project benchmarks/RuntimeAsync.Benchmarks -- --filter "*Async*" --memory
```

L’option `--memory`, documentée parmi les [arguments de BenchmarkDotNet](https://benchmarkdotnet.org/articles/guides/console-args.html), active le diagnostic mémoire. Les [diagnostiqueurs disponibles](https://benchmarkdotnet.org/articles/configs/diagnosers.html) permettent d’examiner les allocations et d’autres aspects de l’exécution.

Je conserve la moyenne, la dispersion, le ratio entre les variantes et les octets alloués par opération. Une faible différence de temps doit être comparée à la variabilité des mesures. Le diagnostic mémoire décrit les allocations, pas directement la mémoire résidente du processus.

Pour un service qui reste longtemps en exécution, la mesure après échauffement est particulièrement utile. Le démarrage et les premières requêtes se mesurent séparément. BenchmarkDotNet documente ces différences dans ses [stratégies d’exécution](https://benchmarkdotnet.org/articles/guides/choosing-run-strategy.html).

## Passer à une API représentative

Le microbenchmark aide à comprendre le mécanisme. L’essai d’API permet de déterminer son importance dans le travail réellement effectué.

Prenons une API d’essai `Dossiers.Api` qui expose `GET /dossiers/{id}`. Elle applique les middlewares habituels, appelle un service, consulte un cache, puis utilise un Repository et EF Core lorsque le dossier doit être lu dans la base de données. Elle retourne un sommaire JSON.

Je conserverais trois profils de lecture.

- Un cache préchargé pour observer le parcours qui termine souvent immédiatement.
- Un cache applicatif désactivé dans le banc d’essai pour exercer la lecture dans la base de données.
- Un mélange représentatif de l’usage réel, avec un taux de succès du cache effectivement mesuré.

Un quatrième profil peut injecter des erreurs de dépendance de manière contrôlée. Il doit être analysé séparément, avec le même taux d’erreurs, les mêmes reprises et la même journalisation entre les variantes.

### Publier deux variantes du même code

Avec l’activation conditionnelle précédente appliquée aux projets concernés, les commandes de [publication .NET](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-publish) peuvent isoler leurs sorties.

```bash
dotnet publish src/Dossiers.Api/Dossiers.Api.csproj -c Release -p:ActiverRuntimeAsync=false --artifacts-path artifacts/compiler -o artifacts/compiler/api
```

```bash
dotnet publish src/Dossiers.Api/Dossiers.Api.csproj -c Release -p:ActiverRuntimeAsync=true --artifacts-path artifacts/runtime -o artifacts/runtime/api
```

Ces chemins désignent le projet d’essai à adapter à votre dépôt. Les deux publications doivent utiliser les mêmes versions, paramètres d’hébergement et limites CPU et mémoire.

Je les exécuterais successivement sur le même environnement, en alternant leur ordre sur plusieurs passages. Le générateur de charge devrait idéalement se trouver sur une autre machine. Une concurrence pour le processeur entre le serveur et le générateur peut brouiller la comparaison.

### Envoyer la même charge offerte

L’exécuteur [constant-arrival-rate de k6](https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/constant-arrival-rate/) permet de programmer les arrivées indépendamment du temps de réponse, tant que le générateur dispose d’assez d’utilisateurs virtuels.

Voici un script pour un jeu d’essai contenant les dossiers 1 à 100, dont la propriété JSON `id` est numérique. Le débit de 200 requêtes par seconde est un paramètre illustratif à adapter à l’environnement.

```javascript
import http from 'k6/http'
import { check } from 'k6'
import execution from 'k6/execution'

const adresse = __ENV.ADRESSE_API || 'http://localhost:5000'

export const options = {
  scenarios: {
    lecture: {
      executor: 'constant-arrival-rate',
      rate: Number(__ENV.DEBIT || 200),
      timeUnit: '1s',
      duration: __ENV.DUREE || '2m',
      preAllocatedVUs: 100,
      maxVUs: 100
    }
  },
  summaryTrendStats: ['avg', 'p(50)', 'p(95)', 'p(99)', 'max'],
  thresholds: {
    checks: ['rate==1'],
    dropped_iterations: ['count==0']
  }
}

export default function () {
  const identifiant = 1 + execution.scenario.iterationInTest % 100
  const reponse = http.get(`${adresse}/dossiers/${identifiant}`)
  let contenu = null

  try {
    contenu = reponse.json()
  } catch {
    contenu = null
  }

  check(reponse, {
    'réponse attendue': x =>
      x.status === 200 && contenu?.id === identifiant
  })
}
```

La [numérotation des itérations de k6](https://grafana.com/docs/k6/latest/javascript-api/k6-execution/) permet ici de parcourir le même jeu de dossiers. Les [checks associés à des seuils](https://grafana.com/docs/k6/latest/using-k6/checks/) font échouer la validation si les réponses ne correspondent pas aux attentes. Le script vise le parcours sans erreur injectée.

Après adaptation de l’authentification et des données au banc d’essai, on peut enregistrer ce contenu dans `charge-dossiers.js`, échauffer chaque variante, puis lancer la mesure.

```bash
k6 run -e DUREE=30s -e DEBIT=200 charge-dossiers.js
k6 run -e DUREE=2m -e DEBIT=200 charge-dossiers.js
```

Les durées sont des points de départ. Il faut vérifier la stabilisation du JIT, des connexions et des caches avant de retenir la fenêtre mesurée. Les résultats de l’échauffement restent séparés.

Si des itérations ne démarrent pas, le débit demandé n’a pas été maintenu. Il faut identifier si la limite vient du serveur ou du générateur avant de comparer les latences. On peut ensuite répéter l’expérience à plusieurs débits pour trouver la capacité qui respecte les objectifs de service.

### Observer aussi le processus serveur

Les mesures côté client doivent être accompagnées des métriques du serveur. Avec [dotnet-counters](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-counters), une collecte peut ressembler à ceci, en remplaçant le PID par celui de l’API.

```bash
dotnet-counters collect --process-id 12345 --counters System.Runtime --format csv --output runtime-async-api.csv
```

Pour chaque variante et chaque profil, je comparerais les mêmes éléments sur une fenêtre stable.

| Mesure | Utilité dans la décision |
| --- | --- |
| Débit réellement traité et taux d’erreurs | Vérifier que les deux variantes rendent le même service |
| Latences p50, p95 et p99 | Observer le parcours courant et les requêtes plus lentes |
| Temps CPU par requête | Évaluer le travail nécessaire à charge équivalente |
| Octets alloués par requête | Relier le gain local au débit d’allocation du service |
| Activité et pauses du GC | Vérifier si la baisse d’allocation produit un effet observable |
| Temps dans les dépendances et taux de succès du cache | Expliquer un écart de comportement entre les passages |

Les octets alloués par requête peuvent être estimés en divisant l’augmentation des allocations du processus par le nombre de requêtes terminées pendant la même fenêtre. Les travaux en arrière-plan doivent être pris en compte dans l’interprétation.

J’éviterais une mesure fondée sur `GC.GetAllocatedBytesForCurrentThread` autour d’un `await` pour caractériser une requête complète. Sa continuation peut reprendre sur un autre thread.

## Relier les résultats du microbenchmark et de l’API

Un ratio de temps très favorable dans BenchmarkDotNet peut devenir un faible écart sur l’API. C’est cohérent lorsque la requête passe l’essentiel de sa durée à attendre une dépendance ou à effectuer un autre travail.

Voici comment j’interpréterais les résultats recueillis.

| Observation | Interprétation à vérifier |
| --- | --- |
| Microbenchmark plus rapide, latence HTTP stable | Le coût optimisé représente probablement une petite part du parcours complet |
| Allocations par requête réduites, CPU réduit à débit égal | Le service peut avoir gagné en efficacité même sans changement visible de latence |
| Amélioration surtout sur le cache préchargé | La terminaison immédiate semble constituer un parcours favorable |
| Gain pendant les erreurs, mais effet faible en fonctionnement normal | La propagation des exceptions contribue probablement au bénéfice |
| Allocations réduites, p99 dégradé | La variante demande une investigation avant adoption |

Ces interprétations restent des hypothèses à confirmer avec les traces et les répétitions. Je retiendrais un gain reproductible, accompagné du même comportement fonctionnel et d’un effet utile sur les objectifs du service.

Une baisse de CPU peut offrir une marge de capacité. Elle ne se traduit en économie d’hébergement que si le dimensionnement ou les règles de mise à l’échelle permettent effectivement de l’exploiter.

## Diagnostic : retrouver plus facilement le code qui nous intéresse

L’amélioration décrite dans la [documentation des piles d’exécution avec Runtime Async](https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-11/runtime#cleaner-live-stack-traces) concerne particulièrement les piles observées pendant l’exécution, par exemple dans un profileur, un débogueur ou avec `StackTrace`. Les méthodes applicatives peuvent apparaître plus directement, avec moins de cadres liés aux machines à états.

Il faut distinguer ces piles des traces d’exception déjà mises en forme par .NET, qui masquent une partie de cette infrastructure. Une différence spectaculaire dans une pile capturée pendant l’exécution ne signifie donc pas que tous les journaux d’exception seront transformés de la même façon.

Le gain opérationnel est de consacrer moins de temps à reconnaître la mécanique asynchrone avant d’identifier le code utile. La corrélation des requêtes, les activités distribuées et les mesures des dépendances restent nécessaires pour comprendre les attentes entre services.

Avant de retenir une variante, je vérifierais aussi les outils réellement utilisés par l’équipe : points d’arrêt, traces, profileur et agent de supervision. Les tests devraient confirmer la propagation des exceptions, l’annulation, les blocs `finally` et le contexte ambiant transporté par `AsyncLocal`.

Cette vérification est d’autant plus pertinente que Runtime Async conserve son statut de préversion pour le code applicatif. Une régression de diagnostic peut coûter davantage à l’exploitation qu’un petit gain dans un microbenchmark.

## Conclusion

Runtime Async donne au runtime une meilleure compréhension des enchaînements asynchrones. Il peut ainsi réduire des tâches intermédiaires, alléger la propagation des exceptions et rendre certaines piles d’exécution plus lisibles, tout en conservant la syntaxe `async` et `await`.

Les gains dépendent du parcours. Les méthodes qui terminent immédiatement, les chaînes comportant plusieurs niveaux et certains scénarios d’exception méritent une attention particulière. Les résultats publiés montrent également que moins allouer ne signifie pas toujours aller plus vite.

Je commencerais par une comparaison contrôlée sous le même .NET 11, puis je chercherais son effet sur une API représentative. Une amélioration reproductible du CPU, des allocations ou des latences, avec un comportement fonctionnel et un diagnostic fiables, fournit une raison concrète d’aller plus loin.
