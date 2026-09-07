---
title: Polly devient-il payant ? Comprendre l’OSMF et ses impacts pour les équipes .NET
date: 2026-09-07 19:00:00 -0400
categories: [outil-developpement]
tags: []
---

## Préambule

Avant d’entrer dans le vif du sujet, je tiens d’abord à remercier toutes les personnes qui prennent le temps de lire, de partager et de commenter les articles que je publie sur ce blogue. Il s’agit de mon premier article depuis mon retour des vacances estivales, alors j’en profite également pour souhaiter un bon retour à tous ceux qui reprennent tranquillement leur rythme habituel après l’été.

Le monde .NET continue d’évoluer à une vitesse impressionnante et les nouveautés ne manquent certainement pas. Comme toujours, je vais essayer de suivre au mieux les différents changements, annonces et tendances qui peuvent avoir un impact concret sur nos projets et nos décisions techniques. Quelques ajustements devraient également arriver prochainement sur mon blogue, notamment dans la façon dont je partage ma veille technologique et mes différentes découvertes, particulièrement sur LinkedIn. Je vous invite donc à rester à l’affût pour la suite.

Depuis maintenant plus d’un an, l’écosystème .NET traverse une période assez particulière. Plusieurs bibliothèques que nous utilisons parfois depuis des années ont revu leur modèle de licence ou, plus largement, leur façon de financer leur développement et leur maintenance.

J’ai déjà eu l’occasion d’aborder ce phénomène sur ce blogue, notamment avec les changements entourant Fluent Assertions, AutoMapper, MediatR, MassTransit ou encore Redis.

Ces différents événements nous rappellent une réalité qui devient de plus en plus difficile à ignorer : **le fait qu’une dépendance soit open source aujourd’hui ne signifie pas que son modèle économique restera nécessairement le même pendant toute la durée de vie de notre application**.

Et voilà maintenant qu’un autre nom particulièrement important de l’écosystème .NET s’ajoute à cette discussion : **Polly**.

Le 14 juillet 2026, l’équipe derrière Polly a annoncé l’adoption de l’**Open Source Maintenance Fee**, ou OSMF. À compter du **16 novembre 2026**, certaines organisations utilisant les releases maintenues de Polly devront verser un montant de **20 $ US par mois** afin de contribuer au financement du projet.

À première vue, 20 $ par mois peut sembler presque anecdotique pour une entreprise. Mais selon moi, ce n’est justement pas le montant qui rend cette annonce intéressante.

Le véritable sujet est plutôt le suivant : **comment doit-on désormais gouverner une dépendance open source lorsqu’une obligation financière peut venir s’ajouter à son utilisation, alors que son code source demeure lui-même sous une licence open source?**

Et surtout : **qu’est-ce que cela change concrètement pour les équipes .NET qui utilisent déjà Polly?**

## Polly : un incontournable pour la résilience dans le monde .NET

Il est difficile de parler de résilience dans l’écosystème .NET sans parler de Polly. Depuis plus d’une décennie, cette bibliothèque permet d’implémenter plusieurs stratégies essentielles lorsqu’une application doit communiquer avec des dépendances externes, notamment le Retry, le Circuit Breaker, le Timeout, le Rate Limiting, le Hedging ou encore le Fallback.

Polly revendique aujourd’hui plus de 14 000 étoiles sur GitHub et plus de deux milliards de téléchargements cumulés à travers ses packages NuGet. Au fil des années, elle est donc devenue une référence dans l’écosystème .NET lorsqu’il est question de gérer les défaillances transitoires et d’améliorer la résilience des applications.

Son importance va toutefois beaucoup plus loin que son package principal.

Depuis .NET 8, Microsoft a considérablement renforcé les mécanismes de résilience proposés autour de `HttpClient` et des applications .NET modernes, notamment avec `Microsoft.Extensions.Resilience` et `Microsoft.Extensions.Http.Resilience`.

Or, détail particulièrement important pour la suite de cet article : **ces bibliothèques Microsoft sont elles-mêmes construites au-dessus de Polly**. Microsoft le précise explicitement dans sa documentation.

Polly n’est donc plus uniquement une bibliothèque que les développeurs choisissent explicitement d’installer. Elle est également devenue une brique sous-jacente de certaines abstractions de résilience proposées directement par Microsoft.

Cette distinction va devenir particulièrement importante avec l’OSMF.

## Qu’est-ce qui change réellement?

Commençons par clarifier un point important : **la licence du code source de Polly ne change pas**.

Le projet demeure open source et son code source reste disponible sous sa licence BSD actuelle. Il reste donc possible de consulter le code, de le modifier, de le forker, de contribuer au projet ou encore de compiler soi-même la bibliothèque conformément aux droits accordés par cette licence.

Ce qui change est plutôt le modèle entourant l’utilisation des **releases binaires maintenues** du projet.

L’Open Source Maintenance Fee repose justement sur une séparation entre deux concepts que l’on a souvent tendance à considérer comme une seule et même chose. D’un côté, il y a **le logiciel open source**, c’est-à-dire le code source lui-même. De l’autre, il y a **le projet open source**, ce qui englobe tout le travail nécessaire pour maintenir, sécuriser, tester, documenter et publier ce logiciel dans le temps.

