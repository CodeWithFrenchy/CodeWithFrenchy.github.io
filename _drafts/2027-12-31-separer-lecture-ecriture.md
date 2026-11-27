---
title: "Query et Command en .NET : séparer clairement la lecture et l’écriture"
date: 2027-12-31 19:00:00 -0400
categories: [architecture, bonnes-pratiques]
tags: [dotnet, csharp, cqrs, command, query, separation-responsabilites, conception-logicielle, maintenabilite]
---

> ⚠️ À RELIRE + valider le format du code

## Préambule

Lorsqu’on ouvre une fiche de dossier, on s’attend à en consulter le contenu. Modifier son responsable, faire avancer son statut ou réaffecter des tâches sont d’autres opérations. Cette distinction devrait être aussi visible dans le code que dans l’interface.

Les **Query** servent à obtenir de l’information. Les **Command** demandent au système d’effectuer une action.

- Query : obtenir de l’information.
- Command : créer, modifier, supprimer ou déclencher quelque chose.

Cette séparation rend les conséquences attendues d’une opération plus faciles à comprendre. Elle constitue aussi un point de départ pour CQRS.

## Une Query exprime un besoin de lecture

Une Query récupère de l’information sans modifier l’état métier du système.

| Query | Intention |
| --- | --- |
| `ObtenirDossierQuery` | Consulter les informations d’un dossier |
| `RechercherIntervenantsQuery` | Trouver les intervenants qui correspondent à des critères |
| `ObtenirTachesQuery` | Afficher les tâches accessibles à l’utilisateur |
| `ObtenirDemandeTransfertDossierQuery` | Consulter une demande de transfert |

Une lecture peut filtrer, paginer, joindre plusieurs tables, calculer un total ou construire une représentation adaptée à un écran. C’est son effet attendu, et non la complexité du calcul, qui détermine sa catégorie.

Par exemple, calculer le nombre de tâches en retard reste une Query tant que le calcul ne modifie pas les tâches. Enregistrer ce nombre comme indicateur officiel dans un historique constitue une opération d’écriture distincte.

Deux consultations peuvent retourner des informations différentes si un autre utilisateur a modifié les données entre-temps. La Query observe cet état, mais ne provoque pas elle-même la modification.

## Une Command exprime une intention d’action

Une Command demande au système d’exécuter une action susceptible de modifier son état métier ou de produire un effet externe.

| Command | Intention |
| --- | --- |
| `CreerDossierCommand` | Créer un nouveau dossier |
| `TransfererTachesCommand` | Réaffecter des tâches |
| `AnnulerDemandeCommand` | Annuler une demande selon les règles applicables |
| `TransfererDemandeTransfertDossierCommand` | Exécuter le transfert prévu par une demande |

Le nom devrait exprimer l’objectif métier. `AnnulerDemandeCommand` décrit mieux l’intention qu’une opération générique nommée `ModifierStatutCommand`.

Une commande peut lire des données pour agir. Elle peut charger un dossier, vérifier son statut, consulter les droits de l’utilisateur et déterminer les tâches concernées. Ces lectures font partie de son exécution et ne la transforment pas en Query.

Une commande exprime une **demande d’action**, pas la preuve que cette action a réussi. Elle peut être refusée si une règle n’est pas respectée. Si le traitement est idempotent, une répétition peut aussi ne produire aucun changement supplémentaire.

## CQS et CQRS : deux niveaux de séparation

Les termes sont proches, mais ne désignent pas exactement la même chose.

