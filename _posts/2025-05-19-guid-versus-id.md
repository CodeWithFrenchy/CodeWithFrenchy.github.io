---
title: GUID vs ID auto-incrémenté - dilemme et solutions en environnement .NET distribué
date: 2025-05-19 13:00:00 -0400
categories: []
tags: [dotnet]
---

## Préambule

Je me souviens encore d’une réunion mouvementée avec un DBA, alors que nous discutions de l’architecture d’un nouveau système **.NET** réparti sur plusieurs services. Lorsque j’ai osé proposer d’utiliser des GUID comme clés primaires au lieu des habituels identifiants auto-incrémentés, j’ai vu des yeux s’écartiller. *« Pourquoi changer une recette qui marche ? »* m’a-t-on lancé. En bon développeur, j’avais moi-même toujours apprécié la simplicité des IDs séquentiels (1, 2, 3, …). Mais dans un environnement **distribué et asynchrone**, cette vieille recette commençait à montrer ses limites.

Dans cet article, je vous propose un retour d’expérience sur ce choix technique. Nous verrons pourquoi les GUID peuvent s’avérer pratiques (et comment les utiliser intelligemment), sans occulter les inconvénients bien réels qui rendent certains DBA méfiants. Installez-vous confortablement, on va parler d’**identifiants**, de **.NET/Entity Framework**, d’**index**, de **sécurité**, de **tests** et de **compromis astucieux**.

## Le dilemme dans un monde distribué

Identifiants auto-incrémentés ou GUID ? Le débat est ancien. Dans un monde monolithique, les identifiants séquentiels (souvent de type `int`) brillent par leur simplicité, leur performance, et leur tri naturel. Mais dès que le système devient **distribué**, **asynchrone**, ou s’étend sur plusieurs bases de données, les problèmes commencent : collisions d’identifiants, coordination nécessaire pour garantir l’unicité, complexité lors des fusions de données ou synchronisations inter-sites.

Les GUID (_Globally Unique IDentifiers_), quant à eux, peuvent être générés localement par l’application, sans dépendance à la base de données, tout en assurant une unicité quasi-absolue (2^128 combinaisons possibles).

## Nouveautés en .NET 9 : les GUID version 7

Depuis la version 9, .NET introduit la méthode `Guid.CreateVersion7()`, qui génère des GUID **ordonnables chronologiquement**. Cela répond à une critique historique des GUID : leur tendance à provoquer de la fragmentation dans les index SQL.

Les GUID v7 combinent un horodatage (48 bits) et des bits aléatoires.

On peut les utiliser avec :

```csharp
Guid guid7 = Guid.CreateVersion7();
```

Ces nouveaux GUID sont plus efficaces pour les insertions en base de données, car ils réduisent significativement les fragmentations d'index et offrent des performances comparables aux IDs séquentiels tout en conservant les avantages des GUID. Bien que EF Core ne les supporte pas encore nativement, il est possible de créer un générateur personnalisé pour les utiliser efficacement.

## Avantages des GUID dans un système distribué

* **Unicité globale** : pas besoin de coordination entre les bases de données ou les services.
* **Génération côté client** : utile pour le mode hors-ligne, les tests ou les systèmes à haute disponibilité.
* **Scalabilité** : pas de bottleneck sur une table centrale.
* **Compatibilité avec la réplication** : les GUID sont idéaux pour la réplication multi-site.
* **Facilite les tests automatisés** : permet de créer des identifiants prédictibles et stables entre exécutions.
* **Uniformisation entre environnements** : les mêmes identifiants peuvent être utilisés en dev, test, staging ou production.

## Cas concrets

* **Fusion de deux bases de données** : sans GUID, risque de collision d’IDs.
* **Microservices** : chaque service peut gérer ses propres identifiants.
* **Tests automatisés** : avec des GUID prévisibles, les assertions et les fixtures deviennent plus stables.
* **Alimentation entre environnements** : les mêmes GUID peuvent être utilisés pour des jeux de données identiques, simplifiant les déploiements.

## Implémentation en EF Core

```csharp
public class Commande {
    public Guid Id { get; set; } = Guid.NewGuid();
    public string Description { get; set; } = "";
}
```

Pour utiliser un GUID séquentiel (côté SQL Server) :

```csharp
modelBuilder.Entity<Commande>()
    .Property(c => c.Id)
    .HasDefaultValueSql("NEWSEQUENTIALID()");
```

## Inconvénients des GUID

* Taille (16 octets vs 4 pour un `int`)
* Moins lisibles pour les humains
* Risque de fragmentation si non ordonnés
* Légère surcharge en performance sur les jointures

## Dialogue avec les DBAs : trouver le bon équilibre

Tous les DBAs ne sont pas réfractaires au changement. Certains apportent même des solutions très pertinentes (index non cluster, fill factor adapté, etc.). Ce qu’ils attendent, c’est que les choix soient justifiés techniquement, mesurés, et documentés. Un DBA averti sera rassuré de savoir que vous utilisez des GUID **v7** plutôt que des GUID aléatoires classiques. Le dialogue avec eux permet de mettre en place une stratégie gagnant-gagnant.

## Sécurité des routes HTTP exposées

Un autre avantage souvent oublié : la **sécurité par l’obscurcité**.

Une route comme :

```
GET /api/citoyens/5/enfants
```

est prévisible. Un utilisateur malveillant peut tester `/6`, `/7`, etc., pour tenter d’accéder à des données qui ne lui sont pas destinées. C’est une attaque de type **IDOR (Insecure Direct Object Reference)**.

Avec des GUID :

```
GET /api/citoyens/6f8d2ac1-3b90-4dc5-9543-f490a8f5d8c2/enfants
```

l’exploration devient presque impossible sans information préalable. Attention : cela **ne remplace pas** une vérification des droits, mais cela constitue une **couche supplémentaire de protection**, conforme aux bonnes pratiques OWASP. Cela décourage aussi les attaques automatisées.

## Conclusion

Utiliser des GUID, ce n’est pas une mode. C’est une réponse technique à des besoins très concrets dans les architectures modernes. Il faut être conscient des implications (taille, fragmentation, lisibilité), mais avec les GUID **v7**, **les bonnes pratiques EF Core**, une **stratégie de tests solide**, et une **communication honnête avec vos DBAs**, ils peuvent devenir un atout puissant. Et dans un monde où la distribution, l’asynchronisme, les tests déconnectés, la sécurité et l’automatisation sont la norme, **c’est un choix de plus en plus stratégique**.

Bon coding, et que vos identifiants restent uniques !
