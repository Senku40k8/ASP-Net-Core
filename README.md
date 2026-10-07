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
- Les bibliothèques front-end (Bootstrap, jQuery, jQuery Validation) sont versionnées dans `wwwroot/lib` comme le fait le gabarit, pour ne pas dépendre d'un gestionnaire de paquets ni d'un CDN. Seuls les fichiers réellement chargés par les pages sont conservés (versions minifiées + licences) : les sources non minifiées, les cartes `.map`, les variantes RTL/ESM et les feuilles partielles du gabarit ont été retirées (~9 Mo de moins dans le dépôt et dans l'artefact déployé).

### Pourquoi un dépôt séparé de l'infrastructure
Le code change souvent, la VM rarement. Avec deux dépôts et deux pipelines, un commit sur l'application ne relance pas le provisionnement. Ce pipeline ne se déclenche que sur `WebApp/*` ou sur son propre YAML. Il suppose que l'infrastructure existe déjà, et s'arrête avec un message clair si le pipeline Dev-Infra n'a pas encore tourné.

### Pipeline
Deux stages, exécutés sur le même agent local que Dev-Infra (pool `Default`, machine Windows) :

**Build**
1. `UseDotNet@2` installe le SDK `10.x` : il doit correspondre au `TargetFramework` du projet (`net10.0`), sinon le build échoue (NETSDK1045).
2. `dotnet publish` en `Release` pour `linux-x64`, en mode self-contained : le runtime .NET est inclus, rien à installer sur la VM.
3. `PublishBuildArtifacts@1` publie le résultat sous le nom `webAppArtifact`.

**Deploy** (seulement si le Build réussit)
1. `DownloadBuildArtifacts@1` récupère `webAppArtifact`. Pas de checkout du code : on déploie exactement ce qui a été construit.
2. L'artefact est compressé et envoyé dans le compte de stockage créé par Dev-Infra.
3. `az vm run-command` exécute un script sur la VM. Ce script télécharge l'archive via un lien SAS, l'installe dans `/opt/webapp` et la lance comme service systemd (`webapp.service`) sur le port 80.
4. L'archive est supprimée du stockage, puis le pipeline vérifie que le site répond en HTTP sur l'IP publique.

Pourquoi `run-command` plutôt que SSH/SCP : l'agent Windows n'a pas de moyen simple de passer un mot de passe à SSH. Une clé SSH aurait été un secret de plus à gérer, et il aurait fallu ouvrir le port 22. Ici, tout passe par la service connection Azure déjà utilisée par Dev-Infra.

L'application tourne sous un utilisateur système `webapp` sans shell. Elle obtient seulement la capacité d'écouter sur le port 80 (`CAP_NET_BIND_SERVICE`), pas les droits root. Le service redémarre automatiquement en cas d'arrêt ou de reboot.

### Gestion des secrets
- Aucun secret dans le dépôt. `appsettings.json` ne contient que la configuration de logs.
- Accès à Azure via la service connection `Azure-8CLD201` (fédération d'identité, aucun secret stocké), limitée au groupe de ressources `MonGroupeDeRessources`.
- La clé du compte de stockage est lue au moment du déploiement, jamais écrite, et masquée dans les logs (`task.setsecret`).
- Le lien SAS donné à la VM est en lecture seule, limité à un seul fichier, HTTPS uniquement, et expire après 30 minutes. Il est lui aussi masqué dans les logs.

## Consignes couvertes
| Consigne | Où |
|---|---|
| Provisionner la VM | Pipeline du dépôt Dev-Infra |
| Déployer l'application sur la VM avec l'artefact | Stage Deploy de ce pipeline (`webAppArtifact`) |
| Agent local | Pool `Default` (agent auto-hébergé) pour les deux pipelines |

## Captures d'écran
Les captures sont réparties entre les deux dépôts :

| Dépôt | Fichier | Contenu |
|---|---|---|
| ASP-Net-Core | `Capture d'écran 2026-10-03 213243.png` | Site en ligne sur la VM, URL `http://74.241.244.205` visible (Brave) |
| Dev-Infra | `Capture d'écran du site web.png` | Même site, URL visible (Edge) |
| Dev-Infra | `Capture d'écran des branches.png` | Historique du pipeline ASP-Net-Core sur `main`, dernier run (déploiement) réussi |

## Sources

**JavaScript / front-end**
- Bootstrap 5 (CSS + `bootstrap.bundle.min.js`, inclut Popper pour le menu repliable) : https://getbootstrap.com/docs/5.3/getting-started/introduction/
- jQuery : https://api.jquery.com/
- jQuery Validation et validation « unobtrusive » utilisées par `_ValidationScriptsPartial.cshtml` : https://jqueryvalidation.org/documentation/ , https://github.com/aspnet/jquery-validation-unobtrusive , https://learn.microsoft.com/aspnet/core/mvc/models/validation#client-side-validation
- Gestion des bibliothèques côté client dans ASP.NET Core (`wwwroot/lib`, LibMan) : https://learn.microsoft.com/aspnet/core/client-side/libman/
- Fichiers statiques et `MapStaticAssets` / `asp-append-version` : https://learn.microsoft.com/aspnet/core/fundamentals/map-static-files , https://learn.microsoft.com/aspnet/core/fundamentals/static-files

**Application .NET**
- Razor Pages : https://learn.microsoft.com/aspnet/core/razor-pages/
- Fichiers `.slnx` : https://learn.microsoft.com/visualstudio/ide/reference/solution-file
- `dotnet publish`, identifiants de runtime (`linux-x64`) et déploiement self-contained : https://learn.microsoft.com/dotnet/core/tools/dotnet-publish , https://learn.microsoft.com/dotnet/core/rid-catalog , https://learn.microsoft.com/dotnet/core/deploying/
- Erreur NETSDK1045 (SDK trop ancien pour le `TargetFramework`) : https://learn.microsoft.com/dotnet/core/tools/sdk-errors/netsdk1045
- Hébergement sous Linux avec systemd et variables `ASPNETCORE_URLS` / `ASPNETCORE_ENVIRONMENT` : https://learn.microsoft.com/aspnet/core/host-and-deploy/linux-nginx , https://learn.microsoft.com/aspnet/core/fundamentals/environments
- Dépendance ICU sous Linux (`libicu`) : https://learn.microsoft.com/dotnet/core/install/linux-ubuntu , https://learn.microsoft.com/dotnet/core/runtime-config/globalization

**Pipeline Azure DevOps**
- Syntaxe YAML (stages, `trigger.paths`, `condition`, `checkout: none`) : https://learn.microsoft.com/azure/devops/pipelines/yaml-schema/
- Agents auto-hébergés Windows : https://learn.microsoft.com/azure/devops/pipelines/agents/windows-agent
- Tâches `UseDotNet@2`, `PublishBuildArtifacts@1`, `DownloadBuildArtifacts@1`, `AzureCLI@2` : https://learn.microsoft.com/azure/devops/pipelines/tasks/reference/
- Masquage d'un secret dans les logs (`##vso[task.setsecret]`) : https://learn.microsoft.com/azure/devops/pipelines/scripts/logging-commands#setsecret-register-a-value-as-a-secret
- Service connection Azure avec fédération d'identité de charge de travail : https://learn.microsoft.com/azure/devops/pipelines/library/connect-to-azure

**Azure CLI / VM**
- `az vm run-command invoke` : https://learn.microsoft.com/cli/azure/vm/run-command , https://learn.microsoft.com/azure/virtual-machines/linux/run-command
- `az storage blob upload` / `generate-sas` / `delete` : https://learn.microsoft.com/cli/azure/storage/blob
- Bonnes pratiques des SAS (durée courte, lecture seule, HTTPS) : https://learn.microsoft.com/azure/storage/common/storage-sas-overview

**Linux / systemd**
- Fichiers d'unité de service (`Restart`, `User`, `WorkingDirectory`) : https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html , https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html
- `AmbientCapabilities` et `CAP_NET_BIND_SERVICE` (port 80 sans root) : https://man7.org/linux/man-pages/man7/capabilities.7.html

## Lancer en local
```
dotnet run --project WebApp
```
