---
title: "Sécurité mémoire en C# : rendre les obligations des API unsafe explicites"
date: 2026-11-17 19:00:00 -0400
categories: [dotnet, securite]
tags: [dotnet, dotnet11, csharp, csharp15, unsafe, interop, securite-memoire, tests]
---

>  ⚠️ OUVRIR LE FICHIER AVANT PUBLICATION + IMAGE

## Préambule

Une méthode reçoit une adresse mémoire et une longueur, puis appelle une bibliothèque native. Sa signature utilise `nint` et `int`. Aucun pointeur n’apparaît dans le code de l’application qui l’appelle.

Pourtant, une longueur incorrecte peut provoquer une lecture au-delà de la zone autorisée. Une adresse conservée trop longtemps peut désigner de la mémoire déjà libérée. Le fait de manipuler une adresse sous la forme d’un entier ne fait pas disparaître ces risques.

La session [.NET Runtime vNext: Memory Safety and Runtime Async de .NET Conf 2026](https://www.youtube.com/watch?v=7udirOvQqLk) présente une évolution de C# qui vise précisément cette situation. Une API peut rendre explicites les obligations de sécurité mémoire qu’elle laisse à son appelant.

Après avoir couvert plusieurs fonctionnalités de [.NET 11](https://codewithfrenchy.com/posts/dotnet11/), je m’intéresse ici à ce changement de contrat. L’objectif est de comprendre les règles, de préparer leur adoption dans une bibliothèque et de vérifier les frontières avec le code natif.

Les nouvelles règles demeurent en préversion avec C# 15 et .NET 11. Cet article s’appuie sur la documentation consultée le 17 novembre 2026. Les exemples illustrent des contrats et une démarche de validation, sans constituer une bibliothèque d’interopérabilité complète.

## Le problème à résoudre se trouve à la frontière des API

Le code managé dispose déjà de protections importantes, notamment le contrôle des indices et la gestion de la durée de vie des objets par le garbage collector. Les accès directs à la mémoire et les appels natifs permettent toutefois de sortir de ce cadre.

Dans le modèle historique, le mot-clé `unsafe` permet surtout d’utiliser certaines constructions dans une portée donnée. Une méthode marquée `unsafe`, dont la signature ne contient aucun pointeur, n’impose pas à son appelant de reconnaître un contrat particulier.

Le [nouveau modèle de sécurité mémoire](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/unsafe-code#the-updated-memory-safety-model-preview) donne une autre responsabilité au modificateur placé sur un membre. Il signale que son utilisation demande un contexte `unsafe`, parce que certaines conditions doivent être garanties par l’appelant.

Le gain est concret pour une revue de code. L’équipe peut repérer les appels qui demandent une justification et examiner cette justification là où les informations nécessaires sont disponibles.

Le compilateur ne démontre cependant pas qu’une adresse est valide. Il ne vérifie pas non plus l’implémentation d’une bibliothèque native. La déclaration rend l’obligation visible, sans constituer une preuve que cette obligation est respectée.

## Activer les nouvelles règles explicitement

L’activation présentée dans les [nouveautés de C# 15](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-15) combine `LangVersion` et la fonctionnalité de compilation `updated-memory-safety-rules`.

Voici un projet minimal destiné au code applicatif.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net11.0</TargetFramework>
    <LangVersion>preview</LangVersion>
    <Features>updated-memory-safety-rules</Features>
    <AllowUnsafeBlocks>false</AllowUnsafeBlocks>
  </PropertyGroup>
</Project>
```

Cet exemple suppose que `Features` ne contient aucune autre option. Dans un dépôt existant, il faut conserver les fonctionnalités déjà configurées.

Les réglages ont des rôles distincts.

| Réglage | Rôle |
| --- | --- |
| `LangVersion` à `preview` | Activer la syntaxe de préversion |
| `Features` avec `updated-memory-safety-rules` | Appliquer les nouvelles obligations et inscrire le choix dans les métadonnées de l’assembly |
| `AllowUnsafeBlocks` | Autoriser ou interdire l’utilisation de `unsafe` dans les sources du projet |

Pour le projet qui contient l’adaptateur natif, `AllowUnsafeBlocks` doit être à `true`.

```xml
<AllowUnsafeBlocks>true</AllowUnsafeBlocks>
```

`LangVersion` à `preview` ne suffit donc pas à faire respecter le contrat côté appelant. La propriété stable `MemorySafetyRules` est annoncée, mais pas encore implémentée selon la [documentation des options du compilateur](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/compiler-options/language#enable-the-updated-memory-safety-rules).

## Séparer le contrat public de l’opération dangereuse

Avec les nouvelles règles, deux usages de `unsafe` doivent être distingués.

| Emplacement | Signification |
| --- | --- |
| Sur la déclaration d’un membre | L’appelant doit respecter des obligations supplémentaires |
| Autour d’un bloc ou d’une expression | Le code situé à cet endroit assume la responsabilité des opérations effectuées |

Le modificateur sur la méthode ne crée plus, à lui seul, un contexte `unsafe` dans son corps. Un appel à une autre API exigeant ce contexte, ou un déréférencement, demande encore un bloc ou une expression explicite.

Une [expression `unsafe`](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/unsafe) permet de limiter cette portée à un seul appel.

La manipulation des pointeurs évolue aussi. Détenir un pointeur et accéder à la mémoire qu’il désigne deviennent deux opérations distinctes. Les déréférencements comme `*p` ou `p[index]` restent des opérations qui demandent un contexte `unsafe`.

Pour une bibliothèque, deux stratégies se présentent. Elle peut transmettre les obligations à ses consommateurs, ou fournir une façade qui les prend en charge à l’intérieur de son implémentation.

Ce choix doit suivre les garanties réellement offertes. Ajouter un bloc `unsafe` uniquement pour faire disparaître un diagnostic laisse le problème entier.

## Un exemple de contrat pour une bibliothèque native

Prenons une bibliothèque fictive qui calcule une empreinte de 32 bits sur une séquence d’octets. L’appel est synchrone. La fonction native lit le tampon sans le modifier et ne conserve pas son adresse après son retour.

Voici la signature de l’API de bas niveau, sans son implémentation.

```csharp
public static unsafe uint CalculerEmpreinteNative(
    nint adresse,
    nuint longueur)
```

Le choix de `nuint` suppose ici que la longueur native utilise `size_t`. Il faut vérifier cette correspondance avec la déclaration réelle de la bibliothèque, comme pour tout contrat binaire.

La longueur représente un nombre d’octets. Pour une entrée vide, notre contrat prévoit que l’adresse est ignorée et qu’aucune lecture n’est effectuée.

Une documentation utile doit préciser les responsabilités de chaque côté.

| Élément | Obligation de l’appelant | Garantie de l’API appelée |
| --- | --- | --- |
| Étendue | Fournir une zone lisible d’au moins `longueur` octets | Lire uniquement cette zone |
| Initialisation | Fournir des octets initialisés | Ne pas lire de mémoire supplémentaire |
| Durée de vie | Maintenir la zone valide et son adresse stable pendant l’appel | Ne pas conserver l’adresse après le retour |
| Concurrence | Empêcher les écritures concurrentes pendant la lecture native | Ne pas modifier le tampon |
| Entrée vide | Fournir une longueur nulle | Ne pas déréférencer l’adresse |

Pour ce contrat fictif, la fonction accepte un alignement d’un octet et ne prend pas possession de la mémoire. Avec une API réelle, ces points doivent être confirmés. Une structure native peut imposer un alignement différent. Une autre fonction peut transférer la propriété du tampon.

### Documenter les obligations avec `safety`

La [balise de documentation XML `safety`](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/xmldoc/recommended-tags#safety) permet de placer les préconditions au-dessus de la signature.

Voici un extrait de documentation à associer à notre méthode.

```csharp
/// <summary>
/// Calcule une empreinte sur une zone de mémoire native.
/// </summary>
/// <param name="adresse">Adresse du premier octet à lire.</param>
/// <param name="longueur">Nombre d’octets à lire.</param>
/// <safety>
/// <para>
/// Si longueur est non nulle, adresse désigne une zone lisible
/// contenant au moins longueur octets initialisés.
/// </para>
/// <para>
/// La zone reste valide à la même adresse pendant tout l’appel.
/// Aucune écriture concurrente ni libération ne peut intervenir.
/// </para>
/// </safety>
/// <remarks>
/// La méthode ne modifie pas la zone et n’en prend pas possession.
/// Elle ne conserve aucune adresse après son retour.
/// Si longueur est nulle, aucune lecture n’est effectuée.
/// </remarks>
```

Le compilateur recopie cette balise dans le fichier XML de documentation. Il n’en vérifie ni la présence ni la validité du contrat. Un analyseur peut compléter ce travail, mais il faut le configurer explicitement.

Pour une bibliothèque distribuée, j’activerais aussi la génération de la documentation.

```xml
<GenerateDocumentationFile>true</GenerateDocumentationFile>
```

Je vérifierais ensuite que le paquet et le site de documentation exposent réellement ces informations. Un contrat présent uniquement dans le dépôt aide moins les équipes qui consomment le paquet.

### Une adresse non nulle ne prouve pas sa validité

L’adaptateur peut vérifier certaines conditions, comme une longueur maximale autorisée ou une adresse nulle accompagnée d’une longueur non nulle.

Ces contrôles ne démontrent pas que l’allocation existe encore, qu’elle appartient à l’appelant ou qu’elle contient assez d’octets. Une valeur `nint` ne transporte pas cette information.

Le même problème apparaît avec une longueur fournie séparément du tampon. Si les deux valeurs viennent de sources indépendantes, leur cohérence doit être établie ailleurs.

Lorsque cette cohérence ne peut pas être établie par l’adaptateur, le contrat doit rester explicite pour son appelant.

## Construire une façade qui prend réellement les obligations en charge

Pour les consommateurs habituels, je proposerais une seconde signature.

```csharp
public static uint CalculerEmpreinte(ReadOnlySpan<byte> octets)
```

Cette surface évite d’exposer une adresse et une longueur indépendantes. L’implémentation dispose d’une vue bornée sur les octets.

Il reste toutefois un point souvent oublié. Une [vue `ReadOnlySpan<byte>`](https://learn.microsoft.com/en-us/dotnet/api/system.readonlyspan-1?view=net-11.0) empêche les écritures à travers cette vue. Elle ne rend pas le tableau sous-jacent immuable et n’interdit pas qu’un autre thread le modifie.

Comme notre contrat natif interdit les écritures concurrentes, je choisirais ici une copie de travail privée avant l’appel. Voici l’algorithme de cette façade, exprimé en pseudocode.

```text
Vérifier la limite de taille acceptée par l’application
Traiter explicitement le cas vide
Copier les octets dans un tableau privé
Épingler ce tableau pendant l’appel synchrone
Déduire l’adresse et la longueur de ce même tableau
Appeler la fonction native dans une portée unsafe minimale
Terminer l’épinglage après le retour de la fonction
Retourner l’empreinte calculée
```

L’[instruction `fixed`](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/statements/fixed) sert à empêcher le déplacement de la mémoire managée pendant l’utilisation du pointeur.

La copie privée assure que le tampon transmis au code natif n’est pas modifié par un autre consommateur. Elle ne rend pas atomique la copie initiale si la source change pendant cette copie. Un résultat correspondant à un instant précis demande une synchronisation supplémentaire.

Au point d’appel, le commentaire peut expliquer la justification concrète. L’extrait suivant représente uniquement l’expression exécutée dans la portée d’épinglage.

```csharp
// SAFETY: le tampon est privé, initialisé et épinglé.
// Sa longueur provient de cette même allocation.
// L’appel natif est synchrone et ne conserve pas l’adresse.
unsafe(CalculerEmpreinteNative(adresse, longueur))
```

Cette justification dépend aussi de la bibliothèque native. Une façade ne peut pas garantir la sécurité d’une fonction qui lit au-delà de la longueur annoncée.

### Assumer le compromis entre copie et performance

Cette façade ajoute une allocation et une copie. Le gain recherché est la simplicité du contrat pour les consommateurs.

Dans un chemin critique, une autre conception peut éviter cette copie, par exemple avec un tampon dont la propriété exclusive est déjà contrôlée par le composant. Il faut alors démontrer cette exclusivité et la durée de vie correspondante.

Je mesurerais séparément le temps de copie, le coût de l’appel natif et les allocations selon la taille des entrées. Les nouvelles règles ne promettent pas une accélération automatique.

## Ce qui change pour l’interopérabilité native

### Choisir explicitement entre `safe` et `unsafe`

Avec le nouveau modèle, une déclaration `extern` doit annoncer son contrat au moyen de `safe` ou de `unsafe`.

Le [modificateur `safe`](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/safe) indique que l’appelant n’a pas à fournir de garantie supplémentaire de sécurité mémoire. Il ne signifie pas que le compilateur a analysé le code natif.

Une fonction qui renvoie simplement un identifiant peut convenir à ce contrat. Une fonction qui reçoit une adresse brute et une longueur demande généralement des obligations supplémentaires.

Les méthodes managées ordinaires n’ont pas besoin d’être toutes annotées `safe`. Dans le projet d’interopérabilité, j’autoriserais explicitement `unsafe`, car les compilateurs récents imposent aussi cette autorisation pour déclarer `safe`. Ce détail renforce l’intérêt de fixer la version du SDK pendant la préversion.

### Examiner aussi le code généré

Les [recommandations d’interopérabilité de .NET](https://learn.microsoft.com/en-us/dotnet/standard/native-interop/best-practices) privilégient `LibraryImport` lorsque le scénario le permet.

Avec la [génération de code P/Invoke](https://learn.microsoft.com/en-us/dotnet/standard/native-interop/pinvoke-source-generation), une partie de l’adaptateur est produite pendant la compilation. La signature écrite à la main ne suffit donc pas pour comprendre toute la frontière.

La [spécification des nouvelles règles](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/unsafe-evolution) distingue notamment le cas où le générateur produit directement une déclaration `extern` de celui où il produit un adaptateur managé. Les déclarations partielles doivent rester cohérentes. La présence d’un choix explicite sur toute façade générée peut aussi relever d’un analyseur.

Pour une migration, je vérifierais les versions du SDK et du générateur, les diagnostics obtenus et le code produit. Modifier directement les fichiers générés ne règle pas leur régénération à la compilation suivante.

### Conserver les vérifications du contrat binaire

Les nouvelles annotations ne corrigent pas une signature native erronée. Il reste nécessaire de vérifier la taille des types, l’encodage des chaînes, la convention d’appel, l’alignement et les règles de libération.

Les champs de certaines structures à disposition mémoire explicite ou particulière demandent également un choix `safe` ou `unsafe`. Ce sujet mérite une attention spéciale pour les unions et les structures partagées avec une bibliothèque C ou C++.

`SafeHandle` reste un outil adapté à la gestion des ressources natives. Il ne prouve toutefois pas qu’une adresse arbitraire et une longueur désignent un tampon valide.

Enfin, un appel qui conserve un pointeur après son retour dépasse le scénario de notre façade. L’épinglage doit couvrir toute la période d’utilisation réelle, y compris les opérations différées et les callbacks. Un bloc `fixed` limité à l’appel initial ne suffit pas.

## Préparer les bibliothèques et leurs consommateurs ensemble

Une annotation n’apporte sa protection à la compilation que si le consommateur applique les règles correspondantes. Le [comportement entre assemblies](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/compiler-options/language#enable-the-updated-memory-safety-rules) dépend donc des deux côtés.

| Appelant | Bibliothèque | Conséquence |
| --- | --- | --- |
| Nouvelles règles | Nouvelles règles | Les marqueurs du contrat sont reconnus |
| Nouvelles règles | Anciennes règles | La compatibilité traite les signatures à pointeurs comme exigeant `unsafe` |
| Anciennes règles | Nouvelles règles | Les anciennes règles restent appliquées, une signature sans pointeur peut rester appelable sans contexte `unsafe` |

Une ancienne API qui utilise seulement `nint` n’est donc pas automatiquement couverte par la règle de compatibilité fondée sur les pointeurs.

Pour un paquet partagé, j’ajouterais des projets consommateurs aux validations. L’un utilise le modèle historique, l’autre active les nouvelles règles. Le changement peut demander des corrections de source chez les consommateurs même si le nom et les paramètres de la méthode restent identiques.

Les notes de version devraient présenter l’obligation ajoutée et, lorsque possible, la façade recommandée pour les appels courants.

## Vérifier le contrat à trois niveaux

### 1. Vérifier ce que le compilateur refuse

Un test de compilation négatif vérifie qu’un programme interdit est effectivement rejeté. Il protège ici contre une régression du contrat ou de sa configuration.

Je conserverais ces petits cas dans le dépôt.

| Cas | Résultat attendu avec les nouvelles règles |
| --- | --- |
| Appeler notre API `unsafe` sans contexte adapté | Diagnostic pour l’appel interdit |
| Déréférencer un pointeur dans une méthode seulement annotée `unsafe` | Diagnostic pour l’opération non entourée d’un contexte `unsafe` |
| Déclarer un membre `extern` sans choix de contrat | Diagnostic demandant `safe` ou `unsafe` |
| Appeler la façade depuis un projet avec `AllowUnsafeBlocks` à `false` | Compilation réussie |

La [documentation des diagnostics](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/compiler-messages/unsafe-code-errors) identifie notamment `CS9362` pour l’utilisation d’un membre marqué `unsafe` sans le contexte requis, et `CS9389` pour une déclaration `extern` sans choix explicite.

Le test devrait vérifier le diagnostic attendu. Une compilation qui échoue parce qu’une référence manque ne valide pas notre protection.

### 2. Vérifier le comportement sur des entrées valides

Pour notre calcul d’empreinte, j’utiliserais une implémentation de référence ou des vecteurs connus afin de comparer les résultats.

Les scénarios comprennent une entrée vide, un octet, plusieurs tailles, des octets nuls au milieu du tampon et la limite de taille acceptée. Si la façade impose une limite applicative, son dépassement doit être rejeté avant tout appel natif.

Une série de tests peut aussi générer des tampons valides de tailles et de contenus variés. L’objectif est de comparer les résultats et de vérifier que les entrées ne sont pas modifiées.

Ces tests devraient couvrir les systèmes et architectures réellement supportés. Une déclaration correcte sur une plateforme peut être incorrecte ailleurs si le contrat natif diffère.

### 3. Examiner les obligations impossibles à réduire à une assertion

La durée de vie du tampon, son exclusivité et la libération d’une ressource demandent aussi une revue de l’implémentation.

Dans notre exemple, je veux pouvoir relier chaque phrase du contrat à un mécanisme précis. La copie privée prend en charge l’exclusivité pendant l’appel. La longueur vient du même tableau. L’épinglage couvre la lecture native. La fonction ne conserve pas l’adresse.

Tester intentionnellement une adresse libérée ou une lecture hors limites peut terminer le processus. Ce type d’essai demande un environnement isolé et des outils adaptés au code natif. Il ne devrait pas être ajouté comme une simple assertion au processus habituel des tests unitaires.

Un ensemble de tests réussis ne constitue pas une preuve générale qu’un pointeur sera toujours valide.

## Une adoption progressive qui aide les revues

Je commencerais par inventorier les appels natifs, les méthodes manipulant des adresses et les abstractions qui exposent une ressource non managée.

Pour chaque frontière, l’équipe devrait pouvoir identifier son propriétaire, son contrat et la façon dont celui-ci est vérifié. Les obligations qui peuvent être prises en charge par une façade devraient l’être à cet endroit.

Les projets applicatifs peuvent ensuite activer les nouvelles règles tout en conservant `AllowUnsafeBlocks` à `false`. Le code qui assume les opérations sensibles reste regroupé dans des composants ciblés.

Si un agent propose d’ajouter `unsafe` pour réparer une compilation, je demanderais la même justification qu’à un développeur. Quelle obligation est satisfaite, par quel mécanisme et avec quelle validation ? Un commentaire qui répète simplement que l’opération est sûre n’apporte pas cette information.

Pour suivre les progrès, je regarderais la proportion des API sensibles documentées, les appels qui passent par une façade et la présence de tests de compilation sur les contrats publics. Le nombre de blocs `unsafe` pris isolément est moins utile. Plusieurs petites portées justifiées peuvent être plus faciles à examiner qu’un seul grand bloc.

## Conclusion

Les nouvelles règles de sécurité mémoire donnent à C# un moyen plus explicite de transmettre les obligations d’une API à ses appelants.

Le bénéfice pour une équipe se trouve dans la qualité des contrats et des revues. Les opérations sensibles deviennent repérables. Les façades peuvent prendre en charge les contraintes courantes. Les tests de compilation vérifient que ces frontières restent visibles pour les consommateurs.

L’adoption demande néanmoins de maîtriser la configuration de préversion, les versions des outils et les garanties réelles du code natif. Le compilateur peut exiger que l’appelant reconnaisse une obligation. Il appartient toujours à l’implémentation, à la documentation et aux validations de montrer comment elle est respectée.
