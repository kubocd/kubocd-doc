# The KuboCD CLI

## kubocd package

Creates a KuboCD package from a manifest and stores it in an OCI image repository.

See [A First Deployment / Package Build](130-a-first-deployment.md/#package-build) for usage examples.

Help excerpt (`kubocd package --help`):

```text
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
      --allowExternalLinks           Allow Git and local charts to read files and file:// dependencies outside their source root; special files remain refused.
      --allowInsecureCredentials     Allow OCI authentication over HTTP, including token services and redirects. Disabled by default; anonymous HTTP token flows remain supported. Does not select HTTP, disable certificate checks, or relax Git protections.
      --disableArchiveLimits         Disable CLI archive resource limits; path checks and archive validation remain enabled.
  -h, --help                         help for package
      --logLevel string              Log level (default "INFO")
      --maxArchiveContentSize int    Maximum logical content bytes per archive, including skipped and duplicate entries. (default 41943040)
      --maxArchiveEntries int        Maximum entries per archive, including implicit parent directories. (default 1000)
      --maxArchiveListingSize int    Maximum bytes per archive listing, including newlines. (default 1048576)
      --maxArchiveMetadataSize int   Maximum bytes per archive metadata file read or decoded. (default 1048576)
      --maxArchiveStreamSize int     Maximum decompressed bytes per archive, including headers, skipped entries and padding. (default 47087616)
  -r, --ociRepoPrefix string         OCI repository prefix (i.e 'quay.io/your-organization/packages'). Can also be specified with OCI_REPO_PREFIX environment variable
  -p, --plainHTTP                    Use plain HTTP instead of HTTPS when pushing image; authentication over HTTP requires --allowInsecureCredentials
  -w, --workDir string               Working directory. Default to $HOME/.kubocd
```

---

## kubocd dump package

Displays the contents of a KuboCD package.

Help excerpt (`kubocd dump package --help`):

```text
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
      --allowExternalLinks           Allow Git and local charts to read files and file:// dependencies outside their source root; special files remain refused.
      --allowInsecureCredentials     Allow OCI authentication over HTTP, including token services and redirects. Disabled by default; anonymous HTTP token flows remain supported. Does not select HTTP, disable certificate checks, or relax Git protections.
  -a, --anonymous                    Connect anonymously to the registry. To check 'public' image status
  -c, --charts                       unpack charts in output directory
      --disableArchiveLimits         Disable CLI archive resource limits; path checks and archive validation remain enabled.
  -h, --help                         help for package
  -i, --insecure                     Use HTTP instead of HTTPS; authentication over HTTP requires --allowInsecureCredentials
      --maxArchiveContentSize int    Maximum logical content bytes per archive, including skipped and duplicate entries. (default 41943040)
      --maxArchiveEntries int        Maximum entries per archive, including implicit parent directories. (default 1000)
      --maxArchiveListingSize int    Maximum bytes per archive listing, including newlines. (default 1048576)
      --maxArchiveMetadataSize int   Maximum bytes per archive metadata file read or decoded. (default 1048576)
      --maxArchiveStreamSize int     Maximum decompressed bytes per archive, including headers, skipped entries and padding. (default 47087616)
  -o, --output string                Output directory; an empty value prints to stdout (requires --charts=false) (default "./.dump")

Global Flags:
  -w, --workDir string   working directory. Default to $HOME/.kubocd
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

Help excerpt (`kubocd render --help`):

```text
Usage:
  kubocd render <Release manifest> [<package manifest>] [flags]

Examples:
	Preview a Release.
	$ render releases/podinfo2-ctx.yaml

	Preview a Release using an alternate package manifest.
	$ kubocd render releases/podinfo1.yaml packages/podinfo-p01.yaml

Flags:
      --allowDNSLookups                Let the package templates resolve host names with resolveDNS, as the controller option of the same name
      --allowExternalLinks             Allow Git and local charts to read files and file:// dependencies outside their source root; special files remain refused.
      --allowInsecureCredentials       Allow OCI authentication over HTTP, including token services and redirects. Disabled by default; anonymous HTTP token flows remain supported. Does not select HTTP, disable certificate checks, or relax Git protections.
      --allowRedirectVerifyDrop        Allow a matching packageRedirect without verify to remove Release signature verification.
      --allowReleaseInsecureSource     Allow Release package sources to use insecure HTTP without a matching packageRedirect.
      --conditionCelCostLimit int      Maximum CEL cost per condition evaluation; 0 disables the cost limit, negative values are refused. (default 1000000)
      --conditionCelTimeout duration   Timeout per condition CEL evaluation; 0 disables this timeout (caller cancellation/deadlines still apply); negative values are refused. (default 1s)
      --debug                          Enable debug logging
      --disableArchiveLimits           Disable CLI archive resource limits; path checks and archive validation remain enabled.
  -h, --help                           help for render
      --kubocdNamespace string         The namespace where the kubocd controller is installed in (To fetch configs resources) (default "kubocd")
      --maxArchiveContentSize int      Maximum logical content bytes per archive, including skipped and duplicate entries. (default 41943040)
      --maxArchiveEntries int          Maximum entries per archive, including implicit parent directories. (default 1000)
      --maxArchiveListingSize int      Maximum bytes per archive listing, including newlines. (default 1048576)
      --maxArchiveMetadataSize int     Maximum bytes per archive metadata file read or decoded. (default 1048576)
      --maxArchiveStreamSize int       Maximum decompressed bytes per archive, including headers, skipped entries and padding. (default 47087616)
  -n, --namespace string               Value to set if release.metadata.namespace is empty (default "default")
  -o, --output string                  Output directory; an empty value prints top-level dumps to stdout and leaves charts/, modules/ and connection append files in the current directory (default "./.render")
  -s, --skipConditions                 Skip condition evaluation; core Secret/ConfigMap and Release target namespace restrictions still apply
      --trace                          Enable trace logging
  -w, --workDir string                 working directory. Default to $HOME/.kubocd
```

To render a package that calls `resolveDNS`, add the `--allowDNSLookups` flag, the counterpart of the controller option.
See [resolveDNS](../reference/550-template-functions.md#resolvedns).

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
| `modules/main/helmRelease.yaml` | The `HelmRelease` that the controller writes for the module. |
| `modules/main/values.yaml` | The values of this `HelmRelease`, patches included: the `specPatch` of the `Config` resources, of the module and of the `Release`. `spec.valuesFrom` and `spec.chart.spec.valuesFiles` are not read. |
| `modules/main/manifests.yaml` | The standard output of `helm template --debug` on the chart with these values, in the namespace and under the release name where Flux installs this `HelmRelease`. When helm succeeds, what it writes on its standard error, such as its debug lines, goes to the standard error of `kubocd render`. When helm fails, `kubocd render` writes no `manifests.yaml`, prints the command and both outputs of helm on its standard output, and stops with an error on its standard error, such as `failed to generate manifest: exit status 1`. |

!!! tip
    It is good practice to run `kubocd render` against your manifests before applying them to the cluster.