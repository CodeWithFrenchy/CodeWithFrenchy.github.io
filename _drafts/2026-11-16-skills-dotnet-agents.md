---
title: "Skills .NET : donner aux agents les bonnes connaissances au bon moment"
date: 2026-11-16 19:00:00 -0400
categories: [dotnet, intelligence-artificielle]
tags: [dotnet, github-copilot, claude-code, agent-skills, mcp, nuget, qualite-code, tests]
---

>  ⚠️ OUVRIR LE FICHIER AVANT PUBLICATION

## Préambule

Un agent peut connaître C#, reconnaître une architecture et produire du code qui compile, tout en passant à côté d’une convention importante du projet. Il peut aussi proposer une API qui appartient à une autre version d’une bibliothèque ou consacrer plusieurs recherches à retrouver une procédure que l’équipe applique régulièrement.

À la .NET Conf 2026, la session [Beyond Autocomplete](https://www.youtube.com/watch?v=R4ZgrOcazT4) illustrait comment fournir ces connaissances au moment où elles deviennent utiles. Les skills occupent une place importante dans cette approche, avec le dépôt `dotnet/skills` et la possibilité d’en distribuer directement dans des paquets NuGet.

Pour une équipe .NET, l’intérêt est concret. On peut transmettre une méthode de diagnostic, documenter l’utilisation d’une bibliothèque interne et réduire certaines recherches répétitives. Encore faut-il distinguer ce qui relève des instructions du dépôt, d’un skill ou d’un outil MCP, puis vérifier que l’ensemble améliore réellement le travail.

Je propose donc de regarder leur rôle, leur mise à disposition et une méthode d’évaluation applicable à nos projets.

## Instructions de dépôt, skills et MCP : trois rôles complémentaires

Ces mécanismes peuvent intervenir dans la même tâche, avec des responsabilités différentes.

| Mécanisme | Rôle principal | Exemple dans une solution .NET |
| --- | --- | --- |
| Instructions de dépôt | Communiquer les conventions et contraintes applicables au projet ou à certains fichiers | Utiliser xUnit, conserver les messages de validation en français et respecter les versions de paquets retenues |
| Skill | Fournir une procédure et des ressources pour une catégorie de tâches | Examiner une requête EF Core en suivant une démarche de vérification définie |
| Outil exposé par MCP | Donner accès à une opération par une interface standardisée | Consulter un résultat de pipeline ou interroger un service de diagnostic autorisé |

### Les instructions décrivent le contexte de travail

Les [instructions personnalisées de GitHub Copilot](https://docs.github.com/en/copilot/concepts/prompting-and-context/custom-instructions) permettent notamment de définir des règles à l’échelle du dépôt dans `.github/copilot-instructions.md`, ou pour certains chemins dans `.github/instructions/`.

Des fichiers comme `AGENTS.md` peuvent également transmettre des consignes aux agents qui les prennent en charge. Les emplacements reconnus, la portée et la combinaison des instructions dépendent du client et de la fonctionnalité utilisée.

J’y placerais les conventions qui doivent accompagner le travail courant. Par exemple, les commandes de validation du projet, les choix de framework de tests et les règles d’architecture. Une procédure détaillée de diagnostic d’un incident particulier trouverait plutôt sa place dans un skill spécialisé.

### Le skill décrit une façon de réaliser une tâche

Le format ouvert [Agent Skills](https://agentskills.io/home) organise une compétence dans un répertoire contenant un fichier `SKILL.md`. Celui-ci décrit son utilité et sa procédure. Il peut être accompagné de références, de gabarits et de scripts.

Le chargement est progressif. L’agent dispose d’abord d’informations permettant d’identifier les skills pertinents. Il peut ensuite lire les instructions complètes et les ressources nécessaires à la tâche.

Un skill peut ainsi expliquer comment rechercher une requête coûteuse, quelles informations recueillir et comment vérifier une amélioration. Il enrichit le contexte de travail du modèle. Il ne réentraîne pas le modèle et ne garantit pas que ses instructions seront toujours suivies correctement.

### MCP expose des capacités et des informations

Le [Model Context Protocol](https://modelcontextprotocol.io/docs/learn/server-concepts) permet à un serveur d’exposer des outils, des ressources et des modèles de requêtes. Un outil définit une opération que le client peut appeler, avec des paramètres et un résultat structurés.

Dans notre exemple, un skill pourrait demander de consulter les résultats d’un pipeline avant de proposer une correction. Un outil MCP fournirait cet accès. L’authentification et les permissions du service détermineraient ce que l’appel peut réellement accomplir.

Un skill peut aussi utiliser une commande locale comme `dotnet test`. MCP n’est donc pas nécessaire à chaque procédure. Inversement, rendre un outil disponible ne fournit pas automatiquement la méthode permettant de l’utiliser efficacement.

## Un exemple : revoir une requête EF Core

Imaginons une demande assez courante : améliorer une recherche de dossiers qui charge trop de données et ralentit lorsque le volume augmente.

Les instructions du dépôt précisent les conventions de la solution. Un skill fournit une démarche de revue. Les outils disponibles permettent de lire le code, d’exécuter les tests et, si l’environnement le permet, d’observer les requêtes envoyées à la base de données.

La [spécification Agent Skills](https://agentskills.io/specification) demande notamment un nom et une description dans un en-tête YAML. Le nom doit correspondre au répertoire du skill. La description aide à déterminer quand le charger.

Voici un exemple volontairement limité à une revue. Il pourrait être enregistré dans `.github/skills/revoir-requete-ef-core/SKILL.md` pour un client Copilot qui reconnaît cet emplacement.

```markdown
---
name: revoir-requete-ef-core
description: >-
  Examiner une requête Entity Framework Core pour identifier des problèmes
  de projection, de suivi, de pagination ou de matérialisation.
  Utiliser lors d'une demande de revue ou de diagnostic d'une requête EF Core.
  Ne pas utiliser pour concevoir une migration de schéma.
---

# Objectif

Produire une analyse étayée et proposer des changements limités au besoin.
Cette procédure de revue ne modifie pas les fichiers.

# Démarche

1. Identifier la version d'EF Core, le fournisseur et les conventions du dépôt.
2. Suivre la requête depuis sa construction jusqu'à sa matérialisation.
3. Vérifier les filtres métier, la projection et l'ordre de pagination.
4. Déterminer si les entités retournées doivent être modifiées avant de
   recommander un changement du comportement de suivi.
5. Rechercher les chargements répétés et les données récupérées inutilement.
6. Examiner le SQL généré et les mesures disponibles lorsque l'environnement
   le permet. Distinguer les observations des hypothèses.
7. Proposer les scénarios de validation et indiquer les contrôles non exécutés.

# Résultat attendu

Pour chaque constat, fournir son emplacement, son effet possible,
la modification proposée et la façon de la vérifier.
```

L’intérêt vient des décisions que cette procédure rend explicites. Elle demande de vérifier si les entités seront modifiées avant de proposer de supprimer leur suivi. Elle exige aussi de distinguer un problème observé d’une hypothèse de performance.

Je garderais ce fichier court. Les particularités d’un fournisseur ou des exemples volumineux peuvent vivre dans des références chargées au besoin. Le skill devient ainsi plus facile à relire, à maintenir et à évaluer.

La portée doit rester claire. Si l’équipe veut automatiser la correction après la revue, elle peut prévoir une procédure distincte avec ses propres validations. Cela évite qu’une demande d’analyse entraîne une modification imprévue du comportement métier.

## Explorer dotnet/skills

Le dépôt [dotnet/skills](https://github.com/dotnet/skills) rassemble des skills maintenus par l’équipe .NET, regroupés en plugins thématiques. On y retrouve notamment des ensembles consacrés à l’accès aux données, au diagnostic, à MSBuild, à NuGet et aux tests.

Leur utilisation ne change pas la version .NET ciblée par l’application. Les procédures doivent tenir compte du SDK et des dépendances du dépôt, comme dans l’exemple précédent.

Un plugin est un regroupement installable. Selon le client, il peut aussi fournir d’autres composants, comme des définitions d’agents ou une intégration avec des outils de langage. Il faut donc examiner ce qui est effectivement chargé dans l’environnement choisi.

Le plugin [dotnet-test](https://github.com/dotnet/skills/blob/main/plugins/dotnet-test/README.md) donne deux exemples utiles. `run-tests` aide à déterminer comment exécuter les tests selon le projet et sa plateforme. `assertion-quality` examine ce que les assertions vérifient réellement.

Ce second cas mérite de l’attention. Le skill [assertion-quality](https://github.com/dotnet/skills/blob/main/plugins/dotnet-test/skills/assertion-quality/SKILL.md) recherche notamment les vérifications superficielles et les comportements insuffisamment couverts. Sa valeur dépend de la pertinence des constats, pas du nombre d’assertions qu’il recommande d’ajouter.

Pour un test qui vérifie une règle de date de fin inclusive, je voudrais par exemple savoir s’il échouerait lorsque la dernière journée est exclue par erreur. Une recommandation utile doit se rattacher à ce comportement précis.

### Installer un ensemble ciblé

Le README propose notamment cette installation dans Copilot CLI ou Claude Code. Ces commandes se saisissent dans l’interface interactive de l’agent.

```text
/plugin marketplace add dotnet/skills
/plugin install dotnet-test@dotnet-agent-skills
```

Après le redémarrage demandé par la procédure, la commande suivante permet de consulter les skills disponibles.

```text
/skills
```

Je commencerais par un plugin correspondant à une difficulté récurrente de l’équipe. Cela rend son utilité plus facile à observer et les éventuels conflits plus faciles à comprendre.

La [documentation des skills dans Copilot](https://docs.github.com/en/copilot/concepts/customizations/agent-skills) distingue les skills du projet et les skills personnels. Elle rappelle aussi que leur disponibilité dépend de l’environnement. Une installation sur le poste d’un développeur ne garantit pas leur présence dans une session exécutée à distance.

Pour une pratique partagée, il faut donc choisir comment l’équipe reçoit les mêmes ressources et comment leurs mises à jour sont suivies.

## Distribuer les connaissances avec un paquet NuGet

### Rapprocher les instructions de la version utilisée

L’article [Delivering Skills in NuGet packages](https://devblogs.microsoft.com/dotnet/delivering-skills-in-nuget-packages/) présente une autre voie de distribution. Un paquet peut contenir les instructions adaptées à sa propre utilisation, en plus des bibliothèques qu’il fournit.

Lors de la restauration, les fichiers du paquet sont extraits dans le cache NuGet. Les connaissances peuvent ainsi accompagner la version effectivement référencée par le projet. Le paquet Markout sert d’exemple dans la démonstration, avec plusieurs skills consacrés à ses usages.

Pour une bibliothèque interne fictive nommée `CodeWithFrenchy.Data`, je pourrais inclure un fichier comme `skills/codewithfrenchy-data-revoir-requete/SKILL.md`. Il documenterait les points d’entrée de cette bibliothèque, ses conventions et les erreurs fréquentes observées dans les projets qui la consomment.

Le contenu pourrait évoluer avec les versions du paquet. Si une méthode de configuration change, les indications destinées aux agents devraient changer avec elle. L’équipe qui maintient la bibliothèque prendrait ainsi en charge la qualité de ses exemples et de ses procédures.

### Télécharger un skill et le rendre disponible sont deux étapes

`dotnet restore` ne garantit pas que l’agent découvrira les skills présents dans le cache NuGet. Il faut encore que son environnement sache les repérer ou qu’ils soient copiés dans un emplacement reconnu.

L’outil [dotnet-inspect](https://www.nuget.org/packages/dotnet-inspect) permet notamment d’inspecter les skills associés aux dépendances directes d’un projet. Une fois l’outil installé, une commande peut prendre cette forme.

```bash
dotnet-inspect project src/GestionDossiers.Api/GestionDossiers.Api.csproj -S Skills
```

L’outil [dotnet-package-skills](https://www.nuget.org/packages/dotnet-package-skills) permet de copier les skills fournis par les dépendances directes dans un répertoire reconnu par l’agent. Au moment de la rédaction, cet outil est en préversion.

Voici un exemple pour une solution fictive `GestionDossiers.slnx`, avec une version explicite de l’outil.

```bash
dotnet tool install --global dotnet-package-skills --version 0.1.0-beta.26530.1
dotnet restore GestionDossiers.slnx
dotnet-package-skills list --source GestionDossiers.slnx
dotnet-package-skills install --source GestionDossiers.slnx --dry-run
```

Après examen du contenu et des changements proposés, la copie peut être effectuée avec la même commande sans `--dry-run`.

La destination par défaut est `.agents/skills`. Elle doit être compatible avec le client utilisé. Il faut également prévoir le rafraîchissement des copies lorsque les dépendances changent et examiner les skills devenus inutiles lorsqu’un paquet est retiré.

Pour une équipe, je privilégierais une mise à jour visible dans une pull request, avec la provenance du contenu et la version du paquet. Les adaptations propres à l’organisation gagneraient à rester dans un skill distinct afin de faciliter les mises à jour du contenu fourni par l’auteur.

## Vérifier l’effet d’un skill sur les résultats

Le [tableau de bord de dotnet/skills](https://dotnet.github.io/skills/) présente des évaluations comparatives selon les skills et les modèles. La conférence insiste sur plusieurs dimensions, dont la réussite des tâches, le temps et les jetons consommés.

Ces résultats donnent des indications pour les scénarios évalués. Pour décider d’une adoption dans mon équipe, je compléterais cette lecture par un essai sur nos propres tâches.

Les [consignes de contribution du dépôt](https://github.com/dotnet/skills/blob/main/CONTRIBUTING.md) accordent d’ailleurs une place importante aux scénarios d’évaluation, aux résultats vérifiables et aux situations dans lesquelles le skill doit rester inactif.

### Préparer une comparaison contrôlée

Je comparerais deux configurations.

- La configuration de référence, avec les instructions du dépôt et les outils habituels.
- La même configuration, à laquelle on ajoute uniquement le skill évalué.

Cette distinction compte lorsqu’un plugin ajoute aussi des outils. Si l’on active simultanément un skill et un nouveau service de langage, l’écart mesuré concerne l’ensemble des changements. Pour attribuer un gain au skill, les autres capacités doivent rester identiques.

Je conserverais le même commit du projet, le même SDK, les mêmes dépendances, le même client et le même modèle. Un routage automatique entre plusieurs modèles compliquerait l’interprétation. Je le désactiverais pour ce premier essai, puis je mesurerais séparément le comportement dans la configuration habituelle de l’équipe.

Chaque exécution commencerait dans une nouvelle session et un espace de travail réinitialisé. L’état des caches et les limites de temps ou de jetons devraient aussi être comparables.

### Choisir des tâches qui représentent le travail réel

Pour notre skill de revue EF Core, je préparerais plusieurs cas distincts.

- Une projection qui récupère beaucoup plus de données que l’écran n’en utilise.
- Une pagination dont l’ordre devient instable lorsque plusieurs éléments ont la même valeur de tri.
- Une lecture d’entités qui seront ensuite modifiées, afin de vérifier la pertinence des recommandations sur le suivi.
- Une requête déjà adaptée au besoin, pour mesurer les changements inutiles proposés.
- Une demande sans rapport avec EF Core, afin de vérifier que le skill n’est pas activé inutilement.

Les requêtes adressées à l’agent resteraient naturelles. Pour tester sa capacité de sélection, je ne mentionnerais pas le nom du skill dans la demande. Un essai où je demande explicitement son utilisation répondrait à une autre question, celle de la qualité de la procédure une fois chargée.

Comme premier pilote, une dizaine de tâches exécutées trois à cinq fois par configuration peut aider à repérer des tendances et des échecs récurrents. Ce volume ne suffit pas à établir un gain général. Si les résultats sont proches, il faut davantage d’observations avant de conclure.

### Définir les critères avant de lancer les essais

Le résultat attendu doit être défini indépendamment de la réponse de l’agent. Pour une tâche de correction, les tests de référence devraient rester hors du périmètre qu’il peut modifier pendant l’essai.

Pour une revue, je préparerais les problèmes connus et les recommandations acceptables. Un constat de performance devrait être accompagné d’une observation ou présenté comme une hypothèse à vérifier. Une suggestion qui modifie le comportement métier serait comptée comme une régression.

| Dimension | Ce que je mesurerais |
| --- | --- |
| Qualité | Problèmes correctement identifiés, résultats attendus satisfaits et absence de régression |
| Pertinence | Faux positifs, recommandations inutiles et respect du périmètre demandé |
| Activation | Présence du skill, chargement sur les tâches pertinentes et absence de chargement hors sujet |
| Efficacité | Temps écoulé, appels d’outils, jetons et coût lorsque ces données sont disponibles |
| Effort humain | Corrections nécessaires et temps de revue avant de pouvoir utiliser le résultat |

Je m’appuierais d’abord sur les contrôles exécutables et les faits vérifiables. Une évaluation par un autre modèle peut aider à examiner des réponses, mais elle doit elle-même être confrontée à des cas connus et à une revue humaine.

Je conserverais également les échecs et les dépassements de temps dans les résultats. Ne regarder que les exécutions réussies masquerait une partie du coût et des limites de la solution.

### Interpréter les résultats sans chercher un gain à tout prix

Un skill peut améliorer la qualité en consommant davantage de jetons. Il peut aussi conserver la même qualité en réduisant les recherches et le temps de revue. Ces deux résultats peuvent avoir de la valeur selon la tâche.

Je regarderais les résultats par catégorie de travail et leur dispersion, en plus d’une médiane globale. Une amélioration sur les requêtes complexes peut coexister avec un surcoût inutile sur les demandes simples.

Si le skill n’apporte aucun bénéfice observable, je vérifierais sa description, son périmètre et son contenu. Il peut être trop général, redondant avec les instructions existantes ou mal adapté au modèle utilisé. Son retrait peut être une bonne décision.

## Maintenir les skills comme une partie de l’outillage

Un skill partagé mérite un responsable, une provenance connue et un historique de changements. Une mise à jour de modèle, de client ou de bibliothèque peut modifier son utilité. Les scénarios d’évaluation doivent donc pouvoir être rejoués.

Je distinguerais aussi les instructions des contrôles réellement appliqués. Une phrase demandant de ne pas modifier une base de données de production ne remplace pas une restriction d’accès. Les permissions des outils, des pipelines et des services doivent faire respecter les limites de l’environnement.

Lorsqu’un skill contient des scripts, leur revue fait partie de l’adoption. Cela vaut aussi pour les fichiers fournis par une dépendance NuGet. Leur présence dans un paquet facilite la distribution, mais ne dispense pas d’en comprendre le comportement.

Enfin, un skill devrait rester cohérent avec les conventions du projet. Si une procédure recommande de changer de framework de tests alors que l’objectif est une simple revue, son périmètre doit être corrigé. Une compétence utile aide l’équipe à accomplir une tâche définie sans introduire des décisions supplémentaires à son insu.

## Conclusion

Les skills permettent de rendre des connaissances techniques et des procédures disponibles au moment où un agent en a besoin. Les instructions du dépôt donnent le contexte, les skills structurent certaines tâches et les outils, dont ceux exposés par MCP, fournissent les capacités nécessaires pour les réaliser.

Avec `dotnet/skills`, les équipes disposent de ressources spécialisées à examiner et à essayer. La distribution par NuGet rapproche également les indications d’utilisation des bibliothèques et de leurs versions.

Le gain reste à démontrer dans le contexte du projet. Un bon point de départ consiste à choisir une difficulté récurrente, à introduire un seul skill et à comparer les résultats avec des critères définis à l’avance.

Pour moi, la réussite se voit dans le travail livré. Des constats plus justes, moins de recherches répétitives, des changements mieux vérifiés et moins de corrections humaines sont des raisons solides de conserver un skill et de le faire évoluer.
