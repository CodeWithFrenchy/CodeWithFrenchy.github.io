---
title: Une dépendance change de licence - comment choisir une alternative et réussir la migration ?
date: 2026-12-31 19:00:00 -0400
categories: [outil-developpement]
tags: []
---

## Préambule

Dans mon article [Changements de licences dans l’écosystème .NET : quelles implications ?](https://codewithfrenchy.com/posts/changements-licences-ecosysteme-dotnet/), j’abordais les motivations des mainteneurs et les conséquences de ces évolutions pour les équipes. Une fois le changement annoncé, une autre question se pose : que fait-on des applications qui utilisent déjà ces outils ?

La réponse engage davantage qu’une ligne dans un fichier projet. Une bibliothèque peut structurer les tests, les transformations de données ou l’exécution des traitements. Un cache peut contenir des sessions actives. Un bus de messages peut porter des processus métier qui dureront encore plusieurs jours après le prochain déploiement.

Remplacer systématiquement les outils devenus commerciaux n’est donc pas une stratégie suffisante. Les conserver sans réexaminer les conditions et les coûts ne l’est pas davantage. Il faut comparer des trajectoires viables : continuer avec une licence adaptée, conserver une version antérieure dans un cadre maîtrisé ou préparer une migration.

Cet article propose une démarche pour les architectes et les responsables techniques, puis l’applique à cinq cas : Redis, Fluent Assertions, AutoMapper, MediatR et MassTransit. L’objectif est de préserver les comportements, la capacité de livraison et la maîtrise des dépendances. Les alternatives retenues servent à illustrer des trajectoires différentes. Cette sélection ne constitue pas un catalogue exhaustif ni un classement universel.

*Les repères de licence et les offres mentionnées ont été vérifiés dans les sources officielles le 23 septembre 2026. L’analyse doit toujours porter sur la version et les conditions effectivement retenues par l’organisation.*

## 1. Comprendre ce qui change réellement

Une annonce de commercialisation ne décrit pas, à elle seule, les obligations de chaque utilisateur. Il faut distinguer le droit d’utiliser une version, l’accès aux nouvelles versions, le support et les mécanismes techniques de validation de licence.

### Licence, gratuité et support sont des sujets différents

Un logiciel open source peut être utilisé commercialement et faire l’objet d’une offre de support payante. Un logiciel dont le code est consultable n’est pas nécessairement open source. Une édition gratuite sous conditions peut imposer des restrictions selon l’organisation, ses revenus ou l’usage du produit.

Le copyleft ne doit pas non plus être résumé à une interdiction d’usage en entreprise. Ses obligations dépendent notamment du texte applicable, des modifications et des modalités de distribution ou d’accès. La décision juridique doit partir de ces faits, avec les personnes compétentes, plutôt que d’une étiquette « autorisé » ou « interdit » appliquée sans contexte.

Enfin, l’absence de blocage technique ne vaut pas autorisation. Une bibliothèque peut fonctionner sans clé valide tout en exigeant une licence pour l’usage concerné. Inversement, acheter une licence ne garantit pas tous les niveaux de support ni tous les droits de redistribution.

### Les repères utiles pour les cinq outils

| Outil | Changement à prendre en compte | Conséquence pour l’évaluation |
|---|---|---|
| Redis | BSD-3-Clause jusqu’à 7.2, puis RSALv2 ou SSPLv1 dès 7.4. Redis 8 ajoute l’option AGPLv3 | Identifier le moteur, sa version, ses modules et le mode d’exploitation |
| Fluent Assertions | Usage commercial soumis à licence à partir de la version 8. Branche 7 conservée open source | Examiner les usages de développement et de test, ainsi que la trajectoire de maintenance |
| AutoMapper | À partir de la version 15, offre commerciale ou option RPL-1.5 selon les conditions applicables | Vérifier l’éligibilité Community, le périmètre des développeurs et les conventions de mapping |
| MediatR | À partir de la version 13, offre commerciale ou option RPL-1.5 selon les conditions applicables | Vérifier les mêmes conditions et inventorier les comportements des pipelines |
| MassTransit | Licence requise à partir de la version 9 | Évaluer l’offre, sa configuration et la profondeur d’intégration du framework |

L’[historique officiel des licences Redis](https://redis.io/legal/licenses/) précise que le moteur lui-même est concerné dès la version 7.4 : le changement ne se limite pas aux modules. Redis 8 propose trois licences au choix. Il ne faut pas traiter leurs obligations comme si elles s’appliquaient toutes simultanément. L’utilisation d’un client .NET pour communiquer avec un serveur Redis ne permet pas, à elle seule, de conclure que toute l’application doit être publiée sous AGPL.

Pour [Fluent Assertions](https://fluentassertions.com/), la version 7 demeure open source et le projet annonce des corrections sur cette branche. La conserver n’équivaut donc pas automatiquement à utiliser un produit abandonné. Il faut examiner la maintenance effectivement disponible.

La [FAQ de Lucky Penny Software](https://luckypennysoftware.com/faq) distingue l’offre commerciale d’AutoMapper et de MediatR, l’admissibilité Community et une voie sous RPL-1.5 avec obligations réciproques. Ces possibilités doivent être évaluées séparément. Une petite équipe n’est pas automatiquement admissible à Community. Le statut de l’organisation compte, notamment pour les organismes gouvernementaux ou quasi gouvernementaux, qui en sont exclus. En consultation, vérifier aussi quelle entité doit détenir la licence et couvrir les intervenants.

La [documentation de licence de MassTransit](https://masstransit.massient.com/configuration/license) confirme le régime de la version 9 et les mécanismes de configuration. Les conditions de renouvellement, de redéploiement et de support doivent être examinées dans l’offre choisie, sans reprendre une ancienne annonce comme un engagement toujours applicable.

## 2. Comparer trois trajectoires, avec leurs coûts et leurs limites

### Continuer avec une licence adaptée

Conserver l’outil permet de préserver les compétences, les intégrations et les comportements déjà qualifiés. Cette option peut être particulièrement avantageuse lorsque l’outil structure une plateforme utilisée par plusieurs équipes.

Le financement des mainteneurs peut soutenir les corrections, la documentation et le développement. Le bénéfice réel dépend toutefois de ce que l’offre garantit : acheter un droit d’usage ne signifie pas nécessairement obtenir une correction prioritaire ou un engagement de disponibilité.

Les coûts comprennent le renouvellement, l’approvisionnement, le suivi du périmètre couvert et les éventuels changements de palier. Il faut aussi savoir ce qui reste permis après expiration. Une organisation qui conserve le composant doit pouvoir expliquer pourquoi cette dépendance reste acceptable et dans quelles circonstances elle serait réévaluée.

### Conserver une version antérieure

Cette option évite une migration précipitée et donne le temps de qualifier les alternatives. Elle peut rester viable tant que la branche répond aux exigences de sécurité, de compatibilité et de maintenance.

Le risque apparaît lorsque « nous restons sur cette version » devient une absence de décision. Les SDK évoluent, les dépendances transitives changent et les intégrations tierces peuvent cesser de prendre en charge l’ancienne branche.

Il faut donc préciser la version autorisée, le responsable du suivi et les déclencheurs d’une nouvelle décision : vulnérabilité non corrigée, incompatibilité avec le prochain runtime, fonctionnalité indispensable ou disparition du support nécessaire. Une date de revue évite de laisser cette situation devenir invisible.

### Migrer vers une alternative

Migrer peut réduire une dépendance commerciale, améliorer la lisibilité du code ou rapprocher le système des standards de l’organisation. Cela peut aussi déplacer les coûts vers le développement et l’exploitation.

Une alternative sans redevance exige toujours du support, des mises à jour et une capacité de diagnostic. Si l’équipe doit reconstruire les garanties de l’ancien outil, le gain financier apparent peut disparaître.

| Trajectoire | Bénéfice principal | Coût ou risque à accepter |
|---|---|---|
| Conserver avec une licence adaptée | Continuité technique et opérationnelle | Conditions commerciales, renouvellement et dépendance au fournisseur |
| Conserver une version antérieure | Temps pour décider, faible changement immédiat | Compatibilité et maintenance à surveiller |
| Migrer | Nouvelle trajectoire technique ou contractuelle | Requalification, apprentissage, coexistence et risque de régression |

### Calculer un coût complet

La comparaison doit utiliser un horizon commun, par exemple celui de la feuille de route du produit. Pour chaque option, intégrer :

- les droits d’usage, le support et l’infrastructure.
- l’analyse, l’adaptation du code et la validation des comportements.
- la formation, les guides et les changements de pipelines.
- l’exploitation de la transition et le retrait de l’ancien outil.
- les fonctionnalités métier reportées pendant ce travail.

Il faut distinguer les dépenses certaines des risques estimés. Une comparaison avec quelques hypothèses explicites et plusieurs scénarios est plus utile qu’un montant unique présenté comme précis. Le coût de migration doit venir d’un pilote représentatif, pas du seul nombre de références NuGet.

## 3. Redis : séparer le choix du moteur de celui de l’exploitation

Redis peut servir à conserver des valeurs recalculables, des sessions, des compteurs, des verrous ou des données exploitées par des traitements asynchrones. Ces usages n’ont pas la même tolérance à une interruption ou à une perte de données.

Avant de choisir un remplaçant, inventorier les commandes, les scripts, les modules, les volumes et les garanties attendues. Vérifier aussi si le problème vient de la licence du moteur ou du coût d’exploitation du service.

### Valkey : un candidat pour conserver une proximité fonctionnelle

[Valkey](https://valkey.io/) est un projet open source sous BSD-3-Clause, issu de Redis et accueilli par la Linux Foundation. C’est un candidat pertinent lorsque l’organisation veut conserver une licence permissive et que ses usages correspondent aux fonctionnalités prises en charge.

L’intérêt est de pouvoir préserver une partie importante de l’intégration cliente. La limite est qu’un protocole commun ne garantit pas l’équivalence de toutes les commandes, de tous les modules ou de tous les formats de persistance entre versions.

La [documentation de migration vers Valkey](https://valkey.io/topics/migration/) constitue le point de départ pour vérifier les chemins de migration compatibles. Un scénario documenté depuis Redis 7.2 ne doit pas être transposé automatiquement à une version plus récente ou à un service enrichi de fonctionnalités spécifiques.

### Azure Managed Redis : déléguer une partie de l’exploitation

[Azure Managed Redis](https://learn.microsoft.com/en-us/azure/redis/overview) propose une autre trajectoire : conserver un service Redis dans un cadre managé par Azure. Cette option peut convenir lorsque les besoins de disponibilité, de maintenance et d’intégration à la plateforme dominent la décision.

Elle conserve un coût récurrent et une dépendance au fournisseur. Les capacités du niveau de service, la connectivité, l’authentification et les contraintes de déploiement doivent être validées. Le passage à un service managé ne constitue pas une sortie de l’écosystème Redis.

### Préserver la sémantique du cache

Pour un cache entièrement reconstructible, une bascule vers une instance vide peut être envisageable. Il faut alors mesurer l’effet du réchauffement sur les bases de données et les services en aval. Un afflux de défauts de cache peut déplacer la panne vers la source de données.

Pour des sessions ou des compteurs métier, perdre les valeurs peut modifier le comportement du produit. La migration doit préserver les données nécessaires, leur expiration et leur cohérence. Deux stockages alimentés simultanément peuvent diverger. Une double écriture ne garantit pas leur synchronisation.

**Ma recommandation :** évaluer Valkey pour une trajectoire permissive compatible avec les usages existants, et Azure Managed Redis pour une trajectoire d’exploitation managée. Dans les deux cas, qualifier les expirations, les évictions, les reconnexions et les pannes avant de conclure que le changement de connexion suffit.

## 4. Fluent Assertions : préserver la force des tests

Une migration d’assertions semble locale : remplacer un package, modifier des espaces de noms, corriger les appels qui ne compilent plus. Son principal risque est pourtant de rendre certains tests moins exigeants.

Une comparaison de graphes d’objets peut ignorer l’ordre, exclure des membres ou utiliser des règles particulières pour les types et les valeurs nulles. Une réécriture qui compare seulement le nombre d’éléments ne protège plus le même comportement.

### Ma préférence : les assertions natives de xUnit

Dans un projet qui utilise déjà xUnit, je privilégie ses assertions natives. Elles permettent d’exprimer directement les comportements attendus tout en limitant les dépendances à maintenir. Pour vérifier des valeurs, des exceptions ou le contenu d’une collection, je préfère généralement cette approche à l’ajout d’une bibliothèque d’assertions supplémentaire.

Cette préférence demande tout de même de préserver la lisibilité des tests. Une comparaison complexe peut nécessiter plusieurs assertions ou un comparateur explicite. Si l’équipe finit par reconstruire une bibliothèque complète de comparaison de graphes, le bénéfice de simplicité devient discutable.

| Option | Intérêt | Limite à examiner |
|---|---|---|
| Assertions natives de xUnit | Mon premier choix dans un projet xUnit, avec des vérifications explicites et aucune bibliothèque d’assertions supplémentaire | Certaines comparaisons complexes demandent davantage de code |
| [AwesomeAssertions](https://github.com/AwesomeAssertions/AwesomeAssertions), sous Apache 2.0 | Proximité avec Fluent Assertions pour un parc de tests qui exploite largement ses fonctionnalités | Dépendance supplémentaire et comportements à revalider selon les versions et les extensions |
| [Shouldly](https://github.com/shouldly/shouldly), sous BSD | Autre syntaxe d’assertions, avec des messages d’échec lisibles | Réécriture des tests et maintien d’une bibliothèque supplémentaire |

AwesomeAssertions reste une option pertinente lorsque le coût de réécriture d’un parc existant est élevé. Je le considère comme un compromis de migration à évaluer, plutôt que comme le remplacement à adopter automatiquement. Le choix doit préserver la précision des tests et leur facilité de maintenance.

Les exemples restent ici en xUnit. Fluent Assertions prend aussi en charge MSTest et NUnit. Un changement de bibliothèque d’assertions n’impose pas un changement de framework de tests.

### Un exemple d’intention à préserver

Supposons qu’un test doive vérifier une collection de lignes, y compris son ordre et le contenu de chaque élément. Une assertion sur sa taille seule serait insuffisante. Avec xUnit, l’intention peut s’écrire ainsi :

```csharp
using Xunit;

public sealed class VerificationLignesTests
{
    public sealed record Ligne(string Reference, int Quantite);

    private static void VerifierLignesAttendues(Ligne[] lignesObtenues)
    {
        Assert.Collection(
            lignesObtenues,
            ligne =>
            {
                Assert.Equal("A-100", ligne.Reference);
                Assert.Equal(2, ligne.Quantite);
            },
            ligne =>
            {
                Assert.Equal("B-200", ligne.Reference);
                Assert.Equal(1, ligne.Quantite);
            });
    }

    [Fact]
    public void VerifierLignesAttendues_LignesConformes_Reussit()
    {
        VerifierLignesAttendues(new[]
        {
            new Ligne("A-100", 2),
            new Ligne("B-200", 1)
        });
    }

    [Fact]
    public void VerifierLignesAttendues_OrdreInverse_LeveUneException()
    {
        Assert.ThrowsAny<Xunit.Sdk.XunitException>(() =>
            VerifierLignesAttendues(new[]
            {
                new Ligne("B-200", 1),
                new Ligne("A-100", 2)
            }));
    }
}
```

Cet exemple illustre la qualification d’une assertion, pas un test métier complet. Le second cas sert à vérifier que la nouvelle formulation rejette bien un résultat incorrect. Dans une migration réelle, il faut choisir les contre-exemples selon les règles de l’ancien test : ordre, doublons, membres absents ou tolérance numérique.

**Ma recommandation :** privilégier les assertions natives de xUnit lorsqu’elles expriment clairement les attentes. Commencer l’évaluation par les assertions complexes et les extensions internes pour identifier les cas où une bibliothèque spécialisée conserve une valeur réelle. Une suite qui reste verte constitue une première indication. Il faut aussi vérifier qu’elle échoue toujours sur les écarts qu’elle devait détecter.

## 5. AutoMapper : choisir un mapping lisible et facile à maintenir

Le coût d’une migration AutoMapper dépend davantage des comportements utilisés que du nombre de profils. Copier quelques propriétés est généralement simple. Reproduire des conventions implicites, des convertisseurs personnalisés et des règles sur les valeurs nulles demande une analyse plus attentive.

Les [offres AutoMapper](https://automapper.io/) permettent de conserver l’outil selon le périmètre de l’équipe et son admissibilité. Cette option mérite d’être chiffrée lorsque les conventions et les transformations sont largement partagées dans l’organisation.

### Mapperly : rendre le code de transformation inspectable

[Mapperly](https://mapperly.riok.app/) génère les transformations à la compilation, sous licence Apache 2.0. Le code produit peut être inspecté, ce qui facilite la compréhension du mapping et réduit le travail de résolution au runtime.

Dans mon article sur [la performance de Mapperly](https://codewithfrenchy.com/posts/performance-mapperly/), je présente une comparaison avec AutoMapper réalisée avec BenchmarkDotNet. Elle illustre l’intérêt de la génération de code sur un scénario de transformation simple. Les résultats restent liés aux versions et au scénario mesurés. Pour une migration, je retiens aussi la lisibilité du code généré et la capacité de l’équipe à comprendre les transformations qu’elle maintient.

Ce modèle demande cependant de traduire les conventions et les personnalisations existantes. Les valeurs nulles, les énumérations, les objets imbriqués et les collections peuvent révéler des différences de comportement. Une propriété ignorée ou une valeur par défaut différente peut modifier le résultat sans empêcher la compilation.

### Le mapping manuel : du contrôle, avec du code à maintenir

Un mapping manuel convient lorsque la transformation est courte, spécifique ou porte une décision métier qui mérite d’être visible. Il évite de cacher une règle importante dans une convention générale.

Il augmente toutefois le volume de code et le risque d’oublier une nouvelle propriété. Une bibliothèque de génération peut rester préférable pour des transformations répétitives. Le mapping manuel et Mapperly peuvent donc cohabiter, selon la complexité et la responsabilité de chaque transformation.

### Migrer par ensembles de transformations

Commencer par un ensemble cohérent de mappings permet de comparer les résultats avant de généraliser. Les tests doivent couvrir les cas nominaux, mais aussi les valeurs absentes, les collections vides et les mises à jour partielles. Pour ces dernières, vérifier notamment qu’une valeur non fournie n’efface pas une information qui devait être conservée.

Si l’application utilise aussi des projections `IQueryable`, vérifier séparément leur traduction et les données chargées. Ce point mérite une validation ciblée lorsqu’il existe dans le projet.

**Ma recommandation :** évaluer Mapperly pour les transformations répétitives et conserver un mapping manuel lorsque cela rend une règle plus claire. Migrer progressivement, en validant les comportements avant de chercher un gain de performance.

## 6. MediatR : migrer les comportements autour des handlers

MediatR est souvent présenté comme un simple intermédiaire entre une requête et son handler. Dans une application réelle, ses pipelines peuvent porter la validation, l’autorisation, les transactions, l’audit ou la journalisation.

La [présentation de MediatR](https://mediatr.io/) rappelle son rôle de médiation en processus. Il ne fournit pas, à lui seul, une livraison durable entre services. CQRS et l’organisation en tranches verticales peuvent également être conservés sans cette bibliothèque.

### Un médiateur minimal sans bibliothèque externe

Lorsque les besoins se limitent à envoyer une requête vers un seul gestionnaire et à récupérer son résultat, il est possible d’implémenter une version très simple du patron Mediator dans le projet. Le patron architectural ne dépend pas de MediatR ni d’un autre package.

Le principe peut rester limité à un contrat de requête, un contrat de gestionnaire et un composant qui dirige chaque requête vers le gestionnaire enregistré. Avec quelques types de requêtes, des associations explicites peuvent suffire. Il n’est pas nécessaire d’introduire immédiatement de la découverte automatique ou un système extensible de pipelines.

Cette approche conserve un point d’entrée commun et une séparation entre l’appelant et le traitement. Le code reste sous le contrôle de l’équipe et ne couvre que les besoins réels du projet. En contrepartie, l’équipe devient responsable de son comportement et de ses tests, notamment lorsqu’aucun gestionnaire n’est enregistré, qu’une opération est annulée ou qu’un traitement échoue. La résolution des gestionnaires doit aussi respecter les durées de vie des dépendances utilisées.

La limite apparaît lorsque les exigences s’accumulent : notifications à plusieurs destinataires, pipelines génériques, stratégies d’exécution ou gestion sophistiquée des erreurs. Si le médiateur interne devient un framework à part entière, une bibliothèque maintenue peut redevenir plus avantageuse. L’objectif est de conserver une implémentation volontairement limitée, avec des responsabilités faciles à expliquer.

### Appeler directement des services applicatifs

Lorsque le médiateur ne fait que transmettre une poignée d’appels, des interfaces applicatives peuvent simplifier le chemin d’exécution. La dépendance devient explicite et le diagnostic plus direct.

Les comportements transversaux doivent alors trouver une place claire : décorateurs, filtres, middleware ou composition applicative. Copier la validation et l’autorisation dans chaque point d’entrée remplace une dépendance externe par une maintenance dispersée.

### Wolverine : une décision plus large

[Wolverine](https://github.com/JasperFx/wolverine) couvre à la fois la médiation et la messagerie. Son intérêt augmente si l’organisation veut faire évoluer ces deux dimensions ensemble. Son adoption demande en contrepartie de comprendre un modèle plus étendu. Elle doit être justifiée comme un choix de plateforme, avec ses propres garanties à qualifier.

**Ma recommandation :** pour des besoins limités, évaluer d’abord un médiateur minimal interne ou des appels directs à des services applicatifs. Conserver un médiateur lorsque le point d’entrée commun apporte une valeur concrète. Vérifier que la simplification préserve la validation, l’autorisation, l’annulation et les transactions. Retenir une bibliothèque plus complète lorsque ses capacités répondent à des besoins établis et justifient son coût de maintenance ou de licence.

## 7. MassTransit : remplacer un système de traitement, pas seulement une API

Un bus peut prendre en charge les tentatives, la livraison différée, la sérialisation, la topologie du broker, les outbox et les sagas. La [documentation MassTransit](https://masstransit.massient.com/) présente cette portée de framework applicatif distribué.

Si ces fonctions sont largement utilisées, l’achat d’une licence doit faire partie de la comparaison. La valeur à préserver comprend les procédures d’exploitation, les outils de test et les connaissances des équipes.

### Trois alternatives, trois compromis

| Option | Intérêt | Limite à examiner |
|---|---|---|
| [Rebus](https://github.com/rebus-org/Rebus) | Bus sous MIT, volontairement sobre, avec intégrations séparées, support et outillage commerciaux possibles | Vérifier chaque capacité utilisée et sa prise en charge par les intégrations retenues |
| [Wolverine](https://github.com/JasperFx/wolverine) | Médiation et messagerie dans un même modèle, projet MIT avec support commercial disponible | Changement de conventions, persistance, transport et garanties à qualifier ensemble |
| [NServiceBus et la plateforme Particular](https://particular.net/pricing) | Offre intégrant messagerie, supervision et récupération des erreurs | Nouvelle dépendance commerciale et migration de modèle, conditions selon l’offre et le périmètre |

Aucune de ces options ne peut être qualifiée de remplacement équivalent sans inventaire des usages. NServiceBus peut répondre à un objectif de support ou d’outillage. Particular propose notamment une édition Community encadrée par des conditions et des limites de capacité. Il ne constitue pas, par principe, une stratégie de suppression des coûts de licence.

De même, un bus plus sobre ne signifie pas nécessairement l’absence de sagas ou de fonctions de résilience. L’évaluation doit porter sur les capacités requises, leur implémentation et les garanties du transport choisi.

### La compatibilité doit être vérifiée sur le broker

Conserver les mêmes classes C# et le même broker ne garantit pas qu’un nouveau consommateur lira correctement les messages existants. Les noms de contrats, enveloppes, en-têtes, identifiants de corrélation et conventions de routage peuvent différer.

Un premier test d’interopérabilité consiste à faire produire un message par l’ancien système et à le traiter avec le nouveau, puis à vérifier le sens inverse si la transition le requiert. Les tests doivent utiliser le transport cible : un transport en mémoire ne reproduit pas ses verrous, ses délais et ses modes de livraison.

### Les sagas en cours ne disparaissent pas au déploiement

Une saga possède un état, des règles de corrélation et parfois des messages différés. Il faut décider si les instances existantes termineront dans l’ancien système ou si leur état sera converti.

Laisser les anciennes instances terminer réduit le risque de conversion, mais prolonge la coexistence. Migrer les états peut raccourcir cette période, au prix d’une transformation et d’une validation plus exigeantes. Les nouvelles instances doivent être dirigées vers un propriétaire clair.

Le même raisonnement s’applique aux messages en erreur, aux outbox non vidées et aux tâches planifiées. Une migration n’est pas terminée parce que tous les nouveaux producteurs utilisent le nouveau package.

**Ma recommandation :** commencer par un processus métier représentatif, avec un cas de reprise et un état persistant s’ils existent dans le système. Comparer les options à partir de cette expérience, y compris la possibilité de conserver MassTransit.

## 8. Organiser une migration qui reste maîtrisable

### Inventorier les usages directs et indirects

La liste des packages est un début. Elle doit être reliée aux dépôts, aux propriétaires et aux fonctions utilisées. Ajouter les packages transitifs, les extensions, les modèles de projets internes et les composants de test.

Classer ensuite les usages : remplacement local, comportement transversal ou infrastructure avec état. Ce classement permet de concentrer l’effort sur les dépendances qui peuvent modifier le fonctionnement du produit.

### Construire un pilote représentatif

Le pilote doit répondre aux questions susceptibles de changer la décision. Un mapping de trois chaînes ne valide pas une migration qui comporte des convertisseurs personnalisés et des objets imbriqués. Un consommateur sans persistance ne valide pas le remplacement d’un moteur de sagas.

Consigner les changements mécaniques, les adaptations de comportement, les écarts non résolus et les opérations à maintenir pendant la transition. Ces observations alimentent l’estimation et le guide de migration.

### Définir des critères d’acceptation adaptés

| Famille | Comportements à préserver | Preuves attendues |
|---|---|---|
| Assertions | Règles de comparaison et écarts rejetés | Cas valides et contre-exemples sur les assertions sensibles |
| Mapping | Valeurs, nullabilité, collections et mises à jour partielles | Comparaison des résultats et tests des transformations sensibles |
| Médiation | Pipelines, durées de vie, transactions et erreurs | Scénarios applicatifs avec leurs comportements transversaux |
| Cache | Expiration, éviction, reconnexion et cohérence | Tests sur le moteur cible, charge et panne représentatives |
| Messagerie | Livraison, reprise, corrélation et états en cours | Interopérabilité, redémarrage et incidents sur le transport cible |

Les comparaisons de performance doivent reprendre des charges utiles et une configuration comparables. Les benchmarks des projets servent à orienter une investigation. Ils ne démontrent pas le gain pour une application donnée.

### Choisir une coexistence compatible avec les effets métier

Pour une transformation pure en mémoire, exécuter temporairement deux implémentations peut permettre de comparer les sorties. Pour un traitement qui débite un compte ou envoie une notification, la même approche pourrait produire deux effets réels.

Un traitement en parallèle doit donc être isolé ou privé d’effets externes lorsqu’il sert seulement à l’observation. Deux consommateurs sur une même file peuvent se partager les messages au lieu de recevoir chacun une copie. Deux abonnements peuvent au contraire dupliquer les traitements. La topologie de test doit correspondre à l’intention.

### Préparer le retour arrière avant la bascule

Revenir au package précédent suffit parfois pour une modification sans état. Cela ne suffit plus si le nouveau système a écrit des données, consommé des messages ou modifié des schémas.

Le plan doit préciser ce que l’ancienne version peut encore lire, quelles écritures doivent être réconciliées et comment traiter le travail déjà effectué. Selon les contraintes, une correction en avant peut être préférable à un retour arrière devenu risqué.

Conserver les artefacts nécessaires au redéploiement et valider les droits d’usage applicables. Définir également les seuils qui déclencheront l’arrêt ou l’ajustement : erreurs, retard de traitement, écarts fonctionnels ou surcharge en aval.

### Construire une checklist adaptée au projet

Une checklist commune aide les équipes à ne pas oublier les engagements de la migration. Elle doit rester proportionnée à la dépendance. Le remplacement d’assertions n’exige pas le plan de reprise d’un système de messagerie avec des sagas persistantes.

- [ ] Identifier les versions exactes, les usages et les responsables concernés.
- [ ] Confirmer les conditions de licence applicables à l’organisation et aux environnements.
- [ ] Documenter les options et les compromis dans le registre de décisions.
- [ ] Valider un pilote avec les cas qui présentent réellement un risque.
- [ ] Définir les critères fonctionnels et opérationnels d’acceptation.
- [ ] Préparer la coexistence et la reprise, y compris les données et traitements déjà engagés.
- [ ] Mettre à jour les commandes, exemples, pipelines et procédures utilisés par les équipes.
- [ ] Fixer les conditions de retrait de l’ancien outil et la prochaine revue de la décision.

La documentation de développement fait partie du résultat. Une migration techniquement correcte peut encore ralentir l’équipe si les nouveaux arrivants suivent un README obsolète ou si seuls deux développeurs connaissent la configuration nécessaire.

### Retirer l’ancien outil

Après validation, supprimer les packages, configurations, droits, tâches et procédures devenus inutiles. Mettre à jour les modèles de projets pour éviter que la dépendance soit réintroduite.

La clôture doit inclure les états historiques et les traitements différés. Une coexistence sans fin ajoute une charge permanente et rend les responsabilités moins lisibles.

## 9. Prévenir les prochaines surprises sans alourdir chaque changement

Une gouvernance utile donne de la visibilité et permet de décider rapidement. Elle n’a pas besoin de soumettre toutes les bibliothèques à un comité identique.

Pour une dépendance structurante, conserver un registre court : responsable interne, versions utilisées, licence, fonctions critiques, support disponible et prochaine revue. Associer architecture, sécurité, juridique et approvisionnement selon les questions à résoudre.

La [gestion centralisée des packages NuGet](https://learn.microsoft.com/en-us/nuget/consume-packages/central-package-management) avec `Directory.Packages.props` facilite la cohérence des versions. Elle ne constitue pas à elle seule un verrouillage complet du graphe restauré. Les mécanismes de restauration reproductible et le contrôle des dépendances transitives doivent être traités séparément.

Les inventaires logiciels et les scans de licences aident à détecter les changements. Ils ne déterminent pas toujours l’éligibilité commerciale, le titulaire du contrat ou la couverture d’un usage particulier. Une alerte doit conduire à une qualification, sans devenir automatiquement une interdiction de mise à jour.

### Documenter la décision et ses conditions de révision

L’inventaire indique ce qui est utilisé. Le [registre de décisions](https://codewithfrenchy.com/posts/registre-decisions/) explique pourquoi l’organisation conserve ou remplace une dépendance. Les deux répondent à des besoins complémentaires.

Pour ce type de choix, consigner le contexte, les options évaluées, les résultats du pilote, les coûts estimés et les compromis acceptés. Préciser aussi ce qui déclencherait une révision, par exemple une fonctionnalité devenue indispensable, une fin de maintenance ou une modification des conditions commerciales.

Cette trace évite qu’une nouvelle équipe interprète une solution transitoire comme un standard permanent. Elle permet aussi de réexaminer le choix à partir de ses hypothèses d’origine, sans recommencer toute l’analyse.

### Placer les abstractions là où elles protègent une responsabilité

Une interface métier autour d’un cache de tarifs peut limiter la propagation des détails techniques. Une interface qui reproduit toute l’API d’un bus conserve souvent ses concepts et ses contraintes, tout en ajoutant du code à maintenir.

Il faut donc chercher à isoler les usages utiles plutôt qu’à promettre l’interchangeabilité totale. Les contrats, les données et les comportements d’exploitation constituent souvent les dépendances les plus difficiles à déplacer.

### Évaluer la santé du projet, y compris son financement

Pour une alternative, examiner les versions publiées, les corrections, la documentation, la compatibilité avec la plateforme et les possibilités de support. Le nombre de téléchargements ou d’étoiles ne prouve pas, seul, la capacité à maintenir le produit.

La gouvernance d’un fork mérite la même attention que celle du projet d’origine. Une licence permissive ouvre des possibilités. Elle ne garantit pas que l’organisation dispose des compétences pour assurer elle-même la maintenance.

Contribuer, financer du support ou soutenir les mainteneurs peut faire partie d’une stratégie de maîtrise des dépendances. La pérennité de l’écosystème reste un intérêt partagé entre ceux qui produisent les outils et ceux qui en dépendent.

## Conclusion

Un changement de licence doit déclencher une décision documentée, proportionnée à l’usage de l’outil. Il peut justifier un achat, une période de maintien sur une branche existante ou une migration. Aucune de ces trajectoires n’est avantageuse indépendamment du contexte.

Le choix d’une alternative commence par les comportements à préserver. Pour les assertions, il s’agit de maintenir la force des tests. Pour le mapping et la médiation, de conserver les règles, les transactions et les performances. Pour le cache et la messagerie, il faut aussi protéger les données, les traitements en cours et les procédures de reprise.

Une migration réussie laisse les équipes avec une solution qu’elles comprennent, qu’elles peuvent exploiter et dont elles acceptent les coûts. C’est sur ce résultat que l’architecture doit être jugée, et sur la capacité à réexaminer sereinement la prochaine évolution de l’écosystème.