Écrire une bibliothèque est une chose. La maintenir pendant dix ans en est une autre. Derrière chaque release, il faut traiter les issues, analyser les vulnérabilités, revoir les pull requests, mettre à jour les dépendances, maintenir les pipelines CI/CD, gérer la signature des artefacts et s’assurer que la bibliothèque continue de fonctionner avec l’évolution de l’écosystème .NET.

C’est précisément ce travail que l’OSMF cherche à financer.

Pour Polly, les modalités annoncées sont actuellement les suivantes :

| Élément                  | Modalité annoncée                                                                |
| ------------------------ | -------------------------------------------------------------------------------- |
| Maintenance Fee          | 20 $ US par mois                                                                 |
| Coût annuel              | 240 $ US                                                                         |
| Facturation              | Une fois par organisation                                                        |
| Nombre de projets        | Sans impact sur le montant                                                       |
| Seuil Polly              | 20 000 $ US de revenus provenant d’au moins un produit ou projet utilisant Polly |
| Début prévu              | 16 novembre 2026                                                                 |
| Paiement                 | GitHub Sponsors                                                                  |
| Support inclus           | Aucun SLA ou support prioritaire                                                 |
| Code source              | Toujours open source                                                             |
| Installations existantes | Continuent de fonctionner                                                        |

Une entreprise ayant cinquante applications utilisant Polly ne paierait donc pas cinquante fois 20 $. Elle paierait 20 $ par mois pour l’ensemble de l’organisation.

Le modèle est également lié aux revenus générés par le produit ou le projet et non simplement au fait que le logiciel soit vendu directement. Un produit gratuit qui contribue malgré tout à générer des revenus par l’intermédiaire de publicité, de services payants, de génération de prospects ou d’un autre modèle économique peut donc également entrer dans le périmètre.

## Alors, Polly devient-il réellement payant?

Depuis l’annonce, on voit régulièrement circuler des formulations comme :

> *« Polly is becoming a paid library. »*

Je trouve cette formulation beaucoup trop simplificatrice.

Polly ne devient pas une bibliothèque commerciale de la même manière qu’une bibliothèque qui abandonnerait complètement sa licence open source. Le code demeure open source. Vous pouvez encore le récupérer, le modifier, le compiler et créer votre propre distribution conformément aux conditions de la licence applicable au code source.

En revanche, l’OSMF considère que les organisations qui utilisent les **binaires officiels maintenus** dans un contexte générant des revenus doivent contribuer financièrement au projet lorsqu’elles atteignent le seuil prévu. La FAQ de l’OSMF indique d’ailleurs qu’une organisation qui ne souhaite pas payer devrait éviter les releases officielles et les références via un gestionnaire de packages, et plutôt travailler directement à partir du code source.

La nuance est donc importante.

> *Polly reste open source, mais l’utilisation de ses releases officielles maintenues dans certains contextes commerciaux devient associée à un Maintenance Fee.*

Cette distinction peut sembler purement sémantique lorsqu’on regarde uniquement le dépôt GitHub. En entreprise, elle ne l’est absolument pas, puisque la majorité des équipes consomment Polly par l’entremise de NuGet et non en compilant elles-mêmes le projet à chaque nouvelle version.

## Le coût réel n’est probablement pas 20 $ par mois

Soyons clairs, pour pratiquement n’importe quelle entreprise développant des logiciels professionnellement, **240 $ US par année représente un montant extrêmement faible**.

Si Polly protège plusieurs dizaines de services contre des pannes transitoires et contribue à la fiabilité d’une plateforme générant plusieurs millions de dollars, il serait difficile d’argumenter sérieusement que la bibliothèque ne vaut pas 240 $ par année.

Mais ce raisonnement oublie quelque chose.

Dans plusieurs grandes organisations, acheter quelque chose qui coûte 20 $ peut être beaucoup plus compliqué que le montant lui-même. Il faut parfois créer un fournisseur, obtenir une approbation budgétaire, déterminer le bon centre de coûts, vérifier les conditions juridiques, valider la méthode de paiement, documenter la dépendance, gérer le renouvellement et conserver une preuve de conformité.

Le véritable calcul ressemble donc davantage à ceci :

> **Maintenance Fee + approvisionnement + gouvernance + validation juridique + inventaire des usages + suivi des versions**

Et ce coût peut facilement dépasser les 240 $ annuels demandés.

C’est particulièrement vrai dans les grandes entreprises, les institutions financières et le secteur public. La FAQ générale de l’OSMF indique d’ailleurs que les agences gouvernementales sont également considérées dans son modèle lorsqu’elles utilisent des projets soumis à un Maintenance Fee.

Il faudra cependant surveiller la façon dont Polly précisera ce cas particulier, puisque son annonce utilise principalement le terme « company ». Pour une organisation gouvernementale, je ne présumerais donc **ni d’une exemption, ni d’une obligation**, sans validation formelle des modalités finales.

Autrement dit, le problème n’est probablement pas que Polly coûte trop cher. Le véritable enjeu est plutôt que **le coût administratif associé à cette dépendance devient soudainement non nul**.

## Le vrai changement : la gouvernance des dépendances

