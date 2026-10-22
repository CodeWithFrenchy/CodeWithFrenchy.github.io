---
title: Redis 8 - Un retour attendu à l’open source avec l’AGPLv3
date: 2025-05-04 19:00:00 -0400
categories: [outil-developpement, architecture]
tags: [redis, redis-8, open-source, agplv3, valkey, bases-de-donnees, cache, performance]
---

Le 1er mai 2025, Redis a fait un virage majeur en revenant à ses racines open source. Redis 8 est désormais disponible sous la licence **AGPLv3**, approuvée par l’Open Source Initiative (OSI). Ce changement intervient après une période de turbulences sur le plan des licences, que j'avais déjà abordée dans un [précédent article](https://codewithfrenchy.com/posts/redis-change-licence/).

## 📜 Une parenthèse fermée : de BSD à SSPL, puis à AGPL

En mars 2024, Redis avait troqué sa licence permissive BSD pour un modèle dual : **RSALv2** et **SSPLv1**. L’objectif était de restreindre l’exploitation commerciale sans contribution équitable, notamment par les grands fournisseurs cloud.

Ce virage avait déplu à la communauté, provoquant l’apparition de forks comme **Valkey**, soutenu par la Linux Foundation. En réponse à ces tensions, Redis 8 est désormais aussi disponible sous **AGPLv3**, une licence réellement open source, tout en conservant les autres options.

## 👨‍💻 Le retour de @antirez et les nouveautés de Redis 8

Le retour de **Salvatore Sanfilippo** ([@antirez](https://x.com/antirez)), créateur de Redis, a été un moment clé. De retour chez Redis en novembre 2024 comme évangéliste développeur, il a participé à la réorientation du projet. Sous sa houlette, Redis 8 propose de vraies avancées techniques :

- **Vector Sets** : un nouveau type de données orienté pour l'IA et recherche vectorielle.
- **Modules intégrés directement dans Redis Core** : JSON, Time Series, Bloom, Cuckoo, Count-min sketch, Top-K, t-digest, etc.
- **Améliorations de performance** : plus de 30 optimisations, avec jusqu’à **87 % de gain de vitesse** sur certaines commandes et un **débit doublé**.

💡 Redis 8 s’aligne donc sur les besoins modernes des développeurs et des infrastructures.

## 🌍 Une communauté rassurée

Le retour à une licence open source a été bien accueilli. Sanfilippo l’a résumé simplement :

> _« Je suis heureux que Redis soit à nouveau un logiciel open source, sous les termes de la licence AGPLv3. »_

Ce geste témoigne d’une volonté de renouer avec les principes du logiciel libre, tout en soutenant une innovation continue.

## 🎯 Ce qu’il faut retenir

- Redis revient à l’open source avec l’**AGPLv3**.
- Redis 8 introduit des fonctionnalités fortes pour l’IA, des modules clés et des vraies optimisations.
- C’est un geste d’ouverture et de réconciliation avec la communauté des développeurs.

📚 Pour creuser le sujet :

- L’annonce officielle : [redis.io/blog/agplv3](https://redis.io/blog/agplv3/)
- Mon analyse du changement de licence de Redis en 2024 : [Redis change de licence – Pourquoi ça fait débat](https://codewithfrenchy.com/posts/redis-change-licence/)
