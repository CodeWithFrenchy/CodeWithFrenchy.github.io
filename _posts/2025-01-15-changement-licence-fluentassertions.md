---
title: Transition de Fluent Assertions vers une licence commerciale
date: 2025-01-15 12:00:00 -0400
categories: [outil-developpement, bonnes-pratiques]
tags: [dotnet, fluentassertions, awesomeassertions, tests-unitaires, open-source, licence-commerciale]
---

[Fluent Assertions](https://fluentassertions.com/), une des bibliothèques .NET les plus populaires pour écrire des assertions fluides et expressives dans les tests unitaires, a récemment franchi une étape importante en [changeant de modèle de licence](https://github.com/fluentassertions/fluentassertions/pull/2943). Depuis la version 8.0.0, la bibliothèque est soumise à une [licence commerciale](https://x.com/ddoomen/status/1879138055484809616), ce qui signifie que les utilisateurs commerciaux doivent désormais acheter une licence pour continuer à l'utiliser.

## Une réaction communautaire : la naissance d'AwesomeAssertions

Ce changement a suscité des réactions mitigées dans la communauté des développeurs. Beaucoup ont exprimé leur désir de continuer à utiliser une solution open source. En réponse à ces besoins, un projet communautaire nommé **AwesomeAssertions** a été lancé. Ce projet vise à offrir une alternative open source, sous licence Apache 2.0, permettant de bénéficier des fonctionnalités clés de Fluent Assertions sans les contraintes liées à une licence commerciale.

## Maintenir Fluent Assertions sous licence open source

Pour les développeurs qui souhaitent continuer à utiliser Fluent Assertions sans adopter la nouvelle licence commerciale, il est possible de verrouiller la version à la 7.0.0, la dernière version disponible sous l'ancienne licence open source.

Voici comment procéder dans votre fichier de projet (.csproj) :

```xml
<PackageReference Include="FluentAssertions" Version="[7.0.0]" />
```

Cela garantit que votre projet reste sur la version 7.0.0, évitant ainsi toute mise à jour involontaire vers une version soumise à la nouvelle licence.

## Alternatives disponibles

L'écosystème .NET propose plusieurs autres bibliothèques et solutions pour écrire des assertions dans les tests unitaires :

- **Fluent Assertions 8.0.0 et versions ultérieures** : Pour ceux qui n'ont pas de problème avec le modèle commercial, ces versions offrent des fonctionnalités avancées, avec un support prioritaire et des garanties additionnelles.

- **[AwesomeAssertions](https://github.com/meenzen/AwesomeAssertions)** : Ce projet communautaire en cours de développement sous licence Apache 2.0 offre une alternative open source aux versions commerciales de Fluent Assertions.

- **[Shouldly](https://github.com/shouldly/shouldly)** : Une bibliothèque d'assertions connue pour ses messages d'erreur lisibles et informatifs en cas d'échec d'une assertion, tout en fournissant une syntaxe simple et intuitive.

- **[TUnit](https://github.com/thomhurst/TUnit)** : Un framework de test moderne et performant pour .NET 8 et versions ultérieures. Bien qu'il soit principalement conçu comme un framework de test complet, il inclut des fonctionnalités d'assertion.

- **Assertions natives** : Utiliser des assertions telles que `Assert.Equal` ou `Assert.True` directement via le framework de test par défaut.

## En savoir plus

Pour ceux qui souhaitent mieux comprendre le contexte, Nick Chapsas a publié une vidéo analysant ce changement et ses implications.

<iframe width="740" height="473" src="https://www.youtube.com/embed/ZFc6jcaM6Ms" title="Testing Entity Framework Core Correctly in .NET" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Conclusion

Le passage de Fluent Assertions à une licence commerciale marque un tournant dans l'écosystème .NET. Bien que cela puisse créer des contraintes pour certains utilisateurs, des alternatives comme `AwesomeAssertions` ou `Shouldly` offrent des solutions adaptées. Chaque équipe doit évaluer ses besoins en termes de licence, de support et de fonctionnalités pour choisir la meilleure option.

💡 Restez à l'affût, je devrais republier bientôt pour partager l'orientation que nous aurons choisie.
