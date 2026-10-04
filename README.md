# ASP-Net-Core

Application web ASP.NET Core et son pipeline de build/déploiement.
L'infrastructure (la VM cible) est dans un dépôt séparé : [Dev-Infra](https://github.com/Senku40k8/Dev-Infra).

## Contenu

| Fichier | Rôle |
|---|---|
| `ASP-Net-Core.slnx` | Solution Visual Studio |
| `WebApp/` | Application web (gabarit Razor Pages par défaut) |
| `azure-pipelines-app.yml` | Pipeline de compilation et de déploiement |

## Choix techniques

### Structure
- Gabarit par défaut « Application web ASP.NET Core » (Razor Pages), sans modification : l'objectif est de valider la chaîne CI/CD, pas le contenu de l'application.
- .NET 10, version du SDK installée sur le poste. Solution au nouveau format `.slnx`.
- `.gitignore` .NET standard : `bin/` et `obj/` ne sont pas versionnés, le pipeline recompile tout.
- Les bibliothèques front-end (Bootstrap, jQuery) sont versionnées dans `wwwroot/lib` comme le fait le gabarit, pour ne pas dépendre d'un gestionnaire de paquets pendant le build.

### Pourquoi un dépôt séparé de l'infrastructure
Le code change souvent, la VM rarement. Avec deux dépôts et deux pipelines, un commit sur l'application ne relance pas le provisionnement. Ce pipeline ne se déclenche que sur `WebApp/*` ou sur son propre YAML.

### Pipeline
1. `UseDotNet@2` installe le SDK `10.x` : il doit correspondre au `TargetFramework` du projet (`net10.0`), sinon le build échoue (NETSDK1045).
2. `dotnet publish` en `Release` vers le dossier d'artefacts.
3. `PublishBuildArtifacts@1` publie le résultat sous le nom `webAppArtifact`, ce qui garde une trace de chaque build.
4. Déploiement sur la VM : étape présente mais pas encore implémentée (copie prévue en SSH/SCP vers l'IP renvoyée par le déploiement Dev-Infra).

Le pipeline utilise le même agent local que Dev-Infra (pool `Default`). Les scripts `script:` y sont exécutés par `cmd.exe`.

### Gestion des secrets
- Aucun secret dans le dépôt. `appsettings.json` ne contient que la configuration de logs.
- Le futur déploiement SSH utilisera une variable secrète ou un fichier sécurisé (Secure Files) d'Azure DevOps pour l'accès à la VM, jamais une valeur en clair dans le YAML.

## Lancer en local
```
dotnet run --project WebApp
```
