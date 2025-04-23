# The KuboCD CLI

## kubocd pack

Permet de créer un package KuboCD à partir d'un manifest et de la stocker dans un repository d'image OCI.

Se reporter au chapitre [A first deployment/Package build](130-a-first-deployment.md/#package-build) pour un exemple d'utilisation

```
Usage:
  kubocd package <Package manifest> [flags]

Aliases:
  package, pack, build

Flags:
  -h, --help                   help for package
  -r, --ociRepoPrefix string   OCI repository prefix (i.e 'quay.io/your-organization/packages'). 
                               Can also be specified with OCI_REPO_PREFIX environment variable
  -p, --plainHTTP              Use plain HTTP instead of HTTPS when pushing image
  -w, --workDir string         Working directory. Default to $HOME/.kubocd
```

---

## kubocd dump package

Permet de consulter le contenu d'un package KuboCD

```
Usage:
  kubocd dump package <package.yaml|oci://repo:version> [flags]

Aliases:
  package, pck, Package, Pck, pack, Pack

Flags:
  -a, --anonymous       Connect anonymously to the registry. To check 'public' image status
  -c, --charts          unpack charts in output directory
  -h, --help            help for package
  -i, --insecure        insecure (use HTTP, not HTTPS)
  -o, --output string   Output dump directory (default "./.dump")

Global Flags:
  -w, --workDir string   working directory. Default to $HOME/.kubocd

```

Exemple:

```
kubocd dump package oci://quay.io/kubodoc/packages/podinfo:6.7.1-p01
```

ou:

```
kubocd dump package podinfo-p01.yaml
```

```
Create .dump/podinfo/status.yaml
Create .dump/podinfo/original.yaml
Create .dump/podinfo/groomed.yaml
Create .dump/podinfo/default-parameters.yaml
Create .dump/podinfo/default-context.yaml
```

Cette command crée un répertoire `.dump` dans le dossier local, qui contient:

- `.dump/podinfo/original.yaml`: Le manifest original, tel que fourni lors du packaging.
- `.dump/podinfo/groomed.yaml` Le manifest dans sa version préparée: Ajout des valeurs par défaut, mise au norme des schémas de paramètres et de contexte, etc...
- `.dump/podinfo/default-parameters.yaml`: Les valeurs de paramètres par défaut. Extrait du schema des paramètres
- `.dump/podinfo/default-context.yaml`: Les valeurs de context par défaut. Extrait du schema du context.

En option, il est aussi possible de récupérer les charts Helm des modules intégrés dans le package:

```
kubocd dump package podinfo-p01.yaml --charts
```

```
--- Handling module 'main':
    Fetching chart podinfo:6.7.1...
    Chart: podinfo:6.7.1
Expand chart podinfo
Create .dump/podinfo/status.yaml
Create .dump/podinfo/original.yaml
Create .dump/podinfo/groomed.yaml
Create .dump/podinfo/default-parameters.yaml
Create .dump/podinfo/default-context.yaml
```

La chart Helm de `podinfo` se trouve dans `.dump/podinfo/charts/main`.

---

## kubocd dump helmRepository

Cette commande permet d'explorer le contenu d'un repository Helm distant

```
Usage:
  kubocd dump helmRepository repoUrl [chartName [version]] [flags]

Aliases:
  helmRepository, hr, HelmRepository, helmrepository, helmRepo, HelmRepo, helmrepo

Flags:
  -c, --chart           unpack charts in output directory
  -h, --help            help for helmRepository
  -o, --output string   Output chart directory (default "./.charts")

Global Flags:
  -w, --workDir string   working directory. Default to $HOME/.kubocd
```


Examples:

#### List les Charts

```
kubocd dump helmRepository https://stefanprodan.github.io/podinfo
```

```
---------------Chart in repo 'https://stefanprodan.github.io/podinfo':
podinfo
```

---

#### List les versions

```
kubocd dump helmRepository https://stefanprodan.github.io/podinfo podinfo
```

```
---------- Versions for 'podinfo':
6.8.0
6.7.1
6.7.0
........
```

---

#### Détail le contenu d'une version


```
kubocd dump helmRepository https://stefanprodan.github.io/podinfo podinfo 6.8.0
```

```
Fetching chart podinfo:6.8.0...

Chart: podinfo:6.8.0

---------------------- Chart.yaml:
apiVersion: v1
appVersion: 6.8.0
description: Podinfo Helm chart for Kubernetes
home: https://github.com/stefanprodan/podinfo
kubeVersion: '>=1.23.0-0'
maintainers:
- email: stefanprodan@users.noreply.github.com
  name: stefanprodan
name: podinfo
sources:
- https://github.com/stefanprodan/podinfo
version: 6.8.0


-------------------- content:
podinfo/Chart.yaml
podinfo/values.yaml
podinfo/templates/NOTES.txt
podinfo/templates/_helpers.tpl
podinfo/templates/certificate.
.........
```

---

#### Download une Chart

L'option `--chart` permet de downloader la chart en question dans le répertoire local `.chart`:

```
kubocd dump helmRepository https://stefanprodan.github.io/podinfo podinfo 6.8.0 --chart
```

```
Fetching chart podinfo:6.8.0...

Chart: podinfo:6.8.0

---------------------- Chart.yaml:
apiVersion: v1
........

-------------------- content:
podinfo/Chart.yaml
podinfo/values.yaml
.......

---------------------- Extract chart podinfo (6.8.0) to ./.charts/podinfo-6.8.0
```

---

## kubocd dump context

Cette commande permet de visualiser le context perçu par une application. Elle implique une accès sur le cluster.

```
Usage:
  kubocd dump context [flags]

Aliases:
  context, ctx, Context, Ctx

Flags:
  -c, --context stringArray      context as 'namespace:name'
  -h, --help                     help for context
      --kubocdNamespace string   The namespace where the kubocd controller is installed in (To fetch configs resources) (default "kubocd")
  -n, --namespace string         namespace (default "default")
      --skipDefaultContext       Don't use default context

Global Flags:
  -w, --workDir string   working directory. Default to $HOME/.kubocd```
```


Examples:

Affiche le context tel que perçu par une application dans le namespace 'default'.

```
kubocd dump context
```

```
---
ingress:
  className: nginx
  domain: ingress.kubodoc.local
storageClass:
  data: standard
  workspace: standard
```

---

Affiche le context tel que perçu par une application dans le namespace `project03` 

> Il est assumé que ce namespace a un context par default, tel que décrit dans les paragraphes précédent)

```
kubocd dump context --namespace project03
```

```
---
ingress:
  className: nginx
  domain: prj03.ingress.kubodoc.local
project:
  id: p03
storageClass:
  data: standard
  workspace: standard
```

---

Fournis une list explicite des contexts à aggrèger:

```
kubocd dump context --skipDefaultContext --context contexts:cluster --context project01:project01
```

```
---
ingress:
  className: nginx
  domain: ingress.kubodoc.local
project:
  id: p01
  subdomain: prj01
storageClass:
  data: standard
  workspace: standard
```

## kubocd render

Cette commande permet de visualiser en amont l'ensemble des resources qui constituera le deployment d'une `Release`.

Elle accède au cluster courant. (Notamment pour récupérer les `Contexts`)

```
Usage:
  kubocd render <Release manifest> [<package manifest>] [flags]

Flags:
  -h, --help                     help for render
      --kubocdNamespace string   The namespace where the kubocd controller is installed in (To fetch configs resources) (default "kubocd")
  -n, --namespace string         Value to set if release.metadata.namespace is empty (default "default")
  -o, --output string            Output directory (default "./.render")
  -w, --workDir string           working directory. Default to $HOME/.kubocd
```

Exemple:

Génère les ressources telles que la `Release` `podinfo2-ctx.yaml` les généreras sur le cluster:

```
kubocd render podinfo2-ctx.yaml
```

```
Create .render/podinfo2/release.yaml
Create .render/podinfo2/configs.yaml
# Pulling image 'quay.io/kubodoc/packages/podinfo:6.7.1-p02'
Expand chart podinfo
Create .render/podinfo2/package.yaml
Create .render/podinfo2/default-parameters.yaml
Create .render/podinfo2/default-context.yaml
Create .render/podinfo2/status.yaml
Create .render/podinfo2/context.yaml
Create .render/podinfo2/parameters.yaml
Create .render/podinfo2/model.yaml
Create .render/podinfo2/roles.yaml
Create .render/podinfo2/dependencies.yaml
Create .render/podinfo2/ociRepository.yaml
Create .render/podinfo2/usage.txt
Create .render/podinfo2/helmRepository.yaml
Create .render/podinfo2/modules/main/helmRelease.yaml
Create .render/podinfo2/modules/main/values.yaml
Create .render/podinfo2/modules/main/manifests.yaml
Contexts: contexts:cluster,contexts:cluster
```

Une petite description des résultats le plus interesants :

| FILES                            | DESCRIPTION                                                                                                                                   |
|----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| release.yaml                     | Le manifest de la `Release` dans sa version préparée: Ajout des valeurs par défaut, etc..                                                     | 
| package.yaml                     | Le manifest du package dans sa version préparée: Ajout des valeurs par défaut, mise au norme des schémas de paramètres et de contexte, etc... |
| default-parameters.yaml          | Les valeurs de paramètres par défaut. Extrait du schema des paramètres                                                                        |
| default-context.yaml             | Les valeurs de context par défaut. Extrait du schema du context.                                                                              |
| context.yaml                     | Le context résultant                                                                                                                          |
| parameters.yaml                  | Les paramètres fournis dans le model de donnée pour le templating.                                                                            |
| model.yaml                       | L'ensemble du model de données pour le templating des values et d'autres paramètres.                                                          |
| roles.yaml                       | La liste des roles remplis pas ce déploiement                                                                                                 |
| dependencies.yaml                | La liste des dépendances de ce déployment.                                                                                                    |
| ociRepository.yaml               | L'objet Flux `OCIRepository` tel qu'il serait crée.                                                                                           |
| usage.txt                        | Le résultat de templating de l'`Usage` du package.                                                                                            |
| helmRepository.yaml              | L'objet Flux `HelmRepository` tel qu'il serait crée.                                                                                          |
| modules/main/helmRelease.yaml    | Pour chaque module, L'objet Flux `HelmRelease` tel qu'il serait crée.                                                                         |
| modules/main/values.yaml         | Les `values`, résultat du templating pour ce module.                                                                                          |
| modules/main/manifests.yaml      | Le résultat d'une commande `helm template --debug ....` pour ce module.                                                                       |
