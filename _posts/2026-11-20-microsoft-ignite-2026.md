---
title: "Microsoft Ignite 2026 : de Copilot aux agents d’entreprise"
date: 2026-11-20 09:00:00 -0400
categories: [Microsoft, Azure]
tags: [Microsoft Ignite, Azure, Copilot, intelligence artificielle, agents]
description: "Les annonces marquantes de Microsoft Ignite 2026 : Copilot, Microsoft IQ, Foundry, Azure, sécurité et gouvernance des agents."
---

À Ignite 2026, Microsoft a présenté plusieurs annonces autour d’une même idée : faire travailler ensemble Copilot, les agents, les données et l’infrastructure de l’entreprise. Le keynote décrit une plateforme où les organisations peuvent utiliser leur propre contexte et garder le contrôle sur les agents qu’elles déploient.

Dans le [keynote d’ouverture](https://www.youtube.com/watch?v=ZR4v1c4CClc), Microsoft articule cette vision autour de trois priorités : garder les personnes au centre, permettre aux organisations de développer leur propre intelligence et établir la confiance nécessaire à un déploiement à grande échelle. Le [Book of News de Microsoft Ignite 2026](https://news.microsoft.com/events/ignite/2026/book-of-news/) précise les annonces et leur disponibilité.

## Copilot devient un point d’entrée pour le travail

Copilot réunit la conversation, la collaboration, la création de contenu et l’exécution de tâches dans une même expérience. Les fonctions de Word, Excel et PowerPoint sont accessibles dans Copilot. Une personne peut demander un document, un classeur ou une présentation en langage naturel, puis poursuivre le travail dans les applications Office au besoin.

Copilot Workspaces permet à une équipe de rassembler documents, échanges et tâches dans un espace partagé. Copilot aide à retrouver le contexte, à voir ce qui a changé et à poursuivre un travail récurrent. Workspaces est offert dans l’application Copilot par l’intermédiaire du programme Frontier.

Autopilot est un agent de productivité qui peut suivre des fils de discussion, exécuter des tâches récurrentes et poursuivre un projet sans attendre une nouvelle demande. Il fonctionne dans le tenant de l’organisation, avec sa propre identité et des accès contrôlés. Comme tout agent persistant, il lui faut un propriétaire, des permissions adaptées et des limites opérationnelles.

Copilot Code permet de créer des applications et des outils internes à partir d’une description en langage naturel. Les projets peuvent s’exécuter dans un environnement isolé et rester dans le tenant de l’organisation. Copilot Managed Runtime prend en charge leur publication et leur gouvernance. Copilot Code est offert dans le programme Frontier. Microsoft prévoit sa disponibilité générale en décembre 2026.

## Microsoft IQ fournit le contexte organisationnel

Microsoft IQ rassemble le contexte provenant des données, des applications et des pratiques internes pour le rendre accessible à Copilot et aux agents développés par l’entreprise.

Le keynote présente plusieurs composantes : Work IQ pour le contexte de travail, Fabric IQ pour le contexte analytique et Foundry IQ pour les connaissances non structurées. Microsoft annonce aussi des centaines de connecteurs et des plug-ins d’affaires qui donnent aux agents accès à des connaissances et à des actions. Le partenariat avec Salesforce en est un exemple. Les équipes peuvent consulter du contexte client et exécuter certaines opérations de Salesforce depuis Copilot et Teams.

Un contexte partagé peut éviter de multiplier les intégrations particulières entre applications. Il faut toutefois définir clairement les accès, la provenance des données et les limites de chaque agent. Les organisations devront aussi désigner les responsables de ce contexte et établir des critères de qualité.

## Foundry soutient l’évaluation et l’amélioration des agents

Microsoft Foundry sert à construire, déployer et faire évoluer des agents. Les annonces mettent l’accent sur une boucle d’amélioration continue. Les équipes peuvent évaluer la qualité, la sécurité, la latence et les coûts, observer les traces de production, regrouper les problèmes récurrents, puis comparer des changements avant leur déploiement.

Elles pourront évaluer des conversations complètes, générer des critères à partir d’exemples vérifiés et annoter les traces avec des évaluations d’experts. Agent Optimizer proposera des changements et les mesurera à partir des évaluations de l’organisation. Certaines fonctions sont disponibles de façon générale. Les annotations humaines et les nouvelles options d’entraînement sont en aperçu.

Ces pratiques rapprochent le cycle de vie d’un agent de celui d’un service logiciel en production. Les équipes doivent mesurer son comportement sur des scénarios représentatifs, surveiller les résultats après sa mise en ligne et vérifier les régressions lorsqu’elles modifient ses instructions, ses outils ou son modèle. Elles peuvent ainsi suivre les effets d’un changement sur la sécurité, les coûts et les cas limites, en plus de la qualité des réponses.

Microsoft annonce plusieurs modèles spécialisés : MAI-Code-2 pour le développement logiciel, MAI-Code-2-Flash pour les tâches de codage à grand volume et MAI-Cyber-2-Flash pour la cybersécurité. Microsoft Decision 1.1, maintenant disponible, cible les décisions structurées comme le routage, la classification et la priorisation. Foundry accueille aussi des modèles de partenaires et des modèles dont les poids sont accessibles. Le choix d’un modèle dépendra de la tâche, de la qualité attendue, de la latence, du coût et des exigences de déploiement.

## Azure adapte son infrastructure aux agents

Les annonces Azure couvrent les besoins des agents en production : calcul, isolation, connectivité, accès aux données et contrôle des coûts.

Azure Cobalt 200, maintenant disponible de façon générale, propose des machines virtuelles à processeurs Arm pour les tâches de calcul autour des modèles, comme l’orchestration et les traitements exécutés par les agents. Microsoft annonce aussi Agent Grid, qui applique des politiques aux interactions entre agents, modèles, outils et services Azure. Agent Sandbox isole le code produit ou exécuté par un agent à l’échelle de la machine virtuelle. Ces deux fonctions sont en aperçu.

Azure Catalyst, disponible de façon générale, regroupe des fonctions de découverte, de planification, de migration et de modernisation de systèmes. L’outil associe Azure Copilot et GitHub Copilot pour aider les équipes à examiner leur parc applicatif et à préparer sa transformation. Les équipes doivent valider les dépendances, les estimations et les choix de migration proposés.

Azure HorizonDB, un service de base de données compatible avec PostgreSQL, est maintenant disponible de façon générale. Il vise les applications transactionnelles exigeantes et comprend des fonctions de recherche vectorielle et d’IA. Microsoft présente également Fabric HorizonDB en aperçu. Ce service PostgreSQL sans serveur vise à rapprocher les données transactionnelles et analytiques dans Fabric. Il pourra suspendre les bases de données inactives et les reprendre à la réception d’une requête.

Le conteneur Azure SQL Database, en aperçu, permettra d’exécuter localement le moteur Azure SQL pour le développement et les tests d’intégration. Microsoft annonce aussi des améliorations de résilience et de migration pour Azure SQL Managed Instance, ainsi que des outils de développement et de protection pour Cosmos DB. Leur intérêt dépendra des besoins des projets, de la compatibilité recherchée, des coûts et des pratiques de livraison en place.

## Agent 365 et Perception ajoutent des contrôles

Quand les agents se multiplient, les équipes doivent savoir lesquels sont en service, qui les exploite, à quoi ils accèdent et combien ils coûtent. Agent 365 fournit un registre des agents et de leurs dépendances, puis applique des politiques de sécurité, de gouvernance et de dépenses.

Agent 365 Business Analytics aide les responsables à examiner les dépenses par service, modèle ou équipe, à comparer des scénarios et à estimer les tâches réalisées ou le temps assisté. Ces indicateurs peuvent guider les décisions, mais ils ne prouvent pas à eux seuls des économies ou un rendement financier. Les organisations devront définir leurs mesures et vérifier les estimations à partir des résultats observés.

Microsoft Perception, annoncé en disponibilité générale, utilise plusieurs agents pour les opérations de sécurité. Certains simulent des attaques, d’autres enquêtent sur des alertes et contribuent à corriger des problèmes à l’aide de playbooks. Le keynote présente des situations où les analystes gardent la main sur les actions sensibles. Pour l’appliquer en entreprise, il faudra définir des permissions minimales, des approbations, des journaux d’audit et des procédures de retour arrière.

Les annonces concernant Microsoft Entra, Intune, Purview et Windows portent aussi sur la gouvernance des agents locaux et hybrides. Elles visent à repérer les agents, à leur attribuer une identité, à contrôler leurs accès réseau et à protéger les données qu’ils peuvent consulter. Windows Server 2028, en aperçu, ajoute des fonctions de sécurité et la possibilité d’exécuter certains modèles près des applications et des données. Les annonces couvrent ainsi l’IA exécutée dans le nuage, sur les appareils et dans des environnements contrôlés.

## Ce que les équipes devraient retenir

Microsoft assemble Copilot, les agents, le contexte organisationnel et les plateformes de sécurité, de gestion des coûts et d’évaluation. Pour les équipes de développement et d’architecture, quelques mesures concrètes se dégagent :

- Choisir des scénarios limités et mesurables avant de déployer des agents persistants.
- Attribuer aux agents des identités, des propriétaires et les permissions minimales requises.
- Évaluer les agents avec des cas vérifiés, puis surveiller leurs traces, leurs coûts et leur comportement en production.
- Distinguer les fonctions généralement disponibles, les aperçus publics et les fonctions réservées à Frontier dans la feuille de route.
- Essayer les plateformes sur des charges de travail représentatives avant de tirer des conclusions à partir de démonstrations ou de benchmarks.

Le keynote rappelle que l’expertise humaine reste nécessaire à mesure que le travail évolue. Les équipes doivent définir les objectifs, concevoir les systèmes, évaluer les résultats et encadrer les actions automatisées. Ces responsabilités déterminent si les agents pourront être exploités de façon fiable.

## Conclusion

À Ignite 2026, Microsoft a présenté Copilot comme point d’entrée, Microsoft IQ comme source de contexte, Foundry comme environnement de création et d’amélioration des agents, et Azure comme infrastructure. Agent 365 et Perception fournissent les fonctions de gestion et de sécurité qui accompagnent leur exploitation.

Les organisations devront évaluer les gains sur leurs propres processus, contrôler les accès et les coûts, et tenir compte du statut des annonces, dont plusieurs sont encore en aperçu. Ces capacités seront utiles si elles répondent à un besoin mesurable et s’intègrent aux pratiques d’exploitation de l’organisation.
