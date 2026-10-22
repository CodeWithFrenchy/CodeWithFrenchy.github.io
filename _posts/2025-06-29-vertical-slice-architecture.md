---
title: Architecture Vertical Slice dans l’écosystème .NET
date: 2025-06-29 20:00:00 -0400
categories: [architecture, bonnes-pratiques]
tags: [dotnet, vertical-slice, clean-architecture, cqrs, mediatr, ddd, modularite, maintenabilite]
---

L’__architecture Vertical Slice__ (ou _architecture en tranches verticales_) est une approche de conception logicielle qui consiste à organiser le code par fonctionnalités verticales plutôt que par couches techniques. Autrement dit, chaque fonctionnalité de l’application est implémentée __de bout en bout__ dans une tranche (un _slice_) comprenant tous les aspects nécessaires : interface utilisateur, logique métier et accès aux données. Chaque tranche verticale est autonome et correspond à un cas d’utilisation distinct, ce qui favorise un développement modulaire centré sur les fonctionnalités. En .NET, cette approche s’accompagne souvent du patron __CQRS__ (_Command Query Responsibility Segregation_) : on sépare les opérations de lecture (_Query_) de celles d’écriture (_Command_) et on les traite différemment. La bibliothèque __MediatR__ est fréquemment utilisée pour mettre en œuvre ce style – elle agit comme un médiateur qui envoie chaque commande ou requête vers son gestionnaire (handler) dédié, permettant de découpler l’émetteur de la requête de son traitement.

## Avantages de l’architecture Vertical Slice

- __Lisibilité et cohésion par fonctionnalité :__ Le code de chaque fonctionnalité est regroupé au même endroit, ce qui le rend plus __facile à comprendre et à maintenir__ qu’une base de code où les éléments d’une même fonctionnalité sont dispersés entre plusieurs couches. Cette forte cohésion interne à la tranche améliore la lisibilité : on peut se concentrer sur un __use case__ à la fois sans être noyé dans la complexité globale.
- __Découplage des fonctionnalités :__ Chaque tranche est __indépendante des autres__, ce qui réduit le couplage du système. Modifier ou ajouter une fonctionnalité impacte peu (voire pas du tout) les autres modules, minimisant les effets de bord. On obtient ainsi un système plus robuste face aux changements, chaque slice pouvant évoluer isolément.
- __Facilité de test :__ Puisque la logique de chaque use case est isolée, on peut tester une tranche verticalement sans dépendre du reste de l’application. Par exemple, il est simple d’écrire des tests unitaires pour un _handler_ donné en simulant ses dépendances (bases de données, services externes, etc.), ce qui améliore la __testabilité__ globale.
- __Développement parallèle accéléré :__ Les équipes peuvent travailler __simultanément sur différentes fonctionnalités__ sans se marcher sur les pieds, étant donné que chaque slice est autonome. Ceci réduit également les risques de conflits de merge et facilite l’intégration du code produit par plusieurs développeurs en parallèle. En grande équipe, segmenter le travail par vertical slices permet d’accélérer le développement en divisant le système en sous-problèmes indépendants.
- __Ajout de fonctionnalités simplifié :__ L’architecture Vertical Slice tend à rendre l’application __extensible par addition plutôt que par modification__. L’ajout d’une nouvelle fonctionnalité se traduit par la création d’une nouvelle tranche (nouvelle requête, nouveau handler, etc.) __sans modifier du code existant__, ce qui limite le risque de régression. Cette propriété s’aligne bien avec le principe _Open/Closed_ – le code en place est fermé aux modifications intempestives, tandis que le système est ouvert à de nouvelles extensions fonctionnelles.
- __Flexibilité d’implémentation :__ Chaque tranche peut adopter l’approche technique la plus adaptée à son cas sans impacter le reste du projet. Par exemple, une fonctionnalité très simple pourra se contenter d’un traitement procédural direct (_transaction script_), tandis qu’une autre plus complexe pourrait utiliser des patterns de domaine riche – le choix peut se faire au niveau de la slice elle-même. De même, chaque tranche pourrait interagir avec des ressources externes différemment (base de données SQL, appel d’un service tiers, etc.) sans casser une “architecture globale” figée. Cette souplesse permet de __personnaliser la solution par fonctionnalité__ et d’éviter les abstractions ou couches supplémentaires inutiles au sein d’une slice donnée.

## Inconvénients et limites de l’architecture Vertical Slice

