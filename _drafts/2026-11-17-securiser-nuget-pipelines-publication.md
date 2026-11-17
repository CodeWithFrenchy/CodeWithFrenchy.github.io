---
title: "Sécuriser ses dépendances NuGet et ses pipelines de publication"
date: 2026-11-17 19:00:00 -0400
categories: [dotnet, securite]
tags: [dotnet, dotnet11, nuget, securite, devops, github-actions, trusted-publishing]
---

>  ⚠️ OUVRIR LE FICHIER AVANT PUBLICATION + IMAGE

## Préambule

Ajouter un paquet NuGet prend quelques secondes. Comprendre ce qu’il introduit dans une application, ce qu’il peut exécuter pendant la compilation et qui peut publier sa prochaine version demande davantage d’attention.

Cette question devient encore plus concrète lorsque des agents proposent des dépendances ou mettent à jour plusieurs projets à la fois. La rapidité d’exécution est utile, mais elle doit s’appuyer sur des règles vérifiables dans le dépôt et dans les pipelines.

Dans la session [Securing Your NuGet Dependencies in the Age of AI de .NET Conf 2026](https://www.youtube.com/watch?v=H54iAyM08VM), John Douglas présente plusieurs évolutions qui touchent autant les équipes qui consomment des paquets que celles qui en publient.

Après mon tour d’horizon de [.NET 11](https://codewithfrenchy.com/posts/dotnet11/), je propose ici une démarche pour relier ces protections. L’objectif est de réduire l’exposition aux versions compromises, de rendre les vulnérabilités visibles et de limiter les accès permettant de publier un paquet.

Les exemples visent des projets utilisant `PackageReference` et un SDK .NET 11 compatible. Certaines protections existaient déjà dans les versions précédentes. Les anciens projets utilisant `packages.config` demandent une vérification distincte, particulièrement pour les scripts d’installation.

## Consommer et publier engagent deux responsabilités

Une équipe qui consomme des paquets choisit du code qui entrera dans son environnement. Une équipe qui publie des paquets fournit ce code à d’autres équipes et doit protéger le chemin qui mène du dépôt à la version distribuée.

Une même organisation occupe généralement les deux positions. Une bibliothèque interne publiée dans Azure Artifacts peut dépendre de composants publics et être utilisée par plusieurs applications métier.

Je distingue les protections selon la question à laquelle elles répondent.

| Protection | Question traitée | Limite à garder en tête |
| --- | --- | --- |
| Délai d’adoption | Cette version vient-elle tout juste de paraître ? | Le temps écoulé ne démontre pas son innocuité |
| Audit des vulnérabilités | Cette version est-elle associée à un avis connu ? | Une compromission récente peut ne pas encore être répertoriée |
| Sources contrôlées et versions verrouillées | D’où viennent les paquets et lesquels sont restaurés ? | Une version compromise peut être reproduite fidèlement |
| Isolation de la compilation | À quoi le code de construction peut-il accéder ? | Le poste de développement reste aussi concerné |
| Trusted Publishing | Quelle identité est autorisée à publier ? | Un workflow autorisé peut lui-même être compromis |

Ces protections se complètent. Leur intérêt vient du fait qu’elles agissent à différents moments du cycle de livraison.

## Encadrer l’adoption des nouvelles versions

### Introduire un délai pour les mises à jour courantes

Installer automatiquement une version dès sa publication réduit le temps disponible pour détecter un incident. Un délai d’adoption laisse une fenêtre aux mainteneurs et à la communauté pour réagir.

La [configuration NuGet](https://learn.microsoft.com/en-us/nuget/reference/nuget-config-file) permet de définir `minPublishAgeHours` par source à partir de NuGet 8.0, du SDK .NET 11.0.100 et de Visual Studio 18.12.

Voici un exemple de `nuget.config` à placer à la racine du dépôt.

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org"
         value="https://api.nuget.org/v3/index.json"
         minPublishAgeHours="72" />
  </packageSources>
  <minPublishAgeExceptions />
  <auditSources>
    <clear />
    <add key="nuget.org"
         value="https://data.nuget.org/v3/index.json" />
  </auditSources>
</configuration>
```

Le délai de 72 heures représente un choix de politique pour cet exemple. Il faut l’adapter au niveau d’exposition et à la capacité de réaction de l’équipe. La section vide `minPublishAgeExceptions` évite de conserver des exceptions héritées d’une autre configuration.

Le fonctionnement du [minimum d’âge de publication](https://learn.microsoft.com/en-us/nuget/consume-packages/minimum-package-publish-age) comporte des limites importantes.

- `dotnet package add` et `dotnet package update` filtrent les versions lorsqu’ils choisissent automatiquement une version.
- Une version demandée explicitement contourne ce filtrage.
- Une restauration ordinaire ne vérifie pas l’âge des versions, y compris celui des dépendances transitives.
- Une référence flottante, comme `1.2.*`, provoque `NU1020` lorsqu’une source applicable impose un délai et qu’aucune exception ne couvre le paquet. La restauration ne choisit pas simplement une version plus ancienne.
- Les outils externes de mise à jour doivent recevoir leur propre configuration.

Ce mécanisme ne constitue donc pas une politique universelle sur tout le graphe de dépendances. Si une organisation exige un âge minimal pour chaque version consommée, elle doit ajouter un contrôle du graphe résolu ou une validation dans son service de distribution de paquets.

Pour un correctif de sécurité urgent, je privilégierais une adoption explicite de la version examinée, accompagnée d’une justification dans la demande de fusion. Une exemption permanente par nom de paquet s’appliquerait aussi à ses prochaines versions.

### Contrôler les sources autant que les versions

Le dépôt devrait déclarer les sources attendues. Dans un environnement mélangeant paquets publics et privés, [Package Source Mapping](https://learn.microsoft.com/en-us/nuget/consume-packages/package-source-mapping) permet d’associer les identifiants de paquets à des sources déterminées.

Par exemple, les paquets `Entreprise.*` peuvent être réservés au registre interne. L’objectif est d’éviter qu’un paquet portant le même identifiant soit récupéré depuis une source inattendue.

Deux limites méritent une attention particulière. Le mappage encadre le téléchargement des paquets, mais toutes les recherches de métadonnées ne le respectent pas. De plus, un paquet déjà présent dans le cache global peut être réutilisé sans refaire cette sélection de source.

Je validerais donc une nouvelle politique avec un cache de paquets isolé et initialement vide. Cette vérification permet de confirmer le comportement attendu sans supprimer le cache de travail de tous les développeurs.

La section `auditSources` de l’exemple sépare aussi la provenance des paquets de celle des informations de vulnérabilité. Elle utilise le point de terminaison de nuget.org réservé à ces informations. Une organisation qui passe par un registre interne doit prévoir l’accès à une source d’audit appropriée.

## Transformer l’audit en règle de livraison

### Inclure les dépendances transitives

Une vulnérabilité peut se trouver plusieurs niveaux sous le paquet référencé directement. Pour rendre l’intention explicite, voici une configuration `Directory.Build.props` fondée sur [NuGet Audit](https://learn.microsoft.com/en-us/nuget/concepts/auditing-packages).

```xml
<Project>
  <PropertyGroup>
    <NuGetAudit>true</NuGetAudit>
    <NuGetAuditMode>all</NuGetAuditMode>
    <NuGetAuditLevel>low</NuGetAuditLevel>
  </PropertyGroup>
</Project>
```

`all` inclut les dépendances transitives. `low` conserve les avis de tous les niveaux de gravité. Le seuil d’affichage et le seuil qui bloque une livraison peuvent ainsi rester différents.

Pour obtenir un rapport exploitable, la commande [dotnet package list](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-package-list) peut produire du JSON. Cet exemple s’exécute dans le répertoire de la solution après sa restauration.

```bash
dotnet package list --vulnerable --include-transitive --no-restore --format json --output-version 1 > vulnerabilites.json
```

Ce rapport sert à l’analyse et au suivi. Pour bloquer une livraison, je configure explicitement les avertissements de restauration qui doivent faire échouer le pipeline.

### Définir ce qui bloque le pipeline

NuGet utilise [des codes distincts selon la gravité](https://learn.microsoft.com/en-us/nuget/reference/errors-and-warnings/nu1901-nu1904). Voici une politique de départ, à ajuster selon les applications.

| Situation | Codes | Traitement proposé |
| --- | --- | --- |
| Vulnérabilité faible ou modérée | `NU1901`, `NU1902` | Avis visible et correction suivie avec un responsable |
| Vulnérabilité élevée ou critique | `NU1903`, `NU1904` | Blocage de la livraison, sauf exception documentée |
| Échec de récupération des données d’audit | `NU1900` | Blocage du contrôle de publication |
| Source d’audit sans informations de vulnérabilité | `NU1905` | Correction de la configuration avant publication |

Avec des fichiers de verrouillage déjà présents, le contrôle peut prendre cette forme.

```bash
dotnet restore Application.slnx --locked-mode --force --no-http-cache "-warnaserror:NU1900,NU1903,NU1904,NU1905"
```

L’option [MSBuild `warnAsError`](https://learn.microsoft.com/en-us/visualstudio/msbuild/msbuild-command-line-reference) accepte une liste séparée par des virgules. Les codes retenus font échouer la commande. Le pipeline doit ensuite respecter son code de sortie et ne pas poursuivre la publication en cas d’échec. Il faut aussi vérifier que les avertissements concernés ne sont pas masqués par la configuration existante.

Dans la [commande de restauration](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-restore), `--force` demande de refaire la restauration et `--no-http-cache` évite de réutiliser le cache HTTP pendant ce contrôle. Le cache global des paquets reste distinct. Cette vérification plus stricte convient à un contrôle de livraison ou à un audit planifié, sans imposer le même coût à chaque compilation locale.

Une indisponibilité de la source d’audit peut alors retarder une publication. C’est un compromis assumé. Pour une intervention urgente, je prévoirais une procédure d’exception identifiée, avec une approbation et une reprise de l’audit, plutôt qu’une désactivation silencieuse dans le dépôt.

### Verrouiller le graphe et réexaminer les avis

Pour une application, les [fichiers `packages.lock.json`](https://learn.microsoft.com/en-us/nuget/consume-packages/package-references-in-project-files#locking-dependencies) rendent les changements du graphe de dépendances visibles dans les revues.

La première restauration peut les créer avec cette commande.

```bash
dotnet restore Application.slnx --use-lock-file
```

Après vérification, les fichiers sont versionnés. Les restaurations de validation utilisent ensuite `--locked-mode`. Lors d’une mise à jour volontaire, le développeur régénère les fichiers et soumet leurs modifications avec celles des références.

Le verrouillage ne constitue pas une validation de sécurité. Il rend la résolution reproductible. Pour une bibliothèque publiée, son propre fichier de verrouillage n’impose pas le graphe que résoudront les applications consommatrices.

Je prévoirais aussi un audit régulier des applications maintenues, même lorsqu’aucun changement de code n’est livré. Un avis peut apparaître après l’intégration du paquet. Un pipeline qui ne s’exécute que sur les demandes de fusion ne couvre pas cette situation.

Enfin, la gravité ne remplace pas l’analyse. Il faut comprendre les conditions d’exploitation, les usages réels et les mesures compensatoires. Lorsqu’une exception est nécessaire, elle devrait viser un avis précis, avec une justification, un responsable et une date de révision. `NuGetAuditSuppress` permet ce ciblage, mais n’applique pas automatiquement une échéance.

## Les scripts de paquets et le code exécuté pendant la compilation

La session de .NET Conf annonce l’arrêt de l’exécution automatique de scripts comme `init.ps1` et `install.ps1` dans Visual Studio, avec une cible annoncée de Visual Studio 18.14. Pour un parc existant, il faut vérifier la disponibilité et le comportement dans la version effectivement déployée.

Cette évolution concerne particulièrement les anciens usages de la console NuGet. Avec `PackageReference`, les scripts `install.ps1` et `uninstall.ps1` sont déjà ignorés, comme le précise la [documentation de migration depuis `packages.config`](https://learn.microsoft.com/en-us/nuget/consume-packages/migrate-packages-config-to-package-reference).

Le gain recherché est concret : l’installation d’une dépendance ne devrait pas déclencher discrètement une étape d’initialisation qui possède les accès du développeur.

Il reste toutefois d’autres points d’exécution. Un paquet peut apporter des [fichiers MSBuild `.props` et `.targets`](https://learn.microsoft.com/en-us/nuget/concepts/msbuild-props-and-targets), y compris par `buildTransitive`. Des tâches de compilation, des analyseurs ou des générateurs peuvent aussi exécuter du code dans l’environnement de construction.

Remplacer un script PowerShell par une cible MSBuild peut améliorer l’intégration technique. Cela ne retire pas le besoin de faire confiance au code exécuté.

Pour les équipes consommatrices, je recommande d’examiner ces éléments lors de l’introduction d’une dépendance sensible et de limiter les accès des agents de compilation. Un job qui restaure et compile des contributions externes ne devrait pas disposer des autorisations de publication.

Pour les auteurs de paquets, les opérations de configuration devraient être explicites, documentées et limitées à ce qui est nécessaire. Modifier l’environnement du développeur ou télécharger des exécutables pendant la compilation augmente le coût de validation pour toutes les équipes consommatrices.

## Publier avec une identité temporaire

### Ce que change Trusted Publishing

[Trusted Publishing sur nuget.org](https://learn.microsoft.com/en-us/nuget/nuget-org/trusted-publishing) permet au pipeline de s’authentifier avec OpenID Connect. La plateforme CI émet un jeton signé, nuget.org vérifie son identité et les conditions de la politique configurée, puis fournit une clé API temporaire valable une heure.

Le gain principal est de retirer la clé de publication durable des secrets à conserver et à renouveler. Il existe toujours un justificatif d’accès pendant l’exécution, mais sa durée de vie est courte.

Pour GitHub Actions, la politique indique le propriétaire, le dépôt et le nom du fichier de workflow. Par exemple, `publication.yml` correspond à `.github/workflows/publication.yml`. Un environnement comme `nuget-production` peut aussi restreindre la politique.

Je recommande d’utiliser cet environnement pour encadrer les publications autorisées et, lorsque le contexte le justifie, demander une approbation avant l’exécution du job.

### Séparer la fabrication de la publication

Je structurerais le pipeline autour de deux jobs exécutés dans des environnements isolés.

Le premier restaure, audite, compile, teste et produit le paquet. Il ne possède pas l’autorisation OIDC permettant de publier. Le candidat est ensuite vérifié, par exemple avec un projet consommateur qui utilise le fichier `.nupkg` réellement produit.

Le second récupère ce même artefact depuis l’exécution autorisée, vérifie son identité et son empreinte attendue, puis le publie. Il ne recompile pas le code et ne relance pas les tâches du projet.

Cette séparation réduit les possibilités d’accès aux justificatifs de publication pendant la construction. Elle ne prouve pas que le paquet est inoffensif. Une empreinte permet de détecter une substitution par rapport à la valeur attendue, à condition que cette valeur provienne elle-même d’un chemin de confiance.

Voici un extrait du job de publication utilisant l’[action officielle `NuGet/login`](https://github.com/NuGet/login). Le SDK et le paquet vérifié doivent déjà être disponibles. Le job `validation` doit exister dans le workflow. Le nom de paquet et la version sont illustratifs.

```yaml
publication:
  needs: validation
  runs-on: ubuntu-latest
  environment: nuget-production
  permissions:
    contents: read
    id-token: write
  steps:
    # Ajouter ici la préparation du SDK et la récupération
    # du paquet validé provenant de la même exécution.

    - name: Obtenir une clé temporaire
      id: nuget
      uses: NuGet/login@v1
      with:
        user: ${{ vars.NUGET_USER }}

    - name: Publier le paquet validé
      shell: bash
      env:
        NUGET_API_KEY: ${{ steps.nuget.outputs.NUGET_API_KEY }}
      run: >-
        dotnet nuget push
        "./artefacts/Entreprise.Outils.1.4.0.nupkg"
        --source "https://api.nuget.org/v3/index.json"
        --api-key "$NUGET_API_KEY"
```

`NUGET_USER` contient le nom du profil nuget.org, pas son adresse courriel. La clé temporaire est demandée juste avant la publication.

L’autorisation `id-token: write` s’applique toutefois à tout le job. Placer l’authentification à la fin ne retire pas ce droit aux étapes précédentes. Les actions de préparation et de récupération des artefacts doivent donc également être contrôlées.

L’extrait conserve `@v1` pour identifier l’action utilisée. Avant sa mise en production, il faut verrouiller ses dépendances. Les [recommandations de sécurité de GitHub Actions](https://docs.github.com/en/actions/reference/security/secure-use) privilégient un fichier de verrouillage des dépendances de workflow et, à défaut, une référence par SHA complet vérifié. Les changements de ces références doivent passer par une revue.

Il faut également protéger les fichiers de workflow, les références Git autorisées à publier et les règles de l’environnement. Un environnement nommé `nuget-production` sans restrictions ne suffit pas. Une contribution externe ne doit pas pouvoir modifier le paquet ou le workflow, puis bénéficier des droits de publication.

Trusted Publishing authentifie le contexte de publication. Il ne remplace ni la revue du code ni la validation du contenu distribué.

### Adapter la démarche à la plateforme

La documentation présente GitHub Actions et GitLab. La session de conférence évoque Azure DevOps comme une intégration en préparation. Il faut donc vérifier la prise en charge effective avant de concevoir une publication vers nuget.org depuis Azure Pipelines.

Une connexion fédérée utilisée pour Azure n’implique pas automatiquement une compatibilité avec Trusted Publishing de nuget.org. De même, publier dans un registre Azure Artifacts relève des mécanismes d’authentification et des autorisations propres à ce service.

Si une clé API reste nécessaire pour nuget.org, sa gestion doit être explicite. Depuis le 17 août 2026, les nouvelles clés ont une durée maximale de 30 jours. Les clés créées avant cette date ont expiré le 1er novembre 2026, selon l’[annonce de réduction de leur durée de vie](https://devblogs.microsoft.com/dotnet/strengthening-nuget-supply-chain-security-reducing-api-key-lifetime/).

Dans ce cas, je limiterais la portée aux paquets nécessaires, conserverais la clé dans le gestionnaire de secrets du pipeline et attribuerais la responsabilité du renouvellement. Une clé exposée doit être révoquée immédiatement. Attendre son expiration laisse une possibilité de publication non autorisée.

## Mettre la démarche en place progressivement

Pour éviter une politique théorique difficile à maintenir, je commencerais par un dépôt représentatif, puis j’étendrais les réglages validés.

| Étape | Équipe qui consomme | Équipe qui publie |
| --- | --- | --- |
| Rendre la situation visible | Inventorier les sources, les dépendances et les avis transitifs | Identifier les propriétaires, les workflows et les justificatifs de publication |
| Encadrer les changements | Configurer le délai d’adoption, le mappage et les fichiers de verrouillage | Protéger les références de publication et séparer construction et publication |
| Faire respecter la politique | Activer les contrôles bloquants et les audits réguliers | Migrer vers Trusted Publishing lorsque la plateforme le permet |
| Préparer les exceptions | Définir le traitement d’un correctif urgent ou d’un avis accepté | Prévoir la révocation des accès et la suspension des publications |

Avant de généraliser, je vérifierais les contrôles avec quelques scénarios ciblés dans un dépôt de test. Un avis élevé connu doit faire échouer la validation. Une source d’audit indisponible doit produire un échec identifiable. Un contexte de publication non autorisé doit être refusé. Une modification du graphe doit être visible dans les fichiers de verrouillage.

Les indicateurs utiles dépassent le nombre d’alertes. Je suivrais la couverture des applications par l’audit, le délai de traitement des avis prioritaires, les exceptions arrivées à échéance et la proportion de publications effectuées sans clé durable.

Le bénéfice opérationnel est de rendre les décisions plus prévisibles. Les développeurs savent quelles mises à jour peuvent suivre le chemin courant, les responsables savent quelles exceptions examiner et les équipes de livraison disposent de contrôles reproductibles.

## Conclusion

Sécuriser NuGet demande de suivre le paquet depuis sa sélection jusqu’à sa publication. Le délai d’adoption encadre les mises à jour courantes, l’audit signale les vulnérabilités connues et les contrôles de sources rendent la résolution plus maîtrisable.

La compilation et la publication demandent ensuite leurs propres protections. Limiter les accès du code exécuté pendant la construction et utiliser une identité temporaire pour publier réduit l’exposition des pipelines.

Pour une équipe .NET, je commencerais par rendre l’audit transitif explicite, vérifier les sources et protéger le chemin de publication. Ces mesures offrent des gains sans exiger de reconstruire immédiatement toute la chaîne de livraison. Leur efficacité dépend surtout de règles appliquées, d’exceptions suivies et d’une capacité à réagir lorsqu’un paquet de confiance cesse de l’être.
