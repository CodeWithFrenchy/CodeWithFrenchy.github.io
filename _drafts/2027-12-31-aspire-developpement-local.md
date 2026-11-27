---
title: Tirer profit d'Aspire - simplifier le développement local
date: 2027-12-31 19:00:00 -0400
categories: [outil-developpement, devops]
tags: [dotnet, aspire, devex, developpement-local, orchestration, observabilite, conteneurs, cloud-native]
---

> ⚠️ À RELIRE + valider le format du code

## Préambule

Il y a quelques mois, j’ai donné une formation intitulée « Tirer profit d’Aspire - Simplifiez votre développement local ». Le sujet rejoint une préoccupation que j’ai depuis longtemps en architecture et en expérience développeur : le temps passé à préparer l’environnement avant de pouvoir travailler sur l’application.

Démarrer une API paraît simple. La situation se complique lorsqu’on y ajoute une interface Web, une base de données, un cache et un traitement asynchrone. Il faut lancer les bons processus, fournir les bonnes configurations et comprendre pourquoi un service attend une dépendance qui ne répond pas.

C’est dans ce contexte qu’Aspire m’intéresse. Il permet de décrire l’environnement applicatif dans le code, de le démarrer de façon cohérente et d’observer les échanges entre ses composants. C’est utile aux développeurs qui connaissent déjà la solution, à ceux qui la découvrent et, de plus en plus, aux agents IA qui interviennent dans le dépôt.

Dans cet article, je reprends les principaux sujets de la formation en suivant un fil conducteur .NET. Je présente les premiers pas avec Aspire, son intégration à une application existante, ses outils de diagnostic et de test, puis ses possibilités de déploiement et d’utilisation avec l’IA. J’aborde aussi les limites à considérer avant d’en faire un standard d’équipe.

## Le développement local fait partie du produit

Dans certaines équipes, une partie du fonctionnement de l’application existe surtout dans la tête des personnes qui y travaillent depuis longtemps.

Il faut savoir quel script lancer avant l’API, quel port utilise un service, quel fichier de configuration copier et quelle commande permet de recréer les données de départ. Le README donne quelques indications, mais les pratiques ont évolué depuis sa dernière mise à jour.

Ces difficultés ralentissent l’arrivée d’un nouveau collègue, compliquent les changements de branche et rendent certains problèmes difficiles à reproduire. Elles peuvent aussi pousser les développeurs à utiliser un environnement partagé, avec les dépendances et les interférences que cela entraîne.

À mes yeux, l’environnement local mérite le même souci de maintenabilité que le reste de la solution. Lorsqu’une modification ajoute une dépendance, le dépôt devrait aussi expliquer comment la démarrer et vérifier qu’elle fonctionne.

Aspire permet de rendre ces informations exécutables. La topologie de l’application devient du code versionné, que l’on peut réviser dans la même demande de fusion que les changements applicatifs.

## Aspire et le rôle de l'AppHost