- __Duplication de code (anti-DRY) :__ À force d’isoler chaque fonctionnalité, on risque de __répéter du code commun__ dans plusieurs slices. Par exemple, des composants d’infrastructure ou des routines similaires peuvent se retrouver dupliqués dans chaque module fonctionnel. Cette redondance va à l’encontre du principe __DRY__ (_Don’t Repeat Yourself_) et peut alourdir la maintenance : une correction de bug ou un changement commun doit être répercuté à plusieurs endroits. En adoptant Vertical Slice, on accepte implicitement une certaine duplication en échange d’une meilleure isolation, mais il faut surveiller que cela ne devienne pas ingérable (d’où le commentaire humoristique __« Adieu DRY ! »__ souvent évoqué sur le sujet).
- __Fragmentation et vision d’ensemble réduite :__ Un risque de cette approche est de morceler l’application en de nombreux petits __silos__ de code. Si la modularité est poussée à l’extrême sans garde-fou, on peut __perdre la vue d’ensemble du système__. Chaque équipe ou développeur pouvant structurer “sa” tranche à sa manière, l’architecture globale peut manquer de cohérence. En l’absence de conventions communes, le code peut diverger d’une tranche à l’autre, rendant la maintenance transversale plus difficile. En outre, il devient délicat de comprendre le flux applicatif global puisque la logique est éclatée en morceaux indépendants. Il est donc crucial de définir des standards (structuration interne des slices, conventions de nommage, etc.) pour conserver une certaine homogénéité.
- __Complexité de gestion des interactions :__ L’architecture Vertical Slice __fonctionne mieux lorsque les fonctionnalités sont bien découplées__. Si, en pratique, vos cas d’utilisation ont de fortes interactions ou partagent des processus communs, la gestion peut devenir compliquée. Par exemple, une transaction métier impliquant plusieurs slices distinctes s’avère difficile à orchestrer proprement avec ce modèle. On devra alors multiplier les contournements (appels de slice A vers slice B, utilisation de librairies tierces pour orchestrer du workflow, etc.), ce qui peut complexifier le code et le rendre inmaintenable. En d’autres termes, Vertical Slice n’est __pas pensée pour des fonctionnalités étroitement liées ou fortement interdépendantes__, et forcer ce modèle dans de tels cas peut conduire à une usine à gaz.
- __Performances et logique transversale :__ Dans la même veine, si une opération utilisateur nécessite de coordonner plusieurs tranches verticales, l’application peut souffrir d’une certaine __surcouche__. Par exemple, devoir appeler successivement plusieurs handlers MediatR pour réaliser une action complexe peut introduire du surcoût (sérialisation/désérialisation de messages, I/O multiples, etc.). Des scénarios transactionnels touchant plusieurs slices seront difficiles à optimiser. Certes, dans la plupart des applications ce surcoût restera négligeable, mais sur un système à très hautes performances ou très complexe, c’est un point à considérer. L’architecture Vertical Slice est surtout plébiscitée pour __des projets modulaires plutôt que pour de grosses transactions multi-domaines__ – lorsqu’on tente de l’appliquer à des contextes qu’elle gère mal (transactions distribuées, règles globales), on se heurte à ses limites.

En résumé, Vertical Slice améliore la __lisibilité locale__ et la __séparation des préoccupations par fonctionnalité__, au prix potentiel de duplications et d’une __fragmentation__ du codebase. Ces inconvénients ne sont pas rédhibitoires, mais requièrent une vigilance et possiblement la mise en place de patterns complémentaires (par exemple un service commun pour factoriser du code partagé si nécessaire, ou un cadre d’architecture global minimal pour guider les devs).

## Quand utiliser l’architecture Vertical Slice ?

Comme tout choix architectural, l’approche Vertical Slice est plus ou moins adaptée selon le contexte. Voici des __situations où son usage est recommandé__, et d’autres où il l’est __moins__ :

### Scénarios recommandés

- __Applications modulaires ou microservices :__ Si votre application peut être découpée en __modules indépendants__ (par exemple par domaine métier ou _bounded context_), l’architecture Vertical Slice s’aligne naturellement. Chaque module ou microservice peut correspondre à un ensemble de slices cohérents. On parle d’ailleurs souvent de _modular monolith_ pour désigner un monolithe structuré en tranches verticales, qui offre une maintenabilité et une compréhension aisée du code. Dans ces architectures modulaires, chaque slice (ou ensemble de slices) représente un composant du système, facilement isolable pour le développement, le déploiement ou les tests.
- __APIs REST simples ou projets de petite envergure :__ Pour une __petite application, un MVP ou un service avec des cas d’utilisation simples__, adopter d’emblée une architecture hexagonale ou Clean complète peut être surdimensionné. Dans ces cas, l’approche Vertical Slice apporte de la simplicité. On évite de créer pléthore de couches et d’abstractions alors que ce n’est pas nécessaire pour un petit projet. En d’autres termes, si vos besoins sont limités et bien circonscrits, « bricoler quelque chose rapidement » en séparant par fonctionnalités est tout à fait raisonnable, et formaliser une couche de domaine complète serait une perte de temps et d’énergie. Vous irez plus vite tout en gardant une bonne organisation par _features_.
- __Développement de nouvelles fonctionnalités isolées :__ Même au sein d’un projet existant, vous pouvez utiliser Vertical Slice pour des __modules très spécifiques ou autonomes__. Par exemple, l’ajout d’une petite API indépendante dans un grand système peut se faire sous forme de slice verticale sans impacter l’architecture globale. De manière générale, si une fonctionnalité n’a quasiment __pas de dépendance sur le reste du système__, la développer comme une unité verticale permet de la livrer rapidement et de façon propre.
- __Équipes multiples sur des domaines différents :__ Si vous avez plusieurs équipes ou développeurs qui travaillent en parallèle sur des __sous-domaines différents__, leur attribuer des vertical slices différentes peut réduire la collision de leurs travaux. Chaque équipe peut évoluer dans “son” périmètre avec moins de risques de modifier du code utilisé par les autres. Dans un contexte Agile, cela facilite la livraison continue de _features_ indépendantes. (Il faut toutefois, comme mentionné, veiller à garder une cohérence globale via des _guidelines_ communes.)