Pendant longtemps, la gouvernance open source dans plusieurs organisations s’est principalement concentrée sur la licence. Lorsqu’une nouvelle dépendance était ajoutée, les questions étaient souvent relativement classiques : est-ce du MIT, Apache 2.0, BSD, GPL ou AGPL? Peut-on utiliser cette bibliothèque dans un produit commercial? Avons-nous certaines obligations particulières de redistribution ou d’attribution?

Dans plusieurs organisations, une partie de ce travail est aujourd’hui automatisée au moyen d’outils de Software Composition Analysis ou à partir d’un SBOM.

L’OSMF ajoute toutefois une nouvelle dimension à cette analyse. Une dépendance peut conserver une licence open source permissive au niveau de son code tout en introduisant des conditions économiques entourant certaines distributions officielles.

Un simple inventaire des licences ne devient donc plus suffisant.

Pour une organisation mature, l’inventaire de ses dépendances devrait progressivement capturer des informations supplémentaires :

| Information                      | Pourquoi la suivre?                               |
| -------------------------------- | ------------------------------------------------- |
| Package                          | Identifier ce qui est réellement consommé         |
| Version                          | Les conditions peuvent évoluer entre les releases |
| Licence                          | Comprendre les droits sur le logiciel             |
| Dépendance directe ou transitive | Déterminant avec l’OSMF                           |
| Modèle économique                | Gratuit, commercial, dual licensing, OSMF, etc.   |
| Source de distribution           | NuGet officiel, build interne, fork               |
| Criticité                        | Évaluer le risque d’un changement futur           |
| Alternative disponible           | Mesurer le coût de sortie                         |
| Responsable                      | Éviter les dépendances sans propriétaire interne  |

Cela rejoint directement ce que j’écrivais déjà dans mon article sur les changements de licences dans l’écosystème .NET : **la sélection d’une dépendance ne devrait plus se faire uniquement sur sa qualité technique actuelle**.

Il faut également comprendre sa pérennité économique et notre capacité à absorber une évolution de son modèle.

## Vous utilisez déjà Polly : devez-vous faire quelque chose?

À court terme, probablement beaucoup moins que ce que certains messages alarmistes pourraient laisser croire.

L’entrée en vigueur annoncée est prévue pour le **16 novembre 2026**, et l’équipe de Polly précise également que les installations existantes ne cesseront pas de fonctionner à cette date.

Il existe néanmoins une nuance importante autour des versions. Au moment où j’écris ces lignes, le 7 septembre 2026, la version stable actuelle de Polly sur NuGet est la **8.7.0**, publiée en juin 2026 et toujours affichée sous licence BSD-3-Clause.

Dans les discussions du projet, Martin Costello a également indiqué que les conditions des packages déjà publiés ne peuvent pas être modifiées rétroactivement et qu’une nouvelle version devrait matérialiser le changement lorsque l’OSMF entrera en vigueur.

Je mettrais toutefois un astérisque important sur cette information. L’équipe Polly n’a pas encore publié toutes les modalités finales entourant les futurs packages NuGet, et certaines discussions montrent encore des interrogations sur la façon exacte dont l’OSMF sera représenté au niveau des artefacts.

Autrement dit, **je ne prendrais pas aujourd’hui une décision architecturale majeure sur la base d’une modalité qui doit encore être précisée avant novembre**.

En revanche, c’est maintenant le bon moment pour inventorier vos usages et comprendre exactement comment Polly entre dans vos différentes applications.

## Commencez par déterminer si Polly est réellement une dépendance directe

C’est probablement l’une des particularités les plus intéressantes du modèle OSMF.

La FAQ indique actuellement qu’une organisation ne paie pas les Maintenance Fees associés aux **dépendances transitives** de ses propres dépendances. Elle est plutôt responsable des projets qu’elle choisit directement d’utiliser.

Dans un projet .NET, l’OSMF propose donc de commencer l’analyse par les `PackageReference` présents dans les fichiers `.csproj`, ainsi que ceux qui peuvent provenir de fichiers comme `Directory.Build.props` ou `Directory.Build.targets`.

Prenons un premier scénario où Polly est directement référencé :

```xml
<PackageReference Include="Polly" Version="..." />
```

Dans ce cas, votre organisation a explicitement choisi Polly comme dépendance. Si les critères financiers sont remplis et que vous utilisez des releases soumises à l’OSMF, le Maintenance Fee devrait s’appliquer.

La situation devient beaucoup plus intéressante lorsqu’une application référence plutôt :

```xml
<PackageReference Include="Microsoft.Extensions.Http.Resilience" Version="..." />
```

Dans ce cas, votre application n’a pas choisi directement Polly. `Microsoft.Extensions.Http.Resilience` dépend de `Microsoft.Extensions.Resilience`, qui dépend notamment de packages Polly.

L’architecture ressemble donc plutôt à ceci :

```text
Votre application
       │
       ▼
Microsoft.Extensions.Http.Resilience
       │
       ▼
Microsoft.Extensions.Resilience
       │
       ▼
Polly
```

Selon la FAQ actuelle de l’OSMF, Polly est ici **une dépendance transitive**.

Dans les discussions officielles entourant l’annonce, cette interprétation a également été confirmée : une entreprise utilisant uniquement `Microsoft.Extensions.Http.Resilience` n’aurait pas à payer directement le Maintenance Fee associé à Polly.