D’abord connu sous le nom de .NET Aspire, [Aspire](https://aspire.dev/get-started/what-is-aspire/) propose une approche orientée code pour composer des applications et leurs dépendances. Il permet de réunir des projets .NET, des applications JavaScript ou Python, des conteneurs et d’autres processus.

L’AppHost est au centre de cette approche. Il décrit les ressources qui composent l’application, les informations dont elles ont besoin et leurs dépendances au démarrage.

Par exemple, l’AppHost peut indiquer qu’une interface Web utilise une API et que cette API dépend d’une base de données. Il peut lancer les projets applicatifs comme des processus locaux et les autres dépendances comme des conteneurs. Les développeurs conservent ainsi leur façon habituelle de déboguer le code, tandis que la préparation de l’environnement est automatisée.

Il faut distinguer ce modèle d’application de son exécution en production. En local, Aspire orchestre les processus et les conteneurs. Après le déploiement, la plateforme cible prend en charge l’exécution, le redémarrage et la mise à l’échelle selon sa propre configuration.

### Faut-il avoir des microservices?

Une application monolithique avec une base de données, un cache et un worker peut déjà bénéficier d’Aspire. Le gain dépend surtout du nombre de manipulations nécessaires pour travailler sur la solution.

Je commencerais donc par examiner les irritants concrets. Une petite application qui démarre déjà facilement a peu à gagner. À l’inverse, une solution composée de quelques projets peut demander beaucoup de préparation manuelle.

### Quelle place par rapport à Docker Compose?

Docker Compose reste pertinent pour décrire et démarrer un ensemble de conteneurs. Aspire ajoute un modèle applicatif qui réunit aussi les projets locaux, leurs connexions et leur observabilité.

| Besoin | Place d'Aspire |
| --- | --- |
| Démarrer uniquement quelques conteneurs | Évaluer le bénéfice par rapport au fichier Compose déjà en place |
| Déboguer plusieurs projets avec leurs dépendances | Les réunir dans un AppHost tout en conservant leur exécution locale |
| Configurer les communications entre services | Déclarer les références et fournir les informations de connexion |
| Suivre une requête entre plusieurs composants | Exploiter l’instrumentation OpenTelemetry et le tableau de bord |
| Réutiliser la topologie dans des tests | Démarrer l’AppHost depuis le projet de tests |

L’adoption peut être progressive. Une équipe peut garder son démarrage avec Compose et commencer par utiliser le tableau de bord Aspire pour observer la télémétrie.

## Créer une première application

### Préparer l'environnement

Pour suivre l’exemple en C#, les [prérequis officiels](https://aspire.dev/get-started/prerequisites/) demandent le SDK .NET 10 pour l’AppHost. Les applications orchestrées peuvent cibler .NET 8 ou une version ultérieure. Ajouter Aspire à une solution .NET 8 n’oblige donc pas à migrer immédiatement tous ses projets vers .NET 10.

Un environnement capable d’exécuter des conteneurs, comme Docker Desktop ou Podman, est nécessaire lorsque les ressources choisies reposent sur des conteneurs. Chaque application doit également disposer de ses propres prérequis.

Visual Studio Code avec l’extension Aspire, Visual Studio et Rider permettent de travailler avec ce modèle. Les Dev Containers et GitHub Codespaces sont aussi des options à évaluer pour standardiser les postes de l’équipe.

La [documentation d’installation du CLI](https://aspire.dev/get-started/install-cli/) présente les différentes méthodes. Dans un environnement .NET, on peut notamment utiliser :

```bash
dotnet tool install --global Aspire.Cli
aspire --version
```

Je recommande de documenter la version retenue par l’équipe et de coordonner les mises à jour du CLI et des packages. Installer automatiquement la dernière version facilite un essai, mais une équipe doit savoir sur quelle configuration elle travaille.

### Générer et lancer la solution

L’exemple reprend le modèle de départ utilisé pendant la formation :

```bash
aspire new aspire-starter -n AspireApp -o AspireApp
cd AspireApp
aspire run
```

Le [guide de création d’une première application](https://aspire.dev/get-started/first-app/?aspire-lang=csharp) détaille ce parcours. Le modèle génère notamment les projets suivants.

| Projet | Responsabilité |
| --- | --- |
| `AspireApp.ApiService` | Fournir une API de démonstration |
| `AspireApp.Web` | Afficher l’interface Blazor et consommer l’API |
| `AspireApp.AppHost` | Décrire les ressources et orchestrer leur démarrage |
| `AspireApp.ServiceDefaults` | Regrouper des configurations techniques communes |

Les détails du modèle peuvent varier selon la version installée. La répartition des responsabilités reste la même : le code applicatif vit dans ses projets, tandis que l’AppHost décrit leur assemblage.

### Lire l'AppHost

Voici la forme de l’AppHost présentée dans la démonstration :

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var apiMeteo = builder.AddProject<Projects.AspireApp_ApiService>("apiservice")
    .WithHttpHealthCheck("/health");

builder.AddProject<Projects.AspireApp_Web>("webfrontend")
    .WithExternalHttpEndpoints()
    .WithHttpHealthCheck("/health")
    .WithReference(apiMeteo)
    .WaitFor(apiMeteo);

builder.Build().Run();
```

`AddProject` déclare un projet comme ressource de l’application. Le nom `apiservice` devient son identifiant logique dans le modèle.

`WithReference(apiMeteo)` transmet au frontend la configuration nécessaire pour découvrir l’API. Côté application, un mécanisme compatible doit utiliser cette configuration. Dans le modèle .NET, les Service Defaults configurent notamment la découverte de services pour les clients HTTP.

`WaitFor(apiMeteo)` exprime une dépendance de démarrage. Aspire attend que la ressource atteigne l’état attendu et, si elle possède des contrôles de santé, que ceux-ci réussissent. Sans contrôle de santé pertinent, un processus démarré peut rester incapable de traiter une demande.

`WithHttpHealthCheck("/health")` indique à Aspire quelle route interroger. L’application doit réellement exposer cette route. La déclaration dans l’AppHost ne crée pas l’endpoint dans l’API.

Enfin, `WithExternalHttpEndpoints()` marque les endpoints HTTP pour une exposition externe, notamment lors de la publication vers une cible qui interprète cette configuration. L’accessibilité locale et l’exposition en production demandent chacune une configuration adaptée.

Ces méthodes répondent à des besoins distincts. Une référence fournit de la configuration, une attente coordonne le démarrage et la résilience pendant l’exécution dépend du comportement des clients et des services. Si la base de données devient indisponible après le lancement, l’application doit savoir gérer cette situation.

## Les Service Defaults comme point de départ commun

Le projet [Service Defaults](https://aspire.dev/get-started/csharp-service-defaults/) regroupe du code de configuration partagé, notamment pour OpenTelemetry, les contrôles de santé, la découverte de services et la résilience HTTP.

C’est un bon point de départ pour une équipe .NET. Il permet d’établir des conventions communes sans recopier la même configuration dans chaque projet. Ce code appartient toutefois à la solution et mérite d’être révisé.

Par exemple, les délais d’attente et les reprises doivent correspondre aux opérations exécutées. Réessayer un appel qui modifie des données exige de réfléchir à l’idempotence et aux conséquences d’une réponse perdue.

Les endpoints de santé demandent la même attention. Le modèle les expose par défaut dans l’environnement de développement. Pour les autres cibles, il faut définir leur accessibilité et ce qu’ils doivent vérifier. Une sonde qui répond systématiquement avec succès renseigne peu sur la capacité réelle d’un service à traiter les demandes.

Je garderais ce projet centré sur les conventions techniques. Les règles métier et les contrats fonctionnels relèvent d’autres responsabilités dans l’architecture.

## Observer l'application pendant qu'on la développe

Au lancement de l’AppHost, le tableau de bord présente les ressources, leur état et leurs points d’accès. Il permet aussi de consulter les sorties de console et la télémétrie des applications instrumentées.

Les informations utiles dépendent du problème rencontré.

| Information | Question à laquelle elle aide à répondre |
| --- | --- |
| État des ressources | Quel composant a échoué au démarrage ou attend une dépendance? |
| Sortie de console | Quelle erreur le processus a-t-il affichée? |
| Journaux structurés | Quelles propriétés et quelles erreurs accompagnent l’opération? |
| Traces distribuées | Par quels services la requête est-elle passée, et où a-t-elle pris du temps? |
| Métriques | Comment évoluent la durée des requêtes, les erreurs ou l’utilisation des ressources? |

Prenons une page qui affiche lentement des données. Une trace peut montrer que le frontend répond normalement, mais que l’API passe l’essentiel de son temps dans un appel externe. Elle peut aussi révéler qu’une reprise prolonge l’attente. On obtient alors une piste précise à vérifier.

Cette visibilité dépend de l’instrumentation. Aspire peut préparer les connexions et les conventions, mais ajouter une ressource à l’AppHost ne produit pas automatiquement toute la télémétrie souhaitée. Les appels personnalisés et les traitements métier peuvent nécessiter leurs propres activités, mesures ou journaux.

Cette approche aide aussi à valider l’observabilité avant la production. On peut vérifier qu’une erreur contient assez de contexte, que les traces se propagent entre les services et que les journaux ne contiennent pas de données sensibles.

### Utiliser uniquement le tableau de bord

Le [tableau de bord autonome](https://aspire.dev/dashboard/standalone/) peut recevoir de la télémétrie sans AppHost. Avec un CLI récent, on peut le démarrer ainsi :

```bash
aspire dashboard run
```

Une image de conteneur permet également de l’exécuter indépendamment. Les applications instrumentées doivent envoyer leur télémétrie vers l’endpoint OTLP configuré. Par défaut, le lancement par le CLI utilise le port 4317 pour OTLP/gRPC et le port 4318 pour OTLP/HTTP.

L’interface conserve une authentification par jeton. Sans service de ressources associé, ce mode sert de visualiseur de télémétrie, avec des fonctionnalités de contrôle des processus plus limitées.

Les données restent en mémoire et leur rétention est limitée. Ce fonctionnement convient au diagnostic local. Pour conserver un historique durable et gérer les alertes de production, il faut prévoir une plateforme d’observabilité adaptée.

## Intégrer les dépendances et les outils locaux

L’AppHost devient particulièrement utile lorsqu’il prend en charge les éléments qui demandent habituellement plusieurs manipulations : une base de données, un cache, un système de messagerie, un émulateur ou une interface d’administration.

Il faut distinguer l’intégration d’hébergement, utilisée par l’AppHost pour déclarer la ressource, de l’intégration cliente, utilisée dans l’application pour s’y connecter. Déclarer une base de données ne configure pas à lui seul tous les accès aux données du code métier.

La préparation des données doit aussi être explicite. L’environnement conserve-t-il un volume entre deux exécutions? Comment recréer une base propre? Qui applique les migrations? Quelles données rendent la démonstration reproductible? Les réponses à ces questions déterminent une bonne partie de la fiabilité du développement local.

### Le cas d'Azure Functions

Pendant la formation, j’ai aussi utilisé une [démonstration autour d’Azure Functions](https://github.com/alexis35115/Demo-Aspire-AzureFunction) pour explorer ce scénario.

L’[intégration officielle Azure Functions](https://aspire.dev/integrations/cloud/azure/azure-functions/azure-functions-get-started/) permet de représenter un projet Functions dans l’AppHost et de lui fournir les informations nécessaires à ses ressources. Elle prend notamment en charge le démarrage local et l’utilisation d’Azurite pour le stockage de l’hôte.

L’intérêt est de réunir le traitement, ses dépendances et les informations de diagnostic dans le même environnement de travail. On peut préparer un jeu de données, déclencher une opération et suivre son comportement avec les autres composants de la solution.

Les bindings, les déclencheurs et les environnements de déploiement ont toutefois leurs propres conditions de support. Il faut les vérifier pour l’application concernée.

Un émulateur reste une approximation du service distant. Les permissions, les restrictions réseau, certains comportements du service et les limites de capacité doivent encore être validés dans l’environnement cible.

## Ajouter Aspire à une application existante

« Aspireifier » une application consiste à décrire son environnement dans un AppHost et à y raccorder progressivement ses composants. Cette démarche peut conserver l’architecture actuelle.

La [documentation d’intégration à l’existant](https://aspire.dev/get-started/add-aspire-existing-app/) propose un parcours manuel et un parcours assisté par un agent. Le CLI peut créer une structure initiale à partir du dépôt :

```bash
aspire init
```

Je commencerais par un scénario que l’équipe exécute souvent, comme une API avec sa base de données. L’objectif initial serait de le faire démarrer de façon fiable sur un autre poste.

J’ajouterais ensuite les dépendances qui exigent le plus de manipulations, puis la télémétrie utile au diagnostic. Les scripts existants peuvent continuer de servir pendant la transition.

On peut mieux évaluer l’adoption lorsqu’elle vise un résultat observable : moins de paramètres à renseigner, moins de problèmes de ports ou un diagnostic plus rapide. La conversion complète du dépôt peut attendre que ce premier périmètre ait démontré son utilité.

## Tester les interactions avec l'AppHost

Une fois l’environnement décrit dans le code, on peut le démarrer depuis un projet de tests. Le package `Aspire.Hosting.Testing` fournit notamment `DistributedApplicationTestingBuilder` pour construire l’application, gérer son cycle de vie et accéder à ses ressources.

On peut ainsi valider des interactions réelles, par exemple un appel HTTP, une écriture dans une base de données ou un traitement déclenché par un système de messagerie. La topologie de développement sert de point de départ, avec les adaptations nécessaires au scénario.

### Un premier exemple avec xUnit

Pour reprendre la solution précédente, le projet de tests doit référencer `AspireApp.AppHost` et le package `Aspire.Hosting.Testing`, dans une version compatible avec l’AppHost. La [documentation des premiers tests Aspire](https://aspire.dev/testing/write-your-first-test/) présente aussi les modèles disponibles. MSTest et NUnit sont également supportés.

Voici un exemple avec les assertions natives de xUnit :

```csharp
using System.Net;
using System.Net.Http.Json;
using System.Text.Json;
using Aspire.Hosting;
using Aspire.Hosting.Testing;
using Xunit;

public sealed class PrevisionsMeteoTests
{
    [Fact]
    public async Task ObtenirPrevisions_ApiDisponible_RetourneDesPrevisions()
    {
        using var annulation = new CancellationTokenSource(
            TimeSpan.FromMinutes(2));

        var appHost = await DistributedApplicationTestingBuilder
            .CreateAsync<Projects.AspireApp_AppHost>(annulation.Token);

        await using var application = await appHost.BuildAsync(annulation.Token);
        await application.StartAsync(annulation.Token);

        await application.ResourceNotifications.WaitForResourceHealthyAsync(
            "apiservice", annulation.Token);

        using var client = application.CreateHttpClient("apiservice", "http");
        using var reponse = await client.GetAsync(
            "/weatherforecast", annulation.Token);

        Assert.Equal(HttpStatusCode.OK, reponse.StatusCode);

        var previsions = await reponse.Content.ReadFromJsonAsync<JsonElement[]>(
            cancellationToken: annulation.Token);

        Assert.NotNull(previsions);
        Assert.NotEmpty(previsions);
    }
}
```

Ce test vérifie le démarrage de l’environnement et la réception d’une réponse exploitable de l’API. Son périmètre reste limité. Pour valider un parcours utilisateur complet, il faudrait d’autres interactions, éventuellement avec un outil de test du navigateur.

L’attente sur l’état de santé évite une temporisation arbitraire. Le délai global borne l’exécution si l’environnement ne démarre pas. Le contrôle de santé doit toutefois refléter les dépendances nécessaires au scénario.

### Choisir le bon niveau de test

Je combinerais plusieurs niveaux selon ce qu’on cherche à vérifier.

| Niveau | Usage privilégié |
| --- | --- |
| Tests unitaires | Valider rapidement les règles métier et les cas limites |
| Tests d’intégration ciblés | Vérifier un accès aux données, un contrat ou une dépendance précise |
| Tests avec l’AppHost | Valider le comportement de plusieurs composants assemblés |

Testcontainers reste pertinent pour maîtriser une ou quelques dépendances dans un test ciblé. Aspire devient particulièrement intéressant lorsque le scénario doit réutiliser le modèle applicatif décrit dans l’AppHost.

Le coût augmente avec le nombre de ressources démarrées. Il faut réserver cet environnement aux scénarios qui le justifient, plutôt que de lancer toute la solution pour chaque règle métier.

### Adapter l'environnement de test

Les services s’exécutent dans des processus distincts. Un substitut créé dans le processus xUnit ne peut donc pas être injecté directement dans le conteneur de dépendances d’une API démarrée séparément.

Il reste possible de préparer l’application pour utiliser une implémentation de test, de remplacer une dépendance par un serveur simulé ou de modifier la configuration fournie à une ressource. Les [scénarios avancés de test](https://aspire.dev/testing/advanced-scenarios/) décrivent notamment la sélection des ressources et la surcharge des variables d’environnement.

Il faut aussi isoler les données. Des ports distincts n’empêchent pas deux tests d’écrire dans la même base de données distante, le même volume ou la même file de messages.

### Exécuter ces tests dans le pipeline

L’agent de construction doit disposer des runtimes requis et pouvoir démarrer les conteneurs utilisés par le scénario. Le téléchargement des images et leur premier démarrage peuvent prendre plus de temps que sur un poste déjà préparé.

Je prévoirais des délais adaptés, des données isolées et la conservation des journaux utiles en cas d’échec. Si le test utilise une ressource distante, l’authentification vers Azure doit aussi être configurée explicitement.

Une exécution locale réussie constitue une première indication. Le pipeline doit encore démontrer que l’environnement se reconstruit de manière fiable, sans dépendre d’un état laissé sur le poste d’un développeur.

## Réutiliser le modèle pour le déploiement

Aspire peut aussi utiliser le modèle applicatif pour préparer un déploiement. La [documentation du déploiement](https://aspire.dev/deployment/deploy-with-aspire/) distingue deux commandes principales.

| Commande | Rôle |
| --- | --- |
| `aspire publish` | Produire des artefacts propres à une cible pour un traitement ultérieur |
| `aspire deploy` | Générer la sortie attendue par la cible, résoudre les paramètres et appliquer le déploiement |

L’AppHost doit inclure une cible et des ressources capables de contribuer aux étapes correspondantes. `aspire deploy` ne consomme pas simplement le résultat d’une commande `aspire publish` exécutée auparavant.

Le [modèle de pipeline Aspire](https://aspire.dev/deployment/pipelines/) organise des étapes et leurs dépendances. La construction des images, la préparation des ressources et le déploiement peuvent ainsi participer au même processus, certaines opérations pouvant s’exécuter en parallèle.

Cette continuité réduit la duplication de la topologie, mais chaque environnement demande encore des décisions sur l’exposition réseau, les identités, le stockage durable, la capacité et les paramètres opérationnels. Une configuration locale ne suffit pas à déterminer ces choix.

Dans une organisation qui possède déjà des modules d’infrastructure et des pipelines approuvés, je vérifierais comment les artefacts et les étapes Aspire s’intègrent aux responsabilités existantes. Il faut notamment éviter que deux outils modifient la même ressource sans convention claire.

Cette direction me paraît prometteuse. Je commencerais toutefois par le développement local et les tests, puis j’évaluerais le déploiement sur une application représentative avec les équipes qui devront l’exploiter. Le support de la cible, les changements de version et les contraintes du réseau doivent faire partie de l’évaluation.

## Travailler avec plusieurs langages et plusieurs dépôts

L’AppHost peut réunir des technologies différentes. Une équipe peut donc conserver une API .NET, une interface JavaScript et un traitement Python dans le même modèle de développement.

Le langage de l’AppHost est un choix distinct. Aspire propose des AppHosts en C# et en TypeScript. Les versions récentes ont aussi enrichi les outils JavaScript, notamment avec le [support de Bun pour les applications Vite](https://devblogs.microsoft.com/aspire/aspire-bun-support-and-container-enhancements/).

Cette ouverture s’étend aux intégrations avec d’autres écosystèmes, dont AWS. Il faut examiner séparément le support d’un service, son éventuelle émulation locale avec un outil comme LocalStack et les possibilités de déploiement. La présence d’une intégration ne garantit pas une expérience identique sur toutes les plateformes.

### Le cas de plusieurs dépôts

La question devient aussi organisationnelle lorsque les composants d’une application sont répartis dans plusieurs dépôts. Quels dépôts doivent être présents? Dans quels dossiers? Quelles révisions fonctionnent ensemble? Qui maintient l’AppHost commun?

Des extensions comme [Aspire.PolyRepo](https://github.com/Dutchskull/Aspire.PolyRepo), présentée dans la formation, visent ce scénario. Le [suivi officiel de la feuille de route](https://github.com/microsoft/aspire/discussions/18023) permet de suivre les travaux sur le support multi-dépôts et les applications de plus grande taille.

Je distinguerais la capacité technique à orchestrer plusieurs composants de l’expérience complète de gestion de leurs dépôts. Cette dernière demande des conventions de clonage, de versionnement et de compatibilité. La solution retenue doit être simple à reproduire sur un nouveau poste.

## Aspire et les agents IA

Un agent qui intervient dans un dépôt doit comprendre comment lancer l’application et vérifier ses changements. L’AppHost lui donne une description explicite des ressources et de leurs relations.

Le CLI et la télémétrie lui permettent ensuite d’observer l’exécution. Il peut démarrer l’environnement, reproduire une erreur, consulter les informations disponibles et vérifier sa correction.

### Démarrer en arrière-plan

Le [mode détaché](https://devblogs.microsoft.com/aspire/aspire-detached-mode-and-process-management/) libère le terminal après le lancement :

```bash
aspire start
aspire ps
aspire describe --format Json
aspire stop
```

Ces commandes démarrent l’application, retrouvent les AppHosts actifs, en inspectent l’état et les arrêtent. Les sorties structurées facilitent leur utilisation dans un script ou par un agent.

### Exécuter plusieurs instances

Le [mode isolé](https://devblogs.microsoft.com/aspire/aspire-isolated-mode-parallel-development/) permet d’exécuter plusieurs instances, par exemple depuis différents worktrees Git :

```bash
aspire run --isolated
```

Aspire attribue des ports distincts et isole la configuration concernée par ce mode. Cela réduit les conflits entre le travail du développeur et les essais d’un agent.

Il faut tout de même vérifier les ressources partagées explicitement. Une base de données distante ou un volume configuré avec un nom fixe conserve son propre cycle de vie. Isoler les ports ne suffit pas à isoler ces données.

### Donner accès aux informations de diagnostic

La [documentation sur le diagnostic avec les agents](https://aspire.dev/dashboard/ai-coding-agents/) décrit l’accès aux données du tableau de bord par le CLI et, en option, par le serveur MCP d’Aspire. Un agent peut consulter l’état des ressources, les journaux structurés et les traces distribuées.

La qualité du contexte fourni fait la différence. Une exception associée à une trace et à un service précis oriente mieux l’investigation qu’un message d’erreur isolé.

On peut aussi relier une erreur observée dans le navigateur à l’appel API qui l’a provoquée. Avec Copilot ou un autre agent compatible, ces informations peuvent alimenter une investigation. L’interface et les fonctions disponibles dépendent de la version et de l’intégration choisies.

Une analyse assistée reste une hypothèse à vérifier. Je demanderais à l’agent d’indiquer ce qu’il a observé, de proposer une correction limitée et de reproduire le scénario après le changement. Il faut aussi tenir compte des données que les journaux rendent accessibles à l’outil utilisé.

### Consulter la documentation depuis le terminal

Les [commandes de documentation Aspire](https://devblogs.microsoft.com/aspire/aspire-docs-in-your-terminal/) permettent de rechercher les API et leurs usages depuis le terminal :

```bash
aspire docs list
aspire docs search redis
aspire docs get redis-integration
aspire docs get redis-integration --format Json
```

Un agent peut ainsi consulter une source à jour au lieu de se fier uniquement aux connaissances de son modèle. Il doit tout de même tenir compte de la version installée dans le dépôt. Une API documentée dans une version récente peut être absente d’un projet qui utilise une version antérieure.

J’ai également abordé la génération de notes de version à partir des demandes de fusion. C’est un autre usage possible des agents dans le cycle de développement, avec sa propre validation éditoriale.

## L'authentification reste une responsabilité explicite

Une application de démonstration peut démarrer correctement tout en exposant des endpoints sans authentification. La découverte de services et la configuration des connexions ne définissent pas les droits d’accès aux opérations métier.

L’[exemple officiel avec Microsoft Entra ID](https://devblogs.microsoft.com/aspire/securing-dotnet-aspire-apps-with-microsoft-entra-id/) présente une intégration avec OpenID Connect côté interface et la validation des jetons côté API. L’acquisition des jetons pour les appels à l’API repose notamment sur Microsoft.Identity.Web et la configuration des applications.

Les agents et les skills peuvent accélérer cette mise en place. Il faut quand même valider les audiences, les permissions et les règles d’autorisation. Une configuration générée demande la même révision que du code écrit manuellement.

Dans un contexte d’entreprise, je traiterais ces éléments dès que l’exemple devient une base de travail pour l’équipe. Ils ont aussi une incidence sur les tests et sur les différences entre les environnements.

## Adopter Aspire avec un périmètre utile

Je commencerais par un essai limité, assez proche du quotidien de l’équipe pour en mesurer les résultats.

1. Choisir un irritant fréquent, comme le démarrage d’une API avec ses dépendances ou le diagnostic d’un traitement asynchrone.
2. Décrire ce périmètre dans l’AppHost, avec la configuration et les données nécessaires au scénario.
3. Demander à une autre personne de l’exécuter. Les manipulations qui restent révèleront ce qui manque dans le dépôt.
4. Ajouter un test d’interaction pertinent pour vérifier que le scénario peut aussi fonctionner dans le pipeline.
5. Évaluer le résultat avant d’élargir : comparer le temps de préparation et de diagnostic, la fiabilité obtenue et l’effort de maintenance.

Les compromis sont concrets. Le poste doit avoir assez de ressources pour exécuter l’environnement. Les intégrations peuvent demander des adaptations. Les packages et les modèles générés évoluent. L’AppHost devient lui-même du code à maintenir.

Il faut aussi choisir ce qui tourne localement. Une grande application distribuée peut nécessiter plusieurs périmètres de travail, avec certaines dépendances partagées ou simulées. Démarrer toute l’application sur chaque poste peut rendre l’expérience plus lourde que nécessaire.

Les [notes de version officielles](https://aspire.dev/whats-new/aspire-13-5/) et la feuille de route aident à distinguer les capacités livrées des travaux en cours. Les [sessions d’Aspire Conf](https://aspire.dev/aspireconf/) permettent aussi d’approfondir plusieurs sujets de la formation.

## Conclusion

Aspire aide à faire fonctionner une application avec les dépendances nécessaires à son développement et à ses tests.

L’AppHost décrit les ressources et leurs relations, tandis que le tableau de bord permet d’observer leur comportement. Les tests peuvent réutiliser ce modèle, qui donne aussi aux agents un contexte plus précis pour intervenir dans la solution.

Cette cohérence demande du travail. Il faut choisir les contrôles de santé, préparer les données, instrumenter les traitements et vérifier les différences avec la production. Aspire fournit des mécanismes pour organiser ce travail et le partager dans le dépôt.

Je recommande de commencer par un scénario qui coûte déjà du temps à l’équipe. Si une autre personne peut le démarrer, le comprendre et en diagnostiquer les erreurs plus facilement, le bénéfice est tangible. L’équipe pourra ensuite élargir l’adoption selon les besoins de la solution.