### Scénarios moins adaptés

- __Grand monolithe avec logique métier partagée :__ Si votre application forme un __monolithe très complexe avec de nombreuses règles métier transverses__, une architecture strictement Vertical Slice risque de montrer ses limites. Par exemple, dans un système bancaire monolithique, des règles de gestion comme “un client ne peut pas dépasser tel découvert” s’appliquent à __plusieurs fonctionnalités__. Les implémenter séparément dans chaque tranche conduirait à de la duplication et possiblement des incohérences. Dans ce genre de contexte, il est souvent préférable d’__unifier la logique métier centrale__ dans un modèle commun (par exexemple, via une couche Domaine partagée, comme le propose _Clean Architecture_). L’architecture “propre” et dérivées (Hexagonale, etc.) sont le fruit de décennies d’expérience pour bâtir des solutions complexes de grande taille, que ce soit en monolithe ou en microservices. Elles apportent un cadre rigoureux pour garantir la cohérence de la logique métier à travers tout le système. À l’inverse, une approche Vertical Slice pure serait hasardeuse ici, car __chaque slice devrait réimplémenter ou appeler ces règles communes__, augmentant les risques d’erreur.
- __Fonctionnalités fortement couplées entre elles :__ Si vos cas d’utilisation __interagissent étroitement__, qu’ils doivent se coordonner ou partager beaucoup de données, une séparation stricte par slices peut être artificielle. Par exemple, une opération complexe pourrait nécessiter l’enchaînement de plusieurs _slices_ (ce qui revient à simuler une transaction répartie). Ce genre d’opération est pénible à réaliser proprement sans couche de service ou orchestrateur commun. Une architecture classique en couches permettrait ici de centraliser cette orchestration dans un service métier, là où Vertical Slice ne fournit pas de réponse évidente. En bref, plus vos fonctionnalités sont interdépendantes, moins l’architecture en tranches verticales est indiquée.
- __Besoins de réutilisation du code métier :__ Dans certains projets, on cherche à construire une __bibliothèque ou un noyau de composants réutilisables__ (par exemple, un moteur de calcul utilisé par plusieurs applications différentes). Dans ce cas, isoler complètement chaque feature n’est pas souhaitable : on veut au contraire factoriser le cœur commun. Vertical Slice, qui cloisonne le code par _use case_, n’est pas orientée vers la mutualisation du code. Si l’un de vos objectifs est de réutiliser une partie significative de la logique dans d’autres contextes ou d’exposer un modèle de domaine cohérent, une architecture du style Clean/DDD sera plus adaptée.
- __Equipe débutante ou hétérogène sans guidelines :__ Enfin, il faut noter qu’une équipe peu expérimentée ou sans discipline pourrait __dériver__ avec une architecture Vertical Slice. Parce qu’elle impose moins de structure prédéfinie qu’une architecture en couches, chaque développeur pourrait être tenté d’organiser « à sa sauce » sa slice. Sans concertation, le code risque de manquer d’uniformité et la maintenance deviendra difficile. Pour une équipe junior, une architecture plus classique (Clean, couches MVC, etc.) sert parfois de filet de sécurité grâce à son cadre clair. Cela ne disqualifie pas Vertical Slice, mais souligne l’importance d’un leadership technique pour guider sa mise en place dans un contexte d’équipe moins expérimentée.

En somme, l’architecture Vertical Slice brille dans les contextes __modulaires, évolutifs et indépendants__, ou lorsque l’on veut __aller vite sur des fonctionnalités ciblées__. En revanche, dès qu’une forte mutualisation ou des invariants globaux sont nécessaires, il faut soit l’adapter (en combinant avec d’autres approches), soit envisager une architecture différente plus appropriée.

