---
title: Essais automatisés d’architecture dans un projet .NET
date: 2026-12-29 20:00:00 -0400
categories: [outil-developpement, architecture]
tags: [dotnet, essais]
---

## Introduction

Dans les projets .NET complexes, par exemple appliquant le _Domain-Driven Design (DDD)_ ou une _Clean Architecture_ stricte, il devient crucial de s’assurer que la structure du code respecte en permanence les règles d’architecture prévues. À mesure qu’une base de code grandit et que de nombreux développeurs contribuent, le risque de dérive architecturale augmente : sans contrôle, on peut introduire des dépendances entre couches qui violent le design initial, avec à la clé une perte de maintenabilité et de lisibilité du code. L’idéal serait de **valider automatiquement l’architecture** pour prévenir ces dérives dès qu’elles apparaissent.

_Exemple de représentation_ Clean Architecture _en cercles concentriques (modèle_ Onion_). Les dépendances pointent vers le noyau du système : la partie cœur métier au centre n’a aucune dépendance sur les couches extérieures, tandis que les couches externes (UI, Infrastructure) peuvent dépendre de ce qui est plus central.

Dans une architecture de type _Clean Architecture_, on s’attend par exemple à ce que la couche _Domaine_ ou _Application Core_ ne dépende d’aucun détail d’infrastructure ou d’interface utilisateur. Ce sont au contraire les couches extérieures (UI, Infrastructure) qui dépendent des abstractions du domaine. Cette organisation en couches strictes assure une meilleure encapsulation : si l’on change l’implémentation de la persistance des données ou du framework de présentation, le cœur de l’application n’en souffre pas. Pour garantir cela dans la durée, les équipes matures mettent en place des **tests automatisés d’architecture** qui vont vérifier en continu que ces règles de dépendances et de structure sont bien respectées.

## Pourquoi des tests d’architecture automatisés ?

Chaque équipe définit des conventions sur la façon d’organiser le code. Souvent ces règles sont documentées sur un wiki ou transmises oralement, mais on sait que les développeurs lisent peu la documentation et que ces règles "non écrites" finissent par varier d’une personne à l’autre. Les revues de code peuvent attraper certains écarts, mais ce processus manuel est lent, fastidieux, et laisse place à des incohérences.

Automatiser la vérification de l’architecture apporte une solution élégante : on transforme ces conventions en tests unitaires _exécutables_. Ainsi, les règles d’architecture de l’équipe sont "gravées dans le marbre" et ne peuvent plus être ignorées ou mal interprétées. Quand un nouveau développeur rejoint le projet ou qu’une refactorisation majeure a lieu, les tests d’architecture servent de garde-fous immédiats.

Les avantages sont multiples : cela crée une **documentation vivante** des décisions d’architecture (les tests décrivent les contraintes à respecter dans le code). On bénéficie des mêmes gains que pour les autres tests automatisés : gain de temps et retour rapide en cas de régression, sans devoir jouer en permanence le rôle du _gendarme_ en revue de code. L’application des règles devient uniforme pour tous les membres de l’équipe, améliorant la cohérence du code. En attrapant très tôt les violations (dès le build ou la PR), on évite que des erreurs de conception ne s’accumulent et ne transforment le code en un "_big ball of mud_" ingérable sur le long terme. En résumé, ces tests automatisés sécurisent vos refactorings et garantissent l’intégrité architecturale du système dans la durée.

## Tour d’horizon des outils disponibles

En environnement .NET, deux principales bibliothèques se partagent le domaine des tests d’architecture automatisés : **NetArchTest** et **ArchUnitNET**.

