# The KuboCD CLI

## kubocd pack

Creates a KuboCD package from a manifest and stores it in an OCI image repository.

See the [A First Deployment / Package Build](130-a-first-deployment.md/#package-build) section for a usage example.

``` { .bash }
Assemble a KuboCd Package from a manifest to an OCI image

Usage:
  kubocd package <Package manifest> [flags]

Aliases:
  package, pack, build

Examples:
	Build and push a package:
	$ kubocd package podinfo-p01.yaml --ociRepoPrefix quay.io/kubodoc/packages

	or
	$ export OCI_REPO_PREFIX=quay.io/kubodoc/packages
	$ kubocd package podinfo-p01.yaml

Flags:
  -h, --help                   help for package
  -r, --ociRepoPrefix string   OCI repository prefix (i.e 'quay.io/your-organization/packages'). Can also be specified with OCI_REPO_PREFIX environment variable
  -p, --plainHTTP              Use plain HTTP instead of HTTPS when pushing image
  -w, --workDir string         Working directory. Default to $HOME/.kubocd
```

---

## kubocd dump package

Displays the contents of a KuboCD package.

``` { .bash }
Dump KuboCD Package

Usage:
  kubocd dump package <package.yaml|oci://repo:version> [flags]

Aliases:
  package, pck, Package, Pck, pack, Pack

Examples:
	Dump package content from an OCI image repository
	$ kubocd dump package oci://quay.io/kubodoc/packages/podinfo:6.7.1-p01

	Dump package content from a manifest
	$ kubocd dump package podinfo-p01.yaml

	Dump package content from a manifest and fetch Helm charts
	$ kubocd dump package podinfo-p01.yaml --charts

Flags:
  -a, --anonymous       Connect anonymously to the registry. To check 'public' image status
  -c, --charts          unpack charts in output directory
  -h, --help            help for package
  -i, --insecure        insecure (use HTTP, not HTTPS)
  -o, --output string   Output dump directory (default "./.dump")

Global Flags:
  -w, --workDir string   working directory. Default to $HOME/.kubocd
```

Example:

``` { .bash .copy }
kubocd dump package oci://quay.io/kubodoc/packages/podinfo:6.7.1-p01
```

or:

``` { .bash .copy }
kubocd dump package packages/podinfo-p01.yaml
```

``` { .bash  }
Create .dump/podinfo/status.yaml
Create .dump/podinfo/original.yaml
Create .dump/podinfo/groomed.yaml
Create .dump/podinfo/default-parameters.yaml
Create .dump/podinfo/default-context.yaml
```

This command creates a `.dump` directory in the local folder, containing:

- `.dump/podinfo/original.yaml`: The original manifest as provided during packaging.
- `.dump/podinfo/groomed.yaml`: The groomed version of the manifest, including default values and normalized parameter/context schemas.
- `.dump/podinfo/default-parameters.yaml`: Default parameter values, extracted from the parameter schema.
- `.dump/podinfo/default-context.yaml`: Default context values, extracted from the context schema.

Optionally, you can extract the Helm charts of the modules embedded in the package:

``` { .bash .copy }
kubocd dump package packages/podinfo-p01.yaml --charts
```

``` { .bash }
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

The Helm chart for `podinfo` is available in `.dump/podinfo/charts/main`.

---

## kubocd dump helmRepository

This command allows you to explore the contents of a remote Helm repository.

``` { .bash  }
Dump helm chart

Usage:
  kubocd dump helmRepository repoUrl [chartName [version]] [flags]

Aliases:
  helmRepository, hr, HelmRepository, helmrepository, helmRepo, HelmRepo, helmrepo

Examples:
	List charts from helm repositories
	$ kubocd dump helmRepository https://stefanprodan.github.io/podinfo

	List all versions for a chart
	$ kubocd dump helmRepository https://stefanprodan.github.io/podinfo podinfo

	View chart information
	$ kubocd dump helmRepository https://stefanprodan.github.io/podinfo podinfo 6.8.0

	Download locally the chart
	$ kubocd dump helmRepository https://stefanprodan.github.io/podinfo podinfo 6.8.0 --chart

Flags:
  -c, --chart           unpack charts in output directory
  -h, --help            help for helmRepository
  -o, --output string   Output chart directory (default "./.charts")

Global Flags:
  -w, --workDir string   working directory. Default to $HOME/.kubocd
```

Examples:

#### List charts

``` { .bash .copy }
kubocd dump helmRepository https://stefanprodan.github.io/podinfo
```

``` { .bash  }
---------------Chart in repo 'https://stefanprodan.github.io/podinfo':
podinfo
```

---

#### List versions

``` { .bash .copy }
kubocd dump helmRepository https://stefanprodan.github.io/podinfo podinfo
```

``` { .bash  }
---------- Versions for 'podinfo':
6.8.0
6.7.1
6.7.0
........
```

---

#### View contents of a specific version

``` { .bash .copy }
kubocd dump helmRepository https://stefanprodan.github.io/podinfo podinfo 6.8.0
```

``` { .bash  }
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

#### Download a chart

Use the `--chart` flag to download the chart into the local `.charts` directory:

``` { .bash .copy }
kubocd dump helmRepository https://stefanprodan.github.io/podinfo podinfo 6.8.0 --chart
```

``` { .bash }
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

!!! tips
    If your helm chart is provided as an OCI resource, you can use `helm pull` to download it locally.

    For example:
    ```
    helm pull oci://quay.io/kubocd/charts/kubocd-ctrl --version v0.2.3
    tar tvzf kubocd-ctrl-v0.2.3.tgz
    ```
    
---

## kubocd dump context

This command displays the application context as perceived by KuboCD. It requires access to the Kubernetes cluster.

``` { .bash }
Dump KuboCD context

Usage:
  kubocd dump context [flags]

Aliases:
  context, ctx, Context, Ctx

Examples:
	Display the context for an application in the default namespace:
	$ kubocd dump context

	Display the context for an application in the project03 namespace:
	$ kubocd dump context --namespace project03

	Aggregate specific contexts:
	$ kubocd dump context --skipDefaultContext --context contexts:cluster --context project01:project01

Flags:
  -c, --context stringArray      context as 'namespace:name'. May be repeated.
  -h, --help                     help for context
      --kubocdNamespace string   The namespace where the kubocd controller is installed in (To fetch configs resources) (default "kubocd")
  -n, --namespace string         namespace (default "default")
      --skipDefaultContext       Don't use default context

Global Flags:
  -w, --workDir string   working directory. Default to $HOME/.kubocd
```

Examples:

Display the context for an application in the `default` namespace:

``` { .bash .copy }
kubocd dump context
```

``` { .bash }
---
ingress:
  className: nginx
  domain: ingress.kubodoc.local
storageClass:
  data: standard
  workspace: standard
```

Display the context for an application in the `project03` namespace:

> It is assumed than this namespace has a default context, as described in previous chapters

``` { .bash .copy }
kubocd dump context --namespace project03
```

``` { .bash }
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

Aggregate specific contexts:

``` { .bash .copy }
kubocd dump context --skipDefaultContext --context contexts:cluster --context project01:project01
```

``` { .bash }
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

---

## kubocd render

This command previews all the resources that will be deployed as part of a `Release`.

It is also a good validation of a new `Package` and/or `Release` before deployment.

It accesses the current Kubernetes cluster (mainly to retrieve `Contexts`).

``` { .bash }
Render a KuboCD release

Usage:
  kubocd render <Release manifest> [<package manifest>] [flags]

Examples:
	Preview a Release.
	$ render releases/podinfo2-ctx.yaml

	Preview a Release using an alternate package manifest.
	$ kubocd render releases/podinfo1.yaml packages/podinfo-p01.yaml

Flags:
  -h, --help                     help for render
      --kubocdNamespace string   The namespace where the kubocd controller is installed in (To fetch configs resources) (default "kubocd")
  -n, --namespace string         Value to set if release.metadata.namespace is empty (default "default")
  -o, --output string            Output directory (default "./.render")
  -w, --workDir string           working directory. Default to $HOME/.kubocd
```


Example:

``` { .bash .copy }
kubocd render releases/podinfo2-ctx.yaml
```

``` { .bash }
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

Key output files:

| FILES                            | DESCRIPTION                                                              |
|----------------------------------|--------------------------------------------------------------------------|
| release.yaml                     | Final `Release` manifest with defaults added                             |
| package.yaml                     | Processed package manifest with normalized parameter and context schemas |
| default-parameters.yaml          | Default parameter values extracted from the parameter schema             |
| default-context.yaml             | Default context values extracted from the context schema                 |
| context.yaml                     | Resulting context after processing                                       |
| parameters.yaml                  | Parameters provided for templating                                       |
| model.yaml                       | Complete data model used for templating values and other properties      |
| roles.yaml                       | List of roles fulfilled by this deployment                               |
| dependencies.yaml                | List of deployment dependencies                                          |
| ociRepository.yaml               | Flux `OCIRepository` object to be created                                |
| usage.txt                        | Templated result of the package usage                                    |
| helmRepository.yaml              | Flux `HelmRepository` object to be created                               |
| modules/main/helmRelease.yaml    | Flux `HelmRelease` object for the main module                            |
| modules/main/values.yaml         | Templated Helm values for the main module                                |
| modules/main/manifests.yaml      | Result of `helm template --debug ...` for the main module                |

!!! tips
    It is of good practice to check a `Release` with this `kubocd render` command before deployment on the cluster.