## Comparaison avec le Clean Architecture

Il est fréquent d’opposer Vertical Slice à le __Clean Architecture__ (architecture “propre” d’après Robert C. Martin), car ces deux approches structurent le code très différemment. En réalité, elles poursuivent des objectifs communs (modularité, maintenabilité) mais via des principes distincts. Comparons-les selon quelques axes clés :

### Principes fondamentaux

Le Clean Architecture (ainsi que les architectures couches classiques, hexagonale, oignon, etc.) s’articule autour de la __séparation des préoccupations horizontale__ et du respect de la __dépendance inverse__. Elle vise à rendre le cœur du logiciel indépendant des détails d’implémentation. Par exemple, la logique métier ne doit dépendre ni d’un framework particulier, ni de la base de données, ni de l’UI. Concrètement, on retrouve dans Clean Architecture des couches bien définies (Entités du domaine, Cas d’usage, Interfaces d’accès aux données, etc.), avec la règle que les dépendances pointent vers l’intérieur du cercle (vers le domaine). L’accent est mis sur la __stabilité du modèle métier__ : il s’agit de protéger les règles de gestion des changements technologiques ou des caprices de l’interface.

L’architecture Vertical Slice, de son côté, est guidée par la __cohérence fonctionnelle__. Son principe central est de __grouper le code par fonctionnalité métier__ de manière autonome. Chaque slice traite __une et une seule fonctionnalité__ et embarque tout ce qu’il faut pour la réaliser. L’idée sous-jacente est d’obtenir une forte cohésion interne (tout le code lié à un cas d’utilisation est assemblé) et une faible dépendance externe (peu de liens avec d’autres parties du système). On couple “verticalement” le long d’un flux fonctionnel, plutôt qu’horizontalement par type de composant. En adoptant Vertical Slice, on accepte un certain cloisonnement par _feature_, au bénéfice d’une flexibilité locale et d’une simplicité de compréhension par cas d’usage.

En résumé, __Clean Architecture met l’emphase sur le domaine et les abstractions stables__, alors que __Vertical Slice met l’emphase sur les features et leur isolation mutuelle__. Clean cherche à éliminer le couplage __technique__ (UI, DB, frameworks) vis-à-vis du métier, tandis que Vertical Slice cherche à éliminer le couplage __fonctionnel__ entre les différentes parties du système.

### Organisation du code et du projet

Si on regarde l’organisation concrète d’un projet .NET dans chaque approche, la différence saute aux yeux. __Clean Architecture__ propose typiquement de structurer la solution en __couches__ ou couches concentriques. On peut avoir par exemple des projets ou _folders_ séparés pour : le domaine (entités et interfaces), les cas d’application (_use cases_ ou services applicatifs), l’interface (contrôleurs API, UI) et l’infrastructure (implémentations des dépôts, accès BD, services externes). Chaque couche a une responsabilité spécifique et des dépendances restreintes (p.ex. l’infrastructure dépend du domaine pour implémenter ses interfaces, mais l’inverse n’est pas vrai). Cette organisation par couches apporte de la clarté sur le rôle de chaque classe, mais peut introduire de la verbosité (de nombreux projets et fichiers pour une seule feature). Il faut naviguer entre plusieurs dossiers pour suivre le fil d’une fonctionnalité donnée.

__Vertical Slice__, à l’opposé, organise le code par __regroupement vertical__. On va créer un dossier (ou un _namespace_) par fonctionnalité ou par module métier, et y placer tous les éléments liés : contrôleur API ou endpoint correspondant, classes de commande/requête, handler MédiatR, modèles spécifiques, etc.. Par exemple, au lieu d’avoir un dossier Controllers avec tous les contrôleurs de l’application, on aura un dossier `Features` contenant des sous-dossiers par fonctionnalité : __Produit__, __Commande__, __Client__, etc., et à l’intérieur de chacun, les fichiers pour créer un produit, mettre à jour un produit, etc. Chaque slice peut ainsi avoir __ses propres modèles ou services internes__ s’il en a besoin, sans impacter les autres slices. Cette organisation par feature facilite la localisation du code – pour toucher à une fonctionnalité, on sait exactement où aller – mais rend plus difficile la réutilisation d’une classe d’une slice à l’autre (puisqu’a priori on évite de le faire). En pratique, il est courant qu’un projet Vertical Slice n’ait qu’un ou deux assemblages (ex: un projet Web et éventuellement un projet pour les contrats), là où une Clean Architecture en comporte plusieurs pour séparer les couches. Cela réduit la complexité initiale (moins de projets .NET à configurer), au prix d’une séparation moins nette entre logique métier et détails techniques.

### Couplage et dépendances

