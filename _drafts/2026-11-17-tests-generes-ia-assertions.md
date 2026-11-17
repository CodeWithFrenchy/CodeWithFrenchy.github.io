---
title: "Tests générés par l’IA : vérifier ce que les assertions protègent réellement"
date: 2026-11-17 19:00:00 -0400
categories: [dotnet, tests]
tags: [dotnet, tests-unitaires, intelligence-artificielle, github-copilot, xunit, mstest, microsoft-testing-platform, mutation-testing]
---

>  ⚠️ OUVRIR LE FICHIER AVANT PUBLICATION + IMAGE

## Préambule

Un agent modifie une règle métier, génère quelques tests et annonce que tout passe. Le code compile, le pipeline est vert et les nouveaux tests portent des noms rassurants. Pourtant, une question demeure : quelles erreurs ces tests seraient-ils capables de détecter ?

La session [Test Smarter: AI-Assisted Testing for Modern .NET](https://www.youtube.com/watch?v=fKnZotxlEec), présentée à la .NET Conf 2026, abordait précisément ce problème. L’IA peut accélérer la production de tests, mais la confiance vient de ce que leurs assertions vérifient réellement. Un test peut exécuter toute une méthode sans protéger la règle métier qui justifie son existence.

Pour une équipe .NET, le gain potentiel dépasse le temps économisé à écrire du code de test. Des assertions pertinentes peuvent rendre les régressions plus visibles, réduire les corrections après livraison et faciliter la revue d’un changement produit par un agent.

Je propose de reprendre le principe de la démonstration avec un calcul de commande, puis d’examiner les scénarios limites, l’aide apportée par les agents et le rôle de Microsoft.Testing.Platform dans cette boucle de validation.

## Partir d’une règle métier explicite

Notre calculateur produit une estimation du montant d’une commande. Pour cet exemple, le contrat est le suivant.

- Le sous-total correspond à la somme du prix unitaire multiplié par la quantité pour chaque ligne.
- La livraison coûte 4,99$ lorsque le sous-total est inférieur à 100$.
- La livraison est gratuite dès que le sous-total atteint 100 $.
- Le total correspond au sous-total auquel on ajoute les frais de livraison.

Les taxes et les promotions sont exclues de ce calcul. Les montants sont représentés avec `decimal` dans l’implémentation .NET.

Une commande contenant deux articles à 24,50$ et un article à 26$ doit donc produire un sous-total de 75$, des frais de livraison de 4,99$ et un total de 79,99$.

Ces valeurs proviennent du contrat. Elles nous donnent une référence indépendante du code que nous voulons vérifier.

Les exemples de tests ci-dessous sont présentés en pseudocode. Ils décrivent les données, l’appel et les assertions à transposer dans le framework du projet. Les fonctions `VérifierÉgalité` et `VérifierVrai` servent uniquement à illustrer les vérifications.

## Deux tests verts qui protègent très peu

### Le test annonce un calcul, mais vérifie seulement un objet

Imaginons ce premier test généré par un agent.

```text
Test Calculer_PlusieursLignes_CombinePrixEtQuantites
    lignes ← [
        Ligne(prixUnitaire: 24.50, quantite: 2)
        Ligne(prixUnitaire: 26.00, quantite: 1)
    ]

    resultat ← Calculer(lignes)

    VérifierVrai(resultat != null)
```

Son nom promet de vérifier les prix et les quantités. Son assertion garantit seulement qu’un objet a été retourné.

Si le calculateur oublie la multiplication par la quantité, le sous-total devient 50,50$. Le test reste vert. S’il oublie complètement la deuxième ligne, le sous-total devient 49$. Le test reste encore vert.

Vérifier qu’un résultat existe peut être pertinent lorsque c’est le comportement attendu. Ici, cette vérification ne suffit pas à protéger le calcul annoncé par le nom du test.

### Le total est positif, même lorsque la livraison manque

Un deuxième test semble couvrir l’ajout des frais.

```text
Test Calculer_SousLeSeuil_AjouteLesFraisDeLivraison
    lignes ← [Ligne(prixUnitaire: 75.00, quantite: 1)]

    resultat ← Calculer(lignes)

    VérifierVrai(resultat.Total > 0)
```

Si le calculateur retourne directement le sous-total de 75 $, la livraison n’a pas été ajoutée. L’assertion réussit quand même.

Elle accepterait aussi un total de 124,99$, obtenu en facturant par erreur 49,99$ de livraison. Le résultat est positif dans les deux cas, mais aucun des deux montants ne respecte le contrat.

Le problème n’est donc pas le nombre de tests. Il manque une vérification capable de distinguer le résultat attendu d’un résultat incorrect plausible.

## Renforcer les assertions existantes

Pour le premier test, nous pouvons conserver les mêmes données et vérifier le sous-total attendu.

```text
Test Calculer_PlusieursLignes_CombinePrixEtQuantites
    lignes ← [
        Ligne(prixUnitaire: 24.50, quantite: 2)
        Ligne(prixUnitaire: 26.00, quantite: 1)
    ]

    resultat ← Calculer(lignes)

    VérifierÉgalité(attendu: 75.00, obtenu: resultat.SousTotal)
```

Une multiplication oubliée ou une ligne ignorée provoque maintenant un échec pour ce scénario. Le test protège le comportement qu’il annonce.

Le deuxième test peut vérifier les frais et leur effet sur le total.

```text
Test Calculer_SousLeSeuil_AjouteLesFraisDeLivraison
    lignes ← [Ligne(prixUnitaire: 75.00, quantite: 1)]

    resultat ← Calculer(lignes)

    VérifierÉgalité(attendu: 4.99, obtenu: resultat.FraisLivraison)
    VérifierÉgalité(attendu: 79.99, obtenu: resultat.Total)
```

Ces deux assertions portent sur le même comportement. La première protège le tarif appliqué. La deuxième protège son ajout au montant final. En xUnit, ces comparaisons correspondent à des appels à `Assert.Equal` avec des valeurs `decimal`.

On obtient un diagnostic plus utile. Un échec sur les frais oriente vers la règle de livraison. Des frais corrects accompagnés d’un total incorrect orientent vers l’assemblage du montant final.

Il n’est pas nécessaire de vérifier tous les champs de l’objet dans chaque test. Les assertions doivent protéger le contrat du scénario, avec assez de précision pour reconnaître une erreur pertinente.

### Éviter de demander au code de confirmer son propre résultat

Une autre faiblesse apparaît lorsque le résultat attendu est calculé par la fonction testée.

```text
attendu ← Calculer(lignes).Total
obtenu ← Calculer(lignes).Total

VérifierÉgalité(attendu, obtenu)
```

Une fonction déterministe qui produit systématiquement un montant erroné satisfait cette comparaison. Le test vérifie une forme de répétabilité, sans établir que le montant est juste.

Même une relation apparemment plus intéressante demande de la nuance.

```text
VérifierÉgalité(
    attendu: resultat.SousTotal + resultat.FraisLivraison,
    obtenu: resultat.Total
)
```

Cette assertion protège la cohérence entre les trois champs. Elle demeure utile, mais elle accepte des frais de livraison incorrects si le total les additionne correctement. Pour protéger le tarif, il faut également une attente indépendante, comme 4,99$ pour la commande de 75$.

## Tester exactement là où la règle change

Supposons maintenant que l’implémentation accorde la livraison gratuite lorsque le sous-total est strictement supérieur à 100$.

Un test à 75$ passe. Un autre à 125$ passe aussi. Les deux chemins du calcul peuvent avoir été exécutés, alors que la commande à exactement 100$ reste facturée incorrectement.

Voici les trois cas qui encadrent cette décision.

| Sous-total | Frais attendus | Total attendu | Comportement protégé |
| --- | --- | --- | --- |
| 99,99$ | 4,99$ | 104,98$ | Le montant immédiatement inférieur au seuil reste payant |
| 100,00$ | 0,00$ | 100,00$ | Le seuil lui-même donne droit à la gratuité |
| 100,01$ | 0,00$ | 100,01$ | La gratuité s’applique au-dessus du seuil |

Ces cas peuvent alimenter un test paramétré qui compare les frais et le total aux valeurs attendues. Le cas à 100$ détecte précisément le remplacement de `>=` par `>`.

La couverture de lignes aide à repérer du code jamais exécuté. La couverture de branches indique si plusieurs chemins ont été parcourus. Aucune des deux ne garantit, à elle seule, que les valeurs utilisées et les assertions protègent la frontière métier.

### Les limites ne se résument pas aux nombres

Dans un projet réel, je chercherais également les transitions qui modifient les droits, les états ou les effets observables.

| Type de règle | Scénarios à distinguer | Vérification utile |
| --- | --- | --- |
| Période d’admissibilité | Veille, dernière journée incluse, lendemain | Décision exacte pour chaque date |
| Autorisation | Permission présente, permission absente | Décision et absence de modification interdite |
| Traitement idempotent | Première demande, répétition de la même demande | Un seul effet métier persistant |
| Filtrage | Aucun élément admissible, un élément, plusieurs éléments | Identités des éléments retenus et exclus |
| Validation | Valeur minimale autorisée et valeur juste en dessous | Acceptation ou erreur conforme au contrat |

Ces attentes doivent être précisées avec le métier. Par exemple, notre règle de livraison ne dit pas si un panier vide est autorisé. Demander à un agent de générer ce test sans clarifier le contrat peut simplement figer une hypothèse qu’il a inventée.

Pour une règle dépendante de l’heure, je privilégie une date fournie explicitement ou une horloge contrôlée. Pour un traitement qui écrit dans une base de données, la vérification peut devoir porter sur l’état enregistré et sur l’absence de doublon. Le niveau du test dépend de la garantie recherchée.

## Introduire une erreur pour vérifier la protection

Une façon concrète d’évaluer une assertion consiste à modifier temporairement le comportement qu’elle est censée protéger.

Dans notre exemple, trois changements sont intéressants.

| Erreur introduite | Test qui devrait échouer |
| --- | --- |
| Ignorer la quantité d’une ligne | Calcul du sous-total avec plusieurs quantités |
| Remplacer les frais de 4,99$ par 49,99$ | Frais et total d’une commande sous le seuil |
| Remplacer `>= 100` par `> 100` | Livraison gratuite à exactement 100$ |

Ce tableau décrit les détections attendues à partir des exemples. Il ne constitue pas un rapport d’exécution.

Pour une correction de bogue, je commencerais par faire exécuter le nouveau test contre la version qui contient le défaut, puis contre la version corrigée. Le premier passage doit échouer sur le comportement visé. Le deuxième doit réussir. Une erreur de compilation ou un environnement indisponible ne démontre pas que le test détecte le bogue.

Les [tests par mutation](https://learn.microsoft.com/en-us/dotnet/core/testing/mutation-testing) systématisent cette idée. Des modifications ciblées sont introduites dans le code, puis la suite est exécutée pour vérifier si elle les détecte. [Stryker.NET](https://stryker-mutator.io/docs/stryker-net/introduction/) permet d’automatiser ce travail dans des projets .NET.

Un mutant qui survit peut révéler un scénario absent ou une assertion insuffisante. Il faut toutefois examiner son effet. Certaines mutations ne changent aucun comportement observable pour les entrées permises par le contrat.

Je commencerais par les règles dont une régression coûte cher, comme un calcul, une autorisation ou une transition d’état. Une analyse ciblée permet de limiter le temps d’exécution et le volume de résultats à examiner.

## Donner un mandat précis à l’agent de test

Dans Visual Studio, [GitHub Copilot testing for .NET](https://learn.microsoft.com/en-us/visualstudio/test/github-copilot-test-dotnet-overview?view=visualstudio) fournit une expérience dédiée à la génération, à la compilation et à l’exécution de tests C#. Elle prend en charge MSTest, NUnit et xUnit. La documentation indique Visual Studio 2026 version 18.3 ou ultérieure et un abonnement Copilot payant comme prérequis.

Le gain est particulièrement concret lorsqu’il faut préparer les données, respecter les conventions du projet et corriger les problèmes de compilation des tests générés. Il reste cependant à déterminer si les attentes traduisent correctement le besoin.

Si le même agent écrit l’implémentation et les tests à partir de sa propre interprétation, une erreur de compréhension peut se retrouver des deux côtés. Je lui fournis donc les exigences et des exemples validés séparément du code.

Voici un mandat réutilisable, à adapter aux chemins du dépôt.

```text
Examine le calculateur de commande et ses tests existants.

Utilise les règles de docs/regles-livraison.md pour déterminer les
résultats attendus. Utilise l’implémentation pour comprendre les API.
Signale toute ambiguïté dans les règles avant d’inventer une attente.

Conserve xUnit, les versions de paquets et les conventions du dépôt.

Commence par identifier les assertions qui laisseraient passer une
erreur métier plausible. Pour chaque constat, décris cette erreur
et explique pourquoi l’assertion actuelle l’accepterait.

Renforce les tests existants lorsque leur scénario est pertinent.
Ajoute les cas manquants autour du seuil de livraison gratuite.
Ne modifie pas le code de production pour faire passer les tests.
Conserve les échecs et explique leur lien avec le contrat.

Exécute les tests concernés, puis la suite pertinente pour le changement.
Rapporte les commandes ou opérations exécutées, les nombres de tests
réussis, échoués et ignorés, ainsi que toute vérification non exécutée.
```

Le résultat attendu de ce travail est un changement que l’équipe peut examiner. Une assertion générée qui échoue peut révéler un défaut de l’application, une attente incorrecte ou une documentation désuète. Ce désaccord mérite une analyse avant toute correction.

### Utiliser les skills pour orienter la revue

Le plugin [dotnet-test du dépôt dotnet/skills](https://github.com/dotnet/skills/blob/main/plugins/dotnet-test/README.md) regroupe des procédures pour générer, exécuter et analyser des tests, ainsi qu’un agent spécialisé. Il comprend notamment `assertion-quality` pour examiner les assertions et `test-gap-analysis` pour rechercher les comportements insuffisamment protégés.

Le skill [assertion-quality](https://github.com/dotnet/skills/blob/main/plugins/dotnet-test/skills/assertion-quality/SKILL.md) fournit un angle de revue particulièrement utile : associer une faiblesse à un comportement incorrect concret que l’assertion accepterait encore.

Je demanderais donc un constat comme « le total de 75$ passe alors que les frais n’ont pas été ajoutés », accompagné du test concerné. Cette observation permet au réviseur de vérifier le raisonnement.

Une appréciation produite par l’agent reste une analyse. Lorsqu’il prédit qu’un test détecterait une modification, cette prédiction doit être distinguée d’un échec réellement observé pendant une exécution.

## Microsoft.Testing.Platform : exécuter et diagnostiquer les tests

La qualité des assertions et l’infrastructure d’exécution interviennent à deux niveaux différents. Pour situer les outils, on peut distinguer trois responsabilités.

| Composant | Responsabilité |
| --- | --- |
| xUnit, MSTest ou NUnit | Définir les tests, leurs données et les assertions |
| Microsoft.Testing.Platform | Héberger l’exécution et fournir des points d’intégration et d’extension |
| Test Explorer ou la ligne de commande | Déclencher les exécutions et présenter les résultats |

[Microsoft.Testing.Platform](https://learn.microsoft.com/en-us/dotnet/core/testing/microsoft-testing-platform-intro), souvent abrégé MTP, s’intègre au projet de tests. Celui-ci devient une application exécutable que l’on peut lancer et diagnostiquer directement, avec une intégration aux outils de développement et aux pipelines.

Son adoption n’impose pas automatiquement un changement de framework. Par exemple, [xUnit.net v3 possède une intégration MTP](https://xunit.net/docs/getting-started/v3/microsoft-testing-platform). Il faut vérifier les versions et la configuration du projet, ainsi que les outils utilisés localement et en intégration continue.

MTP peut rendre les résultats et les incidents d’exécution plus accessibles. Il ne peut pas déduire que les frais devraient être de 4,99$ si les tests ne vérifient jamais cette attente.

### Rendre le mode d’exécution explicite

Le [mode MTP de la commande dotnet test](https://learn.microsoft.com/en-us/dotnet/core/testing/unit-testing-with-dotnet-test) est disponible à partir du SDK .NET 10 et nécessite MTP 1.7 ou ultérieur.

Pour des projets déjà configurés avec MTP, la sélection peut être ajoutée au fichier `global.json` existant en conservant ses autres propriétés.

```json
{
  "test": {
    "runner": "Microsoft.Testing.Platform"
  }
}
```

Cette configuration sélectionne le mode d’exécution. Elle ne convertit pas les projets VSTest. Les projets de tests concernés doivent être compatibles avec MTP.

On peut ensuite cibler explicitement un projet.

```bash
dotnet test --project tests/Boutique.Tests/Boutique.Tests.csproj
```

La [référence de la commande dotnet test avec MTP](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-test-mtp) décrit aussi les contrôles applicables à l’exécution. Avec le SDK .NET 11 et MTP 2.4 ou ultérieur, cet exemple limite la durée et impose un minimum de tests dans le résultat agrégé.

```bash
dotnet test --project tests/Boutique.Tests/Boutique.Tests.csproj --minimum-expected-tests 3 --timeout 2m
```

Le seuil de trois est illustratif et doit correspondre au périmètre visé. Le minimum global compte aussi les tests ignorés. Il faut donc lire les nombres de tests réussis, échoués et ignorés pour savoir ce qui a effectivement été vérifié.

### Distinguer un échec métier d’un incident d’exécution

Une assertion qui échoue, un processus qui plante et une exécution qui reste bloquée demandent des investigations différentes.

Les [extensions de diagnostic MTP](https://learn.microsoft.com/en-us/dotnet/core/testing/microsoft-testing-platform-crash-hang-dumps) peuvent produire des dumps lors d’un arrêt anormal ou d’un blocage. Les paquets `Microsoft.Testing.Extensions.CrashDump` et `Microsoft.Testing.Extensions.HangDump` apportent ces capacités. Leurs options dépendent des extensions installées et de leur compatibilité avec l’environnement.

Pour une équipe, le gain est de disposer d’éléments exploitables lorsque le processus de tests s’arrête sans fournir un échec d’assertion clair. Pour un agent, ces éléments peuvent aider à différencier une régression du code d’un problème de processus ou d’environnement.

Dans la démonstration de la conférence, le Hot Reload permettait également de modifier du code pris en charge et de relancer les tests dans le même processus. Cette démonstration utilisait l’hôte console, avec un lancement depuis Visual Studio. Elle illustrait une boucle de vérification plus courte pour tester une hypothèse, observer l’échec puis rétablir le comportement attendu.

## Ce que je regarderais dans la revue de code

Avant d’accepter des tests générés, je chercherais à établir quelques éléments précis.

1. Le résultat attendu provient d’une règle ou d’un exemple validé indépendamment de l’implémentation.
2. Les assertions vérifient le comportement annoncé par le nom du test.
3. Les cas choisis couvrent les frontières qui modifient la décision métier.
4. Une correction de bogue dispose, lorsque c’est possible, d’un test qui échoue sur la version fautive et réussit sur la correction.
5. Le compte rendu distingue les tests réellement exécutés des analyses, des tests ignorés et des vérifications impossibles.

J’examinerais aussi le coût de maintenance. Une assertion liée à l’ordre interne des appels peut rendre un refactoring difficile si cet ordre ne fait pas partie du contrat. À l’inverse, vérifier qu’aucune écriture interdite n’a eu lieu peut être essentiel pour une règle d’autorisation.

Le nombre d’assertions est donc un indicateur très limité. Leur utilité dépend de la propriété observable qu’elles protègent et de l’erreur qu’elles permettent de reconnaître.

## Conclusion

Les agents peuvent nous aider à écrire des tests, à repérer des scénarios manquants et à revoir des assertions trop permissives. Microsoft.Testing.Platform apporte de son côté une infrastructure pour les exécuter et mieux comprendre les incidents rencontrés.

La confiance repose toutefois sur le lien entre la règle métier, les données choisies et le résultat attendu. Une assertion précise peut transformer la valeur d’un test existant sans ajouter une seule méthode à la suite.

Pour commencer, je choisirais une règle importante, j’identifierais une erreur plausible et je vérifierais que les tests la détectent réellement. C’est une façon concrète de mesurer ce que le résultat vert nous autorise à croire avant de fusionner un changement.
