# The KuboCD Package Resource

A `Package` is an OCI-compliant container image that bundles an application descriptor and one or more Helm charts. As the standardized unit of deployment, it encapsulates everything needed to install and configure an application.

See [A first deployment](../user-guide/130-a-first-deployment.md) for an introductory example.

The OCI image is built from a YAML manifest described below.

## Templating

Some attributes support templating to render dynamic values. These are denoted as:

- **Template(string)**: Renders a string.
- **Template(bool)**: Renders a string convertible to a boolean (e.g., `true`, `false`, `1`, `0`, `t`, `f`).
- **Template(list(string))**: Renders a list of strings.
- **Template(map)**: Renders a map/object.
- **Template(duration)**: Renders a string convertible to a duration (e.g., `5m`, `1h`).

---

## Package

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `apiVersion` | String | **Yes** | The API version. Currently `v1alpha1`. |
| `type` | String | No | The resource type. Must be `Package` if specified. |
| `name` | String | **Yes** | The package name. Used in the OCI repository name. Must be a valid DNS Subdomain name. |
| `tag` | String | **Yes** | The specific tag for the OCI image. |
| `description` | Template(String) | No | A short description of the package. Default description for the `Release`. |
| `usage` | Template(String) | No | Usage information (e.g., access URLs) displayed after deployment. |
| `protected` | Bool | No (Default: `false`) | If `true`, prevents accidental deletion. Default for the `Release`. |
| `schema.parameters` | Map | No | Schema to validate `Release.parameters`. Supports standard JSON/OpenAPI or [KuboCD simplified schema](../user-guide/190-alternate-schema-format.md). |
| `schema.context` | Map | No | Schema to validate the injected `Context`. Supports standard JSON/OpenAPI or [KuboCD simplified schema](../user-guide/190-alternate-schema-format.md). |
| `modules` | List([Module](#packagemodule)) | **Yes** | The list of modules embedded in this package. |
| `roles` | Template(List(String)) | No | The roles this application fulfills. See [Release Dependencies](../user-guide/200-redis.md/#release-dependencies-and-roles). |
| `dependencies` | Template(List(String)) | No | The roles this application depends on. See [Release Dependencies](../user-guide/200-redis.md/#release-dependencies-and-roles). |

---

## Package.module

A `Module` defines a Helm Chart deployment.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `name` | String | - | **Required.** Unique name for the module. Suffix for the generated `HelmRelease`. Use `noname` for no suffix. |
| `source` | Map | - | **Required.** Defines where to fetch the chart. Must contain exactly one of: [helmRepository](#packagemodulesourcehelmrepository), [git](#packagemodulesourcegit), [oci](#packagemodulesourceoci), or [local](#packagemodulesourcelocal). |
| `values` | Template(Map) | `null` | Template for generating the Helm `values.yaml`. |
| `enabled` | Template(Bool) | `true` | Conditional deployment. If `false`, the module is skipped. |
| `targetNamespace` | Template(String) | `{{ .Release.spec.targetNamespace }}` | The namespace where the module is deployed. |
| `onFailureStrategy` | Template(String) | `null` | Strategy for failure handling. See [Deployment Failure](../user-guide/220-deployment-failure.md). |
| `timeout` | Template(Duration) | `{config.defaultHelmTimeout}` | Sets `spec.timeout` in the generated `HelmRelease`. |
| `interval` | Template(Duration) | `{config.defaultHelmInterval}` | Sets `spec.interval` in the generated `HelmRelease` (reconciliation frequency). |
| `dependsOn` | Template(List(String)) | `null` | List of other module names this module depends on. |
| `specPatch` | Template(Map) | `null` | Arbitrary patch applied to the `spec` of the generated Flux `HelmRelease`. |

---

## Package.module.source.helmRepository

Fetches a Chart from an HTTP(S) Helm Repository.

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `url` | String | **Yes** | The repository URL. |
| `chart` | String | **Yes** | The chart name. |
| `version` | String | **Yes** | The chart version. |

---

## Package.module.source.git

Fetches a Chart from a Git Repository.

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `url` | String | **Yes** | The repository URL. |
| `branch` | String | Yes (if tag omitted) | Git branch to fetch. |
| `tag` | String | Yes (if branch omitted) | Git tag to fetch. |
| `path` | String | **Yes** | Relative path to the chart directory (containing `Chart.yaml`). |
| `extraFilePrefixes` | List(String) | No | Include extra files/folders starting with these prefixes (relative to chart path). |

---

## Package.module.source.oci

Fetches a Chart stored as an OCI artifact.

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `repository` | String | **Yes** | The repository URL (without `oci://` prefix or tag). |
| `tag` | String | **Yes** | The image tag. |
| `insecure` | Bool | No (Default: `false`) | If `true`, allows unencrypted (HTTP) connections. |

---

## Package.module.source.local

Fetches a Chart from the local filesystem (relative to the Package manifest).

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `path` | String | **Yes** | Relative path to the chart directory. |