La __Clean Architecture__ excelle à réduire le couplage entre le domaine et les détails d’implémentation. Grâce à l’inversion des dépendances, le cœur métier ne “connaît” pas la base de données, ni le framework web utilisé. Cela permet, par exemple, de changer de base de données ou de technologie d’UI sans toucher au domaine (en théorie tout au moins). Le couplage est ainsi __contrôlé et dirigé__ : les couches haut-niveau dépendent de couches bas-niveau abstraites (interfaces), jamais l’inverse. En revanche, __les fonctionnalités dans Clean Architecture sont couplées via le domaine commun__. Par exemple, deux _use cases_ différents vont manipuler la même classe __Produit__ ou __Commande__. Cela garantit une logique uniforme, mais signifie aussi que ces _use cases_ sont indirectement reliés : toute modification dans la structure d’une entité du domaine peut impacter de multiples fonctionnalités. Le couplage fonctionnel n’est pas éliminé, il est centralisé dans le domaine.

L’approche Vertical Slice, de son côté, cherche à __minimiser le couplage entre fonctionnalités__ au profit d’une __forte cohésion interne__ à chaque fonctionnalité. Idéalement, chaque slice a ses propres entités ou modèles et n’interagit pas directement avec les autres slices. Le couplage technique (ex: appel base de données) n’est pas nécessairement inversé comme en Clean Architecture – une tranche peut très bien appeler directement un ORM ou une requête SQL si c’est le plus simple – mais ce choix n’affecte que cette tranche précise. En réduisant la portée d’influence du code, Vertical Slice fait pencher la balance vers une multitude de petits couplages locaux plutôt qu’un grand couplage central. Le risque est bien sûr d’introduire du __couplage dupliqué__ (plusieurs slices dépendant des mêmes concepts implémentés en parallèle), d’où l’importance de bien délimiter les frontières. On peut résumer en disant que Clean Architecture vise un __faible couplage “vertical” (technique)__, tandis que Vertical Slice vise un __faible couplage “horizontal” (fonctionnel)__. Les deux ne sont pas incompatibles, mais l’accent n’est pas mis au même endroit.

### Scalabilité du code et de l’architecture

Lorsqu’on parle de __scalabilité__ ici, on entend la capacité de l’architecture à supporter la croissance de l’application (plus de fonctionnalités, plus de développeurs, plus de complexité) tout en restant maintenable.

Du côté de Clean Architecture, la structure en couches apporte une __rigueur appréciable pour les projets de grande envergure__. C’est une architecture éprouvée pour organiser des solutions __complexes avec de nombreuses règles métier__ et de gros équipes. En séparant nettement le domaine, les cas d’utilisation et les détails techniques, on peut faire évoluer chacun de ces aspects indépendamment. Clean Architecture favorise la factorisation du code commun, ce qui évite l’effet boule de neige lors de changements globaux. Par exemple, si une règle métier impacte plusieurs cas d’utilisation, on la codera une fois dans le domaine (ou un service commun) plutôt que de devoir la répliquer. Ainsi, quand la complexité augmente, l’architecture en couches offre un cadre qui __encaisse la montée en volume__ (avec cependant le compromis d’une complexité interne plus élevée et d’une courbe d’apprentissage pour les nouveaux développeurs). On note aussi que Clean Architecture améliore la __maintenabilité et l’évolutivité__ du système sur le long terme, car les préoccupations sont bien isolées – ce qui peut justifier son investissement initial dans des projets appelés à grossir.

Du côté de Vertical Slice, la scalabilité se joue différemment. Ajouter de nouvelles fonctionnalités est très simple (comme évoqué, on ajoute du code neuf sans impacter l’existant), ce qui donne une impression de facilité pour __faire croître le périmètre fonctionnel__. De plus, plusieurs équipes peuvent contribuer en parallèle sur des slices différentes, ce qui __scale bien humainement__ (moins de collisions de code). Beaucoup de projets web ou API croissent initialement plus vite avec ce modèle qu’avec une architecture hexagonale plus formelle. Cependant, plus l’application grossit, plus on accumule de slices – et les risques mentionnés de duplication ou d’incohérence peuvent augmenter exponentiellement. Sans discipline, une grosse base de code en Vertical Slices peut devenir difficile à maintenir si chaque tranche a implémenté sa version de la réalité. En outre, __la gestion des fonctionnalités transverses__ (par ex. logging commun, transactions multi-slices, règles de validation globales) peut devenir ardue. En pratique, Vertical Slice peut tout à fait s’appliquer à de grands systèmes, mais souvent __en combinaison avec d’autres patterns__ pour encadrer la complexité. Par exemple, on peut très bien imaginer un grand monolithe modulable où chaque module est un ensemble de vertical slices, tout en conservant un schéma de base commun et des infrastructures partagées. D’ailleurs, les architectes recommandent souvent de __ne pas opposer frontalement Clean Architecture et Vertical Slice__, mais d’envisager d’associer les deux dans un même projet. Par exemple, on peut structurer son code en vertical slices au sein de chaque couche d’un modèle hexagonal (features folders + interfaces de repo, etc.), ou utiliser Vertical Slice pour la partie application tandis que le domaine reste centralisé. Ces deux approches __ne sont pas mutuellement exclusives__ et peuvent se compléter pour obtenir un système à la fois modulaire et consistant.

