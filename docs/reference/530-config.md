# The Config Kubernetes Resource

## Config

### apiVersion

**String, Required:** Always `kubocd.kubotal.io/v1alpha1`.

### kind

**String, Required:** Always `Config`.

### metadata

**Map, Required:** Standard Kubernetes metadata.

### spec

**Config.spec, Required:** See [Config.spec](#configspec) below.

---

## Config.spec

### clusterRoles

**List(string), Default: []**

List of roles fulfilled by external (non-KuboCD) applications. See [Cluster Roles](../user-guide/200-redis.md/#cluster-roles).

### defaultContexts

**List(CrossNamespaceReference), Default: []**

Global default contexts applied to all `Releases` (unless `skipDefaultContext` is used). See [The Context Resource](../user-guide/160-the-context.md).

### defaultNamespaceContexts

**List(string), Default: []**

List of context names to automatically look up in the Release's namespace. If found, they are merged into the deployment context.

### onFailureStrategies

**List(onFailureStrategy), Optional**

Definitions of deployment failure strategies. See [Deployment Failure](../user-guide/220-deployment-failure.md).

### defaultOnFailureStrategy

**String, Optional**

The name of the strategy to use by default.

### defaultHelmTimeout

**Duration, Default: 3m**

Default timeout for Helm deployments (`HelmRelease.spec.timeout`).

### defaultHelmInterval

**Duration, Default: 30m**

Default reconciliation interval for Helm releases (`HelmRelease.spec.interval`).

### defaultPackageInterval

**Duration, Default: 30m**

Default interval for checking OCI package updates (`OCIRepository.spec.interval`).

### specPatch

**Template(map), Optional**

Global patch applied to the `spec` section of all generated Flux `HelmRelease` resources.

### packageRedirects

**Reserved for future extensions.**

### imageRedirects

**Reserved for future extensions.**

---

## onFailureStrategy

### name

**String, Required**

Unique strategy name.

### values

**Map, Required**

Flux-compatible configuration for `install` and `upgrade` strategies.