Cette distinction est particulièrement importante, puisqu’elle pourrait influencer la façon dont les équipes structurent leurs dépendances dans les nouveaux projets .NET.

## Microsoft.Extensions.Http.Resilience est-il donc une façon d’éviter Polly?

Techniquement, non.

Et c’est ici que je nuancerais fortement certaines recommandations qui circulent présentement.

`Microsoft.Extensions.Http.Resilience` **n’est pas une implémentation concurrente de Polly**. Elle utilise Polly. Microsoft le dit explicitement dans sa documentation et les dépendances du package le démontrent également.

Le package offre surtout une couche d’intégration particulièrement intéressante avec `IHttpClientFactory`.

Par exemple :

```csharp
services
    .AddHttpClient("CatalogApi")
    .AddStandardResilienceHandler();
```

permet d’ajouter une pipeline standard combinant notamment timeout, retry, rate limiting et circuit breaker.

Pour une majorité d’appels HTTP classiques dans une application moderne, cette abstraction est excellente. Mais sous le capot, Polly est toujours là.

Passer de :

```text
Application → Polly
```

à :

```text
Application → Microsoft.Extensions.Http.Resilience → Polly
```

n’est donc pas une migration vers un autre moteur de résilience.

C’est plutôt une modification de **la couche avec laquelle votre application choisit directement d’interagir**.

Cette distinction peut néanmoins être architecturalement intéressante. Elle réduit le couplage direct entre votre code applicatif et Polly et confie à Microsoft la responsabilité de gérer une partie de cette intégration.

Mais je déconseille fortement d’effectuer une telle migration uniquement dans le but d’économiser 20 $ par mois.

## Une zone grise : utiliser directement Polly lorsqu’il est transitif

Voici maintenant un scénario beaucoup plus subtil.

Supposons que votre projet référence uniquement :

```xml
<PackageReference Include="Microsoft.Extensions.Http.Resilience" />
```

Polly arrive donc transitivement.

Mais votre code commence ensuite à utiliser directement certains types provenant de Polly, par exemple :

```csharp
ResiliencePipeline
```

Votre `.csproj` ne contient toujours aucun `PackageReference` explicite vers Polly.

Est-ce encore une dépendance transitive du point de vue de l’OSMF, ou est-ce que l’utilisation directe des APIs de Polly transforme cette relation en dépendance directe?

Au 7 septembre 2026, **la réponse n’est pas encore suffisamment claire**.

Cette question précise a été posée dans la discussion officielle de Polly. Elle cherche justement à déterminer si le concept de dépendance directe doit être évalué uniquement à partir du manifeste du gestionnaire de packages ou également à partir de l’utilisation réelle des APIs dans le code.

Aucune clarification suffisamment définitive n’était encore disponible au moment de la rédaction de cet article.

Et c’est exactement le genre de zone grise qui peut devenir problématique dans une grande organisation.

Une règle de conformité doit idéalement pouvoir être automatisée. Si le modèle nécessite qu’un juriste et un architecte analysent chaque `using`, chaque référence d’assembly ou chaque utilisation d’une API pour déterminer si un Maintenance Fee s’applique, l’avantage d’un modèle censé être simple diminue rapidement.

Pour l’instant, je recommanderais donc de ne pas essayer de jouer avec cette zone grise simplement pour éviter le paiement. Si votre application utilise réellement et explicitement les APIs de Polly, il est probablement plus prudent de considérer cette dépendance comme significative et de surveiller les prochaines clarifications du projet.

## Et les applications internes?

Les applications purement internes représentent également un scénario encore imparfaitement défini.

Imaginons une entreprise générant plusieurs millions de dollars de revenus et possédant un portail interne de gestion des employés utilisant Polly. Ce portail n’est pas vendu, n’est pas distribué à des clients et ne génère directement aucun revenu.

Est-il tout de même considéré comme étant utilisé dans une activité génératrice de revenus?

La FAQ générale de l’OSMF utilise une définition relativement large autour des activités génératrices de revenus, alors que l’annonce spécifique de Polly parle plutôt d’un produit ou d’un projet utilisant Polly qui génère des revenus.

La différence peut sembler minime, mais elle devient importante lorsqu’on pense aux milliers d’applications internes présentes dans les grandes organisations.

Cette question a elle aussi été soulevée dans les discussions suivant l’annonce sans qu’une clarification suffisamment définitive soit disponible au moment d’écrire ces lignes.

Pour une application interne, je recommande donc pour l’instant de **documenter le cas plutôt que d’inventer sa propre interprétation**.

## Faut-il migrer loin de Polly?

Ma réponse générale est simple : **non, pas uniquement à cause des 20 $ par mois**.

Imaginons une organisation ayant une centaine de microservices qui utilisent fortement Polly. Les applications sont stables, les stratégies de retry, circuit breaker, timeout et fallback sont bien testées et le comportement de la plateforme est connu en production.

Remplacer cette infrastructure pourrait nécessiter plusieurs dizaines, voire plusieurs centaines d’heures de travail. Effectuer cette migration pour économiser **240 $ par année** serait extrêmement difficile à justifier économiquement.

