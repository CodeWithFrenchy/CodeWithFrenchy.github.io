---
title: "Query et Command en .NET : séparer clairement la lecture et l’écriture"
date: 2027-12-31 19:00:00 -0400
tags: [dotnet, csharp, architecture, cqrs]
---

## Préambule

Lorsqu’on ouvre une fiche de dossier, on s’attend à obtenir son contenu. On ne s’attend pas à modifier son responsable, à faire avancer son statut ou à réaffecter ses tâches.

Cette attente paraît évidente du point de vue de l’utilisateur. Elle devrait aussi l’être lorsqu’on lit le code.

C’est l’intérêt de distinguer les **Query**, qui servent à obtenir de l’information, des **Command**, qui demandent au système d’effectuer une action.

La règle de départ est simple :

- **Query : je veux obtenir quelque chose.**
- **Command : je veux créer, modifier, supprimer ou déclencher quelque chose.**

Cette séparation permet de comprendre les conséquences attendues d’une opération avant d’en parcourir toute l’implémentation. Elle constitue aussi le point de départ d’une approche CQRS.

## Une Query exprime un besoin de lecture

Une Query sert à récupérer de l’information sans modifier l’état métier du système.

Quelques exemples :

| Query | Intention |
| --- | --- |
| `ObtenirDossierQuery` | Consulter les informations d’un dossier |
| `RechercherIntervenantsQuery` | Trouver les intervenants qui correspondent à des critères |
| `ObtenirTachesQuery` | Afficher les tâches accessibles à l’utilisateur |
| `ObtenirDemandeTransfertDossierQuery` | Consulter une demande de transfert |

Une lecture peut être complexe. Elle peut filtrer, paginer, joindre plusieurs tables, calculer un total ou construire une représentation adaptée à un écran.

La complexité du calcul ne détermine pas sa catégorie. Ce qui compte, c’est l’effet attendu de l’opération.

Par exemple, calculer le nombre de tâches en retard reste une Query tant que ce calcul ne modifie pas les tâches. Enregistrer ce nombre comme un indicateur officiel dans un historique constitue une autre opération, avec une responsabilité d’écriture.

Une Query n’est pas non plus une garantie de résultat identique à chaque appel. Deux consultations peuvent retourner des informations différentes parce qu’un autre utilisateur a modifié les données entre-temps. La Query observe cet état, elle ne provoque pas elle-même cette modification.

## Une Command exprime une intention d’action

Une Command demande au système d’exécuter une action susceptible de modifier son état métier ou de produire un effet externe.

Par exemple :

| Command | Intention |
| --- | --- |
| `CreerDossierCommand` | Créer un nouveau dossier |
| `TransfererTachesCommand` | Réaffecter des tâches |
| `AnnulerDemandeCommand` | Annuler une demande selon les règles applicables |
| `TransfererDemandeTransfertDossierCommand` | Exécuter le transfert prévu par une demande |

Le nom devrait exprimer l’objectif métier. `AnnulerDemandeCommand` explique davantage l’intention qu’une opération générique appelée `ModifierStatutCommand`.

Une commande peut avoir besoin de lire des données pour agir. Elle peut charger un dossier, vérifier son statut, consulter les droits de l’utilisateur et déterminer les tâches concernées.

Ces lectures font partie de l’exécution de l’action. Elles ne transforment pas la commande en Query.

Enfin, une commande représente une **demande d’action**, pas la preuve que cette action a réussi. Elle peut être refusée parce qu’une règle n’est pas respectée. Elle peut aussi ne produire aucun changement supplémentaire si l’action a déjà été appliquée et que le traitement a été conçu pour être idempotent.

## CQS et CQRS : deux niveaux de séparation

Les termes sont proches, mais ils ne désignent pas exactement la même chose.

