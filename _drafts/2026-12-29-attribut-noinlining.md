---
title: Empêcher l’inlining en .NET - l’attribut MethodImplOptions.NoInlining expliqué
date: 2026-12-29 20:0:00 -0400
categories: []
tags: [dotnet]
---

## Introduction

L’inspiration m’est venue suite à la lecture de l'article [de Stephen Toub sur les améliorations de performance en .NET 10](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-10/). Plusieurs optimisations du JIT y sont abordées, dont certaines liées à l’inlining de méthodes. Cela m’a donné envie de revenir sur un attribut peu connu mais très utile `[MethodImpl(MethodImplOptions.NoInlining)]`.

## Qu’est-ce que l’inlining et l’attribut _NoInlining_ ?

En .NET, **l’inlining** (ou _inline expansion_) est une optimisation du _just-in-time_ compiler (JIT) qui remplace un appel de méthode par le corps de cette méthode, évitant ainsi le surcoût d’un appel de fonction. Par exemple, si l’on a une méthode simple `int Addition(int x, int y) { return x + y; }` et qu’on l’appelle par `var z = Addition(a, b)`, le JIT peut décider d’**inliner** cet appel, c’est-à-dire le transformer en `var z = a + b;` directement dans le code appelant. Cette substitution est transparente fonctionnellement et vise à améliorer les performances d’exécution.

L’attribut C# `[MethodImpl(MethodImplOptions.NoInlining)]` sert précisément à **empêcher cette optimisation d’inlining** pour une méthode donnée. En appliquant cet attribut (défini dans `System.Runtime.CompilerServices`), on informe le runtime .NET que la méthode ne doit _jamais_ être incorporée dans ses appelants. Concrètement, le JIT **ne remplacera pas l’appel par le code interne** de la méthode marquée `NoInlining`, même si celle-ci est courte ou fréquemment utilisée. La méthode sera donc toujours appelée "normalement", avec sa propre frame de pile d’exécution.

## Pourquoi voudrait-on empêcher l’inlining d’une méthode ?

On pourrait penser que l’_inlining_ est toujours bénéfique, mais il existe plusieurs situations où un développeur expérimenté peut choisir de le désactiver :

