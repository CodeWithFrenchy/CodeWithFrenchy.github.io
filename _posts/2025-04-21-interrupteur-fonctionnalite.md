---
title: Interrupteurs de fonctionnalité
date: 2025-04-21 18:00:00 -0400
categories: [outil-developpement]
tags: [dotnet]
---

## Qu'est-ce qu'un interrupteur de fonctionnalité

Les applications cloud, les microservices et les pratiques DevOps misent sur la vitesse et l’agilité. Les utilisateurs attendent de la réactivité, des nouveautés fréquentes et peu (voire aucun) temps d'arrêt.

Les **interrupteurs de fonctionnalité** sont une technique moderne de déploiement permettant de découpler l’activation d’une fonctionnalité de son déploiement. Une simple modification de configuration suffit à activer une fonctionnalité pour certains utilisateurs ou environnements, sans redémarrer ni redéployer l'application (_en théorie_).

> ⚡ Ils permettent de **dissocier le déploiement de la mise en production**.

Cela permet notamment de tester des fonctionnalités en production sur des groupes restreints d’utilisateurs ou d’activer graduellement une nouveauté.

## Mise en place dans une application .NET

### Installation des packages NuGet

- Pour tout projet :  
  ```bash
  dotnet add package Microsoft.FeatureManagement
  ```

- Pour les APIs :  
  ```bash
  dotnet add package Microsoft.FeatureManagement.AspNetCore
  ```

### Configuration minimale

```csharp
using Microsoft.FeatureManagement;

public class Program 
{
    // ...

    builder.Services.AddFeatureManagement();
}
```

### Utilisation du service `IFeatureManager`

Une fois la configuration en place, vous pouvez interroger facilement un interrupteur de fonctionnalité comme illustré dans l’exemple suivant :

```csharp
public class MonService
{
    private readonly IFeatureManager _featureManager;

    public MonService(IFeatureManager featureManager)
    {
        _featureManager = featureManager;
    }

    public Task<string> ObtenirMessage() => _featureManager.IsEnabledAsync("NouvelleFonctionnalite")
            ? "Bienvenue avec la nouvelle fonctionnalité"
            : "Bienvenue";
}
```

## Sources de configuration

### Fichier `appsettings.json`

```json
{
  "FeatureManagement": {
    "NouvelleFonctionnalite": true
  }
}
```

Ou via une section personnalisée :

```json
{
  "MaSectionFlags": {
    "NouvelleFonctionnalite": true
  }
}
```

```csharp
services.AddFeatureManagement(Configuration.GetSection("MaSectionFlags"));
```

### Variables d’environnement

```bash
FeatureManagement__NouvelleFonctionnalite=true
```

> ✅ Deux underscores requis entre la section et la clé.

### Azure App Configuration

- NuGet :
  ```bash
  dotnet add package Microsoft.Extensions.Configuration.AzureAppConfiguration
  ```

- Configuration :

```csharp
config.AddAzureAppConfiguration(options =>
{
    options.Connect(Environment.GetEnvironmentVariable("ConnectionString"))
           .ConfigureRefresh(refresh =>
               refresh.Register("Settings:Sentinel", refreshAll: true)
                      .SetCacheExpiration(TimeSpan.FromMinutes(1)))
           .UseFeatureFlags();
});
```

## Cas d'utilisation

### Activation conditionnelle d'un contrôleur ou endpoint

```csharp
[FeatureGate(FeatureFlags.NouvelleFonctionnalite)]
[ApiController]
[Route("api/[controller]")]
public class ExempleController : ControllerBase
{
    [HttpGet]
    [FeatureGate(FeatureFlags.NouvelleFonctionnalite)]
    public IActionResult Get() => Ok("Accès autorisé");
}
```

### Composant Blazor (WebAssembly)

```csharp
@if (estFonctionnaliteActive)
{
    <div>La fonctionnalité est active.</div>
}
```

```csharp
[Inject] 
public required IFeatureManager FeatureManager { get; set; }

private bool estFonctionnaliteActive;

protected override async Task OnInitializedAsync()
{
    estFonctionnaliteActive = await FeatureManager.IsEnabledAsync(nameof(FeatureFlags.NouvelleFonctionnalite));

    // ...
}
```

### Redirection conditionnelle (ex. : vers une page 404)

```razor
<NotFound>
    <LayoutView Layout="@typeof(MainLayout)">
        <PageNonTrouvee />
    </LayoutView>
</NotFound>
```

## Autres possibilités

Les interrupteurs ne se limitent pas à un simple état activé ou désactivé. Grâce aux filtres, il est possible d’introduire des comportements plus dynamiques et contextuels.

- [Interrupteurs avancés](https://andrewlock.net/creating-dynamic-feature-flags-with-feature-filters-adding-feature-flags-to-an-asp-net-core-app-part-3/) - Microsoft permet d’ajouter des filtres personnalisés comme une activation basée sur un utilisateur, une région, etc.
- [Interrupteurs personnalisés](https://andrewlock.net/creating-a-custom-feature-filter-adding-feature-flags-to-an-asp-net-core-app-part-4/) - Vous pouvez créer vos propres filtres.

## Bonnes pratiques

### Utiliser un enum pour éviter les chaînes magiques

```csharp
public enum FeatureFlags
{
    NouvelleFonctionnalite,
    FonctionnaliteB
}

await _featureManager.IsEnabledAsync(nameof(FeatureFlags.NouvelleFonctionnalite));
```

### Faire le ménage régulièrement des interrupteurs

Les interrupteurs de fonctionnalité sont par nature temporaires. Une fois qu’une fonctionnalité est pleinement déployée ou qu’un test A/B est terminé, **il est essentiel de retirer les interrupteurs associés du code et de la configuration**. 

Laisser des interrupteurs inactifs ou non utilisés peut nuire à la lisibilité, alourdir la maintenance, et introduire de l'incertitude dans les comportements attendus de l'application.

> 💡 **Astuce** : tenez une liste des interrupteurs actifs avec leur date de création et l'objectif visé. Cela vous aidera à planifier leur retrait à temps.

### Comportement personnalisé lorsqu’une action est désactivée

Par défaut, un contrôleur ou action désactivé retourne un HTTP 404.  

Vous pouvez créer une classe personnalisée qui implémente `IDisabledFeaturesHandler` pour gérer un autre type de réponse.

> 👉 Voir la [documentation](https://github.com/microsoft/FeatureManagement-Dotnet#disabled-action-handling).

## Conclusion

Les interrupteurs de fonctionnalité sont un levier puissant pour améliorer l’agilité des équipes de développement, réduire les risques liés aux déploiements et faciliter l’expérimentation de nouvelles fonctionnalités. Leur intégration dans une application .NET est relativement simple grâce au support natif offert par Microsoft.FeatureManagement.

Cependant, comme tout outil, leur utilisation doit être encadrée par des bonnes pratiques : éviter l’accumulation d’interrupteurs obsolètes, documenter leur usage, et réfléchir à une stratégie de configuration adaptée à l’échelle du projet (locale, via App Configuration, ou centralisée).

Adopter les interrupteurs de fonctionnalité, c’est se donner les moyens de livrer plus rapidement, de manière plus sûre, et avec une meilleure maîtrise du changement. Commencez petit, expérimentez, et adaptez votre approche à vos besoins réels. L’essentiel est de rester intentionnel dans l’usage que vous en faites.