__En synthèse__, Clean Architecture offre une colonne vertébrale solide pour les __systèmes complexes et de long terme__, tandis que Vertical Slice apporte de la __légèreté et de la flexibilité pour le développement orienté fonctionnalité__.

Le tableau ci-dessous résume quelques différences :

Voici une version restructurée, plus fluide et symétrique pour chaque point :

- __Principe directeur__ :
  Clean Architecture repose sur une organisation par couches avec des dépendances stables orientées vers le domaine. Vertical Slice se structure par fonctionnalités, avec une forte cohésion autour de chaque cas d’usage.
- __Organisation du code__ :
  Clean Architecture répartit le code en couches séparées (domaine, application, infrastructure). Vertical Slice regroupe tout le code nécessaire à une fonctionnalité dans un même ensemble (commande, handler, modèles, etc.).
- __Couplage__ :
  Clean Architecture vise un découplage fort vis-à-vis des détails techniques, mais centralise les fonctionnalités autour d’un modèle commun. Vertical Slice réduit le couplage entre fonctionnalités, en favorisant une autonomie locale, au prix potentiel d’une duplication.
- __Échelle et complexité__ :
  Clean Architecture convient bien aux applications complexes ou d’envergure, avec une structure rigide mais robuste à long terme. Vertical Slice est plus adapté aux projets modulaires ou aux fonctionnalités isolées, avec un démarrage rapide mais qui demande un encadrement au fil de la croissance.

Il n’y a pas de « gagnant » absolu : le choix dépend du contexte de votre projet. L’important est de comprendre ces différences pour appliquer le bon dosage. La section suivante illustre concrètement la structure Vertical Slice dans un projet .NET pour mieux ancrer ces concepts.

## Exemple de projet Vertical Slice en .NET (C#)

Pour rendre les choses plus concrètes, prenons un exemple d’application .NET utilisant l’architecture Vertical Slice. Imaginons une API pour gérer des commandes (__Orders__) dans un système e-commerce. Nous allons voir comment structurer le code par fonctionnalités, et comment s’articule l’utilisation de __MediatR__, des __Commands__ et __Queries__.

### Structure par fonctionnalités

Supposons que notre API offre deux opérations principales pour les commandes : __Créer une nouvelle commande__ (__Create Order__) et __Obtenir les détails d’une commande__ (__Get Order Details__). En Vertical Slice, on va créer deux tranches séparées pour ces deux cas d’utilisation. La structure du projet pourrait ressembler à ceci :

```text
📦 MonProjet.API
└── Features
    └── Orders
        ├── CreateOrder
        │   ├── CreateOrderCommand.cs       // Commande pour créer une commande
        │   └── CreateOrderHandler.cs       // Handler pour traiter la création
        └── GetOrderDetails
            ├── GetOrderDetailsQuery.cs    // Requête pour obtenir les détails
            └── GetOrderDetailsHandler.cs  // Handler pour traiter la requête
```

Dans ce schéma, tout ce qui concerne __“Create Order”__ vit dans son propre dossier (sous __Orders/CreateOrder__), et pareil pour __“Get Order Details”__. On pourrait également avoir des sous-dossiers par agrégat ou entité principale (ici __Orders__ regroupe les features liées aux commandes). L’idée est qu’en ouvrant le dossier d’une fonctionnalité, on retrouve __toutes les pièces du puzzle__ pour cette fonctionnalité, plutôt que de devoir chercher dans un dossier Controllers, puis un dossier Services, puis Repository, etc.

### Command, Query et Handler avec MediatR

Voyons maintenant à quoi ressemblent les classes à l’intérieur d’une slice. Nous allons créer un __Query__ pour la lecture des détails d’une commande, et une __Command__ pour la création d’une commande. Nous utiliserons les interfaces de __MediatR__ (`IRequest<T>` et `IRequestHandler<TRequest, TResponse>`) pour définir nos requêtes/commandes et leurs gestionnaires.

__Exemple : Lire les détails d’une commande (Query)__

