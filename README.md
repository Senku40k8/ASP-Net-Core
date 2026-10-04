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

## Lancer en local
```
dotnet run --project WebApp
```
