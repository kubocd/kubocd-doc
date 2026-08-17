# The KuboCD CLI

## kubocd pack

Creates a KuboCD package from a manifest and stores it in an OCI image repository.

See the [A First Deployment / Package Build](130-a-first-deployment.md/#package-build) section for a usage example.

```
Usage:
  kubocd package <Package manifest> [flags]

Aliases:
  package, pack, build

Flags:
  -h, --help                   help for package
  -r, --ociRepoPrefix string   OCI repository prefix (e.g., 'quay.io/your-organization/packages'). 
                               Can also be specified via the OCI_REPO_PREFIX environment variable
  -p, --plainHTTP              Use plain HTTP instead of HTTPS when pushing the image
  -w, --workDir string         Working directory. Defaults to $HOME/.kubocd
```

---

## kubocd dump package

Displays the contents of a KuboCD package.

```
Usage:
  kubocd dump package <package.yaml|oci://repo:version> [flags]

Aliases:
  package, pck, Package, Pck, pack, Pack

Flags:
  -a, --anonymous       Connect anonymously to the registry (useful for checking 'public' image status)
  -c, --charts          Unpack charts in the output directory
  -h, --help            help for package
  -i, --insecure        Insecure (use HTTP instead of HTTPS)
  -o, --output string   Output dump directory (default "./.dump")

Global Flags:
  -w, --workDir string   Working directory. Defaults to $HOME/.kubocd
```

Example:

```
kubocd dump package oci://quay.io/kubodoc/packages/podinfo:6.7.1-p01
```

or:

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

This command creates a `.dump` directory in the local folder, containing:

- `.dump/podinfo/original.yaml`: The original manifest as provided during packaging.
- `.dump/podinfo/groomed.yaml`: The groomed version of the manifest, including default values and normalized parameter/context schemas.
- `.dump/podinfo/default-parameters.yaml`: Default parameter values, extracted from the parameter schema.
- `.dump/podinfo/default-context.yaml`: Default context values, extracted from the context schema.

Optionally, you can extract the Helm charts of the modules embedded in the package:

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

The Helm chart for `podinfo` is available in `.dump/podinfo/charts/main`.

---

## kubocd dump helmRepository

This command allows you to explore the contents of a remote Helm repository.

```
Usage:
  kubocd dump helmRepository repoUrl [chartName [version]] [flags]

Aliases:
  helmRepository, hr, HelmRepository, helmrepository, helmRepo, HelmRepo, helmrepo

Flags:
  -c, --chart           Unpack charts into the output directory
  -h, --help            help for helmRepository
  -o, --output string   Output chart directory (default "./.charts")

Global Flags:
  -w, --workDir string   Working directory. Defaults to $HOME/.kubocd
```

Examples:

#### List charts

```
kubocd dump helmRepository https://stefanprodan.github.io/podinfo
```

```
---------------Chart in repo 'https://stefanprodan.github.io/podinfo':
podinfo
```

---

#### List versions

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

#### View contents of a specific version

```
kubocd dump helmRepository https://stefanprodan.github.io/podinfo podinfo 6.8.0
```

```
Fetching chart podinfo:6.8.0...

Chart: podinfo:6.8.0

---------------------- Chart.yaml:
...
```

---

#### Download a chart

Use the `--chart` flag to download the chart into the local `.charts` directory:

```
kubocd dump helmRepository https://stefanprodan.github.io/podinfo podinfo 6.8.0 --chart
```

...



---

## kubocd dump context

This command displays the application context as perceived by KuboCD. It requires access to the Kubernetes cluster.

```
Usage:
  kubocd dump context [flags]

Aliases:
  context, ctx, Context, Ctx

Flags:
  -c, --context stringArray      Context in the form 'namespace:name'
  -h, --help                     help for context
      --kubocdNamespace string   The namespace where the KuboCD controller is installed (To fetch config resources) (default "kubocd")
  -n, --namespace string         Namespace (default "default")
      --skipDefaultContext       Do not use the default context

Global Flags:
  -w, --workDir string   Working directory. Defaults to $HOME/.kubocd
```

Examples:

Display context for an application in the `default` namespace:

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

Display context for an application in the `project03` namespace:

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

Aggregate specific contexts:

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

---

## kubocd render

This command previews all the resources that will be deployed as part of a `Release`.

It accesses the current Kubernetes cluster (mainly to retrieve `Contexts`).

```
Usage:
  kubocd render <Release manifest> [<package manifest>] [flags]

Flags:
  -h, --help                     help for render
      --kubocdNamespace string   Namespace where the kubocd controller is installed (default "kubocd")
  -n, --namespace string         Namespace to use if release.metadata.namespace is empty (default "default")
  -o, --output string            Output directory (default "./.render")
  -w, --workDir string           Working directory. Defaults to $HOME/.kubocd
```

Example:

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

Key output files:

| FILES                            | DESCRIPTION                                                                                     |
|----------------------------------|-------------------------------------------------------------------------------------------------|
| release.yaml                     | Final `Release` manifest with defaults added                                                    |
| package.yaml                     | Processed package manifest with normalized parameter and context schemas                        |
| default-parameters.yaml          | Default parameter values extracted from the parameter schema                                    |
| default-context.yaml             | Default context values extracted from the context schema                                        |
| context.yaml                     | Resulting context after processing                                                              |
| parameters.yaml                  | Parameters provided for templating                                                              |
| model.yaml                       | Complete data model used for templating values and other properties                            |
| roles.yaml                       | List of roles fulfilled by this deployment                                                     |
| dependencies.yaml                | List of deployment dependencies                                                                |
| ociRepository.yaml               | Flux `OCIRepository` object to be created                                                      |
| usage.txt                        | Templated result of the package usage                                                          |
| helmRepository.yaml              | Flux `HelmRepository` object to be created                                                     |
| modules/main/helmRelease.yaml    | Flux `HelmRelease` object for the main module                                                  |
| modules/main/values.yaml         | Templated Helm values for the main module                                                      |
| modules/main/manifests.yaml      | Result of `helm template --debug ...` for the main module                                      |
