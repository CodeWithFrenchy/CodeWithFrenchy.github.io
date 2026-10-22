---
title: Aide-mémoire - Code propre
date: 2025-01-14 20:00:00 -0400
categories: [bonnes-pratiques, architecture]
tags: [dotnet, csharp, clean-code, solid, qualite-code, maintenabilite, conventions-codage, refactoring]
---

L'objectif de tout développeur est de produire du code qui soit non seulement fonctionnel, mais aussi **propre**, facile à lire et à maintenir. Un code propre permet non seulement d'améliorer la collaboration entre les membres de l'équipe, mais aussi d'assurer la pérennité du projet à long terme. Chaque choix de conception que vous faites peut avoir un impact significatif sur la maintenabilité de votre code, c'est pourquoi il est crucial de bien comprendre ces principes pour élever la qualité de vos projets.

## Le code propre

Le code est propre s'il peut être compris facilement par toutes les personnes qui auront à travailler avec celui-ci. Un code propre peut être lu et amélioré aisément par un autre développeur que son auteur d'origine. Avec la compréhensibilité viennent la lisibilité, la possibilité de changement, l'extensibilité et la maintenabilité.

💡**Note** : L'importance est d'en prendre connaissance plus comme des orientations communes que comme des règles à respecter à tout prix.

## Règles générales