Il faut également considérer le risque introduit par la migration elle-même. La résilience est justement l’une des parties d’une application où une mauvaise configuration peut créer des problèmes particulièrement difficiles à diagnostiquer : retry storms, amplification des appels, mauvais timeouts, circuit breakers mal calibrés, surcharge des dépendances ou duplication d’opérations non idempotentes.

Une bibliothèque mature comme Polly apporte une valeur réelle. Le changement de modèle économique du projet ne fait pas disparaître cette valeur du jour au lendemain.

Dans une plateforme où Polly est déjà bien intégré, testé et observé, la meilleure décision sera probablement simplement de continuer à l’utiliser et de payer le Maintenance Fee si celui-ci s’applique.

## Et construire soi-même les stratégies?

La FAQ OSMF permet explicitement de récupérer le code source et de produire soi-même ses binaires conformément à la licence open source. À première vue, cette option peut sembler intéressante pour une organisation qui souhaiterait éviter le Maintenance Fee.

En pratique, elle déplace surtout le problème.

Dès que vous produisez vous-même vos artefacts, vous devenez responsable de suivre les vulnérabilités, récupérer les correctifs, reconstruire les packages, valider les nouvelles versions, signer les artefacts et maintenir votre propre pipeline de distribution.

Pour une organisation disposant déjà d’une plateforme interne mature de software supply chain, ce scénario peut être envisageable dans certaines circonstances particulières. Pour éviter 240 $ par année, il devient beaucoup moins intéressant.

La même logique s’applique au fork. Créer un fork est extrêmement simple. **Maintenir un fork pendant cinq ans ne l’est pas.**

Il faut donc faire attention à ne pas remplacer un coût financier très faible par un coût opérationnel et technique beaucoup plus important simplement par principe.

## Quelles alternatives dans l’écosystème .NET?

Il n’existe pas aujourd’hui une bibliothèque qui remplacerait Polly de façon évidente dans l’ensemble de ses scénarios. Il est beaucoup plus pertinent d’évaluer les besoins réels de l’application avant de chercher une solution de remplacement.

| Besoin                              | Option à considérer                    | Remarque                                                       |
| ----------------------------------- | -------------------------------------- | -------------------------------------------------------------- |
| Résilience HTTP standard            | `Microsoft.Extensions.Http.Resilience` | Excellent choix pour `HttpClient`, mais repose sur Polly       |
| Pipeline de résilience générique    | Polly                                  | Toujours l’une des solutions les plus complètes                |
| Intégration générique Microsoft     | `Microsoft.Extensions.Resilience`      | Repose également sur Polly                                     |
| Rate limiting uniquement            | `System.Threading.RateLimiting`        | Primitive .NET plus ciblée                                     |
| Timeout simple                      | APIs natives et `CancellationToken`    | Suffisant dans certains scénarios                              |
| Retry simple et très spécifique     | Implémentation ciblée possible         | Attention à ne pas reconstruire une bibliothèque de résilience |
| Circuit breaker ou hedging complexe | Polly                                  | Une implémentation maison est rarement justifiée               |

Pour de la résilience HTTP standard, `Microsoft.Extensions.Http.Resilience` constitue probablement aujourd’hui l’option la plus naturelle dans une nouvelle application .NET. Elle s’intègre directement avec `IHttpClientFactory`, expose une configuration cohérente avec les patterns modernes de .NET et couvre plusieurs scénarios courants sans nécessiter une dépendance directe à Polly.

Pour des pipelines de résilience plus génériques ou avancés, Polly demeure toutefois extrêmement pertinent.

Dans des cas beaucoup plus ciblés, certaines primitives natives de .NET peuvent également être suffisantes. `System.Threading.RateLimiting`, par exemple, peut répondre à un besoin spécifique de rate limiting sans ajouter toute une abstraction de résilience. Des mécanismes comme `CancellationToken` peuvent aussi suffire pour gérer certains scénarios de timeout simples.

Il faut donc éviter une réaction classique à ce genre d’annonce :

> *« Cette bibliothèque ajoute un coût, remplaçons-la immédiatement par du code maison. »*

Quelques lignes permettant de refaire un retry ne remplacent pas dix années de tests, d’optimisation, d’intégration, de documentation et de maintenance.

## Pour les nouveaux projets, la réflexion est différente

Pour une nouvelle application .NET, j’adopterais cependant une approche légèrement différente.

Si la seule raison d’ajouter directement Polly est de rendre quelques appels HTTP plus résilients, **je privilégierais aujourd’hui `Microsoft.Extensions.Http.Resilience`**.

L’intégration avec `IHttpClientFactory` est naturelle, les pipelines standards couvrent beaucoup de besoins courants et votre application dépend directement d’une abstraction soutenue dans l’écosystème Microsoft.

Microsoft recommande d’ailleurs désormais cette approche et considère l’ancien package `Microsoft.Extensions.Http.Polly` comme déprécié.

En revanche, si votre application nécessite des pipelines de résilience génériques, sophistiqués ou réutilisables en dehors de HTTP, utiliser directement Polly demeure parfaitement justifiable.

L’objectif ne devrait donc pas être :

**« Comment éviter Polly? »**

mais plutôt :

