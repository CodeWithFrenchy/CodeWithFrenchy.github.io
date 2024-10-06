---
title: Redis change de licence ! Quelles répercussions pour les développeurs et les entreprises ?
date: 2024-11-01 20:00:00 -0400
categories: [outil-developpement]
tags: []
---

Depuis son lancement, Redis s'est imposé comme l'un des outils de base de données en mémoire les plus populaires. Que ce soit pour le caching, la gestion de files d'attente ou même l’implémentation de structures de données complexes, Redis a été un choix de prédilection pour de nombreuses entreprises et développeurs. Cependant, un récent [changement de licence](https://redis.io/blog/redis-adopts-dual-source-available-licensing/#:~:text=Redis%20announced%20a%20transition%20from%20the%20BSD%203-Clause%20License%20to) soulève des questions : quelles seront les implications pour ceux qui utilisent Redis au quotidien ? Et comment les alternatives comme [Garnet de Microsoft](https://www.microsoft.com/en-us/research/blog/introducing-garnet-an-open-source-next-generation-faster-cache-store-for-accelerating-applications-and-services/?msockid=01a323d7c627619a2ad03099c71c6058) pourraient-elles s'intégrer dans cet écosystème en pleine mutation ?

## Changement de licence : Qu'est-ce qui a changé ?

Redis, historiquement disponible sous licence [BSD](https://fr.wikipedia.org/wiki/Licence_BSD#:~:text=La%20licence%20BSD%20%28Berkeley%20Software%20Distribution%20License%29%20est%20une%20licence%23:~:text=La%20licence%20BSD%20%28Berkeley%20Software%20Distribution%20License%29%20est%20une%20licence), a récemment modifié les conditions d’utilisation de certains modules via la **Redis Source Available License (RSAL)**. Ce changement vise à limiter l’utilisation des modules spécifiques à Redis dans des solutions commerciales fournies en tant que service (*Software-as-a-Service*). L'objectif principal derrière ce changement est de protéger Redis des grands acteurs du cloud, tels qu'Amazon Web Services, qui bénéficiaient des modules Redis sans contribuer en retour au projet ou à son développement (Microsoft soutient l'utilisation de Redis à travers plusieurs initiatives). 

Voici quelques points clés à retenir :

- **RSAL** s'applique uniquement à certains modules Redis (comme RedisSearch ou RedisJSON), mais pas au cœur de Redis, qui reste sous licence BSD.
- Les entreprises peuvent toujours utiliser ces modules gratuitement pour une utilisation interne ou dans des produits on-premise.
- Cependant, la vente de services SaaS basés sur **ces modules nécessite désormais une licence commerciale**.

## Répercussions pour les développeurs et les entreprises

Pour les développeurs travaillant sur des projets open source ou des applications internes, le changement est relativement mineur. Toutefois, pour les entreprises qui intègrent Redis dans des services cloud commercialisés, cela peut entraîner des coûts supplémentaires et des obligations contractuelles. Les startups et les PME utilisant Redis dans des environnements cloud doivent maintenant évaluer attentivement leur conformité à la nouvelle licence.

Impacts potentiels :

- **Coûts accrus** : Les entreprises SaaS pourraient voir leurs frais augmenter si elles choisissent d'intégrer des modules Redis sous **RSAL** dans leurs solutions.
- **Alternative à considérer** : Les projets peuvent chercher à remplacer certains modules Redis par des solutions open source alternatives ou à développer leurs propres implémentations.
- **Écosystème cloud** : Les grands fournisseurs cloud, comme AWS, peuvent se tourner vers des forks de Redis ou encourager l’adoption de solutions alternatives comme **ElastiCache**, leur version propriétaire du service Redis.

## Garnet : Une alternative prometteuse ?

En parallèle de ce changement de licence, de nouveaux outils émergent, notamment [Garnet](https://github.com/microsoft/Garnet) de Microsoft, qui pourrait devenir une alternative sérieuse à Redis dans certains cas d'usage. Garnet est un projet conçu pour être une base de données distribuée ultra-rapide, optimisée pour les applications cloud natives, et offrant des fonctionnalités qui rivalisent avec Redis.

Il se distingue par :

- **Une intégration native avec Azure** : Garnet est conçu pour tirer parti de l'infrastructure Azure, ce qui en fait un choix naturel pour les entreprises déjà investies dans cet écosystème.
- **Une approche axée sur la performance** : Tout comme Redis, Garnet est conçu pour des temps de réponse très bas, ce qui le rend idéal pour des applications nécessitant des données en temps réel.
- **Flexibilité des licences** : Microsoft pourrait proposer une politique de licence plus flexible, qui pourrait être attrayante pour les entreprises concernées par la RSAL de Redis.
- **Compatibilité avec le contrat Redis** : Garnet est entièrement compatible avec le contrat Redis, facilitant ainsi son adoption pour les entreprises qui utilisent déjà cette technologie.

## Conclusion

Le changement de licence de Redis représente une évolution significative dans la gestion des outils open source dans le cloud. Si la **RSAL** peut offrir une meilleure protection pour [Redis Labs](https://redis.io/press/redis-labs-becomes-simply-redis/#:~:text=The%20company%20has%20developed%20an%20expanded%20set%20of%20data%20models) contre les géants du cloud, elle pousse également les entreprises et développeurs à réévaluer leur dépendance à ces outils. Les alternatives comme **Garnet** de Microsoft, ainsi que d'autres bases de données en mémoire, vont probablement gagner en popularité à mesure que l'écosystème continue d’évoluer.

Pour les développeurs et entreprises, l’essentiel est de suivre de près ces changements et d’évaluer les coûts et les bénéfices de chaque solution pour leur propre architecture. Le choix de rester avec Redis, d’explorer Garnet ou même d’adopter une autre solution dépendra largement des besoins spécifiques et des contraintes budgétaires de chaque projet.

Ce changement de paradigme autour de Redis reflète un mouvement plus large dans l’écosystème open source où de plus en plus de projets cherchent à protéger leur modèle économique tout en continuant à offrir des solutions robustes pour les utilisateurs finaux.

💡Découvrez en [vidéo](https://www.youtube.com/watch?v=ot6bNg-nZ3M) l'impact du changement de licence de Redis.
