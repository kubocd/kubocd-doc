# The KuboCD CLI

## kubocd package

Creates a KuboCD package from a manifest and stores it in an OCI image repository.

See [A First Deployment / Package Build](130-a-first-deployment.md/#package-build) for usage examples.

```text
Assemble a KuboCD Package from a manifest to an OCI image

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
  -r, --ociRepoPrefix string   OCI repository prefix (e.g., 'quay.io/org/packages'). Also definable via OCI_REPO_PREFIX env var.
  -p, --plainHTTP              Use plain HTTP instead of HTTPS when pushing image
  -w, --workDir string         Working directory. Defaults to $HOME/.kubocd
```

---

## kubocd dump package

Displays the contents of a KuboCD package.

```text
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
  -a, --anonymous       Connect anonymously to the registry.
  -c, --charts          Unpack charts in output directory.
  -h, --help            help for package.
  -i, --insecure        Use insecure connection (HTTP).
  -o, --output string   Output dump directory (default "./.dump").

Global Flags:
  -w, --workDir string   Working directory. Defaults to $HOME/.kubocd.
```

**Example:**

``` { .bash .copy }
kubocd dump package oci://quay.io/kubodoc/packages/podinfo:6.7.1-p01
```

Output:

```text
Create .dump/podinfo/status.yaml
Create .dump/podinfo/original.yaml
Create .dump/podinfo/groomed.yaml
Create .dump/podinfo/default-parameters.yaml
Create .dump/podinfo/default-context.yaml
```

The `.dump` directory contains:

- `original.yaml`: The original manifest.
- `groomed.yaml`: The normalized manifest with defaults applied.
- `default-parameters.yaml`: Default values extracted from the schema.
- `default-context.yaml`: Default context values extracted from the schema.

To extract embedded Helm charts:

``` { .bash .copy }
kubocd dump package packages/podinfo-p01.yaml --charts
```

---

## kubocd dump helmRepository

Explore the contents of a remote Helm repository.

```text
Usage:
  kubocd dump helmRepository repoUrl [chartName [version]] [flags]

Aliases:
  helmRepository, hr, HelmRepository, helmRepo

Examples:
	List charts from helm repositories
	$ kubocd dump helmRepository https://stefanprodan.github.io/podinfo

	List all versions for a chart
	$ kubocd dump helmRepository https://stefanprodan.github.io/podinfo podinfo

	View chart information
	$ kubocd dump helmRepository https://stefanprodan.github.io/podinfo podinfo 6.8.0

	Download the chart locally
	$ kubocd dump helmRepository https://stefanprodan.github.io/podinfo podinfo 6.8.0 --chart
```

**Examples:**

List charts:

``` { .bash .copy }
kubocd dump helmRepository https://stefanprodan.github.io/podinfo
```

Download a chart:

``` { .bash .copy }
kubocd dump helmRepository https://stefanprodan.github.io/podinfo podinfo 6.8.0 --chart
```

!!! tip
    If a Helm chart is OCI-based, use `helm pull`.

---

## kubocd dump context

Displays the application context as resolved by KuboCD. Requires cluster access.

```text
Usage:
  kubocd dump context [flags]

Aliases:
  context, ctx

Examples:
	Display the context for the default namespace:
	$ kubocd dump context

	Display the context for project03:
	$ kubocd dump context --namespace project03

	Aggregate specific contexts:
	$ kubocd dump context --skipDefaultContext --context contexts:cluster --context project01:project01
```

**Example:**

``` { .bash .copy }
kubocd dump context --namespace project03
```

Output:

```yaml
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

## kubocd dump config

Displays the global KuboCD configuration. Requires cluster access.

```text
Usage:
  kubocd dump config [flags]

Aliases:
  config, Config
```

**Example:**

``` { .bash .copy }
kubocd dump config
```

Output:

```yaml
---
clusterRoles: []
defaultContexts:
- name: cluster
  namespace: contexts
defaultHelmInterval: 30m0s
...
```

---

## kubocd render

Previews all resources that will be deployed by a `Release`. Ideally used to validate a `Package` or `Release` before deployment.

```text
Render a KuboCD release

Usage:
  kubocd render <Release manifest> [<package manifest>] [flags]

Examples:
	Preview a Release.
	$ render releases/podinfo2-ctx.yaml

	Preview a Release using an alternate package manifest.
	$ kubocd render releases/podinfo1.yaml packages/podinfo-p01.yaml
```

**Example:**

``` { .bash .copy }
kubocd render releases/podinfo2-ctx.yaml
```

Output files in the `.render` directory include:

| File | Description |
|------|-------------|
| `release.yaml` | Final `Release` manifest. |
| `package.yaml` | Processed package manifest. |
| `context.yaml` | The full resolved context. |
| `parameters.yaml` | The final parameters. |
| `model.yaml` | The complete data model used for templating. |
| `modules/main/values.yaml` | Templated Helm values. |
| `modules/main/manifests.yaml` | `helm template` output. |

!!! tip
    It is good practice to run `kubocd render` against your manifests before applying them to the cluster.