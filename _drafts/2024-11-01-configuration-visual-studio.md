---
title: Ma configuration de Visual Studio
date: 2024-11-01 20:00:00 -0400
categories: [outil-developpement]
tags: [visual-studio]
---

Pour l'installation de Visual Studio, suivez-le [guide d'installation officiel](https://learn.microsoft.com/fr-ca/visualstudio/install/install-visual-studio?view=vs-2022).

Les **ressources internes** ont accès à une licence MSDN qui donne accès à [Visual Studio **Professional**](https://visualstudio.microsoft.com/fr/vs/professional/).

Los de l'installation, assurez-vous d'[installer la charge de travail](https://learn.microsoft.com/fr-fr/visualstudio/install/install-visual-studio?view=vs-2022#step-4---choose-workloads) suivante :

![image.png](/.attachments/image-f6972f48-3645-4d7a-8fe2-1482b2c19b51.png)

## Configurations obligatoires

### Suivre les conventions de l'équipe

![image.png](/.attachments/image-56a90476-1ee4-4855-9365-27b0bee4db10.png)

> 💡Cette configuration est requise pour considérer les configurations du `.editorconfig` racine.

### Formatter le code lors de la sauvegarde

![image.png](/.attachments/image-160400ec-db23-406b-921a-2c8d18507e7e.png)

## Configurations optionnelles

#### Ajouter des couleurs distinctes aux paires de crochets (parenthèses, accolades ou crochets)

![image.png](/.attachments/image-3070b137-5ece-49a0-8998-2c87a0a772b6.png)

> Référence : <https://visualstudiomagazine.com/articles/2023/02/28/brace-colorization.aspx>

## Extensions

### Extensions obligatoires recommandées

- [SonarLint](https://marketplace.visualstudio.com/items?itemName=SonarSource.SonarLintforVisualStudio2022)
- [Fine Code Coverage](https://marketplace.visualstudio.com/items?itemName=FortuneNgwenya.FineCodeCoverage2022) (si vous n'avez pas accès à Visual Studio Enterprise)

### Extensions optionnelles

- [CleanBinAndObj](https://marketplace.visualstudio.com/items?itemName=dobrynin.cleanbinandobj)
- [Indent Guides for VS 2022](https://marketplace.visualstudio.com/items?itemName=SteveDowerMSFT.IndentGuides2022)

  ![image.png](/.attachments/image-438efbf0-0d1b-4207-9024-f060b532eab6.png)

  ![image.png](/.attachments/image-a69af51d-e134-46b2-b1d2-87ef7d125a5c.png)

- [Productivity Power Tools 2022](https://marketplace.visualstudio.com/items?itemName=VisualStudioPlatformTeam.ProductivityPowerPack2022)
- [Open in Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=MadsKristensen.OpeninVisualStudioCode)
- [File Icons](https://marketplace.visualstudio.com/items?itemName=MadsKristensen.FileIcons)
- [Add New File (64-bit)](https://marketplace.visualstudio.com/items?itemName=MadsKristensen.AddNewFile64)
