---
title: "Moderniser une application .NET avec GitHub Copilot : de l’évaluation à la validation"
date: 2026-11-17 19:00:00 -0400
categories: [dotnet, modernisation]
tags: [dotnet, dotnet11, github-copilot, modernisation, migration, aspnet-core, tests]
---

>  ⚠️ IMAGE

## Préambule

Moderniser une application .NET demande de comprendre ce qui la fait fonctionner aujourd’hui. Les projets et les packages racontent une partie de l’histoire. Les règles métier, les intégrations et les particularités de production complètent le portrait.

Dans la session [GitHub Copilot modernization de .NET Conf 2026](https://www.youtube.com/watch?v=cdbwHdGikhg), Mika Dumont présente la migration d’une application Web Forms vers ASP.NET Core. L’agent évalue la solution, produit un plan, applique des changements et tente de résoudre les problèmes rencontrés. Le développeur peut préciser ses choix, notamment conserver Entity Framework 6 pour limiter les changements dans l’accès aux données.

Cette capacité peut réduire le temps consacré aux transformations répétitives et à l’exploration d’un dépôt. Elle rend aussi les premières expérimentations plus accessibles. Pour une application métier, il reste toutefois à démontrer que le résultat respecte les comportements attendus et les contraintes d’exploitation.

J’ai déjà couvert plusieurs possibilités de la plateforme dans [mon article sur .NET 11](https://codewithfrenchy.com/posts/dotnet11/). Ici, je m’intéresse au chemin pour y arriver. Nous allons suivre un exemple illustratif de migration progressive, de l’inventaire jusqu’à la validation. Il sert à présenter une méthode de travail, sans prétendre reproduire une migration exécutée pour cet article.

## Choisir l’agent et définir son mandat

La documentation distingue deux responsabilités. [GitHub Copilot upgrade](https://learn.microsoft.com/en-us/dotnet/core/porting/github-copilot-upgrade/overview) accompagne les changements de frameworks, de dépendances et de code. [GitHub Copilot modernization](https://learn.microsoft.com/en-us/dotnet/azure/migration/appmod/overview) couvre les migrations vers Azure, avec des scénarios liés aux services, à la configuration et au déploiement. Ce dernier peut faire appel à l’agent d’upgrade lorsqu’une montée de version .NET est nécessaire.

Pour notre migration applicative, le point de départ est l’agent d’upgrade. Les [instructions d’installation](https://learn.microsoft.com/en-us/dotnet/core/porting/github-copilot-upgrade/install) précisent les prérequis selon l’environnement. Dans Visual Studio, l’entrée passe notamment par **Modernize** ou `@Modernize`. Dans Visual Studio Code, on retrouve `@upgrade`.

Avec GitHub Copilot CLI, le [dépôt officiel du plugin](https://github.com/microsoft/upgrade-agent-plugins) fournit cette installation, à effectuer dans la session interactive de Copilot :

```text
/plugin marketplace add microsoft/upgrade-agent-plugins
/plugin install upgrade-agent@upgrade-agent-plugins
/agent
```

On sélectionne ensuite l’agent Upgrade. Ces commandes appartiennent à Copilot CLI et ne sont pas des commandes PowerShell.

L’environnement doit aussi pouvoir compiler l’application existante. Pour une solution Web Forms historique, il faut conserver les outils Windows, les composants Visual Studio et les versions de MSBuild nécessaires. La présence du SDK .NET cible ne garantit pas que les anciens projets se compilent dans cet environnement.

### Un périmètre concret

Prenons une application de gestion de catalogue composée de pages Web Forms sous .NET Framework 4.8, de bibliothèques C#, d’EF6 et d’une base de données SQL Server. Elle utilise Forms Authentication, distingue les lecteurs des gestionnaires et produit un export CSV consommé par un autre système.

L’objectif proposé est un nouvel hôte ASP.NET Core sous .NET 11, avec une interface Blazor côté serveur. Les parcours seront transférés progressivement. Nous conservons Windows comme environnement d’hébergement initial et reportons la migration vers Azure à une étape distincte.

Le choix de la version cible appartient à l’équipe. Il faut vérifier la [politique de support .NET](https://dotnet.microsoft.com/en-us/platform/support/policy), les dépendances et les contraintes de déploiement. Une application .NET Framework 4.8 peut encore être supportée sur un Windows supporté. La justification de sa modernisation doit donc préciser les gains recherchés, par exemple faciliter les évolutions, remplacer une dépendance bloquante ou améliorer l’exploitation.

Pour cet exemple, je retiens trois critères de réussite : préserver les règles du catalogue, permettre une transition par parcours et conserver un retour arrière praticable pendant cette transition.

## Établir un état de référence avant les modifications

Avant de confier les transformations à l’agent, je veux une version connue de l’application et des preuves de son fonctionnement.

Sur une révision Git identifiée, l’équipe reproduit la restauration, la compilation, les tests et la publication avec les commandes du dépôt. Elle conserve les versions des outils, les journaux et les échecs déjà présents. Un test en échec avant la migration doit rester visible dans le bilan, avec son statut et son responsable.

Les tests existants donnent rarement une vision complète d’une application ancienne. J’ajouterais donc quelques tests de caractérisation sur les parcours importants : chercher un produit, modifier son prix, refuser une modification sans autorisation et exporter le catalogue.

Un test de caractérisation capture un comportement observé. L’équipe doit ensuite confirmer que ce comportement mérite d’être conservé. Un défaut historique connu peut faire l’objet d’une correction explicite, avec sa propre décision et ses propres critères d’acceptation.

Le même principe s’applique à la performance. On relève les temps des parcours représentatifs, la consommation mémoire et l’activité SQL dans un environnement contrôlé. Ces observations permettront de détecter une régression et de vérifier les gains annoncés après la migration.

Copilot peut aider à préparer ces tests et à documenter les commandes. La sélection des comportements de référence demande la connaissance du domaine et des usages réels.

## Faire un inventaire qui dépasse les fichiers projet

L’évaluation doit produire des éléments vérifiables. « Ce package semble ancien » apporte peu de valeur. Une observation utile identifie sa version, les projets concernés, les API utilisées et la raison précise d’un remplacement éventuel.

Je demanderais à l’agent de couvrir au moins ces dimensions :

- Les projets, leurs frameworks cibles et leurs relations, y compris les références vers des assemblages internes
- Les dépendances NuGet directes et transitives, les sources privées et les mécanismes de gestion des versions
- Les usages de `System.Web`, les composants Windows, les bibliothèques natives et les extensions chargées dynamiquement
- L’authentification, les autorisations, l’état de session, la configuration et les accès au système de fichiers
- EF6, son fournisseur SQL, les transactions, les procédures stockées et les outils de gestion du modèle
- Les pipelines, les scripts de publication et les traitements déclenchés en dehors de l’application web

Il faut notamment lire `packages.config`, les fichiers projet et les imports MSBuild. Dans un dépôt plus récent, `Directory.Packages.props` ou `Directory.Build.props` peuvent porter des choix qui ne sont pas visibles dans chaque projet.

Le dépôt ne révèle pas nécessairement une tâche planifiée sur un serveur, une configuration IIS modifiée manuellement ou une intégration utilisée une fois par mois. L’inventaire doit être complété avec les équipes qui exploitent l’application.

Voici une demande initiale que l’on peut adapter :

```text
Évalue cette solution pour une migration progressive vers .NET 11.
Utilise le mode guidé et arrête-toi après l’évaluation.

Tu peux produire les documents d’analyse, mais ne modifie pas encore
le code applicatif, les dépendances ni les configurations de déploiement.

Identifie les projets, leurs dépendances et les commandes nécessaires
pour restaurer, compiler, tester et publier l’application actuelle.

Pour chaque incompatibilité, indique le fichier ou la dépendance
concernée, la preuve disponible et une référence officielle pertinente.
Distingue les constats confirmés, les hypothèses et les points inconnus.

Évalue la conservation d’EF6 et du schéma de la base de données.
Signale les décisions concernant l’authentification, les données
et les contrats utilisés par d’autres applications.

Propose les vérifications nécessaires pour lever les incertitudes.
N’invente pas de résultat pour une commande que tu ne peux pas exécuter.
```

L’intérêt de cette première étape est de rendre le périmètre discutable avant d’accumuler des changements. Un rapport qui montre clairement ce qu’il ignore est plus exploitable qu’une estimation précise construite sur des suppositions.

## Transformer les incompatibilités en décisions

Je classe les observations selon ce qu’elles empêchent : compiler, démarrer, préserver un comportement ou exploiter l’application. Cette distinction évite de traiter toutes les erreurs comme des corrections de syntaxe.

Le [catalogue des changements incompatibles de .NET 11](https://learn.microsoft.com/en-us/dotnet/core/compatibility/11) distingue lui aussi les incompatibilités binaires, les incompatibilités de code source et les changements de comportement. Pour un saut de plusieurs versions, l’analyse doit couvrir les versions traversées et les bibliothèques mises à jour.

### Web Forms exige une transformation de l’interface

ASP.NET Core n’exécute pas le modèle Web Forms. Les pages et leur cycle de vie doivent être adaptés à une technologie prise en charge, comme Blazor ou Razor Pages. Les [explications sur la migration incrémentale de Web Forms](https://devblogs.microsoft.com/dotnet/systemweb-adapters-1_2/) précisent cette limite.

L’agent dispose de [scénarios de migration vers Blazor Server](https://learn.microsoft.com/en-us/dotnet/core/porting/github-copilot-upgrade/dotnet-scenarios-and-skills). Il peut préparer des composants et transformer du code. L’équipe doit néanmoins choisir le modèle d’interface et valider les différences de navigation, d’état et d’interaction.

Dans notre catalogue, un événement de page qui chargeait des données à chaque requête doit être réexaminé dans le cycle de vie Blazor. Une apparence identique dans le navigateur ne démontre pas que les données sont rafraîchies au même moment ni que les actions sont autorisées au bon endroit.

### Conserver EF6 peut réduire le périmètre

La migration du runtime n’impose pas systématiquement une migration immédiate vers EF Core. La [documentation EF6](https://learn.microsoft.com/en-us/ef/ef6/what-is-new/) décrit sa prise en charge de .NET moderne ainsi que des limites, notamment autour du concepteur EDMX.

Pour notre scénario, je commencerais par vérifier la version d’EF6, le fournisseur SQL, les API utilisées et les outils nécessaires à la maintenance du modèle. La conservation d’EF6 devient une décision conditionnelle à cette vérification, puis aux tests d’intégration.

Cela permet de reporter une transformation importante des requêtes et du modèle de persistance. Le coût est de maintenir temporairement cette technologie et de planifier la suite. L’équipe doit accepter ce compromis explicitement.

### Une connexion qui échoue peut révéler un changement utile

Le passage de `System.Data.SqlClient` à certaines versions de `Microsoft.Data.SqlClient` peut modifier les exigences de chiffrement de la connexion. Les scénarios de l’agent prennent en compte ce type de différence.

Si la connexion échoue sur un certificat, une correction qui désactive sa validation mérite une revue. L’équipe doit comprendre la configuration attendue, la chaîne de confiance et le nom du serveur. Faire disparaître l’erreur ne suffit pas à démontrer que la nouvelle configuration convient à l’environnement cible.

## Construire un plan par changements vérifiables

Le [parcours officiel d’upgrade](https://learn.microsoft.com/en-us/dotnet/core/porting/github-copilot-upgrade/dotnet-how-to-upgrade-with-github-copilot) conserve les documents de travail sous `.github/upgrades/{scenarioId}`. On y retrouve notamment `assessment.md`, `upgrade-options.md`, `plan.md` et `tasks.md`. Ces documents peuvent être relus et ajustés avant de poursuivre.

Pour une migration importante, je choisirais le mode guidé. Les pauses entre les étapes permettent de confirmer les décisions qui vont orienter les transformations.

Notre plan pourrait prendre cette forme :

| Lot | Travail confié à Copilot | Validation attendue de l’équipe |
| --- | --- | --- |
| Préparation | Documenter l’état initial et proposer les tests manquants | Confirmer les règles métier et les comportements à conserver |
| Bibliothèques | Adapter les projets et les dépendances compatibles | Vérifier les consommateurs et la compatibilité avec les deux hôtes |
| Nouvel hôte | Préparer ASP.NET Core et la configuration de coexistence | Approuver le routage, l’identité et les frontières de confiance |
| Consultation | Migrer la recherche et la fiche produit | Comparer les résultats, les autorisations et la navigation |
| Modification et export | Migrer les écritures et la production du CSV | Vérifier les transactions, la concurrence et le contrat du fichier |
| Livraison | Préparer les changements de pipeline et le rapport de validation | Autoriser le déploiement et confirmer le retour arrière |

L’ordre dépend du graphe de projets. Une bibliothèque partagée ne peut pas simplement cibler .NET 11 si l’application .NET Framework doit encore la consommer. Il faut conserver une compatibilité adaptée ou prévoir un ciblage multiple, puis tester les deux usages.

La [documentation de migration ASP.NET vers ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/migration/fx-to-core/?view=aspnetcore-10.0) explique ces contraintes de dépendances et les compromis d’une migration incrémentale.

### Séparer coexistence et découpage des commits

Créer un nouvel hôte à côté de l’ancien rend la coexistence possible. Pour livrer progressivement, il faut aussi définir quels parcours passent par chaque application, comment l’identité est transmise et comment revenir au parcours précédent.

Un proxy inverse comme YARP peut participer à ce routage, comme le présente le [guide de mise en place d’une migration incrémentale](https://learn.microsoft.com/en-us/aspnet/core/migration/fx-to-core/start?view=aspnetcore-10.0). Son introduction ajoute toutefois un composant à configurer, à observer et à exploiter. Pour une petite application, une bascule complète après validation peut rester plus économique.

Dans notre scénario, nous transférons d’abord les consultations, puis les écritures. Les cookies et les sessions de l’ancienne application ne sont pas supposés compatibles automatiquement avec le nouvel hôte. Cette intégration fait l’objet d’une tâche explicite.

Des commits progressifs restent utiles dans les deux stratégies. Chaque commit doit porter un changement compréhensible et ses vérifications. Une même pull request qui remplace l’interface, l’ORM, le mécanisme d’identité et l’hébergement rendrait les écarts beaucoup plus difficiles à expliquer.

## Exécuter les transformations avec des limites précises

Copilot peut convertir des fichiers projet, ajuster des références, remplacer des appels connus, lancer des compilations et exploiter leurs diagnostics. Ces opérations sont de bonnes candidates à l’automatisation lorsque le résultat attendu est explicite.

Après chaque lot, je veux examiner le diff et les preuves de validation. Une baisse du nombre de tests exécutés, l’ajout de tests ignorés ou la suppression d’une vérification d’autorisation doivent être expliqués.

On peut encadrer l’exécution ainsi :

```text
Exécute uniquement le lot de consultation du catalogue approuvé.

Préserve les règles de filtrage et d’autorisation décrites dans le plan.
Conserve EF6 et le schéma de la base de données selon les décisions validées.

Si une incompatibilité exige un remplacement de bibliothèque ou
une modification de comportement, documente le blocage avant de poursuivre.

N’ignore pas de test et ne modifie pas une assertion métier uniquement
pour obtenir une exécution réussie.

À la fin, fournis les fichiers modifiés, les commandes réellement
exécutées, leurs résultats et les validations encore manquantes.
Prépare un commit limité à ce lot et arrête-toi pour la revue.
```

Le [tableau de bord Upgrade Dashboard](https://learn.microsoft.com/en-us/dotnet/core/porting/github-copilot-upgrade/dashboard) aide à suivre les tâches, les dépendances et l’activité dans GitHub Copilot CLI et l’application GitHub Copilot. Cette vue n’est pas actuellement disponible dans Visual Studio et Visual Studio Code. Les documents du dépôt et les journaux restent donc des éléments importants du suivi, quel que soit l’environnement.

Il faut également accepter que l’évaluation évolue. Une tâche peut révéler un composant oublié ou nécessiter plusieurs sous-tâches. Le plan doit alors être révisé avec l’impact sur les délais et les validations attendues.

## Valider les comportements, les données et l’exploitation

Une compilation réussie confirme une partie de la compatibilité technique. Les tests unitaires apportent une autre preuve. Pour accepter notre migration, nous devons également exercer les parcours qui traversent l’interface, les services et SQL Server.

Voici quelques critères que je définirais avant de laisser l’agent transformer ces parcours :

| Scénario | Résultat à démontrer |
| --- | --- |
| Un lecteur tente de modifier un prix | L’action est refusée côté serveur et aucune donnée métier n’est modifiée |
| Un gestionnaire enregistre un prix autorisé | Le montant, l’audit et les règles d’arrondi respectent le contrat validé |
| Deux gestionnaires modifient le même produit | La stratégie de concurrence choisie est appliquée, sans perte silencieuse d’une modification |
| Une session expire pendant un parcours | L’application applique le comportement convenu et protège les actions suivantes |
| Le catalogue est exporté | L’encodage, les colonnes, les formats et les valeurs restent compatibles avec le système consommateur |
| Un parcours revient vers l’ancien hôte | L’ancienne application peut encore lire et traiter les données produites par la nouvelle |

Ces critères expriment les attentes de notre scénario. Ils doivent être confirmés par l’équipe métier et adaptés à l’application réelle.

### Vérifier les tests produits par l’agent

L’agent peut générer des tests pour accompagner ses transformations. Il faut vérifier ce qu’ils démontrent. Un test qui confirme seulement qu’une page répond ou qu’aucune exception n’est levée protège peu de choses sur les autorisations et les écritures.

Je conserverais des attentes définies avant la transformation, puis je vérifierais qu’une régression volontaire les fait échouer. Par exemple, autoriser temporairement un lecteur à modifier un produit doit provoquer un échec du test correspondant.

Une seconde revue peut aider à repérer des oublis. La preuve reste le lien entre une règle attendue, une assertion et un scénario qui échoue lorsque cette règle est violée.

### Tester avec les dépendances représentatives

Pour les parcours de données, je privilégierais un environnement SQL Server de test avec des données synthétiques représentatives. Un fournisseur en mémoire ne reproduit pas les mêmes contraintes, transactions et comportements de concurrence.

Si les deux applications partagent la même base de données pendant la transition, il faut également vérifier leurs accès simultanés. Des comparaisons d’écritures doivent utiliser des jeux de données isolés pour éviter de déclencher deux fois les mêmes effets.

La validation passe ensuite par un pipeline qui restaure les dépendances et reconstruit les artefacts dans un environnement propre. Cela permet de repérer une référence locale, un composant installé seulement sur le poste du développeur ou une configuration absente du dépôt.

### Mesurer les gains et préparer le retour arrière

Les performances doivent être comparées sur les mêmes parcours, avec un volume de données, une charge et des ressources comparables. On peut suivre les latences aux percentiles 95 et 99, le débit, la mémoire et les appels SQL.

Avec une interface Blazor côté serveur, je mesurerais aussi les interactions après la connexion et le comportement lors des déconnexions. Une mesure du chargement initial ne décrit pas toute l’expérience utilisateur.

Enfin, un retour à l’ancien binaire n’annule pas les écritures déjà effectuées. Même sans changement de schéma, une nouvelle valeur ou une nouvelle interprétation d’un champ peut empêcher l’ancienne application de fonctionner correctement. Le retour arrière doit donc être testé avec des données produites par la version migrée.

L’équipe décide du moment de la bascule selon les résultats observés, les anomalies restantes et les indicateurs disponibles en production. Copilot peut préparer le rapport et les scripts, mais cette décision exige une compréhension des conséquences opérationnelles.

## Évaluer le bénéfice pour l’équipe

Le gain de Copilot se mesure sur l’ensemble du travail. Je suivrais le temps consacré à l’inventaire, aux transformations, aux corrections et à la revue, ainsi que les régressions découvertes après validation.

Une génération rapide peut produire un diff difficile à relire. À l’inverse, des changements plus petits peuvent réduire le temps nécessaire pour comprendre un problème et autoriser une livraison. Le bon indicateur est le délai pour obtenir une migration vérifiée et exploitable.

L’automatisation apporte surtout de la valeur lorsqu’elle libère du temps pour les décisions qui demandent du contexte : choisir une stratégie de coexistence, confirmer un contrat d’intégration, accepter une modification métier et organiser la transition en production.

## Conclusion

GitHub Copilot peut accélérer plusieurs étapes d’une modernisation .NET, depuis l’exploration du dépôt jusqu’aux corrections guidées par les diagnostics. L’évaluation, le plan et les tâches deviennent des éléments que l’équipe peut relire et faire évoluer.

Pour en tirer profit, je commencerais par un état de référence, un périmètre explicite et quelques comportements métier bien protégés. Je confierais ensuite à l’agent des transformations progressives, chacune accompagnée de preuves vérifiables.

La réussite se constate lorsque l’application migrée remplit ses engagements, que ses gains sont mesurés et que l’équipe sait la déployer, l’observer et gérer un retour arrière. C’est à ce niveau que le temps économisé par l’IA peut se transformer en capacité durable à moderniser.
