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
| `defaultPackageInterval` | Duration | `30m` | Default interval for checking OCI package updates (`OCIRepository.spec.interval`). |
| `specPatch` | Template(Map) | `null` | Global patch applied to the `spec` section of all generated Flux `HelmRelease` resources. |
| `packageRedirects` | - | - | **Reserved for future extensions.** |
| `imageRedirects` | - | - | **Reserved for future extensions.** |

---

## onFailureStrategy

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `name` | String | **Yes** | Unique strategy name. |
| `values` | Map | **Yes** | Flux-compatible configuration for `install` and `upgrade` strategies. |

---

## CrossNamespaceReference

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `name` | String | **Yes** | Resource name. |
| `namespace` | String | No | Resource namespace. Defaults to local namespace if omitted. |
