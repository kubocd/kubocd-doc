# The Config Kubernetes Resource

## Config

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `apiVersion` | String | **Yes** | Always `kubocd.kubotal.io/v1alpha1`. |
| `kind` | String | **Yes** | Always `Config`. |
| `metadata` | Map | **Yes** | Standard Kubernetes metadata. |
| `spec` | [Config.spec](#configspec) | **Yes** | The configuration specification. |

---

## Config.spec

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `clusterRoles` | List(String) | `[]` | List of roles fulfilled by external (non-KuboCD) applications. See [Cluster Roles](../user-guide/200-redis.md/#cluster-roles). |
| `defaultContexts` | List([CrossNamespaceReference](#crossnamespacereference)) | `[]` | Global default contexts applied to all `Releases`. See [The Context Resource](../user-guide/160-the-context.md). |
| `defaultNamespaceContexts` | List(String) | `[]` | Context names to automatically look up in the Release's namespace. |
| `onFailureStrategies` | List([onFailureStrategy](#onfailurestrategy)) | `[]` | Definitions of deployment failure strategies. See [Deployment Failure](../user-guide/220-deployment-failure.md). |
| `defaultOnFailureStrategy` | String | `null` | The name of the strategy to use by default. |
| `defaultHelmTimeout` | Duration | `3m` | Default timeout for Helm deployments (`HelmRelease.spec.timeout`). |
| `defaultHelmInterval` | Duration | `30m` | Default reconciliation interval for Helm releases (`HelmRelease.spec.interval`). |
| `defaultPackageInterval` | Duration | `30m` | Default interval for checking OCI package updates (`OCIRepository.spec.interval`). Only applies to a `Release.spec.package.interval` left empty, which the CRD's own `5m` default already fills in, so a Release running under the controller never falls back to this value. `kubocd render`, which reads the manifest without the API server, does. |
| `specPatch` | Template(Map) | `null` | Global patch applied to the `spec` section of all generated Flux `HelmRelease` resources. |
| `packageRedirects` | List([packageRedirect](#packageredirect)) | `[]` | Rewrites the prefix of package repository URLs, applied to the generated `OCIRepository`. |
| `imageRedirects` | List([imageRedirect](#imageredirect)) | `[]` | Rewrites the prefix of image repository URLs, exposed to templates through the `imageRedirect` function. |

---

## onFailureStrategy

| Field      | Type | Required | Description |
|:-----------| :--- | :--- | :--- |
| `name`     | String | **Yes** | Unique strategy name. |
| `strategy` | Map | **Yes** | Flux-compatible configuration for `install` and `upgrade` strategies. |

---

## packageRedirect

Rewrites the prefix of package repository URLs, so a cluster can pull the same
packages from a mirror without editing every `Release`. The redirect applies to
the generated `OCIRepository`, and its remaining fields are carried over to it.

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `oldPrefix` | String | **Yes** | Every package repository whose URL starts with this value is redirected. |
| `newPrefix` | String | **Yes** | The value replacing the matched prefix. |
| `interval` | Duration | No | Interval at which the redirected `OCIRepository` is checked for updates. |
| `timeout` | Duration | No | Timeout for remote OCI operations such as pulling. |
| `provider` | String | No | Authentication provider: `generic` (default), `aws`, `azure` or `gcp`. |
| `secretRef` | [LocalObjectReference](#crossnamespacereference) | No | Secret holding the registry credentials. |
| `certSecretRef` | LocalObjectReference | No | Secret holding the TLS certificates for the registry. |
| `serviceAccountName` | String | No | Service account whose registry credentials are used. |
| `verify` | Map | No | Signature verification for the redirected artifacts. |

---

## imageRedirect

Rewrites the prefix of container image URLs. Unlike `packageRedirect`, this one
is not applied automatically: it is exposed to package templates through the
`imageRedirect` function, so a chart's `image` values go through it.

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `oldPrefix` | String | **Yes** | Every image repository whose URL starts with this value is redirected. |
| `newPrefix` | String | **Yes** | The value replacing the matched prefix. |
| `imagePullPolicy` | String | No | `IfNotPresent`, `Always` or `Never`. |
| `imagePullSecrets` | List(String) | No | Secrets used to pull from the target registry. |

---

## CrossNamespaceReference

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `name` | String | **Yes** | Resource name. |
| `namespace` | String | No | Resource namespace. Defaults to local namespace if omitted. |
