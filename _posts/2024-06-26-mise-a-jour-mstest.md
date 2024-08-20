---
title: Découvrez la nouvelle version de MSTest
date: 2024-06-26 19:00:00 -0400
categories: [outil-developpement]
tags: [dotnet, test]
---

Microsoft vient tout juste de sortir une nouvelle version de MSTest!

## Quoi de neuf ?

- **MSTest.Analyzers** : Plein de corrections de bugs et d'améliorations pour rendre vos analyses de code encore plus efficaces.
- **MSTest.Sdk** : De nouvelles fonctionnalités et des améliorations pour une meilleure performance et intégration des tests.
- **MSTest.Runner** : Vous pouvez maintenant tester vos applications WinUI avec MSTest. Génial, non ?

Pour plus de détails, consultez l'article officiel [ici](https://devblogs.microsoft.com/dotnet/introducing-mstest-34/).

## Les règles à ajouter

Si vous voulez tirer le meilleur parti des nouveaux analyseurs, pensez à ajouter ces règles à votre fichier `editorconfig` :

**Règles de structures** :

  - MSTEST0004 - PublicTypeShouldBeTestClassAnalyzer
  - MSTEST0006 - AvoidExpectedExceptionAttributeAnalyzer
  - MSTEST0015 - TestMethodShouldNotBeIgnored
  - MSTEST0016 - TestClassShouldHaveTestMethod
  - MSTEST0019 - PreferTestInitializeOverConstructorAnalyzer
  - MSTEST0021 - PreferDisposeOverTestCleanupAnalyzer
  - MSTEST0025 - PreferAssertFailOverAlwaysFalseConditionsAnalyzer

**Règles d'usages** :

  - MSTEST0002 - TestClassShouldBeValidAnalyzer
  - MSTEST0003 - TestMethodShouldBeValidAnalyzer
  - MSTEST0005 - TestContextShouldBeValidAnalyzer
  - MSTEST0007 - UseAttributeOnTestMethodAnalyzer
  - MSTEST0008 - TestInitializeShouldBeValidAnalyzer
  - MSTEST0009 - TestCleanupShouldBeValidAnalyzer
  - MSTEST0010 - ClassInitializeShouldBeValidAnalyzer
  - MSTEST0011 - ClassCleanupShouldBeValidAnalyzer
  - MSTEST0012 - AssemblyInitializeShouldBeValidAnalyzer
  - MSTEST0013 - AssemblyCleanupShouldBeValidAnalyzer
  - MSTEST0014 - DataRowShouldBeValidAnalyzer
  - MSTEST0017 - AssertionArgsShouldBePassedInCorrectOrder
  - MSTEST0023 - DoNotNegateBooleanAssertionAnalyzer
  - MSTEST0024 - DoNotStoreStaticTestContextAnalyzer
  - MSTEST0026 - AssertionArgsShouldAvoidConditionalAccessRuleId

## Configuration du timeout pour le cleanup des classes

Il est également possible de configurer un timeout pour le cleanup des classes. Pour cela, il suffit d'ajouter la configuration suivante dans votre fichier `.runsettings` :

```xml
<RunSettings>
  <MSTest>
    <ClassCleanupTimeout>1000</ClassCleanupTimeout>
  </MSTest>
</RunSettings>
```

Pour en savoir plus sur ces règles, jetez un œil à la documentation officielle [ici](https://learn.microsoft.com/en-ca/dotnet/core/testing/mstest-analyzers/usage-rules).