**« À quel niveau de mon architecture est-il pertinent de dépendre directement de Polly? »**

Cette question est beaucoup plus intéressante, puisqu’elle force à réfléchir au niveau de couplage réellement nécessaire entre l’application et la bibliothèque.

## Ma grille de décision

Voici comment j’aborderais aujourd’hui les principaux scénarios en entreprise :

| Situation                                                 | Recommandation                                                               |
| --------------------------------------------------------- | ---------------------------------------------------------------------------- |
| Polly est fortement utilisé dans une plateforme existante | Continuer à l’utiliser et prévoir le paiement si applicable                  |
| Quelques retries HTTP uniquement                          | Évaluer `Microsoft.Extensions.Http.Resilience`                               |
| Nouvelle application HTTP moderne                         | Privilégier l’abstraction Microsoft dans la majorité des cas                 |
| Résilience générique ou avancée                           | Polly direct demeure pertinent                                               |
| Polly est uniquement transitif                            | Pas d’action immédiate selon l’OSMF actuel                                   |
| Code utilisant directement des APIs Polly transitives     | Attendre une clarification officielle                                        |
| Application interne sans revenus directs                  | Documenter et faire clarifier le scénario                                    |
| Organisation avec procurement complexe                    | Commencer les démarches avant novembre                                       |
| Organisation refusant l’OSMF                              | Évaluer migration, build interne ou fork avec leurs coûts réels              |
| Version Polly déjà publiée et stable                      | Pas de migration précipitée; surveiller les prochaines releases              |
| Application critique nécessitant un SLA                   | L’OSMF ne répond pas à ce besoin; rechercher un véritable contrat de support |

La recommandation qui ressort de cette grille est finalement assez simple. Pour une plateforme existante utilisant déjà fortement Polly, je continuerais généralement à l’utiliser. Pour une nouvelle application dont les besoins sont essentiellement liés à `HttpClient`, je privilégierais plutôt l’abstraction Microsoft.

Dans les deux cas, je ne laisserais pas les 20 $ par mois devenir le principal facteur de décision. La qualité de l’architecture, le niveau de couplage, la maturité de la solution et le coût réel d’une migration sont beaucoup plus importants.

## Attention : le Maintenance Fee n’est pas un contrat de support

C’est un autre point que je trouve particulièrement important en entreprise.

Le paiement du Maintenance Fee ne signifie pas que vous achetez un SLA, que vos issues deviennent prioritaires ou qu’un correctif vous sera fourni dans un délai garanti. Vous n’achetez pas non plus un engagement contractuel de support.

L’OSMF finance plutôt la maintenance globale du projet : le temps consacré aux correctifs, à la sécurité, aux dépendances, aux releases et aux différentes activités nécessaires pour garder Polly en bonne santé.

Pour une application critique, il faut donc continuer à distinguer deux besoins très différents : **financer la maintenance d’un projet open source** et **acheter du support pour son organisation**.

Si vos exigences opérationnelles nécessitent une garantie de réponse à une vulnérabilité ou à un incident de production, payer l’OSMF ne résout pas cette problématique. Il faudra toujours rechercher un véritable mécanisme de support correspondant à vos exigences opérationnelles.

Cette distinction est particulièrement importante dans les organisations où une dépendance critique doit être accompagnée d’engagements contractuels précis.

## Une nouvelle responsabilité pour les mises à jour automatiques

Un autre élément auquel les équipes DevOps devront porter attention concerne les mises à jour automatiques des dépendances.

Dependabot, Renovate et les autres outils d’automatisation permettent aujourd’hui d’adopter rapidement les nouvelles versions NuGet. C’est généralement une excellente pratique, notamment pour réduire le délai d’application des correctifs de sécurité et éviter que les dépendances vieillissent pendant plusieurs années.

Mais une nouvelle version peut désormais apporter autre chose que des corrections de bugs, de nouvelles fonctionnalités ou des correctifs de sécurité. Elle peut également introduire **de nouvelles modalités économiques ou contractuelles**.

Les discussions autour de Polly ont justement soulevé le risque qu’une organisation mette automatiquement à jour son package sans réaliser qu’une nouvelle release peut être associée à des conditions différentes.

Cela ne signifie évidemment pas qu’il faut désactiver Dependabot ou Renovate. Cela signifie plutôt qu’un changement de licence, de modèle économique ou de distribution devrait devenir un **signal de gouvernance capable de soumettre une mise à jour à une validation supplémentaire**.

Dans une organisation mature, les mises à jour de dépendances ne devraient donc plus être analysées uniquement sous l’angle technique ou de la sécurité. La dimension légale et économique doit progressivement faire partie de l’équation.

Le SBOM devient alors une pièce importante de ce mécanisme, mais il ne suffit pas à lui seul. Il faut lui associer des règles de gouvernance capables de détecter qu’une dépendance n’est plus consommée exactement sous les mêmes conditions qu’auparavant.

## Un modèle qui pourrait dépasser largement Polly

C’est probablement l’aspect de cette annonce que je surveillerai le plus attentivement dans les prochains mois.

Pris isolément, le montant demandé par Polly est relativement faible et devrait être facile à absorber pour la majorité des organisations concernées. La situation devient toutefois beaucoup plus intéressante si l’OSMF commence à être adopté par plusieurs autres projets open source.

