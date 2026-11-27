---
title: Une dépendance change de licence - comment choisir une alternative et réussir la migration ?
date: 2027-12-31 19:00:00 -0400
categories: [architecture, bonnes-pratiques]
tags: [dotnet, open-source, licence-commerciale, gestion-dependances, migration, gouvernance, redis, fluentassertions, automapper, mediatr, masstransit]
---

> ⚠️ À RELIRE + valider le format du code

## Préambule

Dans mon article [Changements de licences dans l’écosystème .NET : quelles implications ?](https://codewithfrenchy.com/posts/changements-licences-ecosysteme-dotnet/), j’abordais les motivations des mainteneurs et les conséquences de ces évolutions pour les équipes. Ici, je m’intéresse à la suite : que faire des applications qui utilisent déjà ces outils?

La réponse va au-delà d’une ligne dans un fichier projet. Une bibliothèque peut structurer les tests, les transformations de données ou l’exécution des traitements. Un cache peut contenir des sessions actives. Un bus de messages peut transporter des processus métier qui se poursuivront après le prochain déploiement.

Le changement appelle une comparaison entre plusieurs trajectoires : continuer avec une licence adaptée, conserver une version antérieure dans un cadre maîtrisé ou préparer une migration. Chacune a ses coûts et ses contraintes.

Cet article propose une démarche à l’intention des architectes et des responsables techniques, puis l’applique à cinq cas : Redis, Fluent Assertions, AutoMapper, MediatR et MassTransit. Les exemples illustrent différentes trajectoires. Ils ne constituent ni un catalogue exhaustif ni un classement universel.

*Les repères de licence et les offres mentionnées ont été vérifiés dans les sources officielles le 23 septembre 2026. L’analyse doit toujours porter sur la version et les conditions effectivement retenues par l’organisation.*

## 1. Comprendre ce qui change réellement

Une annonce de commercialisation ne décrit pas, à elle seule, les obligations de chaque utilisateur. Il faut distinguer le droit d’utiliser une version, l’accès aux nouvelles versions, le support et les mécanismes techniques de validation de licence.

### Licence, gratuité et support sont des sujets différents

Un logiciel open source peut être utilisé commercialement et faire l’objet d’une offre de support payante. À l’inverse, un logiciel dont le code est consultable n’est pas nécessairement open source. Une édition gratuite peut aussi imposer des restrictions selon l’organisation, ses revenus ou l’usage du produit.

Le copyleft ne signifie pas automatiquement qu’un logiciel est interdit en entreprise. Les obligations dépendent notamment de la licence applicable, des modifications et des modalités de distribution ou d’accès. La décision juridique doit tenir compte de ces faits et être prise avec les personnes compétentes.

L’absence de blocage technique ne vaut pas autorisation. Une bibliothèque peut fonctionner sans clé valide tout en exigeant une licence pour l’usage concerné. De même, l’achat d’une licence ne garantit pas nécessairement tous les niveaux de support ni tous les droits de redistribution.

### Les repères utiles pour les cinq outils

| Outil | Changement à prendre en compte | Conséquence pour l’évaluation |
|---|---|---|
| Redis | BSD-3-Clause jusqu’à 7.2, puis RSALv2 ou SSPLv1 dès 7.4. Redis 8 ajoute l’option AGPLv3 | Identifier le moteur, sa version, ses modules et le mode d’exploitation |
| Fluent Assertions | Usage commercial soumis à licence à partir de la version 8. Branche 7 conservée open source | Examiner les usages de développement et de test, ainsi que la trajectoire de maintenance |
| AutoMapper | À partir de la version 15, offre commerciale ou option RPL-1.5 selon les conditions applicables | Vérifier l’éligibilité Community, le périmètre des développeurs et les conventions de mapping |
| MediatR | À partir de la version 13, offre commerciale ou option RPL-1.5 selon les conditions applicables | Vérifier les mêmes conditions et inventorier les comportements des pipelines |
| MassTransit | Licence requise à partir de la version 9 | Évaluer l’offre, sa configuration et la profondeur d’intégration du framework |

L’[historique officiel des licences Redis](https://redis.io/legal/licenses/) précise que le moteur lui-même est concerné dès la version 7.4, pas seulement les modules. Redis 8 propose trois licences au choix, dont les obligations ne s’appliquent pas toutes simultanément. Utiliser un client .NET pour communiquer avec un serveur Redis ne permet pas, à lui seul, de conclure que toute l’application doit être publiée sous AGPL.

Pour [Fluent Assertions](https://fluentassertions.com/), la version 7 demeure open source et le projet annonce des corrections sur cette branche. La conserver ne signifie donc pas automatiquement utiliser un produit abandonné. Il faut examiner la maintenance effectivement disponible.

La [FAQ de Lucky Penny Software](https://luckypennysoftware.com/faq) distingue l’offre commerciale d’AutoMapper et de MediatR, l’admissibilité Community et une voie sous RPL-1.5 avec obligations réciproques. Ces possibilités doivent être évaluées séparément. Une petite équipe n’est pas automatiquement admissible à Community. Le statut de l’organisation compte, notamment pour les organismes gouvernementaux ou quasi gouvernementaux, qui en sont exclus. En consultation, il faut également vérifier quelle entité doit détenir la licence et couvrir les intervenants.

La [documentation de licence de MassTransit](https://masstransit.massient.com/configuration/license) confirme le régime de la version 9 et ses mécanismes de configuration. Les conditions de renouvellement, de redéploiement et de support doivent être vérifiées dans l’offre choisie, sans interpréter une ancienne annonce comme un engagement toujours applicable.

## 2. Comparer trois trajectoires, avec leurs coûts et leurs limites

### Continuer avec une licence adaptée

Conserver l’outil permet de préserver les compétences, les intégrations et les comportements déjà qualifiés. Cette option peut être avantageuse lorsque l’outil structure une plateforme utilisée par plusieurs équipes.

Le financement des mainteneurs peut soutenir les corrections, la documentation et le développement. Le bénéfice dépend toutefois de ce que l’offre garantit. Acheter un droit d’usage ne signifie pas nécessairement obtenir une correction prioritaire ou un engagement de disponibilité.

Les coûts comprennent le renouvellement, l’approvisionnement, le suivi du périmètre couvert et les éventuels changements de palier. Il faut aussi savoir ce qui reste permis après l’expiration de la licence. L’organisation devrait pouvoir expliquer pourquoi la dépendance reste acceptable et dans quelles circonstances elle sera réévaluée.

### Conserver une version antérieure

Cette option évite une migration précipitée et donne le temps d’évaluer les alternatives. Elle peut rester viable tant que la branche répond aux exigences de sécurité, de compatibilité et de maintenance.

Le risque apparaît lorsque « nous restons sur cette version » devient une absence de décision. Les SDK évoluent, les dépendances transitives changent et les intégrations tierces peuvent cesser de prendre en charge l’ancienne branche.

Il faut préciser la version autorisée, le responsable du suivi et les circonstances qui déclencheront une nouvelle décision : vulnérabilité non corrigée, incompatibilité avec le prochain runtime, fonctionnalité indispensable ou disparition du support nécessaire. Une date de revue évite que la situation passe inaperçue.

### Migrer vers une alternative

Migrer peut réduire une dépendance commerciale, améliorer la lisibilité du code ou rapprocher le système des standards de l’organisation. Cette démarche peut aussi déplacer les coûts vers le développement et l’exploitation.

Une alternative sans redevance exige toujours du support, des mises à jour et une capacité de diagnostic. Si l’équipe doit reconstruire les garanties de l’ancien outil, le gain financier apparent peut disparaître.

| Trajectoire | Bénéfice principal | Coût ou risque à accepter |
|---|---|---|
| Conserver avec une licence adaptée | Continuité technique et opérationnelle | Conditions commerciales, renouvellement et dépendance au fournisseur |
| Conserver une version antérieure | Temps pour décider, faible changement immédiat | Compatibilité et maintenance à surveiller |
| Migrer | Nouvelle trajectoire technique ou contractuelle | Requalification, apprentissage, coexistence et risque de régression |

### Calculer un coût complet

La comparaison doit utiliser un horizon commun, par exemple celui de la feuille de route du produit. Pour chaque option, intégrez :

- Les droits d’usage, le support et l’infrastructure.
- L’analyse, l’adaptation du code et la validation des comportements.
- La formation, les guides et les changements de pipelines.
- L’exploitation de la transition et le retrait de l’ancien outil.
- Les fonctionnalités métier reportées pendant ce travail.

Distinguez les dépenses certaines des risques estimés. Une comparaison fondée sur quelques hypothèses explicites et plusieurs scénarios est plus utile qu’un montant unique présenté comme précis. Le coût de migration devrait être estimé à partir d’un pilote représentatif, pas seulement du nombre de références NuGet.

## 3. Redis : séparer le choix du moteur de celui de l’exploitation

Redis peut servir à conserver des valeurs recalculables, des sessions, des compteurs, des verrous ou des données utilisées par des traitements asynchrones. Ces usages n’ont pas la même tolérance aux interruptions ou à la perte de données.

Avant de choisir un remplaçant, inventoriez les commandes, les scripts, les modules, les volumes et les garanties attendues. Vérifiez aussi si l’enjeu concerne la licence du moteur ou le coût d’exploitation du service.

### Valkey : un candidat pour conserver une proximité fonctionnelle

[Valkey](https://valkey.io/) est un projet open source sous BSD-3-Clause, issu de Redis et accueilli par la Linux Foundation. C’est un candidat pertinent si l’organisation veut conserver une licence permissive et si ses usages correspondent aux fonctionnalités prises en charge.

Une partie importante de l’intégration cliente peut ainsi être préservée. Un protocole commun ne garantit toutefois pas l’équivalence de toutes les commandes, de tous les modules ou de tous les formats de persistance entre les versions.

La [documentation de migration vers Valkey](https://valkey.io/topics/migration/) indique les chemins de migration compatibles. Un scénario documenté depuis Redis 7.2 ne doit pas être transposé automatiquement à une version plus récente ou à un service enrichi de fonctionnalités spécifiques.

### Azure Managed Redis : déléguer une partie de l’exploitation

[Azure Managed Redis](https://learn.microsoft.com/en-us/azure/redis/overview) offre une autre trajectoire : conserver un service Redis dans un environnement managé par Azure. Cette option peut convenir lorsque les besoins de disponibilité, de maintenance et d’intégration à la plateforme dominent la décision.

Elle conserve un coût récurrent et une dépendance au fournisseur. Il faut valider les capacités du niveau de service, la connectivité, l’authentification et les contraintes de déploiement. Passer à un service managé ne signifie pas quitter l’écosystème Redis.

### Préserver la sémantique du cache

Pour un cache entièrement reconstructible, une bascule vers une instance vide peut être envisageable. Mesurez alors l’effet du réchauffement sur les bases de données et les services en aval. Un afflux de défauts de cache peut déplacer la panne vers la source de données.

Pour des sessions ou des compteurs métier, la perte de valeurs peut modifier le comportement du produit. La migration doit préserver les données nécessaires, leur expiration et leur cohérence. Deux stockages alimentés en parallèle peuvent diverger. Une double écriture ne garantit pas leur synchronisation.

Pour une trajectoire permissive compatible avec les usages existants, évaluez Valkey. Pour une exploitation managée, évaluez Azure Managed Redis. Dans les deux cas, qualifiez les expirations, les évictions, les reconnexions et les pannes avant de conclure qu’il suffit de changer la connexion.

## 4. Fluent Assertions : préserver la force des tests

Une migration d’assertions semble locale : remplacer un package, modifier des espaces de noms et corriger les appels qui ne compilent plus. Le principal risque est de rendre certains tests moins exigeants.

Une comparaison de graphes d’objets peut ignorer l’ordre, exclure des membres ou appliquer des règles particulières aux types et aux valeurs nulles. La remplacer par une comparaison du seul nombre d’éléments ne protège plus le même comportement.

### Ma préférence : les assertions natives de xUnit

Dans un projet qui utilise déjà xUnit, je privilégie ses assertions natives. Elles expriment directement les comportements attendus tout en limitant les dépendances à maintenir. Pour vérifier des valeurs, des exceptions ou le contenu d’une collection, je préfère généralement cette approche à l’ajout d’une bibliothèque d’assertions supplémentaire.

Les tests doivent toutefois rester lisibles. Une comparaison complexe peut nécessiter plusieurs assertions ou un comparateur explicite. Si l’équipe finit par reconstruire une bibliothèque complète de comparaison de graphes, le gain de simplicité devient discutable.

| Option | Intérêt | Limite à examiner |
|---|---|---|
| Assertions natives de xUnit | Mon premier choix dans un projet xUnit, avec des vérifications explicites et aucune bibliothèque d’assertions supplémentaire | Certaines comparaisons complexes demandent davantage de code |
| [AwesomeAssertions](https://github.com/AwesomeAssertions/AwesomeAssertions), sous Apache 2.0 | Proximité avec Fluent Assertions pour un parc de tests qui exploite largement ses fonctionnalités | Dépendance supplémentaire et comportements à revalider selon les versions et les extensions |
| [Shouldly](https://github.com/shouldly/shouldly), sous BSD | Autre syntaxe d’assertions, avec des messages d’échec lisibles | Réécriture des tests et maintien d’une bibliothèque supplémentaire |

AwesomeAssertions reste une option pertinente si le coût de réécriture d’un parc existant est élevé. Je le considérerais comme un compromis de migration à évaluer, plutôt que comme le remplacement à adopter automatiquement. Le choix doit préserver la précision des tests et leur facilité de maintenance.

Les exemples de cet article utilisent xUnit. Fluent Assertions prend aussi en charge MSTest et NUnit. Changer de bibliothèque d’assertions n’impose pas de changer de framework de tests.

### Un exemple d’intention à préserver

Supposons qu’un test doive vérifier une collection de lignes, y compris leur ordre et le contenu de chaque élément. Une assertion sur sa taille ne suffirait pas. Avec xUnit, on peut exprimer l’intention ainsi :

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

Cet exemple sert à qualifier une assertion, pas à représenter un test métier complet. Le second cas vérifie que la nouvelle formulation rejette un résultat incorrect. Dans une migration réelle, choisissez les contre-exemples en fonction des règles de l’ancien test : ordre, doublons, membres absents ou tolérance numérique.

Je commencerais l’évaluation par les assertions complexes et les extensions internes, pour repérer les cas où une bibliothèque spécialisée conserve une valeur. Une suite qui reste verte ne suffit pas. Il faut vérifier qu’elle échoue toujours sur les écarts qu’elle doit détecter.

## 5. AutoMapper : choisir un mapping lisible et facile à maintenir

Le coût d’une migration AutoMapper dépend davantage des comportements utilisés que du nombre de profils. Copier quelques propriétés est généralement simple. Reproduire des conventions implicites, des convertisseurs personnalisés et des règles sur les valeurs nulles demande une analyse plus attentive.

Les [offres AutoMapper](https://automapper.io/) permettent de conserver l’outil selon le périmètre de l’équipe et son admissibilité. Cette option mérite d’être chiffrée lorsque les conventions et les transformations sont largement partagées dans l’organisation.

### Mapperly : rendre le code de transformation inspectable

[Mapperly](https://mapperly.riok.app/) génère les transformations à la compilation sous licence Apache 2.0. Le code produit peut être inspecté, ce qui facilite la compréhension du mapping et réduit le travail de résolution au runtime.

Dans mon article sur [la performance de Mapperly](https://codewithfrenchy.com/posts/performance-mapperly/), je présente une comparaison avec AutoMapper réalisée avec BenchmarkDotNet. Elle illustre l’intérêt de la génération de code pour un scénario de transformation simple. Les résultats dépendent toutefois des versions et du scénario mesuré. Pour une migration, j’évaluerais aussi la lisibilité du code généré et la capacité de l’équipe à comprendre les transformations qu’elle maintient.

Il faut traduire les conventions et les personnalisations existantes. Les valeurs nulles, les énumérations, les objets imbriqués et les collections peuvent révéler des différences de comportement. Une propriété ignorée ou une valeur par défaut différente peut modifier le résultat sans empêcher la compilation.

### Le mapping manuel : du contrôle, avec du code à maintenir

Un mapping manuel convient lorsque la transformation est courte, spécifique ou porte une décision métier qui mérite d’être visible. Il évite de cacher une règle importante dans une convention générale.

Il augmente toutefois le volume de code et le risque d’oublier une nouvelle propriété. Une bibliothèque de génération peut rester préférable pour les transformations répétitives. Mapperly et le mapping manuel peuvent donc cohabiter selon la complexité et la responsabilité de chaque transformation.

### Migrer par ensembles de transformations

Commencez par un ensemble cohérent de mappings pour comparer les résultats avant de généraliser. Les tests doivent couvrir les cas nominaux, mais aussi les valeurs absentes, les collections vides et les mises à jour partielles. Pour ces dernières, vérifiez notamment qu’une valeur non fournie n’efface pas une information qui devait être conservée.

Si l’application utilise aussi des projections `IQueryable`, vérifiez séparément leur traduction et les données chargées. Ce point mérite une validation ciblée lorsqu’il existe dans le projet.

J’évaluerais Mapperly pour les transformations répétitives et conserverais un mapping manuel lorsqu’il rend une règle plus claire. La migration devrait avancer progressivement, en validant les comportements avant de chercher un gain de performance.

## 6. MediatR : migrer les comportements autour des handlers

MediatR est souvent présenté comme un simple intermédiaire entre une requête et son handler. Dans une application réelle, ses pipelines peuvent aussi porter la validation, l’autorisation, les transactions, l’audit ou la journalisation.

La [présentation de MediatR](https://mediatr.io/) décrit son rôle de médiation en processus. Il ne fournit pas, à lui seul, une livraison durable entre services. CQRS et l’organisation en tranches verticales peuvent également être conservés sans cette bibliothèque.

### Un médiateur minimal sans bibliothèque externe

Si le besoin se limite à envoyer une requête vers un gestionnaire et à récupérer son résultat, une implémentation simple du patron Mediator peut suffire dans le projet. Le patron architectural ne dépend pas de MediatR ni d’un autre package.

Le principe peut se limiter à un contrat de requête, un contrat de gestionnaire et un composant qui dirige chaque requête vers le gestionnaire enregistré. Avec quelques types de requêtes, des associations explicites peuvent suffire. Il n’est pas nécessaire d’introduire immédiatement de la découverte automatique ou un système extensible de pipelines.

Cette approche conserve un point d’entrée commun et sépare l’appelant du traitement. L’équipe contrôle le code et ne met en œuvre que les besoins réels. En contrepartie, elle devient responsable du comportement et des tests, notamment lorsqu’aucun gestionnaire n’est enregistré, qu’une opération est annulée ou qu’un traitement échoue. La résolution des gestionnaires doit aussi respecter la durée de vie des dépendances utilisées.

La limite apparaît lorsque les exigences s’accumulent : notifications à plusieurs destinataires, pipelines génériques, stratégies d’exécution ou gestion élaborée des erreurs. Si le médiateur interne devient un framework à part entière, une bibliothèque maintenue peut redevenir plus avantageuse. L’implémentation devrait rester limitée, avec des responsabilités faciles à expliquer.

### Appeler directement des services applicatifs

Lorsque le médiateur ne fait que transmettre quelques appels, des interfaces applicatives peuvent simplifier le chemin d’exécution. La dépendance devient explicite et le diagnostic plus direct.

Les comportements transversaux doivent alors trouver une place claire : décorateurs, filtres, middleware ou composition applicative. Copier la validation et l’autorisation dans chaque point d’entrée remplace une dépendance externe par une maintenance dispersée.

### Wolverine : une décision plus large

[Wolverine](https://github.com/JasperFx/wolverine) couvre à la fois la médiation et la messagerie. Son intérêt augmente si l’organisation veut faire évoluer ces deux dimensions ensemble. Son adoption demande en contrepartie de comprendre un modèle plus étendu. Il faut donc la traiter comme un choix de plateforme et qualifier ses garanties.

Pour des besoins limités, j’évaluerais d’abord un médiateur minimal interne ou des appels directs à des services applicatifs. Conservez un médiateur si le point d’entrée commun apporte une valeur concrète. Vérifiez que la simplification préserve la validation, l’autorisation, l’annulation et les transactions. Une bibliothèque plus complète se justifie lorsque ses capacités répondent à des besoins établis et compensent son coût de maintenance ou de licence.

## 7. MassTransit : remplacer un système de traitement, pas seulement une API

Un bus peut prendre en charge les tentatives, la livraison différée, la sérialisation, la topologie du broker, les outbox et les sagas. La [documentation MassTransit](https://masstransit.massient.com/) décrit cette portée de framework applicatif distribué.

Si ces fonctions sont largement utilisées, l’achat d’une licence doit faire partie de la comparaison. Il faut aussi préserver les procédures d’exploitation, les outils de test et les connaissances des équipes.

### Trois alternatives, trois compromis

| Option | Intérêt | Limite à examiner |
|---|---|---|
| [Rebus](https://github.com/rebus-org/Rebus) | Bus sous MIT, volontairement sobre, avec intégrations séparées, support et outillage commerciaux possibles | Vérifier chaque capacité utilisée et sa prise en charge par les intégrations retenues |
| [Wolverine](https://github.com/JasperFx/wolverine) | Médiation et messagerie dans un même modèle, projet MIT avec support commercial disponible | Changement de conventions, persistance, transport et garanties à qualifier ensemble |
| [NServiceBus et la plateforme Particular](https://particular.net/pricing) | Offre intégrant messagerie, supervision et récupération des erreurs | Nouvelle dépendance commerciale et migration de modèle, conditions selon l’offre et le périmètre |

Aucune de ces options ne peut être qualifiée de remplacement équivalent sans inventaire des usages. NServiceBus peut répondre à un objectif de support ou d’outillage. Particular propose notamment une édition Community encadrée par des conditions et des limites de capacité. Elle ne constitue pas, par principe, une stratégie de suppression des coûts de licence.

Un bus plus sobre ne signifie pas nécessairement l’absence de sagas ou de fonctions de résilience. L’évaluation doit porter sur les capacités requises, leur implémentation et les garanties du transport choisi.

### La compatibilité doit être vérifiée sur le broker

Conserver les mêmes classes C# et le même broker ne garantit pas qu’un nouveau consommateur pourra lire correctement les messages existants. Les noms de contrats, les enveloppes, les en-têtes, les identifiants de corrélation et les conventions de routage peuvent différer.

Un premier test d’interopérabilité consiste à faire produire un message par l’ancien système et à le traiter avec le nouveau. Si la transition l’exige, vérifiez aussi le sens inverse. Les tests doivent utiliser le transport cible : un transport en mémoire ne reproduit pas ses verrous, ses délais ni ses modes de livraison.

### Les sagas en cours ne disparaissent pas au déploiement

Une saga possède un état, des règles de corrélation et parfois des messages différés. Il faut décider si les instances existantes termineront dans l’ancien système ou si leur état sera converti.

Laisser les anciennes instances terminer réduit le risque de conversion, mais prolonge la coexistence. Migrer les états peut raccourcir cette période, au prix d’une transformation et d’une validation plus exigeantes. Les nouvelles instances doivent être dirigées vers un propriétaire clairement identifié.

Le même raisonnement s’applique aux messages en erreur, aux outbox non vidées et aux tâches planifiées. La migration n’est pas terminée parce que tous les nouveaux producteurs utilisent le nouveau package.

Commencez par un processus métier représentatif, avec un cas de reprise et un état persistant si le système en possède. Comparez les options à partir de cette expérience, y compris la possibilité de conserver MassTransit.

## 8. Organiser une migration qui reste maîtrisable

### Inventorier les usages directs et indirects

La liste des packages est un point de départ. Reliez chaque dépendance aux dépôts, aux propriétaires et aux fonctions utilisées. Incluez les packages transitifs, les extensions, les modèles de projets internes et les composants de test.

Classez ensuite les usages : remplacement local, comportement transversal ou infrastructure avec état. Cette catégorisation permet de concentrer l’effort sur les dépendances susceptibles de modifier le fonctionnement du produit.

### Construire un pilote représentatif

Le pilote doit répondre aux questions susceptibles de changer la décision. Un mapping de trois chaînes ne valide pas une migration qui comporte des convertisseurs personnalisés et des objets imbriqués. Un consommateur sans persistance ne valide pas le remplacement d’un moteur de sagas.

Consignez les changements mécaniques, les adaptations de comportement, les écarts non résolus et les opérations à maintenir pendant la transition. Ces observations alimentent l’estimation et le guide de migration.

### Définir des critères d’acceptation adaptés

| Famille | Comportements à préserver | Preuves attendues |
|---|---|---|
| Assertions | Règles de comparaison et écarts rejetés | Cas valides et contre-exemples sur les assertions sensibles |
| Mapping | Valeurs, nullabilité, collections et mises à jour partielles | Comparaison des résultats et tests des transformations sensibles |
| Médiation | Pipelines, durées de vie, transactions et erreurs | Scénarios applicatifs avec leurs comportements transversaux |
| Cache | Expiration, éviction, reconnexion et cohérence | Tests sur le moteur cible, charge et panne représentatives |
| Messagerie | Livraison, reprise, corrélation et états en cours | Interopérabilité, redémarrage et incidents sur le transport cible |

Les comparaisons de performance doivent utiliser des charges et une configuration comparables. Les benchmarks des projets peuvent orienter une investigation, mais ne démontrent pas le gain pour une application donnée.

### Choisir une coexistence compatible avec les effets métier

Pour une transformation pure en mémoire, exécuter temporairement deux implémentations peut permettre de comparer leurs sorties. Pour un traitement qui débite un compte ou envoie une notification, la même approche risque de produire deux effets réels.

Un traitement en parallèle doit donc être isolé ou privé d’effets externes lorsqu’il sert uniquement à l’observation. Deux consommateurs sur une même file peuvent se partager les messages au lieu de recevoir chacun une copie. À l’inverse, deux abonnements peuvent dupliquer les traitements. La topologie de test doit correspondre à l’intention.

### Préparer le retour arrière avant la bascule

Revenir au package précédent suffit parfois pour une modification sans état. Cela ne suffit plus si le nouveau système a écrit des données, consommé des messages ou modifié des schémas.

Le plan doit préciser ce que l’ancienne version peut encore lire, quelles écritures doivent être réconciliées et comment traiter le travail déjà effectué. Selon les contraintes, une correction en avant peut être préférable à un retour arrière devenu risqué.

Conservez les artefacts nécessaires au redéploiement et validez les droits d’usage applicables. Définissez aussi les seuils qui déclencheront l’arrêt ou l’ajustement : erreurs, retard de traitement, écarts fonctionnels ou surcharge en aval.

### Construire une checklist adaptée au projet

Une checklist commune aide les équipes à suivre les engagements de la migration. Elle doit rester proportionnée à la dépendance. Le remplacement d’assertions n’exige pas le plan de reprise d’un système de messagerie avec des sagas persistantes.

- [ ] Identifier les versions exactes, les usages et les responsables concernés.
- [ ] Confirmer les conditions de licence applicables à l’organisation et aux environnements.
- [ ] Documenter les options et les compromis dans le registre de décisions.
- [ ] Valider un pilote avec les cas qui présentent réellement un risque.
- [ ] Définir les critères fonctionnels et opérationnels d’acceptation.
- [ ] Préparer la coexistence et la reprise, y compris les données et traitements déjà engagés.
- [ ] Mettre à jour les commandes, exemples, pipelines et procédures utilisés par les équipes.
- [ ] Fixer les conditions de retrait de l’ancien outil et la prochaine revue de la décision.

La documentation de développement fait partie du résultat. Une migration techniquement correcte peut ralentir l’équipe si les nouveaux arrivants suivent un README obsolète ou si seuls quelques développeurs connaissent la configuration nécessaire.

### Retirer l’ancien outil

Après validation, supprimez les packages, configurations, droits, tâches et procédures devenus inutiles. Mettez aussi à jour les modèles de projets pour éviter que la dépendance soit réintroduite.

La clôture doit couvrir les états historiques et les traitements différés. Une coexistence sans fin ajoute une charge permanente et rend les responsabilités moins lisibles.

## 9. Prévenir les prochaines surprises sans alourdir chaque changement

Une gouvernance utile donne de la visibilité et permet de décider rapidement. Elle n’a pas besoin de soumettre toutes les bibliothèques au même comité.

Pour une dépendance structurante, tenez un registre concis : responsable interne, versions utilisées, licence, fonctions critiques, support disponible et prochaine revue. Faites intervenir l’architecture, la sécurité, le juridique et l’approvisionnement selon les questions à résoudre.

La [gestion centralisée des packages NuGet](https://learn.microsoft.com/en-us/nuget/consume-packages/central-package-management) avec `Directory.Packages.props` facilite la cohérence des versions. Elle ne verrouille pas à elle seule tout le graphe restauré. La restauration reproductible et le contrôle des dépendances transitives doivent être traités séparément.

Les inventaires logiciels et les scans de licences aident à détecter les changements, mais ne déterminent pas toujours l’admissibilité commerciale, le titulaire du contrat ou la couverture d’un usage particulier. Une alerte doit déclencher une analyse, pas automatiquement interdire une mise à jour.

### Documenter la décision et ses conditions de révision

L’inventaire indique ce qui est utilisé. Le [registre de décisions](https://codewithfrenchy.com/posts/registre-decisions/) explique pourquoi l’organisation conserve ou remplace une dépendance. Ces outils répondent à des besoins complémentaires.

Pour ce type de choix, consignez le contexte, les options évaluées, les résultats du pilote, les coûts estimés et les compromis acceptés. Précisez aussi ce qui déclencherait une révision, par exemple une fonctionnalité devenue indispensable, une fin de maintenance ou une modification des conditions commerciales.

Cette trace évite qu’une nouvelle équipe interprète une solution transitoire comme un standard permanent. Elle permet de réexaminer le choix à partir de ses hypothèses d’origine, sans reprendre l’analyse depuis le début.

### Placer les abstractions là où elles protègent une responsabilité

Une interface métier autour d’un cache de tarifs peut limiter la propagation des détails techniques. Une interface qui reproduit toute l’API d’un bus conserve souvent ses concepts et ses contraintes tout en ajoutant du code à maintenir.

Cherchez à isoler les usages utiles plutôt qu’à promettre une interchangeabilité totale. Les contrats, les données et les comportements d’exploitation comptent souvent parmi les dépendances les plus difficiles à déplacer.

### Évaluer la santé du projet, y compris son financement

Pour une alternative, examinez les versions publiées, les corrections, la documentation, la compatibilité avec la plateforme et les possibilités de support. Le nombre de téléchargements ou d’étoiles ne prouve pas, à lui seul, la capacité à maintenir le produit.

La gouvernance d’un fork mérite la même attention que celle du projet d’origine. Une licence permissive ouvre des possibilités, mais ne garantit pas que l’organisation dispose des compétences nécessaires pour assurer elle-même la maintenance.

Contribuer, financer du support ou soutenir les mainteneurs peut faire partie d’une stratégie de maîtrise des dépendances. La pérennité de l’écosystème concerne autant ceux qui produisent les outils que ceux qui en dépendent.

## Conclusion

Un changement de licence appelle une décision documentée, proportionnée à l’usage de l’outil. Il peut justifier un achat, le maintien temporaire sur une branche existante ou une migration. Aucune de ces trajectoires n’est avantageuse dans tous les contextes.

Pour choisir une alternative, commencez par les comportements à préserver. Pour les assertions, vérifiez que les tests restent aussi exigeants. Pour le mapping et la médiation, examinez les règles, les transactions et les performances. Pour le cache et la messagerie, incluez les données, les traitements en cours et les procédures de reprise.

Une migration réussie laisse à l’équipe une solution qu’elle comprend, peut exploiter et dont elle accepte les coûts. C’est à partir de ce résultat qu’il faut évaluer l’architecture et réexaminer les prochaines évolutions de l’écosystème.