```csharp
// DTO (Data Transfer Object) représentant le résultat renvoyé au client
public record OrderDto(int OrderId, string ProductName, decimal TotalAmount);

// Requête de lecture pour obtenir les détails d'une commande spécifique
public record GetOrderDetailsQuery(int OrderId) : IRequest<OrderDto>;

// Handler associé à la requête GetOrderDetailsQuery
public class GetOrderDetailsHandler : IRequestHandler<GetOrderDetailsQuery, OrderDto>
{
    private readonly IOrderRepository _repository;

    public GetOrderDetailsHandler(IOrderRepository repository)
    {
        _repository = repository;
    }

    public async Task<OrderDto> Handle(GetOrderDetailsQuery query,
       CancellationToken cancellationToken)
    {
        // Récupérer la commande depuis la base de données (ou autre source)
        var order = await _repository.GetById(query.OrderId);
        if (order is null)
        {
            return null; // ou lever une exception NotFound, selon les besoins
        }

        // Mapper les données de la commande vers le DTO de sortie
        return new OrderDto(order.Id, order.ProductName, order.TotalAmount);
    }
}
```

Dans cet extrait, `GetOrderDetailsQuery` est une simple classe (ici un `record`) qui porte les paramètres nécessaires (l’Id de la commande). Elle implémente `IRequest<OrderDto>` indiquant qu’elle attend une réponse de type `OrderDto`. Le `GetOrderDetailsHandler` contient la logique pour traiter la requête : typiquement, il va chercher la commande dans un dépôt (`IOrderRepository`) et convertir le résultat en DTO. Grâce à MediatR, ce handler sera automatiquement appelé lorsque nous enverrons un objet `GetOrderDetailsQuery` via le médiateur.

__Exemple : Créer une nouvelle commande (Command)__

```csharp
// Commande pour créer une nouvelle commande
public record CreateOrderCommand(int ProductId, int Quantity) : IRequest<OrderDto>;

// Handler associé à la commande CreateOrderCommand
public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, OrderDto>
{
    private readonly IOrderRepository _repository;  // on peut imaginer qu'on utilise un dépôt ou un service de domaine

    public CreateOrderHandler(IOrderRepository repository)
    {
        _repository = repository;
    }

    public async Task<OrderDto> Handle(CreateOrderCommand command,
       CancellationToken cancellationToken)
    {
        // Logique métier simplifiée: créer l'entité Order et l'enregistrer
        var newOrder = new Order 
        {
            ProductId = command.ProductId,
            Quantity = command.Quantity,
            // ... (autres initialisations, calcul du total etc.)
        };

        await _repository.Add(newOrder);

        // Retourner un DTO représentant la commande créée
        return new OrderDto(newOrder.Id, newOrder.ProductName, newOrder.TotalAmount);
    }
}
```

Ici, `CreateOrderCommand` encapsule les informations nécessaires pour créer une commande (par ex. identifiant du produit, quantité). Le `CreateOrderHandler` effectue la création : dans une vraie application, il pourrait appeler des règles de domaine (vérifier le stock, calculer le montant, etc.) et utiliser le dépôt pour sauvegarder la commande en base. À la fin, il retourne un `OrderDto` avec les détails de la nouvelle commande. Remarquez que toute la logique spécifique à "créer une commande" est confinée dans ce handler – si demain une règle change (par ex. limiter la quantité maximale), on viendra l’implémenter ici sans impacter d’autres features.

