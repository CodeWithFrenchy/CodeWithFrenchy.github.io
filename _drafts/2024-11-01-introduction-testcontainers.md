---
title: Introduction aux Testcontainers
date: 2024-11-01 20:00:00 -0400
categories: [outil-developpement]
tags: []
---

Les systèmes logiciels modernes s'attaquent à des problèmes complexes en utilisant une multitude de technologies et d'outils. Rarement un système logiciel fonctionne de manière isolée; il interagit généralement avec des bases de données, des systèmes de messagerie, des fournisseurs de cache, et de nombreux autres services tiers. Dans ce marché hautement concurrentiel, le temps de mise sur le marché est crucial. Les entreprises souhaitent lancer leur produit rapidement, obtenir des retours et itérer en conséquence. Pour atteindre cette agilité, il est essentiel d'avoir un processus solide d'intégration et de déploiement continus (CI/CD). Un élément clé de ce processus sont les tests automatisés, qui garantissent le bon fonctionnement de l'application. 

Les tests unitaires permettent de vérifier la logique métier et les détails d'implémentation en isolant les services externes, mais la majeure partie du code applicatif réside dans l'intégration avec ces services. Pour avoir une confiance totale dans notre application, il est essentiel d'écrire des tests d'intégration en plus des tests unitaires, afin d'assurer la fonctionnalité complète de l'application. 

Historiquement, les tests d'intégration sont considérés comme difficiles en raison des défis liés à la maintenance d'un environnement de test d'intégration. La mise en place de tests d'intégration avec une infrastructure préconfigurée pose plusieurs problèmes, notamment la nécessité de garantir que l'infrastructure est opérationnelle et que les données sont préconfigurées dans un état désiré. De plus, l'exécution de plusieurs pipelines de construction (*build*) en parallèle peut interférer avec d'autres données de test, entraînant des **tests instables**. 

Face à ces défis, certains développeurs se tournent vers des services en mémoire ou des variations embarquées des services requis pour les tests d'intégration. Par exemple, un développeur pourrait utiliser une base de données en mémoire en remplacement de PostgreSQL. Bien que cela soit une amélioration par rapport à l'absence de tests d'intégration, l'utilisation de simulations ou de versions en mémoire présente ses propres problèmes. Les services en mémoire peuvent ne pas avoir toutes les fonctionnalités des services en production ! 

C'est dans ce contexte que les Testcontainers entre en jeu, offrant une solution innovante pour les tests d'intégration avec de véritables services, rendant cette tâche aussi simple que l'écriture de tests unitaires. En utilisant les Testcontainers, les développeurs peuvent créer des environnements de test reproduisant fidèlement leurs systèmes de production, tout en bénéficiant d'une rétroaction rapide et efficace sur leurs modifications. 

## Qu’est-ce qu’un Testcontainer 

Testcontainers est une bibliothèque de test qui offre des API pour faciliter la mise en place de tests d'intégration avec des services réels, encapsulés dans des conteneurs Docker. Grâce à Testcontainers, il est possible d'écrire des tests qui interagissent avec le même type de services que ceux utilisés en production, éliminant ainsi la nécessité d'utiliser des simulations ou des services en mémoire. 

Un test d'intégration typique basé sur Testcontainers fonctionne de la manière suivante : 

- **Avant les tests** :  Vous démarrez vos services requis (bases de données, systèmes de messagerie, etc.) en utilisant l'API de Testcontainers pour lancer des conteneurs Docker. 
- **Configuration** : Vous configurez ou mettez à jour la configuration de votre application pour qu'elle utilise ces services conteneurisés. 
- **Pendant les tests** : Vos tests s'exécutent en utilisant ces services conteneurisés, ce qui garantit un environnement de test qui reflète fidèlement la production. 
- **Après les tests** : Testcontainers s'occupe de détruire ces conteneurs, que les tests aient réussi ou échoué.

![](https://testcontainers.com/guides/introducing-testcontainers/images/testcontainers-lifecycle-simple-2.png)

Source : [What is Testcontainers, and why should you use it?](https://testcontainers.com/guides/introducing-testcontainers/)

La seule exigence pour exécuter des tests basés sur Testcontainers est d'avoir un environnement d'exécution de conteneurs compatible avec l'API Docker. Si vous avez [Docker Desktop](https://www.docker.com/products/docker-desktop/) installé et en cours d'exécution, tout est prêt. 
 
Pour plus d'informations sur les environnements Docker pris en charge par Testcontainers, vous pouvez consulter [la documentation officielle](https://java.testcontainers.org/supported_docker_environment/). 

## Qu’est-ce que les Testcontainers règlent ? 

Les **Testcontainers** résolvent plusieurs problèmes liés aux tests d'intégration en permettant aux développeurs de tester leur application avec de véritables services, ce qui augmente la confiance dans les modifications apportées au code.  

Voici quelques points clés sur ce que Testcontainers améliore : 

1. **Infrastructure de test préconfigurée** : Avec Testcontainers, il n'est pas nécessaire d'avoir une infrastructure de test d'intégration préalablement configurée. L'API de Testcontainers fournit automatiquement les services nécessaires avant l'exécution des tests. Cela signifie que la définition de l'infrastructure se trouve directement à côté du code de test, facilitant ainsi la maintenance et la compréhension.
2. **Isolation des données** : Testcontainers élimine les problèmes de conflit de données, même lorsque plusieurs pipelines de construction (build) s'exécutent en parallèle. Chaque pipeline fonctionne avec un ensemble de services isolés, ce qui prévient les interférences entre les tests. 
3. **Exécution depuis l'IDE** : Les développeurs peuvent exécuter leurs tests d'intégration directement depuis leur IDE, tout comme pour les tests unitaires, sans avoir à pousser les modifications et attendre que l'intégration continue (CI) exécute les tests. Cela améliore le flux de travail et accélère le cycle de rétroaction. 
4. **Nettoyage automatique** : Après l'exécution des tests, Testcontainers s'occupe automatiquement de la suppression des conteneurs, ce qui simplifie la gestion des ressources et réduit le risque d'accumulation de services inutilisés sur le système. 
5. **Compatibilité avec plusieurs langages** : Testcontainers peut être utilisé avec de nombreux langages de programmation populaires, notamment .NET, Java, Go, NodeJS, Rust et Python, avec davantage de supports de langages à venir.

En résumé, Testcontainers fournit une solution pratique et efficace pour les tests d'intégration, permettant aux équipes de développement de gagner en confiance et en agilité dans leur processus de développement. 

## Démonstration 

<iframe width="740" height="473" src="https://www.youtube.com/embed/m7r2qyUabTs" title="Testing Entity Framework Core Correctly in .NET" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Conclusion 

En conclusion, nous avons examiné les défis inhérents aux tests d'intégration, en mettant en lumière les limitations de l'utilisation de mocks ou de services en mémoire, qui peuvent entraîner des incohérences entre les environnements de test et de production. Testcontainers offre une solution puissante à ces problèmes, permettant aux développeurs de réaliser des tests d'intégration avec de véritables services dans des conteneurs Docker isolés. Cela renforce non seulement la fiabilité des tests, mais *streamline* également le processus de test, permettant un retour d'information plus rapide et une plus grande confiance dans les modifications de code. 

Pour des informations supplémentaires et une documentation détaillée sur la mise en œuvre de Testcontainers dans vos projets, n'hésitez pas à visiter [Testcontainers](https://testcontainers.com/).