[NetArchTest](https://github.com/BenMorris/NetArchTest) est un outil qui fournit une API fluide pour définir des règles architecturales sous forme de tests unitaires. Il permet par exemple d’exprimer qu’une classe de service ne doit pas dépendre d’un contrôleur, ou qu’une classe de la couche _Domaine_ ne doit pas se trouver dans le namespace de l’Infrastructure. Cependant, NetArchTest semble en perte de vitesse, aucune mise à jour n’a été publiée depuis 2023, le dépôt GitHub n’ayant plus de commits depuis plus de deux ans. Cela laisse penser que l’outil n’est plus activement maintenu.

[ArchUnitNET](https://github.com/TNG/ArchUnitNET), de son côté, est une librairie plus récente et soutenue par la communauté (créée par TNG Tech) pour tester l’architecture des applications C#. C’est un portage de l’outil Java **ArchUnit**, bien connu dans l’écosystème JVM. ArchUnitNET permet de définir via une API _fluent_ des contraintes solides entre vos classes, modules et namespaces, et de valider automatiquement que votre code reste conforme à l’architecture voulue. L’outil est open-source (Apache 2.0) et bénéficie d’un support actif : au moment où j’écris, le dépôt reçoit des contributions régulières (dernier commit il y a seulement quelques heures !). Face à un NetArchTest vieillissant, le choix d’ArchUnitNET s’est imposé pour notre projet.

## Pourquoi j’ai choisi ArchUnitNET

Voici les principaux atouts qui nous ont fait adopter **ArchUnitNET** :

- **Support actif et pérennité** – ArchUnitNET est activement maintenu par ses contributeurs et aligné sur ArchUnit (Java) qui fait référence. On évite ainsi le risque d’un outil abandonné en rase campagne à l’inverse de NetArchTest, resté figé depuis 2023. La fréquence des commits et des releases d’ArchUnitNET témoigne d’une communauté dynamique (dernière _release_ très récente, projet mis à jour en continu).
- **API fluide et expressive** – La syntaxe proposée par ArchUnitNET est à la fois puissante et lisible. Elle permet de définir finement des règles d’architecture en code C# clair, grâce à un DSL (_Domain-Specific Language_) interne inspiré de ArchUnit Java. Par exemple, on peut écrire que _" les classes du namespace X devraient résider dans telle couche et ne pas dépendre de tel autre namespace "_ en une seule ligne expressive. Cette API fluent rend la création de règles assez naturelle, tout en produisant des messages d’erreur explicites en cas de non-respect (on peut même ajouter un commentaire avec Because("raison") pour documenter la règle violée).
- **Intégration facile dans le projet** – ArchUnitNET s’installe comme n’importe quel package NuGet et s’utilise au sein de vos projets de tests existants. Il propose des extensions pour les frameworks de tests populaires (xUnit, NUnit, MSTest) afin de s’intégrer au mieux dans votre pipeline de tests. En pratique, il suffit d’ajouter le package NuGet et de charger les assemblys de votre application dans le moteur ArchUnitNET, puis d’écrire des tests classiques qui exécutent les vérifications d’architecture. Aucune étape lourde ou outil externe n’est requis, ce qui rend son adoption très simple.
- **Portée fonctionnelle** – ArchUnitNET couvre un large éventail de possibilités : contrôle des dépendances entre assemblies/namespaces, conventions de nommage des classes, structure des couches, règles sur l’héritage, présence d’attributs spécifiques, etc. Cette richesse permet de traduire pratiquement toutes les règles d’architecture imaginables en tests automatisés. De plus, étant un port direct d’ArchUnit, on bénéficie d’un certain recul et de bonnes pratiques déjà éprouvées.

En résumé, ArchUnitNET s’est distingué pour nous par son **support actif**, son **expressivité** et sa **facilité d’adoption** dans notre écosystème .NET, ce qui en fait un choix de confiance pour automatiser nos tests d’architecture.

## Exemple complet avec ArchUnitNET

Pour illustrer l’utilisation d’ArchUnitNET, prenons un exemple fictif de projet .NET organisé selon une architecture en couches façon _Clean Architecture_. Imaginons une application **API Web** qui comporte trois couches principales : **Application**, **Domaine**, **Infrastructure**.

- La couche _Application_ contient la logique applicative (cas d’usage, services applicatifs) et dépend du _Domaine_.
- La couche _Domaine_ encapsule les règles métiers (entités, agrégats, interfaces de référentiels, etc.) et est indépendante.
- La couche _Infrastructure_ fournit les implémentations concrètes (accès aux données, services externes) et dépend de _Application_ et _Domaine_.
- L’API (controllers web) fait office de couche de présentation et interagit avec _Application_.

Notre objectif est d’écrire des tests pour vérifier automatiquement quelques règles d’architecture de ce projet :
  1. **Indépendance des couches** - Par exemple, la couche _Application_ ne doit **jamais** dépendre directement de la couche _Infrastructure_. Ainsi, aucune classe du namespace `MonProjet.Application` ne doit référencer une classe de `MonProjet.Infrastructure` (respect de la dépendance inverse).
  2. **Conventions de localisation** – Les classes de contrôleur ASP.NET (suffixées par `Controller`) doivent résider dans le namespace `MonProjet.Api.Controllers` prévu à cet effet, et pas ailleurs.

Commençons par installer ArchUnitNET dans notre projet de tests. Si vous utilisez **xUnit**, je vous recommande l’extension dédiée ArchUnitNET.xUnit qui fournit tout le nécessaire pour écrire des `[Fact]` vérifiant l’architecture :

```bash
dotnet add package TngTech.ArchUnitNET.xUnit
```

Une fois le package ajouté, nous allons pouvoir définir l’architecture à analyser dans nos tests. ArchUnitNET va charger nos assemblies (.dll) et construire un modèle interne des types et de leurs dépendances. On peut ensuite formuler des règles via l’API fluent. Typiquement, on crée une classe de test _ArchitectureTests_ où l’on initialise en static l’objet Architecture et éventuellement quelques _providers_ de types réutilisables représentant nos couches :

```csharp
using ArchUnitNET.Domain;
using ArchUnitNET.Loader;
using ArchUnitNET.Fluent;
using static ArchUnitNET.Fluent.ArchRuleDefinition;
using Xunit;

namespace MonProjet.ArchTests
{
    // Chargement initial de l'architecture (assemblies Application, Domain, Infrastructure)
    public class ArchitectureTests
    {
        private static readonly Architecture Architecture = new ArchLoader()
            .LoadAssemblies(
                System.Reflection.Assembly.Load("MonProjet.Application"),
                System.Reflection.Assembly.Load("MonProjet.Domain"),
                System.Reflection.Assembly.Load("MonProjet.Infrastructure")
            )
            .Build();

        // Définition de "filtres" réutilisables pour nos couches
        private readonly IObjectProvider<IType> ApplicationLayer = 
            Types().That().ResideInNamespace("MonProjet.Application").As("Couche Application");

        private readonly IObjectProvider<IType> InfrastructureLayer = 
            Types().That().ResideInNamespace("MonProjet.Infrastructure").As("Couche Infrastructure");

        // ... on pourrait définir de même DomainLayer, etc.

        // 1. Test de règle d'architecture : Application ne dépend pas d'Infrastructure
        [Fact]
        public void Application_NeDoitPasDependre_DeInfrastructure()
        {
            IArchRule regleCoucheApp =
                Types().That().Are(ApplicationLayer)
                      .Should().NotDependOnAny(InfrastructureLayer)
                      .Because("la couche Application doit rester indépendante de l'Infrastructure");

            regleCoucheApp.Check(Architecture);
        }

        // 2. Test de convention : les Controllers sont dans le namespace MonProjet.Api.Controllers
        [Fact]
        public void Controllers_DansNamespaceControllers()
        {
            IArchRule regleControllers =
                Classes().That().HaveNameEndingWith("Controller")
                        .Should().ResideInNamespace("MonProjet.Api.Controllers")
                        .Because("les contrôleurs web doivent être déclarés dans MonProjet.Api.Controllers");

            regleControllers.Check(Architecture);
        }
    }
}

```

Dans le premier test (`Application_NeDoitPasDependre_DeInfrastructure`), on utilise la méthode fluide `Types().That().Are(ApplicationLayer)` pour sélectionner tous les types de la couche Application, puis on exprime la contrainte `Should().NotDependOnAny(InfrastructureLayer)`. Autrement dit, aucun de ces types ne doit avoir de dépendance vers un type de la couche **Infrastructure**, ce qui empêche par exemple une classe applicative d’utiliser directement un repository concret défini dans l’infrastructure. Le `.Because("...")` permet d’ajouter un message explicatif qui s’affichera si la règle est violée, ce qui documente le pourquoi de la contrainte (ici "la couche Application doit rester indépendante"). Enfin, on termine par Check(Architecture) pour exécuter la règle sur l’architecture chargée. Si une dépendance interdite est détectée, le test échouera en listant précisément quelles classes posent problème, facilitant la correction.

Le second test (`Controllers_DansNamespaceControllers`) illustre une vérification de convention de nommage et de localisation. On sélectionne toutes les classes dont le nom se termine par _Controller_ puis on impose qu’elles résident dans le namespace `MonProjet.Api.Controllers`. Là encore, si par mégarde un contrôleur était placé dans un autre namespace (ou mal nommé), le test le signalerait immédiatement. Ce genre de règle agit comme une documentation vivante sur l’organisation attendue des composants : si quelqu’un crée un nouveau contrôleur au mauvais endroit, le test le rappellera à l’ordre sans attendre la revue de code.

Ces tests d’architecture s’exécutent comme n’importe quels tests unitaires via votre _runner_ habituel (ou en CI). Lorsqu’une règle échoue, ArchUnitNET fournit un message d’erreur explicite listant les éléments fautifs et la raison fournie. L’équipe dispose donc d’un _feedback_ clair et actionnable pour corriger l’architecture avant même que le code n’atterrisse en production.

## Checklist : Ce que je valide via les tests d’architecture

Pour récapituler, voici une **checklist** de quelques règles que j’automatise dans les projets .NET à l’aide de tests d’architecture :

- **Séparation stricte des couches** – Aucune dépendance illégitime entre couches n’est tolérée. Par exemple, la couche _Domaine_ n’importe pas de classe de l’Infrastructure, l’UI n’accède pas directement à la DAL sans passer par la couche Application, etc. Chaque niveau ne connaît que les contrats (interfaces) définis aux niveaux inférieurs, jamais les implémentations concrètes des niveaux supérieurs.
- **Conventions de nommage et de localisation** – Les classes de contrôleur ASP.NET MVC/Web API doivent être suffixées par _Controller_ et résider dans le namespace dédié (exemple : `MonProjet.Api.Controllers`). De même, on peut imposer que les classes de service se terminent par _Service_, que les interfaces commencent par un _I_, etc., afin de garder une cohérence dans tout le code.
- **Pas de dépendances transverses non désirées** – On vérifie qu’aucun composant ne "court-circuite" l’architecture en appelant du code qu’il ne devrait pas. Par exemple, un service de l’UI ne va pas directement appeler une classe de _Domaine_ sans passer par Application, un contrôleur ne doit pas accéder directement à la base de données, etc. Ces règles encapsulent les _bonnes pratiques_ de l’architecture pour éviter les couplages cachés.
- **Isolation du cœur métier** – Les couches externes (_Interface Adapters_, comme les contrôleurs/web, ou la couche Infrastructure) ne doivent pas référencer directement les entités du domaine. On préfère qu’elles passent par des _DTOs_ ou des interfaces. Cela évite de fuiter la logique interne du domaine vers l’extérieur et renforce le principe d’inversion des dépendances (le domaine expose des abstractions que les autres implémentent, sans quoi c’est le domaine qui se mettrait à dépendre de détails externes).
- **Autres conventions spécifiques** – Tout ce qui relève de règles d’architecture propres à votre projet peut faire l’objet d’un test. Par exemple, vérifier que _les repositories implémentent bien les interfaces du domaine_, que _les classes dans tel namespace sont_ sealed _ou encore qu’aucun service applicatif n’est statique_. L’important est de capturer ces invariants architecturaux sous forme de tests pour s’assurer qu’ils seront vrais en permanence.

Bien sûr, la checklist ci-dessus dépend du contexte de chaque projet. À vous d’identifier les points névralgiques de votre architecture et de les couvrir par des tests adéquats. L’avantage est qu’une fois en place, ce filet de sécurité tourne en continu et vous alerte au moindre écart.

## Conclusion

Après plusieurs mois d’utilisation, je peux confirmer que les tests d’architecture automatisés apportent une réelle valeur à long terme. Au début, cela demande un petit investissement (formaliser les règles, écrire les tests), mais cet effort est largement rentabilisé par la suite. Notre code reste plus **propre** et **cohérente** : fini les dépendances hasardeuses introduites par inadvertance, car le pipeline de build les attrape immédiatement. Cela facilite aussi les **refactorings** de grande ampleur, on peut restructurer le code en confiance, en sachant que si une règle d’architecture est brisée, un test rouge nous le signalera aussitôt.

Mon retour d’expérience est que ces tests jouent un rôle de **garde-fou invisible** mais nécessaires. Ils soulèvent les problèmes d’architecture très en amont (dès la PR ou même avant, en local) et évitent des revues de code interminables sur la structure du projet. L’équipe gagne en sérénité et peut se concentrer sur le métier, sachant que le respect des principes d’architecture est surveillé automatiquement.

Pour les équipes .NET déjà bien rodées qui souhaitent franchir un cap supplémentaire en qualité, je conseille d’expérimenter les tests d’architecture. Commencez petit à petit, avec quelques règles simples sur les dépendances entre couches par exemple, puis élargissez la portée progressivement. Impliquez l’équipe dans la définition de ces règles, cela peut même servir de base à des discussions saines sur l’architecture souhaitée. Intégrez enfin ces tests au pipeline CI/CD afin qu’ils tournent à chaque commit et empêchent toute régression architecturale.

En conclusion, les tests d’architecture automatisés sont un outil de plus dans l’arsenal des équipes .NET matures pour **assurer la pérennité du design logiciel**. Ils apportent une forme de qualité logicielle souvent négligée, en faisant vivre les principes d’architecture au même titre que le code métier. Si votre projet commence à prendre de l’ampleur, n’hésitez pas à les adopter : votre architecture (et vos coéquipiers) vous remercieront sur le long terme !