__Intégration dans un contrôleur ou un endpoint :__ Grâce à MediatR, nos controllers restent très simples. Par exemple, un contrôleur Web API pour créer une commande pourrait ressembler à :

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IMediator _mediator;

    public OrdersController(IMediator mediator)
    { 
        _mediator = mediator; 
    }

    [HttpPost]
    public async Task<IActionResult> CreateOrder([FromBody] CreateOrderCommand command)
    {
        OrderDto result = await _mediator.Send(command);
        return CreatedAtAction("GetOrder", new { id = result.OrderId }, result);
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetOrder(int id)
    {
        OrderDto resultat = await _mediator.Send(new GetOrderDetailsQuery(id));

        return resultat is null ? NotFound() : Ok(result);
    }
}
```

On voit que le contrôleur ne contient quasiment aucune logique : il se contente de transmettre la commande/requête au médiateur (`_mediator.Send(...)`) et de retourner la réponse appropriée (ici un code 201 Created ou 200 OK). Toute la “vraie” logique est implémentée dans nos handlers, ce qui correspond bien à l’esprit Vertical Slice : __chaque requête HTTP correspond à un use case géré par un handler dédié__. Cette structure rend les controllers minces et faciles à maintenir, et déplace le poids de la logique dans les slices où elle est plus facile à tester.

### Points à noter dans cet exemple

- On a un fichier par requête/commande et un par handler, mais on aurait pu regrouper la commande et son handler dans le même fichier (c’est une question de style). L’important est que la séparation est faite par cas d’utilisation et non par type d’objet.
- Les classes `OrderDto`, `CreateOrderCommand`, `GetOrderDetailsQuery`, etc., sont __spécifiques à leur slice__. Aucune d’entre elles n’est utilisée en dehors de la fonctionnalité en question. Cela garantit que modifier l’une n’impactera pas d’autres parties du système par effet de bord inattendu.
- Chaque handler pourrait avoir ses propres __validations__ (par exemple, vérifier que `Quantity` est positive dans `CreateOrderHandler`). On pourrait utiliser des _Behavior_ MediatR ou des filtres, mais souvent chaque slice gère ses validations soit inline, soit via des composants dédiés (par exemple un `FluentValidation` validator par commande). L’essentiel est que, là encore, la validation métier d’un use case vit à côté de ce use case.
- On peut tout à fait utiliser des patterns de conception à l’intérieur d’un vertical slice. Par exemple, si la logique de création de commande devenait très complexe, on pourrait introduire une couche de __domaine__ (entité __Order__ avec des méthodes, service de domaine, etc.) __au sein__ de cette slice. Vertical Slice n’interdit pas d’écrire du code propre ! Il dit juste : « ne le rends pas __partagé__ d’office si ce n’est pas nécessaire ». Vous pouvez donc combiner Vertical Slice et principes DDD/Clean à un niveau fin, en créant une mini-architecture hexagonale interne à une tranche si le besoin s’en fait sentir.

## Réflexion et questions ouvertes

Nous avons exploré ce qu’est l’architecture Vertical Slice, ses atouts et ses faiblesses, ainsi que son positionnement vis-à-vis de l’architecture Clean. Pour conclure cet article, il convient d’insister qu’il n’existe pas de solution universelle – __le choix dépend de votre contexte spécifique__. Voici quelques questions ouvertes pour alimenter votre réflexion et vous aider à évaluer la pertinence de Vertical Slice pour votre projet :

- __Vos fonctionnalités sont-elles réellement indépendantes ?__ Partagez-vous beaucoup de règles métier et de données entre différentes parties de l’application, ou bien pouvez-vous facilement isoler des __slices__ sans créer de doublons massifs ? Si votre domaine est très connecté, une architecture par couches pourrait mieux convenir. Si au contraire vous pouvez dessiner des frontières nettes, Vertical Slice peut apporter de la clarté.
- __Quelle complexité anticipez-vous à long terme ?__ S’il s’agit d’un petit service ou d’un module simple, Vertical Slice vous fera gagner du temps et de la souplesse. En revanche, pour un produit stratégique qui va évoluer sur des années avec de nombreuses fonctionnalités, envisagez-vous que l’absence d’un modèle central puisse devenir un frein (duplication, incohérences) ? Faut-il prévoir une combinaison d’approches pour grandir sereinement ?
- __Quelle importance accordez-vous à la réutilisation et aux abstractions ?__ Préférez-vous du code dupliqué mais simple et lisible, ou une factorisation poussée quitte à introduire des couches d’abstraction supplémentaires ? La première approche favorise Vertical Slice, la seconde s’aligne plus avec Clean/DDD. En fonction de votre priorité (rapidité de développement vs. rationalisation du code), l’une ou l’autre approche prendra l’avantage.
- __Votre équipe et votre organisation sont-elles prêtes ?__ Considérez le niveau d’expérience de vos développeurs et la structure de votre équipe. Sont-ils à l’aise pour évoluer sans le filet d’une architecture standard ? Vont-ils suivre des conventions communes pour ne pas que chaque slice devienne un microcosme isolé ? Avez-vous les outils pour documenter et surveiller la cohérence de l’ensemble ? L’architecture Vertical Slice demande une certaine discipline dans un contexte d’équipe pour éviter l’effet « tour de Babel » où chacun code dans son coin. À l’inverse, une équipe responsable et bien synchronisée pourra prospérer grâce à la liberté qu’elle offre.

En répondant à ces questions, vous serez en mesure de peser le __pour et le contre__ de l’architecture Vertical Slice dans votre cas particulier. N’hésitez pas à expérimenter à petite échelle, voire à combiner des approches (comme structurer votre code en slices tout en conservant un noyau de domaine commun pour les invariants critiques). L’important est de choisir une architecture qui __sert au mieux les besoins de votre projet et de votre équipe__ – que ce soit Vertical Slice, Clean Architecture, une combinaison des deux, ou toute autre variante architecturale. Après tout, une architecture n’est réussie que si elle vous permet de livrer un logiciel de qualité, maintenable et évolutif, dans les délais et avec le sourire de l’équipe 😊.

👉 Pour approfondir la réflexion et découvrir un point de vue éclairant sur le sujet, je vous recommande vivement la vidéo de Milan Jovanović.

<iframe width="740" height="473" src="https://www.youtube.com/embed/Az4Z0HJYl-U?si=5irssXM9jNAyxTVM" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>