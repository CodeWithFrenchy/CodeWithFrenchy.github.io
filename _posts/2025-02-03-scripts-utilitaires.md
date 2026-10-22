---
title: Scripts utilitaires
date: 2025-02-03 20:00:00 -0400
categories: [outil-developpement, automatisation]
tags: [powershell, windows, git, robocopy, scripts, productivite, maintenance]
---

Les scripts suivants sont conçus pour simplifier et automatiser diverses tâches courantes liées à l'administration d'une machine de développement.

Ils couvrent un éventail d'opérations allant de la gestion de fichiers à la maintenance de dépôts Git, en passant par l'optimisation des performances de copie et le nettoyage de répertoires volumineux.

Ces outils sont particulièrement utiles pour gérer des environnements de développement complexes et maintenir une machine propre et performante.

## Créer un fichier de taille exacte

Ce script permet de créer un fichier d'une taille exacte, ce qui peut être utile pour tester des systèmes de fichiers ou des transferts de données.

Voici un exemple pour créer un fichier texte d'un 1 Mo :

```ps1
$file = New-Object -TypeName System.IO.FileStream -ArgumentList "D:\TestFile.txt", "Create", "ReadWrite"
$file.SetLength(1Mb)
$file.Close()
```

Ce fichier peut être utilisé pour des tests de performance ou pour vérifier le comportement de systèmes de fichiers avec des fichiers de taille spécifique. Notez toutefois que ce fichier n'aura pas de contenu valide ou de structure particulière, ce qui peut le rendre inapproprié pour certaines applications nécessitant des données significatives.

## Copier efficacement avec Robocopy

[Robocopy](https://learn.microsoft.com/fr-fr/windows-server/administration/windows-commands/robocopy), un outil intégré à Windows, est souvent plus performant que l'Explorateur de fichiers pour copier des fichiers, surtout lorsqu'il s'agit de gros volumes de données. L'Explorateur de fichiers Windows ne tire pas pleinement parti de tous les cœurs du processeur, ce qui peut le rendre moins efficace pour les transferts de fichiers volumineux ou complexes. De plus, Robocopy utilise les ressources de manière plus optimale, offrant ainsi des vitesses de copie améliorées et une gestion des erreurs plus robuste, ce qui le rend particulièrement adapté pour les tâches de grande envergure.

Voici un exemple d'utilisation :

```ps1
robocopy "C:\repertoireSource" "D:\repertoireCible" /E /r:0 /w:0
```

Dans cet exemple :

- **/E** - copie les sous-répertoires, y compris les répertoires vides.
- **/r:0** - fixe le nombre de tentatives de reprise à zéro, évitant ainsi les retards dus aux erreurs.
- **/w:0** - définit le temps d'attente entre les tentatives à zéro, ce qui accélère encore le processus.

## Supprimer un répertoire avec un chemin trop long

L'Explorateur de fichiers Windows peut rencontrer des problèmes lorsqu'il s'agit de supprimer des répertoires dont le chemin est trop long. En effet, Windows a une **limite de 260 caractères** pour les chemins de fichiers, ce qui peut rendre la suppression de tels répertoires compliquée directement depuis l'interface graphique.

Pour contourner cette limitation, vous pouvez utiliser le script suivant avec Robocopy, qui permet de supprimer le contenu d'un répertoire même si le chemin est trop long :

```ps1
robocopy "C:\RepertoireVide" "C:\RepertoireASupprimer" /purge
```

Cela peut être particulièrement pratique pour effectuer le ménage des répertoires node_modules, qui ont souvent des chemins de fichiers très longs en raison de la structure de leurs dépendances.

## Nettoyer vos référentiels Git de vos branches fusionnées

Ce script automatise le nettoyage des branches fusionnées dans plusieurs référentiels Git. Il parcourt tous les répertoires du dossier courant, identifie ceux contenant un dépôt Git, synchronise les branches distantes, puis supprime les branches locales fusionnées, tout en préservant les branches principales comme **master** et **main**. C’est un outil pratique pour maintenir des référentiels propres et éviter l'encombrement avec des branches inutilisées.

Exemple :

```ps1
.\cleanbranch.ps1
```

## Supprimer les répertoires consommant beaucoup d'espace

Il est fréquent que certains répertoires tels que `TestResults`, `StrykerOutput`, `node_modules`, `bin`, et `obj` occupent une grande quantité d'espace disque. Il est donc utile d'effectuer un nettoyage régulier pour libérer de l'espace.

Par défaut, le script cible ces répertoires qui sont connus pour consommer beaucoup d'espace. Vous pouvez soit confirmer la suppression de ces répertoires par défaut, soit sélectionner manuellement ceux que vous souhaitez supprimer. Une fois les répertoires choisis, le script s'occupe de les supprimer ainsi que tout leur contenu, en explorant également les sous-répertoires de manière récursive.

Exemple :

```ps1
.\cleanFolder.ps1
```

## Mettre à jour des dépôts Git

Ce script met à jour tous les dépôts Git dans un répertoire en exécutant `git pull` pour chacun. Il liste les sous-dossiers, affiche un message en vert pour chaque mise à jour, et effectue la synchronisation avec le dépôt distant.

Exemple :

```ps1
.\updateAll.ps1
```

## Conclusion

En intégrant ces scripts utilitaires à votre processus de travail, vous pouvez considérablement simplifier la gestion de votre machine de développement. Ils vous permettront d'optimiser les performances de votre environnement, de maintenir vos projets propres et organisés, et de gagner du temps en automatisant des processus répétitifs.

Pour accéder à tous ces scripts, consultez le [référentiel complet sur GitHub](https://github.com/alexis35115/scripts-utilitaires).

💡**Note** : Utilisez Windows Terminal avec PowerShell Core pour éviter des problèmes d'encodage de caractères.

N'hésitez pas à explorer et personnaliser ces outils en fonction de vos besoins spécifiques !
 