Le principe [Command Query Separation, ou CQS](https://martinfowler.com/bliki/CommandQuerySeparation.html), distingue les méthodes qui retournent une information sans modifier l’état observable de celles qui modifient cet état. Dans sa formulation stricte, une commande ne retourne pas de valeur.

CQRS, pour *Command Query Responsibility Segregation*, applique une séparation des responsabilités aux modèles utilisés pour lire et pour écrire.

Dans une application de gestion de dossiers, on peut ainsi avoir :

- Un modèle de lecture qui expose les données nécessaires aux écrans.
- Un modèle d’écriture qui porte les opérations métier et les règles à préserver.

Il ne suffit donc pas d’ajouter les suffixes `Query` et `Command` à deux classes pour obtenir cette séparation. Le comportement et les modèles utilisés doivent suivre l’intention annoncée.

Cette organisation peut rester locale. Le [modèle CQRS décrit par l’Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs) prévoit notamment des modèles distincts qui partagent une seule base de données.

Une même application ASP.NET Core peut donc utiliser EF Core, une base de données SQL Server et des traitements séparés pour les lectures et les écritures. CQRS n’impose ni microservices, ni système de messagerie, ni Event Sourcing.

## Un exemple concret : consulter puis transférer un dossier

Prenons un scénario où une demande de transfert existe déjà.

L’utilisateur ouvre cette demande pour consulter le dossier concerné, son responsable actuel et son statut. Il choisit ensuite un nouveau responsable et confirme le transfert.

Ces deux intentions méritent deux opérations distinctes.

### Consulter la demande

La Query ne contient que l’information nécessaire pour identifier la demande :

```csharp
public sealed record ObtenirDemandeTransfertDossierQuery(Guid IdDemande);
```

La réponse correspond au besoin de lecture :

```csharp
public sealed record DemandeTransfertDossierDto(
    Guid IdDemande,
    Guid IdDossier,
    Guid IdResponsableActuel,
    string CodeStatut);
```

Les records sont scellés parce que ces contrats ne sont pas conçus pour être hérités.

Voici un extrait de handler avec EF Core et un constructeur principal de C# 12. On suppose que `DossiersDbContext` et les entités existent dans l’application, et que le pipeline applicatif vérifie le droit de consulter la demande avant cet appel.

```csharp
using System;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.EntityFrameworkCore;

public sealed class ObtenirDemandeTransfertDossierHandler(DossiersDbContext contexte)
{
    public Task<DemandeTransfertDossierDto?> Handle(
        ObtenirDemandeTransfertDossierQuery requete,
        CancellationToken cancellationToken) =>
        contexte.DemandesTransfertDossier
            .Where(demande =>
                demande.Id == requete.IdDemande)
            .Select(demande =>
                new DemandeTransfertDossierDto(
                    demande.Id,
                    demande.IdDossier,
                    demande.IdResponsableActuel,
                    demande.CodeStatut))
            .SingleOrDefaultAsync(cancellationToken);
}
```

Le traitement projette les données vers un DTO. Il ne réaffecte aucune tâche et ne change aucun statut. Ici, `null` signifie qu’aucune demande ne correspond à l’identifiant.

Cette projection ne matérialise aucune entité dans le résultat. EF Core n’a donc pas d’entité à suivre pour cette requête, conformément à sa [documentation sur le suivi des entités](https://learn.microsoft.com/en-us/ef/core/querying/tracking).

Si une autre lecture matérialise des entités, `AsNoTracking()` peut être approprié. Cette option configure le suivi, elle ne transforme pas le contexte en connexion protégée contre les écritures.

La [documentation .NET sur les lectures en CQRS](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/cqrs-microservice-reads) illustre cette liberté de construire une réponse adaptée au consommateur. Le choix entre EF Core, Dapper ou un autre accès aux données reste indépendant de la séparation des responsabilités.

### Exécuter le transfert

La commande porte les paramètres de l’action :

```csharp
public sealed record TransfererDemandeTransfertDossierCommand(
    Guid IdDemande,
    Guid IdResponsableDestination);
```

Dans notre scénario, son traitement doit :

1. Vérifier que l’utilisateur peut exécuter le transfert.
2. Charger l’état nécessaire à la décision.
3. Vérifier que la demande est encore transférable et que le responsable de destination est admissible.
4. Appliquer les changements de responsable, de statut et d’affectation des tâches.
5. Enregistrer les modifications dans la transaction prévue pour cette opération.

Les règles exactes dépendent du domaine. L’intérêt de la commande est de leur donner un point d’entrée explicite.

Une consultation de la demande ne devrait pas effectuer une partie de ce transfert « pour préparer l’écran ». Si une préparation réserve le dossier ou change son statut, elle doit être modélisée comme une action à part entière.

## Sans effet de bord : préciser ce que l’on veut préserver

Pour le code applicatif, je formule la règle ainsi : **une Query ne doit pas provoquer de changement métier**.

Une lecture peut produire des journaux, des métriques ou alimenter un cache technique. Elle n’est donc pas nécessairement une fonction pure au sens fonctionnel du terme.

Ces mécanismes doivent rester distincts des conséquences métier de l’opération.

Par exemple, enregistrer une trace d’accès pour l’observabilité ne revient pas à marquer officiellement une demande comme prise en charge. Dans le second cas, on fait avancer un processus métier.

Cette distinction existe aussi dans les [méthodes sûres de HTTP](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.2.1), dont la sémantique de lecture autorise certains effets techniques, comme la journalisation. Un `GET` ne devrait toutefois pas déclencher un transfert de dossier demandé implicitement par la consultation.

Voici quelques situations qui méritent d’être explicites :

| Opération | Catégorie |
| --- | --- |
| Consulter une demande | Query |
| Marquer une demande comme lue lorsque cela change son suivi métier | Command |
| Calculer un aperçu de transfert sans le conserver | Query |
| Réserver des tâches pour un transfert à venir | Command |
| Vérifier la disponibilité d’un responsable | Query |
| Affecter un dossier à ce responsable | Command |
| Envoyer une notification par un système de messagerie | Command |

Une action peut donc être une commande même si elle n’appelle jamais `SaveChangesAsync()`. Un effet externe compte lui aussi.

Inversement, le seul nom d’une méthode ne protège pas contre les écritures. Un handler nommé `ObtenirDossier` peut appeler un service qui modifie le dossier. La séparation doit tenir sur tout le chemin d’exécution.

## Une commande peut-elle retourner un résultat ?

Dans une application CQRS, je trouve utile de permettre au traitement d’une commande de retourner un compte rendu.

On peut vouloir connaître l’identifiant créé, le résultat d’une validation ou les informations nécessaires pour confirmer l’action à l’utilisateur.

Cette convention s’écarte du CQS strict sur la valeur de retour, tout en conservant la séparation des responsabilités de lecture et d’écriture. La [documentation .NET sur les handlers de commandes](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/microservice-application-layer-implementation-web-api) présente d’ailleurs des retours de résultat, y compris pour les échecs de validation.

Pour un transfert terminé avec succès, le compte rendu pourrait être :

```csharp
public sealed record TransfertDossierReponse(
    Guid IdDemande,
    Guid IdDossier,
    Guid IdResponsableDestination,
    int NombreTachesTransferees);
```

Ce contrat décrit ici le succès. Les refus métier et les conflits doivent être représentés par le mécanisme de résultat ou d’erreur retenu dans l’application.

Le fait de retourner cette réponse ne transforme pas le transfert en Query. Son objectif reste l’exécution d’une action.

Il faut aussi distinguer une action terminée d’une action simplement acceptée. Si le transfert est confié à un traitement différé, la réponse initiale devrait indiquer son acceptation et permettre d’en suivre l’avancement. Elle ne devrait pas annoncer un transfert réussi avant son exécution.

## Une lecture préalable ne garantit pas que la commande sera acceptée

Supposons que l’écran affiche une demande au statut « En attente ».

Avant que l’utilisateur confirme le transfert, une autre personne peut annuler cette demande ou l’affecter ailleurs. La commande doit donc vérifier les conditions applicables au moment de son exécution.

La validation dans l’interface améliore l’expérience utilisateur. Elle ne remplace pas les contrôles côté serveur.

Même après une nouvelle lecture côté serveur, un changement concurrent reste possible avant la sauvegarde. La séparation Query/Command ne résout pas ce problème à elle seule.

Avec EF Core, un [jeton de concurrence](https://learn.microsoft.com/en-us/ef/core/saving/concurrency), par exemple un `rowversion` SQL Server correctement configuré, peut permettre de détecter certaines mises à jour concurrentes. Il faut ensuite traiter le conflit selon les règles du cas d’usage.

La transaction et le contrôle de concurrence doivent couvrir les données dont dépend la décision. Un jeton posé sur une seule ligne ne protège pas automatiquement toutes les règles portant sur plusieurs tables.

La même vigilance s’applique aux reprises. Recevoir deux fois une commande de transfert ne devrait pas réaffecter deux fois les tâches ou envoyer deux fois la même notification. Le comportement attendu en cas de répétition doit être conçu, puis vérifié.

## CQRS reste un choix de conception

La séparation est particulièrement utile lorsque les besoins de consultation diffèrent des règles de modification.

Un écran peut avoir besoin d’une liste aplatie avec des totaux. Le traitement d’un transfert a plutôt besoin de préserver des règles sur les dossiers, les demandes et les tâches. Ces deux représentations peuvent évoluer différemment.

En revanche, une fonctionnalité CRUD simple ne justifie pas nécessairement des dizaines de contrats et de handlers. [Martin Fowler rappelle que CQRS peut ajouter une complexité importante](https://martinfowler.com/bliki/CQRS.html) lorsqu’il est appliqué sans besoin suffisant.

Je commencerais donc par rendre les intentions claires, puis par séparer les modèles lorsque cela facilite réellement le travail.

La distribution vient ensuite, si elle répond à un besoin. Lorsqu’un modèle de lecture est alimenté de manière asynchrone dans une autre base de données, il peut afficher temporairement un état antérieur à la dernière écriture. Cette incohérence éventuelle doit être prise en compte dans l’expérience utilisateur.

Un médiateur, des dossiers nommés `Queries` et `Commands`, ou un système de messagerie peuvent accompagner la conception. Ils ne garantissent pas, à eux seuls, que les responsabilités sont bien séparées.

## Une règle de révision de code facile à partager

Pour appliquer cette convention en équipe, je regarderais trois éléments.

**Le nom annonce-t-il l’intention ?** Une lecture commence naturellement par `Obtenir`, `Rechercher`, `Lister` ou `Calculer`. Une action utilise plutôt `Creer`, `Transferer`, `Annuler`, `Affecter` ou `Confirmer`.

**Le comportement respecte-t-il cette intention ?** Une Query ne doit pas modifier un statut, réserver une ressource métier ou déclencher une notification fonctionnelle.

**Les tests vérifient-ils les conséquences attendues ?** Pour une lecture, on vérifie les données retournées et l’absence de mutation métier. Pour une commande, on vérifie les transitions autorisées, les refus et les effets attendus. Les scénarios de concurrence ou de répétition s’ajoutent lorsqu’ils font partie du risque réel de l’opération.

Le suffixe sert de convention de lecture. La garantie vient de l’implémentation et de sa vérification.

## Conclusion

Une Query permet de consulter le système sans en modifier l’état métier. Une Command exprime une action à exécuter et porte les conséquences de cette action.

Dans notre exemple, `ObtenirDemandeTransfertDossierQuery` retourne la demande. `TransfererDemandeTransfertDossierCommand` exécute le transfert selon les règles applicables.

Cette distinction rend les intentions visibles et les traitements plus prévisibles. Elle fournit une base utile pour CQRS, avec un niveau de séparation que l’on peut adapter aux besoins réels de l’application.
