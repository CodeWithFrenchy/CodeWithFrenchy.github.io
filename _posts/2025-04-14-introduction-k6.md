---
title: Introduction à K6
date: 2025-04-14 20:00:00 -0400
categories: [outil-developpement]
tags: [k6, essais]
---

## Qu'est-ce que k6 ?

[k6](https://k6.io/) est un outil open-source de test de charge et de performance, conçu pour aider les développeurs à évaluer la fiabilité de leurs systèmes, notamment les API, microservices et sites web. Développé par [Grafana Labs](https://grafana.com/), k6 permet de simuler des comportements utilisateur réalistes et d'identifier les potentielles faiblesses avant le déploiement en production. 

Pourquoi utiliser k6 ?

- Si vous êtes habitué à [Apache JMeter](https://jmeter.apache.org/), k6 vous apportera une approche plus moderne et développeur-friendly avec des tests écrits en JavaScript. 🙌
- Son [CLI](https://grafana.com/docs/grafana-cloud/testing/k6/author-run/use-the-cli/) facilite son intégration dans les pipelines d'intégration continue.
- Capacité à générer des [rapports détaillés](https://grafana.com/docs/k6/latest/get-started/results-output/) sur les performances, aidant à identifier les goulets d'étranglement et les points à optimiser.

## Types d’essais de charge

- 🔼 **Test de montée en charge** (_Ramp-up test_) - Augmente progressivement le nombre d’utilisateurs pour voir à quel moment l’application commence à ralentir.
- 💥 **Test de stress** (_Stress test_) - Envoie plus de requêtes que la capacité normale pour voir comment l’application réagit sous la surcharge.
- ⏳ **Test d’endurance** (_Soak test_) - Vérifie si l’application reste stable après plusieurs heures/jours sous charge continue.
- ⚡ **Test de pointe** (_Spike test_) - Simule une augmentation soudaine du trafic pour voir si le système peut absorber les pics de charge.

La liste ne s'arrête pas là ! Il existe également d'autres types de tests complémentaires, comme :
 
- 🧨 **Tests de chaos et de résilience** (_Chaos and Resilience Testing_) - Simulent des pannes ou des conditions extrêmes pour évaluer la capacité d’un système à résister aux défaillances et à se rétablir automatiquement.
- 🖧 **Tests d’infrastructure** (_Infrastructure Testing_) - Vérifient la performance et la fiabilité des composants sous-jacents tels que les serveurs, les bases de données, les réseaux et le stockage cloud. Ils aident à identifier les goulets d’étranglement et à optimiser les ressources.  
- 🌐 **Tests de performance du navigateur** (_Browser Performance Testing_) - Mesurent la vitesse d’affichage et le temps de chargement des pages web du point de vue de l’utilisateur final. Ces tests permettent d’optimiser les performances côté client et d'améliorer l'expérience utilisateur. Outils populaires : [Lighthouse] et [k6 Browser](https://grafana.com/docs/k6/latest/using-k6-browser/).  

>💡 En combinant ces différents types de tests, on s’assure d’une application robuste, performante et résiliente, prête à affronter toutes les conditions ! 

Il est essentiel d’évaluer le retour sur investissement (_ROI_) avant de multiplier les tests : toutes les applications n’ont pas besoin de tests de charge, de résilience ou d’infrastructure avancés. L’important est d’adapter la stratégie de test aux risques et aux exigences de l’application pour éviter des efforts inutiles.

## Mise en place de k6

k6 est multiplateforme et compatible avec Windows, Linux et macOS. Pour l’[installer](https://grafana.com/docs/k6/latest/set-up/install-k6/), utilisez le gestionnaire de paquets adapté à votre système d’exploitation.

Sur Windows :

``` ps1
winget install k6 --source winget
```

>💡Notons également qu'il est possible d'exécuter k6 depuis un [conteneur](https://grafana.com/docs/k6/latest/set-up/install-k6/#docker).

## Lancer un simple test de charge sur une API REST

Faire un test de charge sur une API REST (ex: `https://example.com/api/products`) pour simuler 10 utilisateurs virtuels pendant 30 secondes.

Script `test.js` :

``` js
import http from 'k6/http';
import { sleep, check } from 'k6';

export const options = {
  vus: 10,           // 10 utilisateurs virtuels
  duration: '30s',   // pendant 30 secondes
};

export default function () {
  const res = http.get('https://example.com/api/products');

  check(res, {
    'status est 200': (r) => r.status === 200,
    'réponse contient produits': (r) => r.body.includes('product'),
  });

  sleep(1);
}
```

Exécution du script :

``` bash
k6 run test.js
```

🙌 Tu verras des [statistiques](https://grafana.com/media/docs/k6-oss/k6-results-stdout.png) en temps réel comme le nombre de requêtes réussies, les échecs, le temps de réponse moyen, etc.

## Comprendre les résultats d'exécution de k6

Lorsqu’un test de charge est exécuté avec k6, un rapport détaillé est généré en console, affichant plusieurs indicateurs clés de performance. Ces métriques permettent d’évaluer la stabilité, la rapidité et la robustesse du système testé.  
 
### Résumé des métriques principales 

À la fin d'un test, k6 affiche un tableau de résultats contenant les indicateurs suivants :  
 
- `http_reqs` : Nombre total de requêtes HTTP effectuées pendant le test.  
- `http_req_duration` : Temps moyen de réponse des requêtes HTTP, généralement mesuré en millisecondes.  
- `http_req_failed` : Pourcentage de requêtes ayant échoué.  
- `vus` (Virtual Users) : Nombre d'utilisateurs simultanés simulés à un instant donné.  
- `iterations` : Nombre total d'itérations de script exécutées.

### Analyse des temps de réponse

L’un des éléments les plus critiques est le `http_req_duration`, qui mesure le temps de réponse des requêtes. 

k6 fournit des valeurs utiles comme :  

- Moyenne (`avg`) : Temps de réponse moyen sur l’ensemble des requêtes.  
- Médiane (`med`) : Le temps de réponse qui sépare la moitié des requêtes les plus rapides des plus lentes.  
- 95e percentile (`p(95)`) : 95% des requêtes ont eu un temps de réponse inférieur à cette valeur.  
- Maximum (`max`) : Le temps de réponse le plus élevé observé.  
 
Ces données aident à identifier des problèmes comme des pics de latence ou des temps de réponse anormalement longs.

### Vérification des échecs et des erreurs 

L’indicateur `http_req_failed` permet de voir si certaines requêtes ont échoué (timeouts, erreurs serveur, etc.). Un taux élevé peut indiquer une saturation du backend ou une mauvaise configuration de l’application.  
 
### Interprétation des tendances et optimisations possibles 

- Si le temps de réponse médian est bas, mais le max est élevé, cela signifie que certaines requêtes subissent des latences importantes, peut-être dues à un goulot d’étranglement.  
- Un taux d’échec élevé peut indiquer un problème de scalabilité ou un manque de ressources côté serveur.  
- Si la charge CPU ou mémoire du serveur augmente rapidement, il pourrait être utile de mettre en place un autoscaling ou d’optimiser les requêtes et le caching.  

## Conclusion

Dans un contexte où les performances applicatives sont de plus en plus scrutées, **k6** s’impose comme un outil moderne, léger et puissant pour réaliser des **essais de charge** adaptés aux réalités d’aujourd’hui. Que ce soit pour valider la scalabilité d’une API, détecter des goulets d’étranglement, ou s’assurer que l’expérience utilisateur reste fluide en période de pointe, k6 offre une solution accessible aussi bien aux développeurs qu’aux équipes DevOps.

Grâce à sa syntaxe en JavaScript, son intégration facile dans les pipelines CI/CD, et la richesse des métriques fournies, il devient un excellent allié pour intégrer les tests de performance dès les premières étapes du cycle de développement — un des piliers des pratiques modernes comme le [Shift Left Testing](https://datascientest.com/shift-left-testing-tout-savoir).

Il est cependant essentiel de rappeler que **tous les projets ne nécessitent pas le même niveau de rigueur** en matière de tests de performance. Adapter les types d’essais (stress, endurance, montée en charge…) aux enjeux métier, à l’infrastructure cible et aux attentes des utilisateurs permet d’optimiser l’effort tout en maximisant le retour sur investissement.

En résumé, que vous souhaitiez prévenir les mauvaises surprises en production ou simplement renforcer la résilience de vos systèmes, **k6 est un excellent point de départ pour professionnaliser vos essais de charge**. Et comme toujours : testez tôt, testez souvent, et testez intelligemment.
