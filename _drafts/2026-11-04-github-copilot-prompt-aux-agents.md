---
title: "GitHub Copilot : du premier prompt aux agents dans un projet .NET"
date: 2026-11-04 19:00:00 -0500
categories: [intelligence-artificielle, developpement]
tags: [github-copilot, github, dotnet, csharp, agents-ia, mcp, bonnes-pratiques]
---

## Préambule

Tu ouvres une solution .NET pour corriger une règle métier. Le changement semble simple, jusqu’à ce que tu découvres ses dépendances dans un service, une requête Entity Framework Core, quelques tests et la documentation de l’API.

GitHub Copilot peut t’aider à comprendre ce parcours, préparer une correction et vérifier ses conséquences. Pour en tirer quelque chose d’utile, il faut toutefois savoir quel contexte lui fournir, quelles actions lui confier et comment évaluer son travail.

C’est ce que nous allons construire dans cet article. Nous partirons des suggestions dans l’éditeur pour aller jusqu’aux agents, aux fichiers d’instructions, aux procédures réutilisables et aux connexions vers des outils externes. Une API ASP.NET Core de gestion de mandats servira de fil conducteur.

L’objectif est de pouvoir utiliser Copilot avec une méthode reproductible, en gardant la maîtrise du comportement livré.

> Les fonctionnalités et les références ont été vérifiées le 3 novembre 2026. Les exemples de personnalisation ciblent principalement VS Code. Leur [disponibilité](https://docs.github.com/en/copilot/get-started/about-github-copilot) dépend du client, du moteur d’agent, de l’abonnement et des politiques de ton organisation.

## Comprendre ce que GitHub Copilot fait réellement

[GitHub Copilot](https://docs.github.com/en/copilot/get-started/about-github-copilot) propose plusieurs formes d’assistance. Une suggestion dans l’éditeur complète du code pendant que tu écris. Une conversation permet d’expliquer une méthode ou d’explorer une solution. Un agent peut prendre en charge plusieurs étapes, comme rechercher les fichiers pertinents, les modifier, exécuter des outils et examiner leurs résultats.

Dans notre projet de mandats, ces usages correspondent à des besoins différents.

| Besoin | Utilisation possible |
| --- | --- |
| Compléter un mapping entre une entité et un DTO | Suggestions dans l’éditeur |
| Comprendre pourquoi un mandat disparaît trop tôt | Conversation avec le code pertinent |
| Évaluer les impacts d’une correction | Planification |
| Corriger la règle et ajouter les tests associés | Agent avec accès aux outils nécessaires |
| Examiner le changement avant intégration | Revue assistée, puis validation de l’équipe |

### Modèle, moteur, interface et environnement

Quatre notions méritent d’être séparées.

| Notion | Ce qu’elle désigne |
| --- | --- |
| Modèle | Le système qui traite le contexte et produit une réponse ou une demande d’appel d’outil |
| Moteur d’agent, ou *harness* | Le composant qui organise les échanges avec le modèle, les outils et les règles de fonctionnement |
| Interface | L’endroit où tu interagis avec l’outil, par exemple VS Code, Visual Studio, le terminal ou GitHub |
| Environnement d’exécution | L’endroit où les fichiers sont modifiés et où les commandes s’exécutent |

Le modèle ne lance pas lui-même `dotnet test`. Il peut demander l’exécution d’une commande. [Le moteur traite cette demande](https://code.visualstudio.com/docs/agents/concepts/language-models) selon les outils et les permissions disponibles, puis lui transmet le résultat. L’agent peut alors adapter sa prochaine action.

Choisir un modèle Claude dans Copilot ne signifie donc pas utiliser Claude Code. De la même manière, travailler dans un terminal local ne prouve pas que le modèle s’exécute sur ton ordinateur. Dans VS Code, **Local** désigne aussi [un moteur précis](https://code.visualstudio.com/docs/agents/run/agent-harnesses), ce qui ajoute une ambiguïté au vocabulaire.

## Préparer le projet et commencer simplement

Avant de déléguer une modification, assure-toi que le projet possède un point de départ compréhensible.

1. Connecte le client compatible avec le compte qui possède l’accès Copilot.
2. Vérifie les fonctionnalités et les modèles autorisés par ton organisation.
3. Ouvre le dépôt et identifie les instructions déjà présentes.
4. Vérifie le SDK, les dépendances et les commandes de validation.
5. Consulte l’état Git pour distinguer le travail existant de la modification à venir.

Pour notre solution fictive `GestionMandats`, les premières vérifications pourraient être les suivantes.

```bash
git status --short
dotnet --info
dotnet restore GestionMandats.sln
dotnet build GestionMandats.sln --no-restore
dotnet test GestionMandats.sln --no-build
```

Ces commandes supposent une solution utilisant VSTest et un environnement où les tests peuvent s’exécuter. Adapte-les à ton dépôt, notamment si celui-ci utilise Microsoft Testing Platform, des flux NuGet privés ou des services nécessaires aux tests d’intégration. [`--no-build`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-test) n’est pertinent ici que si la compilation précédente a réussi.

Cette préparation évite un diagnostic fréquent mais trompeur. Si les tests échouaient déjà avant l’intervention de Copilot, il faut le savoir pour évaluer correctement le changement.

### Suggestions et actions rapides

Les suggestions dans l’éditeur conviennent bien aux transformations courtes et répétitives. Un mapping, une structure de test déjà connue ou une adaptation locale peuvent en bénéficier.

Prends néanmoins le temps de vérifier la proposition. Pour un DTO, le risque peut être aussi simple qu’une propriété oubliée ou une valeur par défaut qui masque une donnée manquante. Pour une requête, il peut s’agir d’un filtre métier supprimé.

Tu peux aussi utiliser les actions proposées par ton client pour expliquer une sélection ou préparer un message de commit. Dans ce dernier cas, compare le texte au diff réellement indexé. Une description convaincante doit encore correspondre au contenu du commit.

Si les suggestions te distraient, ajuste leur comportement ou désactive-les pendant certaines tâches. L’intérêt de l’outil se mesure au travail utile qu’il facilite.

## Donner le bon contexte à Copilot

Une demande n’arrive pas seule au modèle. Elle peut être accompagnée de l’historique, d’instructions, de fichiers, d’une sélection dans l’éditeur et de résultats d’outils.

Dans VS Code, [la recherche et l’indexation](https://code.visualstudio.com/docs/agents/concepts/context) aident l’agent à retrouver des éléments du dépôt. Elles ne signifient pas que toute la solution est insérée dans chaque requête. Un service essentiel à la compréhension du problème peut encore manquer au contexte.

Pour analyser un mandat déclaré inactif trop tôt, fournis ou fais rechercher les éléments qui expliquent le comportement.

- L’entité `Mandat` et ses dates.
- La méthode qui détermine si le mandat est effectif.
- La requête EF Core qui sélectionne les mandats.
- Les tests existants.
- La règle métier attendue.
- Un exemple de résultat incorrect.

Une question de suivi comme « ajoute aussi le cas sans date de fin » peut s’appuyer sur la conversation en cours. Lorsqu’on change complètement de problème, mieux vaut repartir avec un contexte adapté. Dans une longue session, [certaines informations peuvent être résumées](https://code.visualstudio.com/docs/agents/concepts/context) ou perdre en précision. Conserve les décisions durables dans le dépôt.

### Comprendre les tokens

Un token est une unité de traitement du modèle. Il peut correspondre à un mot, à une partie de mot ou à de la ponctuation. Un identifiant C# et une phrase française ne se découpent pas nécessairement de la même manière.

Les tokens servent notamment à mesurer la taille du contexte et celle des réponses. Selon le modèle, le raisonnement interne utilise aussi des tokens. Une grande [fenêtre de contexte](https://code.visualstudio.com/docs/agents/concepts/language-models) offre plus de capacité, sans garantir que tous les détails seront correctement utilisés.

La conséquence pratique est simple. Ajouter un fichier pertinent peut éviter plusieurs mauvaises tentatives. Ajouter des milliers de lignes sans rapport peut augmenter le volume traité sans clarifier la tâche.

### LLM, SLM et choix du modèle

Les termes **LLM**, pour *Large Language Model*, et **SLM**, pour *Small Language Model*, décrivent des catégories de modèles de langage. Les [modèles plus compacts](https://github.com/microsoft/generative-ai-for-beginners/blob/main/19-slm/README.md) peuvent demander moins de ressources. Leur intérêt dépend toutefois de la tâche, de leur entraînement et de leur capacité à utiliser les outils disponibles.

Le nombre de paramètres d’un modèle et le volume de ses données d’entraînement sont deux choses différentes. Il faut aussi éviter de déduire sa qualité uniquement de sa taille ou de son classement dans un benchmark.

Un modèle peut produire une réponse plausible mais incorrecte, et [ses connaissances peuvent être dépassées](https://code.visualstudio.com/docs/agents/concepts/language-models). Pour une API ou une option récente, demande une vérification dans la documentation officielle correspondant à la version du projet.

Pour ton équipe, compare les modèles sur quelques tâches représentatives.

| Tâche d’évaluation | Ce que tu observes |
| --- | --- |
| Expliquer une requête EF Core | Exactitude de l’explication et prise en compte des filtres |
| Compléter des tests métier | Pertinence des assertions et des cas limites |
| Analyser une régression | Capacité à relier les fichiers et à vérifier les hypothèses |
| Préparer une correction | Qualité du diff, temps de révision et reprises nécessaires |

Je te conseille de choisir selon le coût total d’une tâche validée. Une réponse rapide perd son intérêt si elle demande ensuite plusieurs corrections.

## Passer d’une question à une tâche réalisable

Les appellations **Ask**, **Plan**, **Edit** et **Agent** reviennent souvent dans les discussions sur Copilot. Elles permettent de comprendre plusieurs usages, mais ne correspondent pas à quatre boutons disponibles partout.

| Mode ou rôle | Usage | Exemple dans notre API |
| --- | --- | --- |
| Ask | Comprendre et explorer | Expliquer le calcul de validité d’un mandat |
| Plan | Préparer une intervention | Identifier les fichiers, les risques et les tests |
| Edit, lorsqu’il est proposé | Modifier un périmètre ciblé | Ajuster quelques fichiers sélectionnés |
| Agent | Réaliser une suite d’actions | Corriger la règle, lancer les tests et examiner les résultats |

La documentation actuelle de VS Code décrit notamment [Ask, Plan et Agent](https://code.visualstudio.com/docs/agents/run/agent-harnesses) pour les moteurs Copilot et Local. Vérifie l’expérience disponible dans ton client plutôt que de supposer que tous les libellés d’un exemple s’y retrouvent.

### Commencer par comprendre

Pour une anomalie encore mal comprise, une première demande peut rester limitée à l’analyse.

```text
Explique comment cette API détermine qu’un mandat est effectif.

Retrouve les règles dans le domaine et dans les requêtes EF Core.
Indique les fichiers et les méthodes concernés.
Signale les différences de comportement entre ces implémentations.
N’effectue aucune modification.
```

Cette demande permet d’établir ce qui existe avant de prescrire une correction. Si la réponse semble incohérente, demande les éléments du dépôt qui la justifient.

### Formuler une demande exploitable

Un prompt utile décrit le résultat attendu et donne un moyen de reconnaître qu’il a été atteint. Je recommande d’y faire apparaître l’objectif, le périmètre, les règles métier, les contraintes et les vérifications. Cette structure rejoint [les recommandations de GitHub](https://docs.github.com/en/copilot/concepts/prompting-and-context/prompt-engineering) sur le contexte et la décomposition des tâches.

Voici une demande complète pour notre scénario.

```text
Dans la solution GestionMandats, corrige la sélection des mandats effectifs.

Règles métier
- La date de début est incluse.
- La date de fin est incluse.
- Une date de fin absente signifie qu’aucune fin n’est prévue.
- L’évaluation utilise la date métier fournie à la méthode.

Périmètre
- Examine le domaine, la requête EF Core et les tests associés.
- Conserve les contrats publics et le schéma de la base de données.
- Réutilise les abstractions et les bibliothèques du dépôt.
- Ne modifie pas les autres règles de sélection.

Travail attendu
- Présente d’abord les impacts et un plan court.
- Après validation du plan, réalise la correction.
- Ajoute des tests xUnit couvrant les limites de dates.
- Si la requête EF Core change, vérifie son comportement avec
  le fournisseur de base de données utilisé par le projet.

Compte rendu
- Résume les fichiers modifiés.
- Donne les commandes exécutées et leurs résultats.
- Indique les vérifications impossibles à réaliser.
```

Pour une petite correction déjà comprise, tu peux alléger ce prompt. Pour une modification qui touche des habilitations ou un contrat public, une étape de planification explicite aide à résoudre les ambiguïtés avant l’écriture du code.

Faire reformuler la demande par une IA peut aussi être utile. Relis toutefois les exigences ajoutées. Une reformulation ne doit pas inventer une règle métier ou élargir discrètement le périmètre.

### Définir les résultats avant de générer les tests

Prenons un mandat qui commence le 2 novembre 2026 et se termine le 27 novembre 2026.

| Date de début | Date de fin | Date évaluée | Effectif |
| --- | --- | --- | --- |
| 2026-11-02 | 2026-11-27 | 2026-11-01 | Non |
| 2026-11-02 | 2026-11-27 | 2026-11-02 | Oui |
| 2026-11-02 | 2026-11-27 | 2026-11-15 | Oui |
| 2026-11-02 | 2026-11-27 | 2026-11-27 | Oui |
| 2026-11-02 | 2026-11-27 | 2026-11-28 | Non |
| 2026-11-02 | Absente | 2026-11-01 | Non |
| 2026-11-02 | Absente | 2030-01-01 | Oui |

Dans une expression LINQ où les dates sont des `DateOnly` et où `DateFin` est nullable, le prédicat attendu pourrait prendre cette forme.

```csharp
x => x.DateDebut <= dateEvaluation &&
     (x.DateFin == null || x.DateFin >= dateEvaluation)
```

Ce fragment illustre la règle à intégrer dans la requête existante. Il ne décrit pas à lui seul les autres filtres, les autorisations ou la provenance de la date métier.

Ce tableau sert aussi à réviser les tests générés. Si l’agent recopie une comparaison incorrecte dans le calcul du résultat attendu, le test peut confirmer le bogue. L’attendu doit venir de la règle métier.

Un bon test sur la dernière journée doit échouer si on remplace accidentellement la comparaison inclusive par une comparaison stricte.

## Mettre les conventions durables dans des instructions

Tu ne devrais pas avoir à répéter à chaque conversation que les messages de validation sont en français ou que les classes de tests sont déclarées `sealed`.

Les fichiers d’instructions permettent de conserver ce contexte dans le dépôt. Les emplacements suivants sont courants dans les expériences Copilot compatibles.

| Fichier | Intention |
| --- | --- |
| `.github/copilot-instructions.md` | Conventions générales du projet |
| `.github/instructions/*.instructions.md` | Consignes ciblées sur certains fichiers ou certaines tâches |
| `AGENTS.md` | Consignes pour les agents qui prennent en charge ce format |

La portée et la découverte dépendent du moteur utilisé. Un fichier présent dans le dépôt n’est pas nécessairement chargé par tous les outils. Dans VS Code, [les instructions personnalisées](https://code.visualstudio.com/docs/agent-customization/custom-instructions) ne s’appliquent pas aux suggestions de code affichées pendant la saisie.

### Décrire les choix propres au projet

Voici un exemple de contenu pour `.github/copilot-instructions.md`.

```markdown
# Contexte du projet

L’API GestionMandats utilise ASP.NET Core et Entity Framework Core.
Les dates métier des mandats sont représentées avec DateOnly.
Les bornes de validité sont inclusives.

# Conventions

- Utiliser des noms métier et des messages de validation en français.
- Conserver les noms usuels des patterns, comme Repository et Handler.
- Réutiliser les abstractions existantes.
- Propager les CancellationToken dans les appels asynchrones.
- Éviter les nouvelles dépendances pour une modification locale.

# Validation

- Consulter le README pour les prérequis et les commandes.
- Exécuter les tests pertinents après une modification.
- Rapporter les vérifications effectuées et les limites rencontrées.
```

Le contenu utile est celui que l’agent risque de mal déduire. Une décision d’architecture, une commande particulière ou une subtilité métier valent davantage qu’une longue liste de conseils génériques.

### Cibler les tests C#

Dans `.github/instructions/tests-csharp.instructions.md`, on peut préciser les conventions des tests.

```markdown
---
applyTo: "**/*Tests.cs"
---

- Utiliser xUnit et les bibliothèques déjà présentes.
- Déclarer les classes de tests sealed.
- Respecter Arrange-Act-Assert sans commentaires de séparation.
- Nommer les tests selon la condition, l’action et le résultat.
- Vérifier les comportements observables et les limites métier.
- Utiliser FakeTimeProvider lorsque le code dépend du temps.
- Éviter les attentes réelles pour simuler le passage du temps.
```

La propriété [`applyTo`](https://code.visualstudio.com/docs/agent-customization/custom-instructions) respecte cette casse. Son motif sert à appliquer les instructions aux fichiers correspondants. Vérifie la découverte du fichier dans ton client et son effet sur une petite tâche.

Une IA peut proposer une première version de ces fichiers à partir du dépôt. Relis les règles avant de les conserver et évite les copies contradictoires entre `AGENTS.md` et les autres instructions.

Enfin, une consigne textuelle reste une consigne. Pour les règles vérifiables, conserve les analyseurs, le formatage et les contrôles de CI.

## Réutiliser les demandes, les rôles et les procédures

Les instructions, les prompts, les agents personnalisés et les skills peuvent se ressembler au premier coup d’œil. Leur différence devient plus claire lorsqu’on part du besoin.

| Besoin | Mécanisme adapté |
| --- | --- |
| Appliquer les conventions C# du dépôt | Instructions |
| Relancer une demande prédéfinie à la demande | Prompt file compatible |
| Réutiliser une procédure avec ses ressources | Skill |
| Définir un rôle et les outils qui lui sont accessibles | Agent personnalisé |
| Accéder à un système externe | Connexion MCP |

Ces [mécanismes](https://code.visualstudio.com/docs/agents/concepts/customization) peuvent être combinés. Par exemple, un agent de documentation peut suivre les instructions du dépôt et utiliser un skill pour appliquer un gabarit.

### Les prompt files pour les demandes récurrentes

Un prompt file conserve une demande que tu déclenches explicitement. Dans VS Code, les fichiers `.prompt.md` de projet sont placés dans `.github/prompts/`.

La documentation actuelle réserve leur exécution aux [sessions du moteur Local](https://code.visualstudio.com/docs/agent-customization/prompt-files) dans la vue Chat. Pour réutiliser une procédure avec un autre moteur compatible, un skill peut être plus approprié.

Par exemple, `.github/prompts/actualiser-readme.prompt.md` pourrait contenir ceci.

```markdown
---
description: Actualiser le README selon un changement validé
agent: agent
---

Identifie le diff à documenter.
Si sa référence est ambiguë, demande la branche ou les commits concernés.

Mets à jour uniquement les sections du README touchées.
Vérifie les commandes dans le dépôt.
Utilise les noms réels des projets et des paramètres.
N’invente aucune fonctionnalité ni aucun prérequis.

Résume les modifications et les points restant à confirmer.
```

Dans une session compatible, tu peux ensuite lancer `/actualiser-readme` et préciser le changement concerné.

L’enregistrement du fichier ne déclenche aucune mise à jour après un commit. Pour cela, il faut une automatisation distincte. Le prompt représente ici une demande réutilisable, exécutée lorsque tu l’invoques.

### Les agents personnalisés pour les rôles spécialisés

Un agent personnalisé regroupe des instructions de rôle et, selon l’environnement, un modèle et une sélection d’outils.

Dans VS Code, un exemple de réviseur .NET peut être enregistré dans `.github/agents/reviseur-dotnet.agent.md`. Les noms d’outils ci-dessous correspondent à [l’exemple documenté par VS Code](https://code.visualstudio.com/docs/agent-customization/custom-agents). Vérifie leur prise en charge dans le moteur retenu.

```markdown
---
name: ReviseurDotNet
description: Examiner un changement .NET et signaler les défauts démontrés
tools: ['read', 'search']
---

Examine le diff fourni comme contexte et les fichiers pertinents.

Priorise les éléments suivants
- Régressions métier
- Autorisations et données sensibles
- Gestion de la concurrence
- Annulation des opérations asynchrones
- Requêtes coûteuses et chargements inutiles

Pour chaque constat, indique
- Le fichier et l’emplacement
- Le scénario qui déclenche le problème
- La conséquence
- Une correction proposée

Distingue les défauts démontrés des hypothèses.
N’affirme pas avoir exécuté des tests sans résultat fourni.
```

Cet agent sert à examiner un changement. Avec les seuls outils de lecture et de recherche, il ne peut pas exécuter la suite de tests. Le résultat doit rester cohérent avec cette limite.

Le même principe peut s’appliquer à un rédacteur de documentation ou à un réviseur d’accessibilité. Donne à chacun un livrable précis.

Pour un rôle en lecture seule, retire réellement les outils d’écriture et d’exécution. Une phrase qui demande de ne rien modifier ne fournit pas la même restriction qu’[une configuration d’outils adaptée](https://code.visualstudio.com/docs/agent-customization/custom-agents).

### Les skills pour les procédures réutilisables

Un **skill** est un ensemble de consignes spécialisées, accompagné au besoin de scripts, d’exemples ou de gabarits. Son point d’entrée est un fichier `SKILL.md`.

Dans les usages Copilot concernés, [les skills de projet](https://code.visualstudio.com/docs/agent-customization/agent-skills) peuvent être placés dans `.github/skills/`. Leur description aide l’agent à reconnaître les tâches pertinentes. Le contenu détaillé est chargé lorsque nécessaire. La découverte d’un skill ne garantit toutefois pas son invocation à chaque occasion.

Pour une équipe .NET, une procédure de revue des migrations EF Core constitue un bon exemple. Voici un point de départ pour `.github/skills/revue-migration-efcore/SKILL.md`.

```markdown
---
name: revue-migration-efcore
description: Examiner une migration EF Core avant son intégration.
---

1. Identifier la migration, le DbContext et le fournisseur de base de données.
2. Lire les changements de modèle et le snapshot associé.
3. Repérer les suppressions, changements de type et nouvelles contraintes.
4. Examiner les effets possibles sur les données existantes.
5. Vérifier les index, les valeurs par défaut et les colonnes obligatoires.
6. Utiliser la procédure du dépôt pour examiner le SQL généré.
7. Distinguer ce qui a été exécuté de ce qui reste à tester.
8. Documenter le retour arrière et les pertes de données possibles.
9. Produire un avis de revue sans appliquer la migration à une base de données.
```

Ce dossier peut ensuite accueillir un gabarit de rapport ou un script approuvé de génération SQL. Le skill conserve alors une procédure complète avec les ressources nécessaires pour la suivre.

Dans un client compatible, tu peux [invoquer ce skill](https://code.visualstudio.com/docs/agent-customization/agent-skills) avec `/revue-migration-efcore` et préciser la migration concernée. Vérifie dans l’activité de l’agent qu’il utilise effectivement la procédure.

D’autres usages sont possibles, comme analyser une requête lente avec un protocole de mesure ou vérifier l’accessibilité d’une interface Blazor.

Examine les scripts et les ressources avant de réutiliser un skill externe. Sa présence n’ajoute aucune autorisation à la tâche et ne garantit pas la qualité de sa procédure.

## Choisir où l’agent travaille

Le niveau d’interaction et le lieu d’exécution influencent la façon de déléguer.

| Exécution | Tâche adaptée | Point à vérifier |
| --- | --- | --- |
| Interactive dans l’environnement de développement | Diagnostic et ajustements fréquents | Le contexte et les réponses intermédiaires |
| En arrière-plan sur la machine | Travail délimité pendant une autre activité | La disponibilité du processus et l’isolation des fichiers |
| Dans le cloud | Tâche réalisable dans un environnement distant | Le SDK, les dépendances, le réseau et les accès nécessaires |

Pour notre API, expliquer une anomalie reste souvent confortable en mode interactif. Ajouter des tests dont les cas sont déjà définis peut se prêter à une exécution en arrière-plan. Une correction reproductible décrite dans une issue peut être déléguée dans le cloud si l’environnement convient.

Les expériences GitHub permettent de travailler sur des tâches distantes et de [poursuivre certaines sessions dans plusieurs clients](https://docs.github.com/en/copilot/concepts/copilot-surfaces/copilot-on-github). Vérifie néanmoins ce que le transfert conserve. Les fichiers locaux non commités, les secrets et les services de ta machine ne deviennent pas automatiquement disponibles à distance.

### Utiliser un worktree pour séparer les modifications

Un [worktree Git](https://git-scm.com/docs/git-worktree) permet de disposer d’un autre répertoire de travail lié au même dépôt. On peut ainsi conserver son activité dans un dossier et laisser une tâche distincte avancer dans un autre.

Par exemple, depuis un dépôt propre, cette commande crée une branche et un répertoire de travail à partir du commit courant.

```bash
git worktree add -b copilot/tests-mandats ../gestion-mandats-tests HEAD
```

Le nouvel environnement ne reprend pas automatiquement les modifications non commitées du dossier initial. Il peut aussi nécessiter sa propre restauration des dépendances.

Dans la fenêtre Agents de VS Code, [le choix entre dossier courant et worktree](https://code.visualstudio.com/docs/agents/run/agents-window) dépend de l’expérience. L’exécution en arrière-plan n’implique donc pas toujours une séparation des fichiers. Un worktree ne restreint pas non plus, à lui seul, les commandes, les accès réseau ou les fichiers accessibles au processus.

Deux agents dans deux worktrees peuvent encore viser la même base de données. Pour les migrations ou les tests qui modifient des données, il faut aussi isoler les ressources utilisées.

### Où se situe Codespaces

GitHub Codespaces fournit un environnement de développement hébergé. Un dev container permet d’en définir les outils et les prérequis de façon reproductible.

Tu peux y utiliser Copilot, mais [Codespaces](https://docs.github.com/en/codespaces/about-codespaces/what-are-codespaces) n’est pas en soi un agent autonome. Pour une solution .NET, son intérêt peut être de fournir le SDK et les outils attendus sans refaire toute la configuration sur chaque poste.

## Observer l’application et revenir sur un changement

### Le navigateur comme moyen de vérification

Lorsqu’une fonctionnalité touche une interface, le code et les tests unitaires ne racontent pas toute l’histoire.

Les [outils de navigateur compatibles](https://code.visualstudio.com/docs/agents/run/browser-tools) peuvent permettre à l’agent d’ouvrir l’application, d’interagir avec elle et d’examiner le résultat. Dans VS Code, des outils de navigateur sont intégrés et ne nécessitent pas systématiquement un serveur MCP externe.

Pour un formulaire Blazor de création de mandat, tu peux demander de vérifier le parcours avec des données de test.

```text
Ouvre le formulaire de création de mandat en développement.

Vérifie la saisie au clavier, l’affichage des erreurs et le focus
après une validation refusée.

Utilise uniquement les données de test prévues dans le dépôt.
Décris les interactions réalisées et les vérifications restantes.
```

Une capture d’écran aide à comprendre l’apparence. Elle ne démontre pas que le formulaire fonctionne au clavier, que le message est annoncé par un lecteur d’écran ou que l’API refuse une requête invalide.

### Les checkpoints et leurs limites

Dans les sessions compatibles, [un checkpoint](https://code.visualstudio.com/docs/agents/run/review-code-edits) permet de revenir à un état antérieur des fichiers concernés et de la conversation. C’est utile lorsqu’une modification prend une mauvaise direction.

Ce mécanisme ne remplace pas Git et ne restaure pas les systèmes externes. Une migration appliquée, un déploiement effectué ou une donnée supprimée nécessite le mécanisme de récupération du système concerné.

Dans notre exemple, restaurer le fichier de migration ne remet pas automatiquement la base de données dans son état précédent.

## Connecter des systèmes externes avec MCP

**MCP**, pour *Model Context Protocol*, permet à un client compatible d’accéder aux capacités exposées par un serveur. Celui-ci peut proposer des outils ou des ressources pour consulter une documentation, examiner un schéma ou interagir avec un service.

Le client découvre les capacités, [l’agent choisit un outil pertinent](https://code.visualstudio.com/docs/agent-customization/mcp-servers), puis son résultat peut enrichir le contexte. La connexion ne charge pas automatiquement toutes les données du service dans la conversation.

Un skill explique une procédure. Un serveur MCP expose une capacité. Un agent peut utiliser les deux pour une même tâche.

### Un exemple avec une base de données de développement

Supposons qu’une variante de notre API ASP.NET Core utilise une base de données PostgreSQL hébergée dans un projet Supabase de développement.

L’exemple suivant utilise le format VS Code du fichier `.vscode/mcp.json`, conservé pour compatibilité avec le moteur Local. La procédure guidée le marque désormais comme déprécié. Pour une nouvelle configuration, utilise [**MCP: Add Server**](https://code.visualstudio.com/docs/agent-customization/mcp-servers) et choisis la destination proposée pour ton moteur.

Le [format VS Code](https://code.visualstudio.com/docs/agents/reference/mcp-configuration) utilise la clé `servers`. Le format portable utilise `mcpServers`, ce qui exige d’adapter la structure si tu changes de configuration.

```json
{
  "servers": {
    "supabase-dev": {
      "type": "http",
      "url": "https://mcp.supabase.com/mcp?project_ref=VOTRE_PROJET_DEV&read_only=true"
    }
  }
}
```

Remplace la référence du projet, établis la connexion et suis le parcours d’authentification proposé. Le paramètre [`project_ref`](https://supabase.com/docs/guides/ai-tools/mcp) limite le périmètre au projet sélectionné. `read_only=true` demande l’exécution des requêtes avec un utilisateur PostgreSQL en lecture seule. Ces paramètres appartiennent à Supabase, et ne sont pas des options universelles de MCP.

On peut ensuite formuler une demande limitée au schéma.

```text
Inspecte le schéma du projet de développement connecté.

Décris les relations et les contraintes autour de la table des mandats.
Compare-les avec les configurations EF Core fournies comme contexte.

N’interroge pas les données métier.
Propose les vérifications nécessaires sans appliquer de migration.
```

Ce type d’accès peut aider à rapprocher le modèle EF Core de la structure réellement disponible. Il faut encore vérifier que l’outil examine le bon projet et la bonne version du schéma.

### Les permissions restent à définir

MCP ne garantit pas automatiquement le moindre privilège. [Les droits effectifs](https://code.visualstudio.com/docs/agent-customization/mcp-servers) dépendent du serveur, du compte connecté, des outils exposés et des restrictions du client. Un serveur lancé localement peut aussi exécuter du code avec les droits du processus.

Pour cet exercice, je recommande un projet de développement avec des données fictives et les seuls outils nécessaires. La lecture seule réduit les possibilités de modification, mais laisse possible la lecture et la divulgation de données accessibles.

Le contenu retourné par un outil doit également rester une donnée à examiner. Une ligne dans un document ou un commentaire qui demande d’ignorer les consignes et de transmettre des secrets constitue une tentative d’[injection de prompt](https://supabase.com/docs/guides/ai-tools/mcp#security-risks), pas une autorisation.

## Garder les secrets et les contrôles de sécurité sous maîtrise

### Un fichier ignoré par Git peut rester accessible

Un fichier `.env` permet de déplacer des valeurs de configuration, mais sa présence ne protège pas les secrets qu’il contient.

De même, [`.gitignore`](https://git-scm.com/docs/gitignore) sert à désigner des fichiers que Git doit ignorer dans son fonctionnement habituel. Il ne retire pas les droits de lecture d’un processus et ne fait pas disparaître un fichier déjà suivi. Il faut donc distinguer l’exclusion du suivi Git et la protection des accès.

Évite de placer des secrets dans les prompts, les captures, les sorties de commandes et les fichiers d’instructions. Si un secret est exposé, sa suppression du texte doit s’accompagner de sa révocation ou de sa rotation.

### Les particularités d’un projet .NET et Azure

En développement ASP.NET Core, [User Secrets](https://learn.microsoft.com/en-us/aspnet/core/security/app-secrets) permet de conserver des valeurs hors du répertoire du projet. Ce mécanisme ne chiffre pas les secrets et ne constitue pas un coffre inaccessible aux autres processus qui disposent des droits nécessaires.

Pour un déploiement Azure, privilégie une authentification adaptée au service, par exemple une identité managée lorsqu’elle est prise en charge. Lorsqu’un secret reste nécessaire, un coffre comme [Azure Key Vault](https://learn.microsoft.com/en-us/aspnet/core/security/key-vault-configuration) permet d’en organiser l’accès. Le choix doit tenir compte des identités réellement utilisées par l’application et par les outils de développement.

Dans une revue assistée, demande aussi d’examiner la journalisation. Un ajout de diagnostic qui écrit les paramètres SQL ou les données personnelles peut introduire un problème même si la correction fonctionnelle est bonne.

### Distinguer les fonctions de sécurité GitHub

Les outils de sécurité répondent à des besoins différents.

| Fonction | Rôle |
| --- | --- |
| Dependabot alerts | Signaler des dépendances présentant des vulnérabilités connues |
| Dependabot security updates | Proposer des mises à jour correctives lorsque possible |
| Dependabot version updates | Proposer des mises à jour selon une configuration et une fréquence |
| Secret scanning | Détecter des secrets selon les mécanismes pris en charge |
| Push protection | Bloquer certains envois contenant des secrets détectés |
| Code scanning, notamment avec CodeQL | Analyser le code pour repérer des vulnérabilités |

Leur disponibilité dépend de la fonction, du dépôt, de la configuration et des produits activés. Un abonnement Copilot ne signifie pas que tous [ces contrôles](https://docs.github.com/en/code-security/getting-started/github-security-features) sont en place.

Les expériences [Autofix](https://docs.github.com/en/code-security/concepts/code-scanning/autofix-for-code-scanning) peuvent proposer des corrections à partir d’alertes. Les modalités et les conditions d’accès varient selon l’expérience. Les modifications proposées restent à examiner et à tester.

Un agent personnalisé appelé « Security Agent » peut compléter une revue. Il ne remplace pas ces contrôles, les tests d’autorisation ou les protections du pipeline.

## Comprendre les coûts et l’option BYOK

### Tokens, crédits IA et requêtes premium

Ces termes désignent des choses différentes.

| Terme | Signification |
| --- | --- |
| Tokens | Unités traitées ou produites par les modèles |
| Crédits IA | Unité utilisée par le régime de facturation Copilot selon l’utilisation |
| Requêtes premium | Unité d’un régime historique encore applicable à certains abonnements |

Dans le [régime de facturation selon l’utilisation](https://docs.github.com/en/copilot/concepts/billing-and-usage/individuals/billing), la consommation dépend notamment du modèle et des tokens traités. Les échanges agentiques peuvent enchaîner plusieurs appels pour une seule tâche. La documentation indique que les complétions de code et les suggestions de prochaine modification ne consomment pas de crédits IA et sont illimitées sur les offres payantes. Les limites de l’offre gratuite sont différentes.

Le [régime historique des requêtes premium](https://docs.github.com/en/copilot/reference/copilot-billing/request-based-billing-legacy/copilot-requests) concerne encore certains abonnements annuels existants. Consulte les modalités réellement appliquées à ton compte avant de calculer un budget.

Pour [maîtriser l’utilisation](https://docs.github.com/en/copilot/tutorials/optimize-ai-usage), fournis les éléments pertinents dès le départ, limite la tâche et définis son critère de fin. Lorsqu’un agent répète les mêmes tentatives, corrige le contexte ou le diagnostic avant de simplement lui demander de continuer.

La métrique utile pour une équipe reste le coût du changement validé, incluant la révision et les reprises. Le nombre de lignes générées renseigne peu sur cette valeur.

### Utiliser sa propre clé de fournisseur

[**BYOK**](https://code.visualstudio.com/docs/agent-customization/language-models), pour *Bring Your Own Key*, permet d’utiliser sa propre configuration d’accès auprès d’un fournisseur compatible. Dans VS Code, les possibilités dépendent du moteur, du fournisseur et des politiques de l’organisation. Une configuration BYOK ne configure pas automatiquement tous les moteurs ni toutes les fonctions de Copilot.

Avant de retenir cette option en entreprise, précise le fournisseur réellement appelé, les modèles permis, la destination des données et le budget associé. Ne présume pas qu’un fournisseur tiers bénéficie automatiquement des mêmes conditions que ton abonnement Copilot.

Cela vaut aussi pour un intermédiaire comme OpenRouter. Vérifie sa compatibilité avec le client et le moteur choisis avant de considérer l’intégration comme acquise.

## Organiser un cycle de travail reproductible

Tous ces mécanismes prennent leur sens dans une façon de travailler cohérente. Pour notre correction de validité des mandats, je recommande le parcours suivant.

| Étape | Résultat attendu |
| --- | --- |
| Comprendre | Localiser la règle et reproduire le comportement incorrect |
| Cadrer | Confirmer les dates inclusives et les contraintes |
| Planifier | Identifier les fichiers et les tests nécessaires |
| Réaliser | Produire une correction limitée au besoin |
| Vérifier | Examiner les tests, les commandes et les résultats |
| Réviser | Relire le diff et les conséquences indirectes |
| Intégrer | Passer par le processus de revue et la CI du dépôt |

La planification peut tenir en quelques lignes pour une petite tâche. Elle devient plus détaillée lorsque les impacts touchent les données, les contrats ou les habilitations.

Avant d’intégrer le changement, demande un compte rendu concret.

```text
Résume le comportement avant et après la correction.

Liste les fichiers modifiés et explique leur rôle dans le changement.
Indique les commandes de validation exécutées et leur résultat.
Signale explicitement les tests non exécutés et pourquoi.
Mentionne les impacts possibles sur les contrats, les données,
les performances et la journalisation.
```

Lis aussi le diff toi-même. Vérifie notamment qu’aucun test utile n’a été retiré pour obtenir un résultat vert, qu’aucune autorisation n’a été assouplie et qu’aucune dépendance inutile n’a été introduite.

Si l’agent demande de poursuivre, examine ce qui l’a arrêté. Une limite d’itérations, une approbation et un problème de configuration demandent des réponses différentes. La bonne décision dépend de l’avancement observable.

Pour commencer dans une équipe, je privilégierais une petite correction réelle, des instructions courtes et une commande de tests fiable. Les prompts réutilisables, les skills et les intégrations externes pourront ensuite répondre aux besoins qui se répètent.

## Conclusion

GitHub Copilot peut accompagner une grande partie du travail de développement, depuis la compréhension du code jusqu’à la préparation d’un changement vérifiable.

Son efficacité dépend beaucoup de ce que tu lui fournis. Une règle métier explicite, les bons fichiers, des conventions cohérentes et des tests utiles donnent à l’agent une tâche qu’il peut réellement traiter.

Les différents mécanismes ont chacun leur rôle. Les instructions conservent les conventions. Les prompts rendent certaines demandes réutilisables. Les agents personnalisés définissent un rôle et ses outils. Les skills structurent des procédures. MCP donne accès à des capacités externes.

Dans un projet .NET, commence par un besoin concret et un résultat observable. Ajuste ensuite le contexte et le niveau d’autonomie selon les résultats obtenus.

Le changement reste sous la responsabilité de l’équipe qui l’intègre. La valeur de Copilot se mesure donc au code compris, validé et maintenable que tu peux livrer avec son aide.