Imaginons qu’une organisation utilise directement quelques dizaines de bibliothèques importantes et que dix, vingt ou même cinquante d’entre elles adoptent progressivement un modèle similaire. Le coût financier total pourrait encore rester raisonnable pour une grande entreprise. Le véritable problème risque plutôt de se situer du côté administratif.

Chaque projet pourrait avoir son propre mécanisme de paiement, ses propres critères d’admissibilité, sa propre date de renouvellement et certaines subtilités quant aux produits ou aux usages concernés.

On passerait alors d’un problème relativement simple de gestion des licences open source à une forme beaucoup plus distribuée de gestion contractuelle des dépendances logicielles.

Ce scénario soulève également une question intéressante pour les équipes responsables des plateformes et de la gouvernance : jusqu’où voulons-nous gérer ces décisions projet par projet? Si ce type de modèle devient fréquent, il faudra probablement voir apparaître de nouveaux mécanismes permettant de centraliser ou d’automatiser davantage ces contributions.

L’OSMF cherche justement à créer un lien plus direct entre les entreprises qui bénéficient d’un projet open source et les personnes qui le maintiennent. L’objectif est difficile à critiquer en soi. La maintenance open source a un coût, même lorsque celui-ci est invisible pour les consommateurs du logiciel.

Pour fonctionner à grande échelle, le modèle devra cependant être suffisamment clair, prévisible et automatisable pour pouvoir être intégré aux processus des organisations qu’il souhaite justement faire contribuer.

## Une tendance plus large dans l’open source

Polly n’est évidemment pas un événement isolé.

Au cours des dernières années, nous avons vu de nombreux projets expérimenter différentes approches afin de trouver un équilibre entre accessibilité du code et financement de leur développement. Certains sont passés vers des licences commerciales ou du dual licensing, d’autres vers des modèles open core ou source available, tandis que certains projets ont choisi des licences comme l’AGPL afin de conserver leur caractère open source tout en protégeant davantage leur modèle économique.

J’ai déjà abordé plusieurs de ces changements dans mon précédent article consacré à l’évolution des licences dans l’écosystème .NET. Redis représente également un exemple intéressant. Après avoir abandonné son ancienne licence BSD pour adopter des licences beaucoup plus controversées, Redis 8 est finalement revenu vers une licence reconnue comme open source avec l’AGPLv3.

La situation de Polly est néanmoins différente. Il ne s’agit pas ici de fermer le code source ou de transformer directement la bibliothèque en produit commercial. Le projet tente plutôt de maintenir sa licence open source tout en créant un mécanisme permettant aux organisations qui bénéficient économiquement de son travail de contribuer à son financement.

C’est justement pour cette raison que je pense qu’il faut éviter de résumer l’annonce à **« encore une bibliothèque .NET qui devient payante »**. Cette formulation est simple et attire certainement l’attention, mais elle masque une problématique beaucoup plus profonde.

Pendant longtemps, une partie importante de notre industrie a fonctionné sur un modèle assez particulier. Des bibliothèques utilisées par des milliers d’entreprises et parfois intégrées à des systèmes générant des milliards de dollars de revenus pouvaient être maintenues essentiellement sur le temps personnel de quelques développeurs.

Les entreprises bénéficiaient gratuitement du logiciel, alors que les mainteneurs absorbaient une grande partie du coût nécessaire pour garder le projet fonctionnel, sécurisé et compatible avec l’évolution de l’écosystème.

Ce modèle possède lui aussi des limites.

Il ne faut donc probablement pas être surpris de voir apparaître de nouvelles expériences comme l’OSMF. Certaines réussiront, d’autres non, et les modèles évolueront probablement encore beaucoup dans les prochaines années.

Mais une chose semble de plus en plus claire : **le financement de l’open source devient lui aussi un élément de l’architecture logicielle que les organisations devront apprendre à considérer**.

## Ce que je recommande aux organisations

À la lumière de l’annonce de Polly, ma première recommandation serait de ne surtout pas lancer une migration massive simplement en réaction à l’introduction de l’OSMF. Pour la majorité des organisations qui utilisent déjà Polly en production, le coût et le risque associés à une migration seraient probablement beaucoup plus importants que les 240 $ US demandés annuellement. La priorité devrait plutôt être de comprendre précisément où Polly est utilisé et sous quelle forme.

La première étape consiste donc à inventorier les applications qui possèdent une dépendance directe à Polly et à les distinguer de celles où la bibliothèque arrive transitivement, notamment par l’intermédiaire de `Microsoft.Extensions.Http.Resilience`. Cette distinction est importante puisque le modèle actuel de l’OSMF ne traite pas de la même façon les dépendances directes et transitives. Il faut ensuite déterminer quels produits ou projets peuvent entrer dans le périmètre financier annoncé, tout en documentant séparément les cas qui demeurent plus difficiles à interpréter, comme certaines applications purement internes ou certains contextes gouvernementaux.

