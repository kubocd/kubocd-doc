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
| `defaultHelmTimeout` | Duration | `2m` | Default timeout for Helm deployments (`HelmRelease.spec.timeout`). The `Config` installed by the `kubocd-ctrl` chart (`conf01`) sets `3m0s`. |
| `defaultHelmInterval` | Duration | `30m` | Default reconciliation interval for Helm releases (`HelmRelease.spec.interval`). |
| `defaultPackageInterval` | Duration | `30m` | Takes the place of a [`Release.spec.package.interval`](./510-release.md#releasespecpackage) of `0s`, the interval at which the package of the `Release` is checked for updates. A `Release` that does not set this interval gets `5m` from the API server. `kubocd render`, which reads the `Release` from a file, applies `defaultPackageInterval` to an absent interval too. |
| `specPatch` | Map | `null` | Global patch applied to the `spec` section of all generated Flux `HelmRelease` resources. It is not a template. |
| `packageRedirects` | List([packageRedirect](#packageredirect)) | `[]` | Package reference rewrites, applied to the generated `OCIRepository`. |
| `imageRedirects` | List([imageRedirect](#imageredirect)) | `[]` | Image reference rewrites, applied by templates through the [`imageRedirect`](./550-template-functions.md#imageredirect) function. |

---

## Several Configs

The controller reads every `Config` of its namespace, and combines them in name order:

- `clusterRoles`: the roles of every `Config`.
- `defaultContexts`, `defaultNamespaceContexts`, `packageRedirects` and `imageRedirects`: the lists of every `Config`,
  one after the other. The first redirect that matches applies.
- `onFailureStrategies`: the strategies of every `Config`. A strategy replaces an earlier one of the same name.
- `defaultHelmTimeout`, `defaultHelmInterval`, `defaultPackageInterval` and `specPatch`: the value of the last `Config`
  that sets one. A `Config` without `specPatch` leaves the `specPatch` of an earlier one in effect, and the patches of
  several `Config` resources are not merged.
- `defaultOnFailureStrategy`: when several `Config` resources set it, they must give the same value. It must name a
  strategy of `onFailureStrategies`.

When the `Config` resources fail to load, for instance with two different `defaultOnFailureStrategy` values, the
controller logs the error and keeps the previous configuration. At startup, it stops instead.
[`kubocd dump config`](../user-guide/180-kubocd-cli.md#kubocd-dump-config) shows the combined configuration.

---

## packageRedirect

Rewrites the prefix of package references, for instance to pull the packages from a mirror without changing the
`Release` manifests. `Config` resources are taken in name order, and the first redirect whose `oldPrefix` matches
`<repository>:<tag>` of `Release.spec.package` applies. The controller applies it to the `OCIRepository` it generates,
and `kubocd render` fetches the package from the redirected reference.

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `oldPrefix` | String | **Yes** | A package whose `<repository>:<tag>` starts with this value is redirected. |
| `newPrefix` | String | **Yes** | Replaces `oldPrefix`. If empty, the reference is kept and only the settings below apply. |
| `interval`, `timeout`, `provider`, `secretRef`, `certSecretRef`, `proxySecretRef`, `verify`, `serviceAccountName`, `insecure`, `suspend`, `ignore` | - | No | Set on the generated `OCIRepository` in place of the same fields of [Release.spec.package](./510-release.md#releasespecpackage). A field left empty here is left empty in the `OCIRepository`. |

---

## imageRedirect

Rewrites the prefix of container image references. It is not applied automatically: a template applies it to an image
with the [`imageRedirect`](./550-template-functions.md#imageredirect) function. `Config` resources are taken in name
order, and the first redirect whose `oldPrefix` matches the image applies.

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `oldPrefix` | String | **Yes** | An image whose reference starts with this value is redirected. |
| `newPrefix` | String | **Yes** | Replaces `oldPrefix`. If empty, the reference is kept and only the pull policy and secrets apply. |
| `imagePullPolicy` | String | No | `IfNotPresent`, `Always` or `Never`. |
| `imagePullSecrets` | List(String) | No | Names of the pull secrets. |

---

## onFailureStrategy

| Field      | Type | Required | Description |
|:-----------| :--- | :--- | :--- |
| `name`     | String | **Yes** | Unique strategy name. |
| `strategy` | Map | **Yes** | Flux-compatible configuration for `install` and `upgrade` strategies. |

---

## CrossNamespaceReference

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `name` | String | **Yes** | Resource name. |
| `namespace` | String | No | Resource namespace. Defaults to local namespace if omitted. |
