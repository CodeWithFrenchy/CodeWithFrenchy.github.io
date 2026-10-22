---
title: L'importance d'un registre de décisions
date: 2024-11-06 20:00:00 -0400
categories: [architecture, bonnes-pratiques]
tags: [registre-decisions, adr, gouvernance, documentation, prise-de-decision, collaboration]
---

## Préambule

Je ne pensais pas publier un article aujourd'hui, mais je me suis laissé inspirer ! La majorité ne le sait pas, mais j'ai changé de mandat depuis peu. Je me retrouve maintenant dans un ministère où les ressources et les pratiques ne sont pas toutes en place pour affronter des projets d’envergure informatique. Dans ce genre de contexte, on réalise rapidement que certaines choses que l’on prend pour acquises sont loin d’être la norme ! Ce n’est pas par manque de volonté ; plusieurs personnes dans l’organisation occupent des rôles multiples, remplissant parfois les tâches d’une autre ressource en plus de leur propre mission.

Cette situation me rappelle une conférence de Roy Osherove, [Team Leadership in the Age of Agile](https://youtu.be/sSdMcqxkEdo?si=x4MAB1Vh155er8U5&t=2424), où il explique qu'une équipe constamment en mode "survie" peine à surmonter les problèmes quotidiens. Impossible donc, pour une équipe dans cette situation, d’améliorer ses processus par elle-même ; c’est pourquoi il mentionne souvent qu’une intervention externe est indispensable pour aider l’équipe à sortir de cette impasse.

Pour donner un peu de contexte, lors de mon dernier mandat, j'ai eu la chance d'être impliqué activement dans un groupe d'experts qui avait pour mission de définir des orientations, de développer et maintenir des outils communs, et surtout, de prévenir les problèmes avant même qu’ils n’apparaissent. Nous avons instauré, au fil du temps, divers processus, documentations et outils pour alléger les tâches des équipes de développement.

C'est dans cette même optique que j'ai voulu continuer cette pratique dans mon nouveau mandat en recommandant la mise en place d’un **registre de décisions** !

Mais pourquoi, allez-vous me demander ? C’est précisément ce que je veux explorer dans cet article 😉.

## Qu'est-ce qu'un registre de décisions ?

Un registre de décisions est un document centralisé (parfois même un outil) où sont consignées toutes les décisions importantes prises dans le cadre d'un projet, qu'elles soient techniques, stratégiques ou organisationnelles. Chaque décision y est décrite avec son contexte, les options envisagées, les raisons du choix final, ainsi que la date et les participants à la prise de décision.

Il permet de garder une trace claire de chaque orientation prise pour diverses raisons : justifications futures, onboarding des nouveaux membres de l’équipe ou, tout simplement, éviter de revenir sans cesse sur des sujets déjà tranchés.

Ce registre n’a pas besoin d’être complexe, mais il doit être à jour et facilement accessible par l'équipe pour être utile. On peut utiliser un simple document partagé ou opter pour des outils de gestion plus avancés si l’ampleur du projet le justifie. Dans mon précédent mandat, par exemple, notre registre était intégré dans un wiki dans Azure DevOps.

## Les gains

La mise en place d'un registre de décisions offre plusieurs avantages significatifs :

1. **Transparence et clarté** : Toutes les décisions sont visibles et documentées, ce qui réduit les interprétations ou les malentendus.

2. **Gain de temps** : Plus besoin de revisiter sans cesse les mêmes discussions ou de redébattre des choix déjà faits. L’équipe peut consulter le registre pour comprendre pourquoi une décision a été prise et se concentrer sur les prochaines étapes.

3. **Responsabilité et engagement** : En documentant chaque décision avec les noms des participants, chacun s’investit davantage dans le processus, sachant que son avis est pris en compte et consigné.

4. **Traçabilité** : Dans un contexte de révision, de retour en arrière ou même d’audit, le registre sert de référence pour comprendre la logique derrière chaque étape du projet.

5. **Support à la continuité** : En cas de changement de personnel, le registre facilite la transition et l’onboarding des nouveaux arrivants en leur offrant un historique clair.

## Un exemple

Prenons un exemple du monde .NET. Vous avez peut-être remarqué que certaines méthodes portent le suffixe `Async`.

La question s'est posée : est-ce systématique de l'ajouter ?

Voici une décision prise pour clarifier cette pratique :

**2024-11-06 - Nomenclature des méthodes asynchrones**

- **Contexte :** Devons-nous systématiquement ajouter le suffixe `Async` aux méthodes asynchrones ?
- **Décision :** Comme toutes nos méthodes sont asynchrones, le suffixe `Async` devient redondant, donc il n’est pas recommandé de l’ajouter systématiquement. Nous suivons l’orientation préconisée par [MediatR](https://github.com/jbogard/MediatR/blob/master/src/MediatR/IPublisher.cs). Cependant, si une interface doit proposer des versions synchrone et asynchrone d’une même méthode, le suffixe `Async` est recommandé pour identifier clairement la méthode asynchrone. 
- **Conséquences :** Aucun impact notable dans ce cas-ci.

## Conclusion

Un registre de décisions est bien plus qu'un simple document administratif. Dans les environnements complexes et les projets d'envergure, il devient un outil essentiel de transparence, de cohérence et d'efficacité pour toute l’équipe. Il permet non seulement de gagner du temps, mais également de créer un espace de confiance où chacun peut retrouver les raisons et le contexte des choix passés, sans devoir constamment revenir en arrière. 

En documentant nos décisions, nous créons un cadre de référence clair et solide pour éviter les erreurs de répétition, faciliter l’intégration des nouveaux membres et renforcer la continuité dans les projets. En fin de compte, ce petit investissement de temps pour maintenir un registre peut faire toute la différence dans la gestion d’un projet réussi.

Alors, pourquoi ne pas intégrer un registre de décisions dès aujourd'hui dans vos pratiques ?
