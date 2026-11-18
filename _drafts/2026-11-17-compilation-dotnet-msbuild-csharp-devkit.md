---
title: "Compilation .NET : où gagner du temps avec MSBuild et C# Dev Kit"
date: 2026-11-17 19:00:00 -0400
categories: [dotnet, outillage]
tags: [dotnet, dotnet11, msbuild, csharp-dev-kit, vscode, performance, devops, ci-cd]
---

>  ⚠️ IMAGE

## Préambule

On modifie quelques lignes, on lance un test et on attend. Parfois, le test lui-même dure quelques millisecondes, alors que la préparation de son exécution prend plusieurs secondes. Sur une grande solution, ce temps se répète assez souvent pour devenir une friction importante dans la journée.

À la .NET Conf 2026, deux sessions examinaient ce problème sous des angles complémentaires : [More Processes, More Problems, sur MSBuild multithread](https://www.youtube.com/watch?v=9WDTK828qtg) et [C# Dev Kit 11: Faster Foundations for .NET Development](https://www.youtube.com/watch?v=s7p7nuSvY3s).

Leurs améliorations touchent plusieurs étapes : préparer les informations du projet, répartir le travail, déterminer ce qui doit être reconstruit et propager les fichiers produits. Pour en tirer parti, il faut savoir quelle étape nous fait réellement attendre.

Après mon [tour d’horizon de .NET 11](https://codewithfrenchy.com/posts/dotnet11/), je propose donc de regarder ces mécanismes plus précisément, puis de mesurer séparément leur effet sur le poste de développement et sur les pipelines. Les chiffres présentés proviennent des démonstrations et publications citées. La méthode de mesure proposée reste à exécuter sur votre solution.

## Identifier le temps que l’on veut réduire

Dans les échanges quotidiens, on utilise souvent « compilation » pour désigner toute la commande de build. Pourtant, le compilateur C# n’en représente qu’une partie.

| Étape | Travail réalisé | Symptôme possible |
| --- | --- | --- |
| Restauration | Résoudre les dépendances et préparer les informations NuGet | Attente variable selon le réseau et les caches |
| Évaluation et préparation | Lire les projets, les imports, les propriétés et les références | Une solution lente à ouvrir ou à préparer |
| Exécution du build | Planifier les projets, parcourir les cibles et lancer les tâches nécessaires | Un build coûteux même sans changement |
| Compilation et génération | Compiler les sources, exécuter les analyseurs et générateurs concernés | Un projet précis domine la durée |
| Propagation des sorties | Copier les assemblies, ressources et fichiers de contenu | Beaucoup de travail après une petite modification |

Les outils de développement ont aussi besoin des informations du projet sans nécessairement produire un exécutable. IntelliSense doit connaître les fichiers, les références, les symboles conditionnels et les paramètres du compilateur. Ces opérations expliquent une partie du travail effectué à l’ouverture d’une solution.

La première mesure devrait donc distinguer au moins trois délais : le temps avant de pouvoir travailler dans l’éditeur, le temps de build et le temps total avant d’obtenir le résultat d’un test.

## MSBuild multithread : réduire le coût de coordination

### Le parallélisme existe déjà

MSBuild sait depuis longtemps construire plusieurs projets en parallèle. Dans son modèle habituel, les nœuds de travail peuvent être répartis dans plusieurs processus. Le paramètre `-m`, ou `-maxCpuCount`, permet de contrôler leur concurrence.

Chaque processus doit charger les composants nécessaires et échanger des informations avec les autres nœuds. Sur une solution contenant beaucoup de projets et de frameworks cibles, cette coordination peut devenir une part visible de la durée du build.

Le mode multithread regroupe les nœuds du moteur dans un même processus, avec des threads de travail. Il réduit ainsi certains chargements répétés et remplace une partie des communications entre processus par des échanges internes.

Cette approche est particulièrement intéressante lorsqu’il y a peu de travail utile à effectuer. Lors d’un build sans changement, le moteur doit encore déterminer que les sorties sont à jour. Accélérer cette vérification peut améliorer la boucle quotidienne même si aucune compilation C# n’est nécessaire.

### Activer le mode pour l’évaluer

La [documentation de MSBuild multithread](https://learn.microsoft.com/en-us/visualstudio/msbuild/update-task-multithreaded) présente le paramètre `-mt`. Elle classe encore ce mode comme expérimental et ne le recommande pas pour les builds de production à la date de cette rédaction.

Avec un SDK .NET 11 compatible, on peut comparer les deux configurations en conservant la même limite de concurrence.

```bash
dotnet build Solution.slnx -c Debug --no-restore -m:4 -mt:false
```

```bash
dotnet build Solution.slnx -c Debug --no-restore -m:4 -mt:true
```

Ces commandes supposent une restauration déjà effectuée. Le nombre quatre est un point de départ pour l’expérience, à adapter aux ressources disponibles.

Les options répondent à deux questions différentes. `-m:4` fixe une limite de concurrence du moteur. `-mt:true` change l’organisation des nœuds. Aucun des deux ne garantit que quatre projets auront toujours du travail exécutable en parallèle. Le graphe de dépendances impose aussi des contraintes.

Dans la session, l’équipe montrait une réduction d’environ un quart du temps de build à froid sur Windows et de plus de 10 % sur Linux pour Orchard Core. Ce sont des résultats du scénario présenté, avec sa solution, ses configurations et ses machines. Ils donnent une raison d’essayer le mode sur son propre dépôt, sans fournir un pourcentage applicable à tous les projets.

### Le serveur participe aussi au résultat

Les [notes MSBuild de .NET 11 Preview 6](https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/preview6/msbuild.md) décrivent l’activation automatique du serveur MSBuild avec `-mt`. Ce processus durable permet de réutiliser de l’état entre les invocations.

Une comparaison entre les deux configurations peut donc refléter à la fois la nouvelle organisation du moteur et la réutilisation du serveur. Attribuer tout l’écart au seul passage des processus aux threads serait trop précis par rapport à ce qui est mesuré.

Sur un poste de développement, les builds successifs peuvent profiter de cet état conservé. Sur un agent CI neuf qui effectue une seule compilation, une partie de cet avantage ne peut pas être amortie de la même façon.

### Les tâches personnalisées restent importantes

Partager un processus implique aussi de partager son état. Une tâche qui modifie le répertoire courant, les variables d’environnement ou un champ statique peut interférer avec une autre tâche.

Le contrat présenté par Microsoft utilise l’attribut `MSBuildMultiThreadableTask` pour déclarer une tâche compatible. L’interface `IMultiThreadableTask` permet de recevoir un `TaskEnvironment` pour les opérations qui demandent un contexte propre à la tâche. Ajouter l’attribut exige donc d’avoir vérifié le comportement concurrent du code.

Les tâches qui ne sont pas adaptées peuvent être isolées dans des processus dédiés. Ce mécanisme préserve la compatibilité, mais conserve une partie du coût des communications et du démarrage de processus.

Je commencerais par inventorier les tâches internes et les paquets qui ajoutent des étapes au build. Une solution chargée de génération, de packaging ou d’intégrations historiques peut présenter un profil différent d’un ensemble de bibliothèques C#.

## C# Dev Kit : réutiliser les informations du projet

La session C# Dev Kit décrit une autre source de travail répété. Plusieurs outils ont besoin des mêmes informations : fichiers sources, références, frameworks cibles et options de compilation. Recalculer ces informations à chaque ouverture ou opération consomme du temps et de la mémoire.

La nouvelle architecture peut enregistrer ces informations dans des caches de projets, appelés `LSCache` dans la présentation, puis les relire rapidement. Elle évite ainsi de reconstruire systématiquement le modèle du projet pour rendre les services de langage disponibles.

Le gain concerne notamment la réouverture d’une solution et les interactions entre les outils qui produisent et consomment ces caches. Il faut distinguer le premier chargement, qui doit préparer les informations, des chargements suivants.

### Un cache de projet n’est pas un cache de binaires

Les mécanismes de réutilisation ne stockent pas tous la même chose.

| Mécanisme | Ce qui est réutilisé | Limite à garder en tête |
| --- | --- | --- |
| Cache NuGet | Les paquets téléchargés | Le projet doit encore disposer de ses informations de restauration |
| Cache de projet pour l’outillage | Les informations nécessaires pour comprendre le projet | Les changements de configuration doivent être pris en compte |
| État du serveur de build | Des composants chargés et de l’état conservé en mémoire | Il dépend de la durée de vie du processus |
| Sorties incrémentales | Des résultats déjà produits et encore valides | Leur validité dépend des entrées et des règles du build |

Copier les paquets NuGet sur un nouvel agent ne lui donne donc pas automatiquement le même état qu’un poste ayant déjà chargé et compilé la solution.

Pour évaluer les caches de projets, je vérifierais aussi le comportement après une modification de `.csproj`, de `Directory.Build.props`, de `Directory.Packages.props` ou du SDK sélectionné. Le résultat doit devenir disponible rapidement tout en reflétant la nouvelle configuration.

### Les chiffres doivent conserver leur périmètre

Dans [A faster, lighter C# Dev Kit](https://devblogs.microsoft.com/dotnet/faster-lighter-csharp-dev-kit/), Microsoft publie notamment ces résultats pour la solution Aspire de 407 projets.

| Mesure publiée | C# Dev Kit 3.3 | C# Dev Kit 11.0 |
| --- | ---: | ---: |
| Solution entièrement chargée | 84,1 s | 3,0 s |
| Build sans modification | 34,4 s | 0,88 s |
| Build après modification d’un fichier C# | 36,1 s | 2,1 s |
| Mémoire des processus du serveur Dev Kit | 2 072 Mo | 316 Mo |

La refonte regroupe aussi plusieurs services dans un serveur compilé avec Native AOT. Les chiffres agrègent donc plusieurs changements d’architecture.

La mesure mémoire exclut le processus Roslyn et VS Code. Elle ne représente donc pas toute la mémoire utilisée par l’environnement de développement.

L’article du 6 octobre 2026 présentait la version 11.0 sur le canal de préversion. Pour reproduire une comparaison, je noterais les versions exactes de VS Code, de C# Dev Kit et de l’extension C#, ainsi que leur canal. Le numéro du SDK ne suffit pas à identifier cette configuration.

## Les builds incrémentaux : éviter le travail devenu inutile

Les [compilations incrémentales de MSBuild](https://learn.microsoft.com/en-us/visualstudio/msbuild/incremental-builds) s’appuient notamment sur les entrées et les sorties des cibles. Lorsque les sorties sont à jour par rapport aux entrées, une cible peut être ignorée.

Une cible personnalisée qui réécrit un fichier généré à chaque invocation peut invalider ce mécanisme, même si le contenu logique ne change pas. Un générateur qui ignore ses dépendances réelles peut, à l’inverse, conserver une sortie devenue incorrecte.

Je regarderais donc les cibles qui déclarent mal leurs `Inputs` et `Outputs`, les fichiers réécrits inutilement et les tâches qui s’exécutent systématiquement. Une amélioration à ce niveau peut bénéficier au poste de développement comme au pipeline.

### BuildX : ignorer, copier ou reconstruire

Dans la session C# Dev Kit, le nom BuildX désigne le mécanisme présenté pour accélérer ces décisions. Selon les changements observés, un projet peut être ignoré, recevoir des copies de fichiers ou nécessiter un build.

Prenons un service qui dépend d’une bibliothèque. Une modification interne à cette bibliothèque peut laisser son API publique inchangée. Les projets consommateurs n’ont alors pas nécessairement besoin d’être recompilés, mais ils doivent recevoir la nouvelle version de la bibliothèque à l’exécution.

Les assemblies de référence et les mécanismes incrémentaux permettent déjà d’éviter certaines recompilations. L’accélération décrite réduit aussi le travail nécessaire pour prendre cette décision et effectuer les copies.

La [documentation de Build Acceleration dans Visual Studio](https://github.com/dotnet/project-system/blob/main/docs/build-acceleration.md) explique cette distinction : l’environnement peut copier directement les sorties lorsque l’appel à MSBuild ne servirait qu’à réaliser ces copies. Le fonctionnement présenté dans C# Dev Kit suit cette logique.

Cela explique pourquoi un gain spectaculaire sur un build lancé par l’éditeur ne doit pas être automatiquement attribué à une commande `dotnet build` exécutée dans un terminal ou un pipeline. Lors de la conférence, l’extension de ces mécanismes à d’autres outils était présentée comme une direction de travail.

## Une méthode de mesure reproductible

### Fixer les conditions de comparaison

Je commencerais avec un commit identifié, un répertoire d’essai isolé et un SDK précis. Le fichier [global.json](https://learn.microsoft.com/en-us/dotnet/core/tools/global-json) permet de fixer ce dernier, indépendamment du framework ciblé par l’application.

```json
{
  "sdk": {
    "version": "11.0.100",
    "rollForward": "disable"
  }
}
```

La version doit correspondre au SDK installé et retenu pour l’expérience. Il faut aussi noter le système d’exploitation, les ressources CPU et mémoire, le stockage, les frameworks cibles et la configuration Debug ou Release.

Les propriétés et fichiers de réponse MSBuild existants peuvent influencer les commandes. Je conserverais donc également la configuration du dépôt et les variables d’environnement pertinentes dans le compte rendu.

### Définir les scénarios avant de chronométrer

| Scénario | État de départ | Question évaluée |
| --- | --- | --- |
| Build complet | Sorties absentes, restauration effectuée, serveurs arrêtés | Combien coûte une construction complète avec un moteur non échauffé ? |
| Build sans changement | Sorties à jour, serveurs échauffés | Combien coûte la vérification qu’il n’y a rien à reconstruire ? |
| Modification interne | Build initial terminé, changement d’implémentation précis | Quel est le délai de la boucle courante ? |
| Modification d’API publique | Build initial terminé, ajout public qui conserve une solution compilable | Comment le changement se propage-t-il aux consommateurs ? |

« À froid » doit être défini. Arrêter les serveurs de build ne vide ni le cache de fichiers du système d’exploitation ni le cache NuGet. Pour le premier scénario, je garderais les paquets disponibles afin de mesurer le build séparément du téléchargement.

La commande [dotnet build-server shutdown](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-build-server) permet d’arrêter les serveurs associés aux builds. Elle convient à un environnement d’essai isolé, où les autres outils ne vont pas les relancer pendant l’expérience.

### Automatiser le cas sans changement

Le script suivant, destiné à PowerShell 7, mesure un build sans modification après trois passages d’échauffement. Il alterne l’ordre des modes et exporte les durées en CSV dans un répertoire temporaire extérieur au dépôt. Il suppose que le dépôt est déjà positionné sur le commit choisi et que les tâches de build ne modifient pas les sources suivies.

```powershell
param(
    [string]$Solution = "Solution.slnx",
    [ValidateRange(2, 100)]
    [int]$Repetitions = 8,
    [ValidateRange(1, 256)]
    [int]$Noeuds = 4
)

$ErrorActionPreference = "Stop"
$DossierMesures = Join-Path ([System.IO.Path]::GetTempPath()) ("mesures-build-" + [guid]::NewGuid().ToString("N"))
New-Item -ItemType Directory -Path $DossierMesures -Force | Out-Null

& dotnet --info > (Join-Path $DossierMesures "dotnet-info.txt")
& dotnet restore $Solution
if ($LASTEXITCODE -ne 0) {
    throw "La restauration a échoué"
}

$ArgumentsBuild = @(
    "build"
    $Solution
    "-c"
    "Debug"
    "--no-restore"
    "-m:$Noeuds"
    "-v:quiet"
    "-tl:off"
)

$Resultats = @(
    foreach ($Repetition in 1..$Repetitions) {
        $Modes = if ($Repetition % 2 -eq 1) {
            @("false", "true")
        } else {
            @("true", "false")
        }

        foreach ($Mode in $Modes) {
            & dotnet build-server shutdown | Out-Null
            if ($LASTEXITCODE -ne 0) {
                throw "L’arrêt des serveurs a échoué"
            }

            foreach ($Passage in 1..3) {
                $Journal = Join-Path $DossierMesures "chauffe-$Repetition-$Mode-$Passage.log"
                & dotnet @ArgumentsBuild "-mt:$Mode" *> $Journal
                if ($LASTEXITCODE -ne 0) {
                    throw "Un build d’échauffement a échoué"
                }
            }

            $Journal = Join-Path $DossierMesures "mesure-$Repetition-$Mode.log"
            $Chronometre = [System.Diagnostics.Stopwatch]::StartNew()
            & dotnet @ArgumentsBuild "-mt:$Mode" *> $Journal
            $CodeSortie = $LASTEXITCODE
            $Chronometre.Stop()

            if ($CodeSortie -ne 0) {
                throw "Le build mesuré a échoué"
            }

            [pscustomobject]@{
                Repetition = $Repetition
                Multithread = $Mode
                Noeuds = $Noeuds
                DureeMs = [math]::Round($Chronometre.Elapsed.TotalMilliseconds, 2)
            }
        }
    }
)

$Resultats | Export-Csv (Join-Path $DossierMesures "resultats.csv") -NoTypeInformation -Encoding utf8
Write-Host "Résultats : $DossierMesures"
```

La mesure inclut le lancement de la commande et son retour au terminal. C’est un délai perceptible dans la boucle de travail. Elle exclut la restauration, l’arrêt des serveurs et les passages d’échauffement.

Huit répétitions constituent un point de départ. Je comparerais la médiane et la dispersion des deux groupes, puis j’augmenterais le nombre de passages si l’écart reste proche de la variabilité observée.

Pour mesurer une modification interne, il faut adapter la préparation de chaque passage : reconstruire l’état de référence, appliquer le même patch, puis chronométrer une seule compilation. Ensuite, annuler ce patch et reconstruire l’état initial hors mesure. Répéter simplement la commande après la première compilation mesurerait surtout des builds sans changement.

Le scénario d’API publique suit la même démarche avec un autre patch. Le build complet demande, lui, de restaurer un état sans sorties avant chaque observation. Un répertoire d’essai fraîchement préparé permet de mieux contrôler cet état qu’un nettoyage dont les effets seraient supposés.

### Expliquer les écarts avec les journaux

Une durée plus faible indique un résultat. Les [journaux binaires MSBuild](https://learn.microsoft.com/en-us/visualstudio/msbuild/obtaining-build-logs-with-msbuild) aident à comprendre le travail qui a changé.

```bash
dotnet build Solution.slnx -c Debug --no-restore -m:4 -mt:false -bl:../classique.binlog
```

```bash
dotnet build Solution.slnx -c Debug --no-restore -m:4 -mt:true -bl:../multithread.binlog
```

Je recueillerais ces journaux lors de passages de diagnostic séparés, en reconstituant le même scénario avant chaque commande. La campagne chronométrée conserverait une instrumentation identique dans les deux modes.

L’analyse devrait identifier les projets et cibles exécutés, les compilations réellement lancées, les copies et les tâches qui dominent le parcours. Les durées de tâches parallèles ne s’additionnent pas directement pour expliquer le temps total. Le chemin critique et les périodes d’attente comptent aussi.

La mémoire doit couvrir l’arbre de processus pertinent. Observer seulement le processus principal peut masquer le coût des processus de tâches, du serveur de compilation ou de l’éditeur.

## Mesurer C# Dev Kit séparément

Le script précédent mesure la ligne de commande. Pour C# Dev Kit, je reproduirais les scénarios depuis l’interface qui utilise effectivement ses mécanismes d’accélération.

Je noterais le délai avant que le fichier actif dispose de ses services de langage, puis celui nécessaire au chargement de toute la solution. Ces deux moments répondent à des besoins différents.

Ensuite, je mesurerais un build sans changement, un build après le patch interne et le délai total pour lancer un test représentatif. Les mêmes extensions, le même fichier actif et le même état de cache doivent être utilisés entre les passages.

Une première ouverture sans cache disponible et une réouverture avec cache valide doivent rester deux résultats distincts. Si des caches sont déjà présents dans le dépôt ou réutilisés entre worktrees, il faut le noter plutôt que qualifier automatiquement un nouveau répertoire de chargement à froid.

## Le poste de développement et le pipeline ne cherchent pas toujours le même gain

### Sur le poste de développement

Le gain le plus utile peut être une boucle plus courte : modifier, vérifier, reprendre le travail. Une petite économie répétée plusieurs dizaines de fois peut compter davantage qu’une amélioration du build complet rarement exécuté.

Je privilégierais donc le temps avant le premier résultat utile, la mémoire occupée lorsque plusieurs outils sont ouverts et la réactivité pendant un build.

La concurrence demande aussi une limite partagée. Quatre agents qui lancent chacun un build avec `-m:4` ne disposent pas automatiquement d’un budget global de quatre nœuds. Ils peuvent se concurrencer pour le CPU, la mémoire et les accès disque.

La session C# Dev Kit illustrait précisément ce problème. Dans l’expérience présentée, réduire la concurrence excessive améliorait même le débit global des builds. Le nombre de tâches terminées par minute et la réactivité du poste sont donc plus utiles que le seul nombre de builds lancés simultanément.

### Dans les pipelines

Sur un agent éphémère, les paquets restaurés, les sorties de build et les processus persistants n’ont pas la même durée de vie que sur un poste de développement. Je mesurerais séparément le checkout, la restauration, le build, les tests et la publication.

Dans Azure Pipelines, la [mise en cache des paquets NuGet](https://learn.microsoft.com/en-us/azure/devops/pipelines/artifacts/caching-nuget?view=azure-devops) peut réduire les téléchargements. Elle ne suffit pas à produire le fichier `project.assets.json` d’un workspace neuf. Une restauration demeure nécessaire pour préparer le projet, même lorsque les paquets sont déjà disponibles.

La [commande dotnet build](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-build) effectue normalement une restauration implicite. Séparer les étapes rend leur coût plus lisible.

```bash
dotnet restore Solution.slnx
dotnet build Solution.slnx -c Release --no-restore -m:4
```

Les étapes suivantes peuvent réutiliser les sorties lorsque leurs options le permettent et que la configuration, le framework cible et les autres propriétés correspondent. Désactiver un build sans disposer des sorties attendues produit un pipeline incorrect.

Je ne traiterais pas un archivage général de `bin` et `obj` comme un cache universel. La validité de ces sorties dépend du code, des dépendances, du SDK, des chemins et des paramètres de compilation. Le temps de téléchargement et d’enregistrement du cache doit aussi entrer dans le bilan.

Pour MSBuild multithread, un pipeline d’essai permet de reprendre la comparaison avec les ressources réelles de l’agent. La validation doit couvrir les sorties attendues, les tests, le packaging et la publication utilisés par le projet. Le statut expérimental du mode reste une contrainte distincte du résultat de performance obtenu.

## Conclusion

MSBuild multithread, les caches de projets et les builds incrémentaux agissent sur des coûts différents. Le premier réduit une partie de la coordination du moteur. Les caches évitent de recalculer des informations utiles à l’outillage. Les mécanismes incrémentaux permettent de conserver le travail déjà valide et d’accélérer sa propagation.

Pour une équipe, le meilleur point de départ est un scénario fréquent et mesurable : rouvrir la solution, relancer un test après une petite modification ou construire un commit sur un agent neuf.

Je conserverais les gains qui se répètent dans ce scénario, avec des sorties correctes et une consommation de ressources acceptable. C’est cette mesure qui permet de décider si l’effort doit porter sur le moteur, le cache, une cible personnalisée ou l’organisation du pipeline.