Pour les organisations qui devraient vraisemblablement être concernées par le Maintenance Fee, je commencerais également les démarches administratives avant l’entrée en vigueur du modèle en novembre. Le montant est faible, mais ce n’est pas nécessairement le cas du processus permettant de le payer. Dans certaines grandes organisations, la création d’un fournisseur, la validation juridique ou l’autorisation d’un paiement récurrent peuvent prendre beaucoup plus de temps que prévu. Ce serait assez ironique de devoir consacrer plusieurs jours de travail à régler une dépense annuelle de 240 $ simplement parce qu’elle n’a pas été anticipée.

À moyen terme, je pense toutefois que l’annonce de Polly devrait provoquer une réflexion plus large sur la gouvernance des dépendances open source. Les SBOM, les outils de Software Composition Analysis et les processus de mise à jour automatisée sont aujourd’hui principalement utilisés pour suivre les vulnérabilités et les licences. Il devient maintenant pertinent d’y intégrer également les changements de modèles économiques, de distribution ou de maintenance. Une nouvelle version d’un package ne devrait plus être évaluée uniquement en fonction de ses changements techniques : certaines mises à jour peuvent aussi modifier les conditions sous lesquelles l’organisation pourra continuer à l’utiliser.

Enfin, je recommanderais de documenter une stratégie de sortie pour les dépendances réellement structurantes de vos plateformes. Cela ne signifie pas qu’il faut construire une abstraction autour de chaque package NuGet ou prévoir systématiquement de remplacer toutes les bibliothèques tierces. Ce serait probablement contre-productif. En revanche, lorsqu’une dépendance occupe une place critique dans des dizaines ou des centaines d’applications, il est raisonnable de savoir quelles alternatives existent, quel serait le coût d’une migration et jusqu’à quel point votre architecture est couplée à cette technologie.

Ce principe dépasse largement Polly. Une architecture mature ne devrait pas nécessairement être capable de remplacer toutes ses dépendances facilement, mais elle devrait au minimum **comprendre les conséquences si l’une d’elles changeait de modèle demain**.

## Ce qu’il faut retenir

Polly ne disparaît pas et son code source ne passe pas derrière un paywall. Les applications existantes ne cesseront pas subitement de fonctionner le 16 novembre 2026 et, pour la majorité des organisations utilisant déjà largement la bibliothèque, je ne vois aucune raison de lancer une migration précipitée uniquement pour éviter un coût annuel de 240 $ US.

L’annonce demeure néanmoins importante, parce qu’elle ajoute une dimension supplémentaire à la manière dont nous devons désormais évaluer nos dépendances. Pendant longtemps, il était possible de regarder principalement la licence d’un projet, sa popularité, sa qualité technique, son niveau de maintenance et son historique de sécurité. Ces critères restent évidemment essentiels, mais ils ne racontent plus nécessairement toute l’histoire.

Il devient également important de comprendre le modèle économique du projet, la manière dont ses releases sont distribuées, les conditions pouvant être associées à leur utilisation et surtout la capacité de notre organisation à réagir si ces conditions évoluent pendant la durée de vie de notre système.

Dans le cas de Polly, la question la plus intéressante n’est donc probablement pas de savoir si la bibliothèque vaut 20 $ par mois. Pour une entreprise qui dépend réellement de Polly pour assurer la résilience de ses applications en production, la réponse est assez facile à défendre.

La question beaucoup plus importante serait plutôt : **savons-nous réellement de quelles dépendances notre organisation dépend, sous quelles conditions elles sont utilisées et ce que nous ferions si ces conditions changeaient demain?**

C’est là que se situe, selon moi, le véritable enseignement de cette annonce.

Notre relation avec l’open source doit progressivement évoluer d’une confiance implicite vers une forme de **confiance éclairée**. Utiliser une dépendance open source ne devrait pas simplement signifier vérifier qu’elle possède beaucoup d’étoiles sur GitHub ou qu’elle est largement utilisée dans l’industrie. Nous devons également comprendre qui la maintient, comment le projet est financé, quelle place il occupe dans notre architecture et quel niveau de risque nous sommes prêts à accepter si son modèle évolue.

Et il faut également reconnaître l’autre côté de l’équation.

Une bibliothèque comme Polly ne se maintient pas gratuitement. Si quelques centaines de dollars par année permettent à un projet critique de continuer à recevoir des correctifs, des mises à jour de sécurité et suffisamment d’attention de la part de ses mainteneurs, payer cette contribution peut être non seulement raisonnable, mais également représenter une **très bonne décision d’architecture**.

## Pour aller plus loin

- Annonce officielle : [Introducing the Open Source Maintenance Fee for Polly](https://thepollyproject.org/2026/07/14/polly-osmf-announcement.html)
- Documentation OSMF : [Open Source Maintenance Fee – FAQ](https://opensourcemaintenancefee.org/consumers/faq/)
- Documentation Microsoft : [Resilient app development in .NET](https://learn.microsoft.com/en-us/dotnet/core/resilience/)
- Discussion communautaire : [Polly – OSMF adoption #3183](https://github.com/App-vNext/Polly/discussions/3183)
- Article précédent : [Changements de licences dans l’écosystème .NET – quelles implications ?](https://codewithfrenchy.com/posts/changements-licences-ecosysteme-dotnet/)
- Article sur Redis : [Redis 8 – Un retour attendu à l’open source avec l’AGPLv3](https://codewithfrenchy.com/posts/redis-redevient-open-source/)