Le principe [Command Query Separation, ou CQS](https://martinfowler.com/bliki/CommandQuerySeparation.html), distingue les méthodes qui retournent une information sans modifier l’état observable de celles qui modifient cet état. Dans sa formulation stricte, une commande ne retourne pas de valeur.

CQRS, pour *Command Query Responsibility Segregation*, sépare les responsabilités des modèles utilisés pour lire et pour écrire. Dans une application de gestion de dossiers, on peut ainsi avoir :

- Un modèle de lecture qui expose les données nécessaires aux écrans.
- Un modèle d’écriture qui porte les opérations métier et les règles à préserver.

Ajouter les suffixes `Query` et `Command` à deux classes ne suffit pas à obtenir cette séparation. Les comportements et les modèles doivent correspondre à l’intention annoncée.

Cette organisation peut rester locale. Le [modèle CQRS décrit par l’Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs) prévoit notamment des modèles distincts qui partagent une seule base de données.

Une même application ASP.NET Core peut donc utiliser EF Core, une base de données SQL Server et des traitements séparés pour les lectures et les écritures. CQRS n’impose ni microservices, ni système de messagerie, ni Event Sourcing.

## Un exemple concret : consulter puis transférer un dossier

Dans ce scénario, une demande de transfert existe déjà. L’utilisateur consulte le dossier concerné, son responsable actuel et son statut, puis choisit un nouveau responsable et confirme le transfert. Ces intentions correspondent à deux opérations distinctes.

### Consulter la demande

La Query contient uniquement l’information nécessaire pour identifier la demande :

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

Ces records sont scellés parce que les contrats ne sont pas conçus pour être hérités.

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

Le traitement projette les données vers un DTO sans réaffecter de tâche ni changer de statut. Ici, `null` signifie qu’aucune demande ne correspond à l’identifiant.

Comme la projection ne matérialise aucune entité, EF Core n’a pas d’entité à suivre pour cette requête, conformément à sa [documentation sur le suivi des entités](https://learn.microsoft.com/en-us/ef/core/querying/tracking).

Si une autre lecture matérialise des entités, `AsNoTracking()` peut être approprié. Cette option configure le suivi, mais ne transforme pas le contexte en connexion protégée contre les écritures.

La [documentation .NET sur les lectures en CQRS](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/cqrs-microservice-reads) montre comment construire une réponse adaptée au consommateur. Le choix entre EF Core, Dapper ou un autre accès aux données reste indépendant de la séparation des responsabilités.

### Exécuter le transfert

La commande porte les paramètres de l’action :

```csharp
public sealed record TransfererDemandeTransfertDossierCommand(
    Guid IdDemande,
    Guid IdResponsableDestination);
```

Dans ce scénario, son traitement doit :

1. Vérifier que l’utilisateur peut exécuter le transfert.
2. Charger l’état nécessaire à la décision.
3. Vérifier que la demande est encore transférable et que le responsable de destination est admissible.
4. Appliquer les changements de responsable, de statut et d’affectation des tâches.
5. Enregistrer les modifications dans la transaction prévue pour cette opération.

Les règles exactes dépendent du domaine. La commande leur donne un point d’entrée explicite.

La consultation de la demande ne devrait pas effectuer une partie du transfert pour « préparer l’écran ». Si cette préparation réserve le dossier ou change son statut, elle doit être modélisée comme une action distincte.

## Sans effet de bord : préciser ce que l’on veut préserver

Pour le code applicatif, je formule la règle ainsi : une Query ne doit pas provoquer de changement métier.

Une lecture peut produire des journaux, des métriques ou alimenter un cache technique. Elle n’est donc pas nécessairement une fonction pure au sens fonctionnel du terme. Ces mécanismes doivent rester distincts des conséquences métier.

Par exemple, enregistrer une trace d’accès pour l’observabilité ne revient pas à marquer officiellement une demande comme prise en charge. Cette dernière opération fait avancer un processus métier.

La distinction apparaît aussi dans les [méthodes sûres de HTTP](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.2.1), dont la sémantique de lecture autorise certains effets techniques, comme la journalisation. Un `GET` ne devrait pas déclencher implicitement un transfert de dossier.

| Opération | Catégorie |
| --- | --- |
| Consulter une demande | Query |
| Marquer une demande comme lue lorsque cela change son suivi métier | Command |
| Calculer un aperçu de transfert sans le conserver | Query |
| Réserver des tâches pour un transfert à venir | Command |
| Vérifier la disponibilité d’un responsable | Query |
| Affecter un dossier à ce responsable | Command |
| Envoyer une notification par un système de messagerie | Command |

Une action peut donc être une commande même si elle n’appelle jamais `SaveChangesAsync()`. Un effet externe compte aussi.

Le nom d’une méthode ne suffit pas à garantir l’absence d’écriture. Un handler nommé `ObtenirDossier` peut appeler un service qui modifie le dossier. La séparation doit être respectée sur tout le chemin d’exécution.

## Une commande peut-elle retourner un résultat?

Dans une application CQRS, je trouve utile que le traitement d’une commande retourne un compte rendu. Il peut contenir l’identifiant créé, le résultat d’une validation ou les informations nécessaires pour confirmer l’action à l’utilisateur.

Cette convention s’écarte du CQS strict sur la valeur de retour, tout en conservant la séparation des responsabilités entre lecture et écriture. La [documentation .NET sur les handlers de commandes](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/microservice-application-layer-implementation-web-api) présente aussi des retours de résultat, notamment pour les échecs de validation.

Pour un transfert terminé avec succès, le compte rendu pourrait être :

```csharp
public sealed record TransfertDossierReponse(
    Guid IdDemande,
    Guid IdDossier,
    Guid IdResponsableDestination,
    int NombreTachesTransferees);
```

Ce contrat décrit le succès. Les refus métier et les conflits doivent être représentés par le mécanisme de résultat ou d’erreur retenu dans l’application.

Le transfert reste une commande, car son objectif est d’exécuter une action. Si le traitement est différé, la réponse initiale devrait indiquer que la commande a été acceptée et permettre d’en suivre l’avancement. Elle ne devrait pas annoncer un transfert réussi avant son exécution.

## Une lecture préalable ne garantit pas que la commande sera acceptée

Supposons que l’écran affiche une demande au statut « En attente ». Avant que l’utilisateur confirme le transfert, une autre personne peut l’annuler ou l’affecter ailleurs. La commande doit donc vérifier les conditions applicables au moment de son exécution.

La validation dans l’interface améliore l’expérience utilisateur, mais ne remplace pas les contrôles côté serveur. Même après une nouvelle lecture côté serveur, une modification concurrente peut survenir avant la sauvegarde. La séparation Query/Command ne résout pas ce problème à elle seule.

Avec EF Core, un [jeton de concurrence](https://learn.microsoft.com/en-us/ef/core/saving/concurrency), comme un `rowversion` SQL Server correctement configuré, peut détecter certaines mises à jour concurrentes. Il faut ensuite traiter le conflit selon les règles du cas d’usage.

La transaction et le contrôle de concurrence doivent couvrir les données dont dépend la décision. Un jeton posé sur une seule ligne ne protège pas automatiquement les règles qui portent sur plusieurs tables.

La même vigilance s’applique aux reprises. Recevoir deux fois une commande de transfert ne devrait pas réaffecter les tâches ou envoyer plusieurs fois la même notification. Le comportement attendu en cas de répétition doit être conçu et vérifié.

## CQRS reste un choix de conception

La séparation est utile lorsque les besoins de consultation diffèrent des règles de modification.

Un écran peut nécessiter une liste aplatie avec des totaux. Le traitement d’un transfert doit préserver des règles sur les dossiers, les demandes et les tâches. Ces représentations peuvent évoluer différemment.

Une fonctionnalité CRUD simple ne justifie pas nécessairement des dizaines de contrats et de handlers. [Martin Fowler rappelle que CQRS peut ajouter une complexité importante](https://martinfowler.com/bliki/CQRS.html) lorsqu’il est appliqué sans besoin suffisant.

Je commencerais par rendre les intentions claires, puis je séparerais les modèles lorsque cela facilite réellement le travail.

La distribution vient ensuite si elle répond à un besoin. Lorsqu’un modèle de lecture est alimenté de manière asynchrone dans une autre base de données, il peut afficher temporairement un état antérieur à la dernière écriture. L’expérience utilisateur doit tenir compte de cette incohérence éventuelle.

Un médiateur, des dossiers nommés `Queries` et `Commands` ou un système de messagerie peuvent accompagner la conception. Ils ne garantissent pas à eux seuls une bonne séparation des responsabilités.

## Une règle de révision de code facile à partager

En révision de code, vérifiez trois choses :

- Le nom annonce-t-il l’intention? Une lecture commence naturellement par `Obtenir`, `Rechercher`, `Lister` ou `Calculer`. Une action utilise plutôt `Creer`, `Transferer`, `Annuler`, `Affecter` ou `Confirmer`.
- Le comportement respecte-t-il cette intention? Une Query ne devrait pas modifier un statut, réserver une ressource métier ou déclencher une notification fonctionnelle.
- Les tests couvrent-ils les conséquences attendues? Pour une lecture, vérifiez les données retournées et l’absence de mutation métier. Pour une commande, vérifiez les transitions autorisées, les refus et les effets attendus. Ajoutez des scénarios de concurrence ou de répétition lorsqu’ils correspondent aux risques réels de l’opération.

Le suffixe sert de convention de lecture. La garantie vient de l’implémentation et de sa vérification.

## Conclusion

Une Query permet de consulter le système sans modifier son état métier. Une Command exprime une action à exécuter et porte les conséquences de cette action.

Dans notre exemple, `ObtenirDemandeTransfertDossierQuery` retourne la demande. `TransfererDemandeTransfertDossierCommand` exécute le transfert selon les règles applicables.

Cette distinction rend les intentions visibles et les traitements plus prévisibles. Elle fournit une base utile pour CQRS, que l’on peut adapter aux besoins réels de l’application.