- **Pour améliorer la traçabilité et le debug** : En mode _Release_, le JIT effectue des optimisations qui peuvent compliquer le diagnostic. L’_inlining_ en fait partie : une méthode inlinée **n’apparaîtra plus dans la pile d’appel (stack trace)** lors d’une exception ou d’une trace d’exécution. Par exemple, en `Debug` la pile listant une suite d’appels affichera toutes les méthodes, mais en `Release` une méthode qui a été _inline_ peut disparaître du `StackTrace`. Cela rend l’analyse des logs ou des exceptions plus difficile, car on ne voit pas cette étape intermédiaire. En marquant explicitement une méthode avec `NoInlining`, on s’assure qu’elle **restera visible comme une étape distincte** dans la _stack trace_, même en `Release`. Ceci facilite le _debugging_ et le support en production (où le code est optimisé) en préservant une séparation claire des appels dans les traces.
- **Pour un meilleur profilage et instrumentation** : De manière similaire, les outils de profilage ou de `tracing` (_Application Performance Monitoring_) peuvent passer à côté des méthodes inlinées. Puisqu’une méthode _inline_ n’a plus de point d’entrée/sortie propre à surveiller, un profiler peut ne pas la comptabiliser séparément. Par exemple, [New Relic](https://newrelic.com/fr) indique que si une méthode annotée pour le suivi de transaction n’apparaît pas dans les rapports, il faut _désactiver son inlining_ en lui ajoutant `[MethodImpl(MethodImplOptions.NoInlining)]`. Cela garantit que chaque appel sera bien tracé individuellement. Bref, **pour toute méthode critique dont on souhaite mesurer le temps d’exécution ou la fréquence d’appel de façon précise**, désactiver l’_inlining_ est utile afin qu’elle soit visible dans les résultats de profilage. (Notons que certains profileurs offrent aussi une option globale pour désactiver l’_inlining_ lors des mesures.)
- **Pour des raisons fonctionnelles ou de correctivité** : Plus rarement, le comportement de certaines méthodes peut dépendre de la présence d’un vrai cadre d’appel. C’est le cas des méthodes qui examinent la pile d’exécution ou l’assembly appelant. Un exemple notable est la méthode .NET `Assembly.GetCallingAssembly()`, celle-ci est marquée en interne avec `MethodImplOptions.NoInlining` afin de retourner l’assembly de son appelant réel. Si elle était inlinée, le _caller_ direct disparaîtrait et le résultat serait incorrect. De même, si vous écrivez une méthode utilitaire qui utilise `StackTrace` ou `MethodBase.GetCurrentMethod()` pour enregistrer le contexte d’appel, vous devez empêcher son _inlining_ pour qu’elle capture le bon niveau d’appel. En résumé, **dès qu’une logique dépend de la structure de la pile d’appels, il est nécessaire d’empêcher l’inlining** pour garantir le fonctionnement attendu.
- **Pour des besoins de tests de performance spécifiques** : Dans certains cas de micro-benchmarking, on souhaite isoler le coût exact d’un appel de méthode. Or, le JIT pourrait optimiser agressivement et _inline_ une petite méthode au point d’en éliminer quasiment le coût d’appel. En marquant la méthode `NoInlining`, on force le runtime à conserver l’appel, ce qui permet de mesurer par exemple le coût d’une fonction en elle-même, ou au contraire d’éviter qu’une fonction vide ne soit complètement optimisée. Ce cas d’usage est surtout pertinent dans des contextes de tests ou d’analyse fine du comportement du JIT.

**En pratique, le JIT .NET décide automatiquement s’il doit inliner une méthode en fonction de nombreux critères** (taille du code IL, complexité du contrôle de flux, présence d’exceptions, type des arguments, etc.). On fait généralement confiance à ces heuristiques du runtime pour obtenir le meilleur équilibre performance/taille du code. Un développeur n’a donc pas besoin d’annoter chaque petite méthode avec _NoInlining_ ou _AggressiveInlining_, au contraire, ces attributs ne s’emploient qu’avec parcimonie lorsqu’on a une raison particulière de forcer ou d’empêcher l’inlining. En effet, **inhiber l’inlining a des contreparties** qu’il faut connaître.

## Impact sur les performances et autres implications

Empêcher l’_inlining_ d’une méthode aura logiquement un **impact sur les performances** de l’application. Dans la plupart des cas, l’_inlining_ améliore les performances en évitant le coût d’un appel de fonction (allocation d’une stack frame, sauts, etc.) et en offrant potentiellement au JIT plus d’opportunités d’optimisation globale (par exemple, propagation de constantes ou élimination de code mort à travers l’appel). Forcer une méthode à ne pas être _inline_ signifie que chaque appel continuera d’entraîner ce petit surcoût. Sur un très grand nombre d’appels, cela peut se traduire par une exécution un peu moins efficace qu’elle aurait pu l’être. Les documentations de _profiling_ notent d’ailleurs que **désactiver l’inlining peut ralentir l’application**, puisque normalement l’_inlining_ **accélère les méthodes appelantes** en éliminant le coût d’appel.

À l’inverse, empêcher l’_inlining_ peut **réduire la taille du code natif généré** dans certains cas, en évitant de dupliquer le corps d’une méthode à chaque site d’appel. Cependant, le JIT prend déjà en compte ce facteur : il évite d’inliner les méthodes trop volumineuses pour ne pas faire exploser la taille du code. Autrement dit, si le JIT juge qu’une fonction est assez petite et critique pour être inlinée, il y a de fortes chances que ce soit effectivement bénéfique de l’inliner. **Utiliser NoInlining sans raison valable peut donc dégrader les performances globales** de votre application.

En ce qui concerne le **débogage et le profilage**, l’impact est plutôt positif du point de vue de la visibilité du code (comme discuté plus haut). En mode `Debug`, le compilateur JIT désactive de lui-même la plupart des optimisations (y compris l’_inlining_) pour faciliter le debug pas-à-pas. Mais en mode `Release`, si vous avez besoin d’investiguer un problème complexe, vous pourriez temporairement ajouter `NoInlining` à certaines fonctions pour mieux les isoler dans les traces. De même, pour un profilage précis, sacrifier un peu de performance en désactivant l’_inlining_ sur quelques méthodes peut être un compromis utile afin de **collecter des métriques plus fines**. Il faut simplement garder à l’esprit que ces modifications peuvent influencer légèrement les _timings_ mesurés (puisque on réintroduit le coût d’appel), mais au moins les méthodes d’intérêt apparaîtront explicitement dans les résultats du profiler.

En résumé, **n’utilisez `[MethodImpl(NoInlining)]` que lorsque c’est nécessaire**. Le gain de contrôle ou de clarté obtenu (sur la stack trace, le profilage, la logique métier, etc.) doit justifier la perte potentielle de performance. Pour un composant critique, mesurez toujours l’impact d’un tel attribut afin de confirmer qu’il apporte le bénéfice escompté.

## Exemple d’utilisation de MethodImplOptions.NoInlining

Pour illustrer un cas d’usage, considérons un scénario de monitoring des performances. Supposons que nous ayons une méthode de traitement de données critiques et que nous voulions mesurer précisément son temps d’exécution, ainsi que la faire apparaître distinctement dans nos traces de log. Nous pouvons la marquer avec `[MethodImpl(MethodImplOptions.NoInlining)]` pour garantir qu’elle ne sera pas intégrée au code appelant par le JIT. Cela permet d’obtenir une mesure plus fiable et d’isoler son coût dans un profiler ou une trace.

```csharp
using System;
using System.Diagnostics;
using System.Runtime.CompilerServices;

public class DataService 
{
    [MethodImpl(MethodImplOptions.NoInlining)]
    public void TraiterDonnees(Data donnees)
    {
        // Logique de traitement des données...
        Console.WriteLine("Traitement des données en cours...");

        // Par exemple, on simule du travail
        System.Threading.Thread.Sleep(100); 
    }
}

public class Program 
{
    static void Main()
    {
        var service = new DataService();
        var chrono = Stopwatch.StartNew();

        service.TraiterDonnees(new Data());
        
        chrono.Stop();
        Console.WriteLine($"Traitement terminé en {chrono.ElapsedMilliseconds} ms");
    }
}
```

Dans cet exemple, l’attribut _NoInlining_ appliqué à `TraiterDonnees` assure que le chronomètre mesure bien l’appel complet de la méthode (même si elle était très courte) et que le profiler verra une entrée séparée pour `DataService.TraiterDonnees`. De plus, si une exception était lancée à l’intérieur, la _stack trace_ inclurait clairement cette méthode, ce qui peut aider au débogage en production. Un développeur pourrait utiliser ce schéma pour des méthodes dont il souhaite analyser précisément le comportement ou éviter toute optimisation agressive qui compliquerait l’observation du programme.

## Autres options courantes de MethodImplOptions

L’attribut `[MethodImpl]` sert de manière générale à configurer la façon dont une méthode est gérée par le runtime. Outre `NoInlining`, l’énumération **MethodImplOptions** propose d’autres indicateurs utiles :

- **AggressiveInlining** – Indique au JIT qu’il _devrait_ inliner la méthode si possible (c’est l’inverse de `NoInlining`). Cet attribut est une suggestion forte pour optimiser le code, souvent utilisée pour les très petites méthodes critiques.
  > ⚠️ **Attention** : Microsoft prévient que l’utiliser à tort et à travers peut nuire aux performances, en entraînant par exemple une explosion du code ou en dépassant certaines limites d’optimisation du JIT. Il faut donc l’employer judicieusement et mesurer son effet.
- **NoOptimization** – Demande au JIT de _ne pas optimiser_ la méthode du tout. Cela empêche non seulement l’_inlining_, mais aussi les autres optimisations JIT. C’est généralement utilisé dans des scénarios de _debugging_ approfondi ou de contournement de bugs du compilateur JIT. En mode `Debug` classique, ce flag est souvent implicite, mais on peut l’appliquer manuellement si besoin d’isoler une méthode en `Release` pour enquête (par exemple, traquer un problème de codegen).
- **Synchronized** – Indique que la méthode est _moniteur_ (synchronisée) : le runtime va acquérir un verrou (`lock`) automatique sur l’instance courante ou sur la classe (pour une méthode `static`) à chaque appel. Cela garantit qu’un seul `thread` exécute la méthode à la fois. Néanmoins, cet usage est déconseillé pour les API publiques, car il peut causer des problèmes de _deadlocks_ si du code externe prend aussi des verrous sur l’objet ou le type en question. On lui préfère en général des mécanismes de lock explicites gérés par l’application.

Parmi les autres flags moins courants, citons _AggressiveOptimization_ (introduit plus récemment pour signaler que la méthode doit toujours être fortement optimisée, en contournant par exemple l’exécution _tiered_), ou _PreserveSig_ (utilisé en interop pour conserver la signature d’origine). Toutefois, la plupart des développeurs n’auront à manipuler que **NoInlining** et éventuellement **AggressiveInlining**, ces deux-là étant les plus liés aux performances d’exécution.

En conclusion, `[MethodImpl(MethodImplOptions.NoInlining)]` est un outil pour garder le contrôle sur l’optimisation d’une méthode** dans l’environnement .NET. Il permet de _désactiver l’inlining_ lorsque celui-ci serait nuisible à la lisibilité du code lors du _debugging_, à la fiabilité de certaines fonctionnalités ou à la précision du profilage. Bien qu’il faille l’utiliser avec parcimonie en raison de son impact sur les performances, cet attribut fait partie des connaissances avancées qu’un développeur C# expérimenté peut exploiter pour affiner le comportement du _runtime_. En comprenant pourquoi et comment le JIT _inline_ les méthodes, on est mieux armé pour décider quand il est judicieux de le prévenir et ainsi écrire du code à la fois efficace **et** maintenable du point de vue diagnostic.
