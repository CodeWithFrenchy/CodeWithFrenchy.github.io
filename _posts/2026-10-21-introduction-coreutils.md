---
title: "Coreutils pour Windows : retrouver ses outils Unix dans son terminal"
date: 2026-10-21 19:00:00 -0400
categories: [outil-developpement, windows]
tags: [coreutils, powershell, rust, devex]
---

## Préambule

Quand on développe avec .NET, on peut facilement passer d’un poste Windows à un conteneur Linux, puis à un pipeline de déploiement. Les outils applicatifs restent familiers, mais les commandes utilisées pour explorer des fichiers ou analyser un journal varient selon l’environnement.

Pour une intervention ponctuelle, retrouver un `tail`, un `grep` ou un `sha256sum` directement dans son terminal Windows peut simplifier le travail.

C’est l’intérêt du projet [Coreutils pour Windows](https://github.com/microsoft/coreutils), maintenu par Microsoft. Il propose des utilitaires de style Unix exécutés nativement sous Windows.

Ce qui m’intéresse ici, c’est la possibilité de conserver des habitudes utiles entre plusieurs environnements, tout en gardant une maîtrise claire des commandes exécutées.

## Ce que contient le projet

Le [guide de contribution](https://github.com/microsoft/coreutils/blob/main/CONTRIBUTING.md) décrit trois composants principaux.

| Composant | Rôle |
| --- | --- |
| `uutils/coreutils` | Réimplémentation en Rust des utilitaires GNU Coreutils, avec des commandes comme `ls`, `cp`, `head` et `tail` |
| `uutils/findutils` | Outils `find` et `xargs` |
| `uutils/grep` | Recherche de texte avec `grep` et ses variantes |

Le dépôt Microsoft rassemble ces composants et prend en charge les adaptations Windows, l’installateur et un exécutable capable de distribuer les appels vers différents utilitaires.

Cette distinction donne aussi un repère pour contribuer. Un problème de comportement dans une commande peut relever du projet uutils correspondant, alors qu’un problème d’installation ou d’intégration Windows peut relever du dépôt Microsoft.

Le projet [uutils/coreutils](https://github.com/uutils/coreutils) vise la compatibilité avec GNU Coreutils. Sa documentation précise cependant que certaines options peuvent manquer ou présenter des différences. L’objectif de compatibilité doit donc être accompagné d’une validation des usages qui comptent pour l’équipe.

## Installer Coreutils pour Windows

Au 21 octobre 2026, le [README](https://github.com/microsoft/coreutils#coreutils-for-windows) présente toujours le projet comme une préversion. La [dernière version publiée](https://github.com/microsoft/coreutils/releases/tag/v2026.9.3) est `2026.9.3`.

L’installation proposée passe par WinGet.

```powershell
winget install Microsoft.Coreutils
```

L’installateur permet notamment de gérer l’ajout au `PATH`, comme le montre sa [configuration](https://github.com/microsoft/coreutils/blob/main/coreutils.iss). Après l’installation, ouvrez un nouveau terminal et vérifiez les exécutables disponibles.

```powershell
Get-Command ls.exe, tail.exe, grep.exe, sha256sum.exe -All |
    Select-Object Name, Source
```

La colonne `Source` permet de repérer leur emplacement. C’est particulièrement utile sur un poste qui possède déjà d’autres distributions de ces outils.

Pour PowerShell, le projet demande **la version 7.4 ou ultérieure** et recommande **7.6 ou ultérieure** pour la prise en charge de `~`. La version de la session se vérifie ainsi.

```powershell
$PSVersionTable.PSVersion
```

## Bien comprendre la résolution des commandes dans PowerShell

La [documentation PowerShell sur la priorité des commandes](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_command_precedence?view=powershell-7.6) rappelle qu’un alias est normalement prioritaire sur une fonction, une cmdlet ou un exécutable.

Un nom familier comme `ls` peut donc désigner l’alias de `Get-ChildItem`.

```powershell
Get-Command ls -All
Get-Command ls.exe -All
```

Le projet ajoute une subtilité. Son [intégration à PowerShell](https://github.com/microsoft/coreutils#powershell-command-parsing) réécrit les saisies interactives avec PSReadLine, sans supprimer les alias. `Get-Command ls` peut donc encore montrer un alias. Le comportement interactif ne constitue pas une garantie pour un script `.ps1`.

Pour les exemples suivants, j’utilise explicitement les noms terminés par `.exe`.

```powershell
ls.exe -lah .
```

Cela contourne les alias habituels comme `ls`, mais plusieurs exécutables peuvent encore porter le même nom dans le `PATH`. Dans une automatisation, je privilégierais un chemin explicite vers la distribution retenue.

Par exemple, si l’installation se trouve à cet emplacement.

```powershell
$repertoireCoreutils = 'C:\Program Files\coreutils\bin'

& "$repertoireCoreutils\ls.exe" -lah .
```

Le chemin est à adapter à l’installation réelle. Cette précision rend le choix de l’outil visible dans le script.

## Trois usages concrets dans un environnement .NET

Les exemples ci-dessous ciblent PowerShell 7.4 ou ultérieur. Ils supposent que les exécutables du package Microsoft sont accessibles par le `PATH` et que les fichiers indiqués existent.

### Consulter un journal applicatif

Pour afficher les 50 dernières lignes d’un journal écrit par une application.

```powershell
tail.exe -n 50 '.\journaux\application.log'
```

Pour suivre les nouvelles lignes pendant une reproduction locale.

```powershell
tail.exe -f '.\journaux\application.log'
```

Les options `-n` et `-f` sont décrites dans la [documentation de `tail`](https://uutils.org/coreutils/docs/utils/tail.html). La première choisit le nombre de lignes, la seconde suit l’évolution du fichier.

J’y vois un bon complément pour examiner rapidement un fichier local pendant un diagnostic. Pour comprendre un incident distribué, les traces, les métriques et la corrélation des requêtes conservent évidemment leur place.

### Filtrer les dernières lignes pertinentes

On peut ensuite composer deux commandes pour chercher le mot `erreur` dans les 200 dernières lignes du journal.

```powershell
tail.exe -n 200 '.\journaux\application.log' |
    grep.exe -i 'erreur'
```

L’option `-i` ignore les différences entre majuscules et minuscules. Ce comportement fait partie de l’interface décrite dans le [manuel GNU Grep](https://www.gnu.org/s/grep/manual/grep.html), que le projet [uutils/grep](https://github.com/uutils/grep) réimplémente en Rust.

Ici, l’ordre des commandes compte. On limite d’abord la lecture aux dernières lignes, puis on filtre. Une recherche dans l’ensemble du journal répondrait à un autre besoin.

Pour intégrer cette recherche à un pipeline CI, il faut aussi définir ce que signifie l’absence de résultat. Le manuel GNU distingue un code de sortie `1`, qui indique qu’aucune ligne ne correspond, d’une erreur d’exécution. Cette convention mérite une vérification avec la version utilisée et le traitement des erreurs du pipeline.

### Calculer et vérifier l’empreinte d’un artefact

Pour un fichier ZIP produit à partir d’une publication .NET, on peut calculer son empreinte et la conserver dans un fichier.

```powershell
sha256sum.exe --binary '.\artefacts\portail.zip' > '.\portail.sha256'
```

La vérification se fait ensuite avec `--check`, depuis le même répertoire de travail puisque le manifeste contient un chemin relatif.

```powershell
sha256sum.exe --check '.\portail.sha256'
```

La [documentation de `sha256sum`](https://uutils.org/coreutils/docs/utils/sha256sum.html) présente ces deux modes. Cet usage peut servir à comparer un artefact à une empreinte de référence connue.

Le fichier d’empreinte doit lui-même provenir d’une source de confiance. S’il est remplacé avec l’artefact, la comparaison seule ne permet pas d’établir l’authenticité de celui-ci.

## Choisir entre Coreutils, PowerShell et WSL

Je choisirais surtout selon la nature du travail.

| Besoin | Choix que je privilégierais |
| --- | --- |
| Explorer des fichiers et filtrer du texte avec des commandes Unix familières | Coreutils pour Windows |
| Manipuler des objets, leurs propriétés et des modules d’administration | PowerShell |
| Exécuter des scripts Bash et des applications Linux dans leur environnement | WSL |

PowerShell conserve notamment l’intérêt de ses [pipelines d’objets](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pipelines?view=powershell-7.6). Lorsqu’une commande retourne déjà des données structurées, travailler avec leurs propriétés évite d’avoir à analyser leur présentation textuelle.

La même documentation explique que PowerShell 7.4 préserve les flux d’octets entre commandes natives et lors de leur redirection vers un fichier. Cette propriété ne s’étend pas automatiquement à un pipeline qui introduit des transformations PowerShell.

[WSL](https://learn.microsoft.com/en-us/windows/wsl/about), de son côté, fournit un environnement Linux avec ses distributions, ses applications et ses outils. Je le conserverais pour les scénarios qui dépendent de cet environnement, notamment la validation de scripts destinés à Linux.

## Conclusion

Coreutils pour Windows est un projet intéressant pour les développeurs qui travaillent régulièrement entre Windows et Linux. Les usages les plus convaincants sont souvent simples, comme consulter un journal, filtrer du texte ou vérifier un artefact avec des commandes déjà connues.

Pour une équipe .NET, je l’envisagerais comme un complément à l’environnement de développement. Son adoption devient plus solide lorsque la version, le shell et les chemins des exécutables sont explicites.

Le gain recherché est concret. Réduire les petites frictions du quotidien, tout en conservant des scripts dont le comportement reste compréhensible et reproductible.