- Suivez les [conventions de codage](https://learn.microsoft.com/fr-ca/dotnet/csharp/fundamentals/coding-style/coding-conventions) C#
- Suivez les [conventions générales](https://learn.microsoft.com/fr-ca/dotnet/standard/design-guidelines/general-naming-conventions) d'affectation de noms
- Pensez [SOLID](https://fr.wikipedia.org/wiki/SOLID_%28informatique%29)
- Pensez KISS ([Keep it simple stupid](https://en.wikipedia.org/wiki/KISS_principle)). Plus simple est toujours mieux. Réduisez au maximum la complexité.
- Suivez-la règle du *boy-scouts* ([The Boy Scout Rule](https://blog.octo.com/tag/boy-scout-rule#:~:text=Martin%20nous%20pr%C3%A9sente%20un%20principe%2cvous%20l%27avez%20trouv%C3%A9%20%C2%BB.)). **Toujours laisser un endroit dans un état meilleur que celui où vous l'avez trouvé.**
- Cherchez toujours la cause première d'un problème.

## Règles de conception

- Conservez les données configurables à des niveaux élevés.
- Préférez le polymorphisme à if/else ou switch/case.
- Séparez le [code multithread](https://stackify.com/c-threading-and-multithreading-a-guide-with-examples/#:~:text=This%20comprehensive%20guide%20will%20dive%20deep%20into%20C%23%20threading%20and).
- Empêchez la surconfigurabilité ([over-configurability](https://en.wikipedia.org/wiki/Convention_over_configuration)).
- Utilisez l'[injection de dépendance](https://fr.wikipedia.org/wiki/Injection_de_d%C3%A9pendances).
- Suivez-la [loi de Déméter](https://fr.wikipedia.org/wiki/Loi_de_D%C3%A9m%C3%A9ter#:~:text=La%20loi%20de%20D%C3%A9m%C3%A9ter%20(en%20anglais%20Law%20of%20Demeter%20ou)). Une classe ne doit connaître que ses dépendances directes.

## Conseils de compréhension

- Être cohérent. Si vous faites quelque chose d'une certaine manière, suivez ce patron par la suite.
- Utilisez des variables explicatives.
- Encapsulez les conditions aux cas limites ([boundary conditions](https://www.simscale.com/docs/simwiki/numerics-background/what-are-boundary-conditions/)). Les conditions aux cas limites sont difficiles à suivre. Mettez le traitement pour eux en un seul endroit.
- Préférez les objets de valeur ([value objects](https://en.wikipedia.org/wiki/Value_object)) dédiés au type primitif.
- Évitez les dépendances logiques. N'écrivez pas de méthodes qui fonctionnent correctement en fonction de quelque chose d'autre dans la même classe.
- Évitez les conditions négatives.

## Conventions de nommage

- Choisissez des noms descriptifs et sans ambiguïté.
- Utilisez le français, sauf si les directives de la compagnie indiquent de coder en anglais. Il n'est cependant pas nécessaire de traduire des termes techniques spécifiques au framework ou un patron de conception tel qu'un "repository".
- Faire une distinction significative.
- Utilisez des noms prononçables.
- Utilisez des noms consultables.
- Remplacez les [nombres magiques](https://refactoring.guru/fr/replace-magic-number-with-symbolic-constant) par des constantes nommées.
- Évitez les encodages. N'ajoutez pas de préfixes ni d'informations de type.
- Évitez de mettre les acronymes tels que "XML" en majuscule, favorisez le format "Xml".

## Règles pour les fonctions

- Petites.
- Font qu'une chose.
- Utilisez des noms descriptifs.
- Préférez moins d'arguments.
- N'ont pas d'effets secondaires.
- N'utilisez pas d'arguments de type *flags*. Divisez la méthode en plusieurs méthodes indépendantes qui peuvent être appelées depuis le client sans l'indicateur.

## Règles pour les commentaires

- Essayez toujours de vous expliquer en code.
- Ne soyez pas redondant.
- N'ajoutez pas de bruit évident.
- N'utilisez pas d'accolade fermante pour commenter.
- Ne commentez pas le code. **Retirez-le simplement.**
- Utilisez comme explication l'intention.
- Utilisez comme clarification du code.
- Utilisez comme avertissement des conséquences.

## Structure du code source

- Séparez les concepts verticalement.
- Le code associé doit apparaître verticalement dense.
- Déclarez les variables proches de leur utilisation.
- Les fonctions dépendantes doivent être proches.
- Des fonctions similaires doivent être proches.
- Placez les fonctions dans le sens descendant.
- Gardez les lignes courtes.
- N'utilisez pas l'alignement horizontal.
- Utilisez un saut de ligne pour associer des éléments liés et dissocier des éléments faiblement liés.
- Ne cassez pas l'indentation.

## Objets et structures de données

- Masquez la structure interne.
- Préférez les structures de données.
- Évitez les structures hybrides (moitié objet et moitié données).
- Devrait être petit.
- Fait qu'une chose.
- Petit nombre de variables.
- La classe de base ne doit rien savoir de ses dérivés.
- Vaut mieux avoir plusieurs fonctions que de passer du code dans une fonction pour sélectionner un comportement.
- Préférez les méthodes non statiques aux méthodes statiques.

## Essais

- Une assertion logique par test
- Lisible
- Rapide
- Indépendant
- Répétable

## Code smells

- **Rigidité** - Le logiciel est difficile à changer. Un petit changement provoque une cascade de changements.
- **Fragilité** - Le logiciel tombe en panne à de nombreux endroits en raison d'un seul changement.
- **Immobilité** - Vous ne pouvez pas réutiliser des parties du code dans d'autres - projets en raison des risques impliqués et des efforts importants.
- Complexité inutile.
- Répétition inutile.
- Opacité. Le code est difficile à comprendre.

💡**Note** : Au besoin, référez-vous à cet [article complet](https://refactoring.guru/fr/refactoring/smells) pour plus de détails.

## Conclusion

En conclusion, adopter des pratiques de code propre n'est pas seulement une bonne pratique, mais une nécessité dans le développement logiciel moderne. En intégrant les principes évoqués dans cet article, vous favoriserez non seulement la lisibilité et la maintenabilité de votre code, mais vous contribuerez également à l'efficacité de votre équipe. Un code bien structuré permet de réduire les erreurs et de faciliter les mises à jour futures, tout en rendant l'intégration de nouveaux développeurs plus fluide. En fin de compte, investir dans la propreté du code est un investissement dans la durabilité de vos projets, garantissant ainsi leur succès à long terme. Rappelez-vous que la quête d'un code propre est un processus continu qui demande réflexion, pratique et engagement.

## Références

- <https://www.perforce.com/blog/sca/what-code-quality-overview-how-improve-code-quality>
- <https://gist.github.com/wojteklu/73c6914cc446146b8b533c0988cf8d29>
