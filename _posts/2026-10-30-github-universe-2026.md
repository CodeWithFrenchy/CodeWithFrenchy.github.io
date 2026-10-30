---
title: "GitHub Universe 2026 : les agents s’installent dans toute la chaîne de livraison"
date: 2026-10-30 10:00:00 -0400
categories: [github, intelligence-artificielle]
tags: [github-universe, github-copilot, agents-ia, devops, securite, gouvernance, azure]
---

## Préambule

En regardant les deux keynotes de GitHub Universe 2026, une évolution m’a particulièrement marqué. Après l’assistant qui suggère du code et l’agent auquel on délègue une tâche, GitHub propose maintenant un environnement pour coordonner plusieurs agents et suivre leur travail dans toute la chaîne de livraison.

Le développeur doit toujours comprendre le besoin, prendre les décisions et assumer ce qui est livré. Une part croissante de son travail consiste toutefois à préparer le contexte, orienter l’exécution et vérifier les résultats.

Dans cet article, je reviens sur les annonces des [keynotes du premier jour](https://www.youtube.com/watch?v=MZfqId_rrB0) et du [deuxième jour](https://www.youtube.com/watch?v=ZSMFrM54H2Q), en les croisant avec les publications officielles. Une [version condensée des annonces](https://www.youtube.com/watch?v=xbCa0aa4tRM) permet aussi de revoir rapidement les principaux points.

Les disponibilités et conditions présentées ici correspondent aux informations publiées au 30 octobre 2026. Plusieurs nouveautés sont encore en préversion et certaines annonces mises en valeur pendant Universe avaient été publiées plus tôt en octobre.

## Des chiffres impressionnants, à interpréter correctement

GitHub compte désormais plus de 225 millions de développeurs. Pendant le keynote, l’entreprise souligne qu’un nouveau développeur rejoint la plateforme chaque seconde, pour une deuxième année consécutive.

Le [rapport Octoverse 2026](https://github.blog/news-insights/octoverse/octoverse-commits-pushes-reach-new-highs-in-2026-as-developers-build-with-agents/) présente notamment les volumes suivants pour les douze mois se terminant le 30 septembre 2026.

| Indicateur | Volume annoncé |
| --- | ---: |
| Pushes de code | 17,9 milliards |
| Commits | 34,3 milliards |
| Pull requests fusionnées | 1,3 milliard |
| Contributions publiques | 2,5 milliards |

Les termes comptent. Les 34,3 milliards désignent des commits, tandis que les 1,3 milliard correspondent à des pull requests fusionnées. Ces volumes ne représentent pas uniquement du travail réalisé par des agents.

Le même rapport indique qu’en septembre, plus de 40 % des pull requests commencent par une première version produite avec un agent, contre 17 % en mars. Il observe aussi huit fois plus de commits en moyenne dans les nouveaux dépôts contenant un fichier de configuration d’agent que dans les autres nouveaux dépôts.

J’y vois un changement d’échelle, mais pas une mesure directe de productivité. Le volume de commits dépend aussi du découpage du travail, des points de sauvegarde et des habitudes des agents. Pour une équipe, je regarderais surtout le délai jusqu’à la livraison, les défauts introduits, le temps de révision et le coût des changements acceptés.

## Copilot app et Projects : coordonner le travail entre plusieurs dépôts

La [GitHub Copilot app](https://github.blog/changelog/2026-10-28-github-copilot-app-universe-releases/) rassemble les projets, les sessions et les résultats des agents. Avec Copilot Projects, un objectif peut mobiliser plusieurs dépôts et différentes sources de contexte, comme des maquettes ou des échanges d’équipe.

La démonstration illustre bien l’intérêt de cette approche. On peut transmettre des précisions pendant que le travail avance. L’orchestrateur coordonne les agents et conserve le contexte du projet, plutôt que d’exiger une conversation indépendante pour chaque sous-tâche.

**Copilot Projects est disponible en préversion publique.** L’annonce d’une disponibilité ouverte pendant le keynote ne signifie donc pas que la fonctionnalité a atteint la disponibilité générale.

Pour une application composée d’une API .NET, d’une interface Angular et d’un dépôt d’infrastructure Azure, le potentiel est concret. Une évolution fonctionnelle peut nécessiter des changements coordonnés dans les trois dépôts. Le projet pourrait réunir le contrat d’API, les règles métier et les contraintes de déploiement dans un même contexte de travail.

Cela exige toutefois un bon découpage. Des agents peuvent produire simultanément plusieurs pull requests sans garantir leur compatibilité une fois réunies. Je conserverais des tests de contrat, une validation d’intégration et un responsable de la livraison complète.

### Des intégrations et des outils de vérification

Les [connecteurs vérifiés par GitHub](https://github.blog/changelog/2026-10-28-connect-your-tools-to-copilot-without-all-the-setup/), en préversion publique, simplifient l’accès à des services comme Slack, Figma, Sentry et Cloudflare. L’authentification peut être réutilisée dans plusieurs expériences Copilot. Les politiques de l’organisation et les autorisations du service connecté continuent de s’appliquer.

Les canvas présentés dans le keynote donnent aussi aux agents des surfaces de travail, notamment un navigateur et un terminal. Voir l’agent ouvrir l’application et vérifier un parcours est intéressant. Je demanderais néanmoins que les vérifications importantes deviennent des tests reproductibles, conservés avec le projet.

Enfin, la [continuité des sessions cloud](https://github.blog/changelog/2026-10-28-work-from-anywhere-with-a-new-connected-copilot-cloud-experience/) permet de retrouver le travail depuis VS Code, le CLI, l’application Copilot, le Web ou GitHub Mobile, avec des points d’entrée dans Slack et Teams. Cette expérience est annoncée en disponibilité générale sur les offres Copilot payantes.

## Copilot Fusion : choisir un niveau d’effort et déléguer le choix des modèles

[Copilot Fusion](https://github.blog/changelog/2026-10-28-route-and-orchestrate-models-with-github-copilot-fusion/) est l’une des annonces les plus structurantes. Disponible généralement, il choisit et coordonne les modèles selon la tâche et le niveau d’effort demandé, notamment Standard, High ou Max.

La [documentation de Fusion](https://docs.github.com/en/copilot/concepts/models/copilot-fusion) décrit trois modes d’exécution.

| Mode | Fonctionnement |
| --- | --- |
| Single | Un modèle traite directement la tâche |
| Cascade | Une première proposition est évaluée, puis confiée à un modèle plus puissant au besoin |
| Critique | Une proposition est examinée par d’autres modèles, puis révisée à partir de leurs observations |

Fusion orchestre des modèles. Copilot Projects coordonne le travail des agents. Cette distinction aide à comprendre les deux niveaux d’orchestration présentés pendant l’événement.

GitHub rapporte des résultats proches de Claude Code avec Opus 5.5 Max sur l’Artificial Analysis Coding Agent Index, avec environ la moitié du coût API moyen par tâche et 40 % moins de temps. Ces résultats, rapportés dans l’annonce, concernent ce benchmark. Je les prendrais comme une invitation à comparer Fusion sur nos propres scénarios.

Les modèles utilisés restent soumis aux politiques administratives. La facturation additionne leur consommation et les offres Copilot payantes bénéficient d’une réduction de 30 % sur ces coûts jusqu’au 30 novembre 2026, selon la documentation.

Mon approche serait de commencer avec Standard pour le travail courant, puis d’évaluer les niveaux supérieurs sur des tâches complexes. Le bon indicateur est le coût d’un résultat accepté, en incluant les reprises et la validation humaine.

## Memory et Dreaming : entretenir le contexte devient un travail à part entière

Dans la démonstration, une application a changé de technologie, mais ses instructions font encore référence à l’ancienne implémentation. C’est un problème très réaliste. Un agent qui suit fidèlement une documentation périmée peut produire un résultat cohérent avec de mauvaises consignes.

[GitHub intelligence](https://github.blog/changelog/2026-10-28-github-intelligence-puts-your-code-context-and-history-to-work/) réunit notamment trois capacités.

- **Memory**, disponible généralement sur les offres Copilot payantes, conserve des préférences et des conventions entre les sessions.
- **Dreaming**, en préversion publique et sur activation explicite, analyse le travail passé pour proposer des améliorations au contexte.
- **Code Search** permet de retrouver du code et des informations utiles au-delà du dépôt cloné localement.

Dreaming peut modifier des mémoires. Pour les instructions et les skills versionnés dans un dépôt, les changements sont proposés dans des pull requests à réviser. Ses exécutions consomment des crédits IA.

À mon avis, cette annonce donne une responsabilité supplémentaire aux équipes. Les instructions doivent avoir un propriétaire, une portée claire et un processus de révision. Une préférence personnelle ne devrait pas devenir automatiquement une norme d’architecture.

Le dépôt [skills de Matt Pocock](https://github.com/mattpocock/skills), mis en valeur pendant le keynote et affichant environ 307 000 étoiles au moment de la rédaction, illustre l’intérêt pour ces pratiques réutilisables. Je m’en servirais comme source d’inspiration, en adaptant les consignes aux conventions, aux tests et aux contraintes du projet.

## Copilot Automations : du travail récurrent, avec un budget explicite

Copilot Automations permet de déclencher du travail dans le cloud à la demande, selon un horaire ou à partir d’événements. Les [nouveaux déclencheurs et contrôles](https://github.blog/changelog/2026-10-28-new-tools-triggers-and-cost-controls-for-copilot-automations/) étendent cette possibilité aux appels API externes, aux outils MCP et aux enchaînements d’automatisations.

Les cas d’usage les plus convaincants, pour moi, sont ceux qui produisent un résultat facile à vérifier. Par exemple, préparer un diagnostic après un échec de pipeline, proposer la classification d’une anomalie ou rédiger une mise à jour de documentation à partir de changements fusionnés.

Ces exécutions consomment des crédits IA. GitHub permet maintenant de fixer des limites par exécution, par jour, par semaine ou par mois. Une limite par exécution interrompt le travail concerné. Les limites périodiques empêchent de nouveaux départs une fois le plafond atteint.

Les [automatisations peuvent aussi proposer leurs propres améliorations](https://github.blog/changelog/2026-10-28-copilot-automations-now-learn-from-outcomes-and-self-improve/), en s’appuyant sur les résultats observés. Les modifications proposées doivent être examinées avant leur application.

Je commencerais avec des permissions limitées, un budget réduit et une sortie sous forme de proposition. J’ajouterais les actions d’écriture après avoir mesuré la fiabilité sur un volume représentatif.

### Une autre option pour les équipes qui travaillent avec GitHub Actions

[GitHub Agentic Workflows passe également en disponibilité générale](https://github.blog/changelog/2026-10-28-github-agentic-workflows-generally-available/). Ce projet permet de décrire des tâches en Markdown et de les compiler en workflows GitHub Actions. Il apporte notamment des permissions en lecture seule par défaut, une isolation d’exécution et une validation des actions proposées.

Pour une équipe qui veut versionner ses automatisations et les intégrer à sa gouvernance Actions, cette approche mérite une évaluation distincte de Copilot Automations. Il faut comparer les possibilités d’exécution, les permissions et les coûts selon le scénario.

## Code review Max : la révision devient un enjeu de capacité

Produire davantage de code augmente la quantité de changements à comprendre. La révision peut rapidement devenir le point qui limite le débit de l’équipe.

[Copilot code review Max](https://github.blog/changelog/2026-10-28-copilot-code-review-merge-with-confidence-with-max-and-approvals/), en préversion publique, ajoute une analyse plus approfondie aux niveaux Lite et Balanced. Plusieurs agents utilisant des modèles de fournisseurs différents examinent les changements, puis leurs observations sont consolidées. Ce niveau demande davantage de temps et de crédits.

La même annonce rend les approbations Copilot généralement disponibles. Lorsqu’elles sont activées, elles peuvent compter parmi les approbations exigées par le dépôt.

Cette possibilité mérite une décision de gouvernance explicite. Pour un changement touchant l’authentification, les habilitations ou une migration de données, je conserverais une approbation humaine obligatoire. L’analyse automatisée peut améliorer la révision, mais la responsabilité du changement reste celle de l’équipe.

Je privilégierais aussi des pull requests de taille raisonnable. Une analyse plus puissante aide peu si le changement mélange une refonte, une migration et plusieurs nouvelles règles métier.

Enfin, il faut distinguer l’accès à la fonctionnalité de sa consommation. Le nouveau modèle Enterprise, décrit plus loin, ouvre une voie sans abonnement Copilot distinct pour les utilisateurs admissibles. Cela ne rend pas Code review Max universellement accessible sans conditions ni coûts.

## Sécurité : plusieurs mécanismes complémentaires

Les annonces de sécurité couvrent différents moments du développement. Les regrouper sous une seule appellation comme « Copilot scanning » ferait perdre des distinctions utiles.

| Fonctionnalité | Rôle | Statut annoncé |
| --- | --- | --- |
| [AI Scan pour les pull requests](https://github.blog/changelog/2026-10-15-ai-scan-for-pull-requests-generally-available/) | Compléter CodeQL pour des langages et frameworks qu’il ne couvre pas | Disponibilité générale depuis le 15 octobre |
| [AI Scan pour les dépôts](https://github.blog/changelog/2026-10-28-ai-scan-for-repositories-in-public-preview/) | Rechercher des vulnérabilités à l’échelle d’un dépôt | Préversion publique |
| [Protection des pushes avec détection IA](https://github.blog/changelog/2026-10-28-push-protection-beyond-predefined-secret-patterns/) | Bloquer aussi des secrets sans format prédéfini, selon leur contexte | Préversion publique |
| [Extension de sécurité de Copilot app](https://github.blog/changelog/2026-10-28-security-extension-for-the-github-copilot-app/) | Examiner les alertes et demander une analyse ou une correction depuis l’application | Préversion publique |
| [Analyse des applications IA](https://github.blog/changelog/2026-10-28-code-scanning-for-ai-applications-in-public-preview/) | Détecter certains risques propres aux agents, dont les injections indirectes de prompt | Préversion publique |

Quelques limites influencent directement une adoption en entreprise. [AI Scan pour les pull requests](https://github.blog/changelog/2026-10-15-ai-scan-for-pull-requests-generally-available/) doit être activé et ses observations sont consultatives. Elles ne font pas échouer le contrôle de code scanning et ne bloquent pas la fusion par les règles de dépôt. Les pull requests provenant de forks ou créées par Dependabot sont exclues de cette analyse.

Il serait donc inexact de conclure que toutes les pull requests sur GitHub sont automatiquement révisées et sécurisées par l’IA.

La [nouvelle protection des pushes](https://github.blog/changelog/2026-10-28-push-protection-beyond-predefined-secret-patterns/) est désactivée par défaut. Elle nécessite l’activation des fonctions requises de détection et de protection. Les [analyses AI Scan des dépôts](https://github.blog/changelog/2026-10-28-ai-scan-for-repositories-in-public-preview/) consomment pour leur part des crédits Code Security AI, avec des conditions d’accès liées au type de dépôt et aux produits de sécurité souscrits.

Mon interprétation est que GitHub cherche à rapprocher la détection du moment où le risque apparaît. Pour en tirer parti, il faut préserver plusieurs contrôles complémentaires, dont les tests, l’analyse statique, la gestion des dépendances et la révision des changements sensibles.

### La chaîne d’approvisionnement reçoit aussi des améliorations

Les [fichiers de verrouillage des workflows Actions](https://github.blog/changelog/2026-10-28-github-actions-workflow-lockfiles-are-generally-available/) passent en disponibilité générale. Ils permettent de conserver des références lisibles tout en exécutant les commits verrouillés des actions et de leurs dépendances transitives. C’est un gain de reproductibilité qui mérite de l’attention, même sans agents IA.

Deux préversions concernent aussi les dépendances applicatives. Le [package firewall](https://github.blog/changelog/2026-10-20-github-package-firewall-in-public-preview/), annoncé le 20 octobre, filtre les téléchargements selon les politiques de l’entreprise et prend actuellement en charge npm. L’[analyse d’accessibilité des vulnérabilités Dependabot](https://github.blog/changelog/2026-10-28-prioritize-dependabot-alerts-with-reachability-analysis/) recherche les chemins d’appel vers des fonctions vulnérables en JavaScript et TypeScript. Ces possibilités peuvent intéresser une équipe Angular, mais leur portée actuelle ne doit pas être étendue à NuGet par supposition.

## GitHub Enterprise : l’accès évolue, la consommation reste à financer

L’[annonce sur l’accès Copilot avec GitHub Enterprise](https://github.blog/changelog/2026-10-28-github-enterprise-now-includes-access-to-github-copilot/) réduit la nécessité d’un abonnement Copilot distinct pour les utilisateurs titulaires d’une licence Enterprise dans le nouveau modèle.

Au lancement, cette offre concerne GitHub Enterprise Cloud et les entreprises accompagnées par une équipe commerciale GitHub ou Microsoft. L’ouverture aux abonnements en libre-service est progressive. Les offres Copilot Business et Copilot Enterprise demeurent disponibles.

Ce modèle n’inclut pas d’allocation mensuelle de crédits IA par siège. Les usages IA concernés sont facturés dès la première requête sur le budget de l’entreprise. La complétion de code et les suggestions de prochaine modification continuent toutefois de fonctionner même sans budget disponible.

Pour une organisation, cela déplace une partie de la gestion vers la consommation. Je comparerais les modèles sur une période réelle d’utilisation, en séparant les usages interactifs, les révisions et les automatisations. Les coûts d’exécution cloud et les produits de sécurité doivent également entrer dans l’évaluation, selon les fonctions retenues.

L’enjeu est de savoir combien coûte le travail livré avec un niveau de qualité acceptable, puis d’attribuer cette dépense aux bonnes équipes.

## L’infrastructure Git évolue pour absorber cette charge

GitHub annonce avoir ajouté quatre millions de cœurs CPU, 160 pétaoctets de stockage rapide et 50 térabits par seconde de capacité réseau en 2026. L’entreprise vise aussi le début de 2027 pour terminer sa migration vers un déploiement Azure multirégion, selon sa [présentation de la stratégie et des investissements](https://github.blog/news-insights/company-news/github-universe-2026-evolving-every-layer-of-the-stack/).

La [refonte de l’infrastructure Git](https://github.blog/engineering/architecture-optimization/building-git-infrastructure-for-agent-scale-development/) repose notamment sur la séparation du stockage et du calcul, la réduction de la coordination et le déplacement de la maintenance hors du chemin des requêtes. Azure Blob Storage porte les données durables, tandis que des composants de calcul peuvent évoluer indépendamment pour servir les lectures.

Le problème est intéressant du point de vue de l’architecture. Dans un système où chaque réplique de lecture participe aussi aux écritures, ajouter de la capacité de lecture peut ralentir les pushes. Séparer les responsabilités permet de réduire ce couplage.

L’échéance annoncée concerne la migration de l’infrastructure de GitHub. Elle ne constitue pas une date limite imposant aux équipes de migrer leurs dépôts vers un nouveau format Git.

J’y vois aussi un rappel pour nos propres plateformes. Avant de multiplier les agents, il faut mesurer la capacité des pipelines, des environnements de test et du processus de fusion à absorber leur travail.

## Deuxième journée : expérimenter, vérifier et mieux planifier

La deuxième keynote m’a semblé moins centrée sur les lancements de produits. Elle met davantage en valeur la communauté, l’expérimentation et la manière de travailler avec les agents.

Les badges programmables étaient particulièrement réussis. La [démonstration de Martin Woodward](https://www.youtube.com/watch?v=ZSMFrM54H2Q&t=1208s) présente un appareil basé sur un RP2350, des capteurs et un environnement de simulation. Le passage sur le podomètre combine télémétrie, collecte de données, entraînement d’un petit modèle et exécution sur le badge.

Ce que je retiens surtout est la boucle de vérification. L’agent peut agir, observer le résultat et corriger sa proposition. Dans nos projets, cela renforce l’intérêt d’un environnement reproductible, de tests fiables et de diagnostics accessibles. Ces bonnes pratiques bénéficient autant aux développeurs qu’aux agents.

### Chopin : explorer la planification collective

[Chopin](https://githubnext.com/projects/chopin/), présenté par GitHub Next, explore un espace où plusieurs personnes et un agent peuvent rédiger un plan ensemble. Documents partagés, commentaires, recherches et décisions doivent permettre de construire une compréhension commune avant l’implémentation.

Pour moi, c’est une piste intéressante. Quand plusieurs agents peuvent réaliser rapidement des changements, une ambiguïté dans le besoin peut se propager dans plusieurs dépôts. Investir dans la qualité du plan et des critères d’acceptation devient encore plus utile.

Le statut doit toutefois rester clair. Le projet a été présenté publiquement dès septembre 2026 et son [README](https://github.com/githubnext/chopin/blob/main/README.md) le décrit comme un prototype de recherche expérimental. Ce n’est pas un service GitHub pris en charge pour la production. Je le garderais dans un environnement d’exploration pour évaluer ses idées et leur pertinence pour les pratiques d’équipe.

## Suivre Universe à distance demande encore quelques détours

Mon bémol concerne l’accès rapide au contenu technique lorsqu’on suit l’événement à distance. Les keynotes présentent bien la direction, mais les équipes ont aussi besoin des détails pour préparer leurs expérimentations et transmettre les apprentissages à l’interne.

Il faut toutefois nuancer l’idée que seules les keynotes sont accessibles en ligne. La [formule virtuelle officielle](https://githubuniverse.com/pricing) prévoit aussi l’accès aux sessions enregistrées lorsqu’elles sont disponibles. Le [récapitulatif de Microsoft Command Line](https://commandline.microsoft.com/github-universe-2026-recap/) présente plusieurs sujets techniques, dont la sécurité MCP, la maîtrise du contexte et l’évaluation des modèles. Le dépôt [virtual-universe-26](https://github.com/octodemo/virtual-universe-26), relevé dans mes notes, reste également une piste à explorer.

## Conclusion

GitHub Universe 2026 confirme une ambition qui dépasse la génération de code. GitHub veut réunir le contexte, la planification, l’exécution, la révision et la sécurité dans un environnement où les développeurs coordonnent le travail de plusieurs agents.

Les annonces les plus intéressantes répondent à des difficultés déjà visibles dans les équipes. Choisir les modèles, maintenir les instructions, coordonner plusieurs dépôts et absorber la révision deviennent des activités importantes à mesure que les agents prennent de la place.

Mon principal constat est que les pratiques logicielles restent déterminantes. Des objectifs précis, des tests utiles, des changements compréhensibles et une responsabilité claire donnent aux agents de meilleures conditions pour contribuer.

J’évaluerais donc cette nouvelle génération d’outils sur sa capacité à livrer des changements fiables à un coût maîtrisé. C’est là que les démonstrations d’Universe doivent maintenant faire leurs preuves dans nos projets.
