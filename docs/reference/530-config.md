# The Config Kubernetes resource.

## Config

### apiVersion

**String, required:** Always `kubocd.kubotal.io/v1alpha1`

### kind

**String, required:** Always `Config'

### metadata

**Map, required:** Refer to the Kubernetes API documentation for the fields of the metadata field.

### spec

**Config.spec, required:** See [Config.spec](#configspec) below

---

## Config.spec

### clusterRoles

**List(string), default: []:** List of roles fulfilled by non-KuboCD application. 
See [Cluster roles](../user-guide/200-redis.md/#cluster-roles)

### defaultContexts

**List(CrossNamespaceReference), default[]:** A list of context which will be used by all `Release`, except the one with the 
`skipDefaultContext` flag.
Refer to [The Context Resource](../user-guide/160-the-context.md) for more explanation. 

### defaultNamespaceContexts

**List(string), default: []:** A list of context name. When a `Release` is deployed in a namespace, if a context of 
this name exists in the namespace, it will be used, merged with default one(s) if existing. This can be skipped with the `Release.skipDefautContext` flag

Refer to [The Context Resource](../user-guide/160-the-context.md) for more explanation. 

### onFailureStrategies

**list(onFailureStrategy), optional**

A list of strategies to apply in case of HelmRelease deployment failure. Refer to the [Deployment failure](../user-guide/220-deployment-failure.md) chapter for more explanation.

### defaultOnFailureStrategy

**string, optional**

The failure strategy to use by default.

### defaultHelmTimeout

**duration, default: 2mn**

will set the value of `HelmRelease.spec.timeout` by default, thus providing timeout on Helm deployment.

### defaultHelmInterval

**duration, default: 30mn**

will set the value of `HelmRelease.spec.interval` by default, thus providing interval to which reconcile the helmRelease

### defaultPackageInterval

**duration, default: 30mn**

Default value for `Release.spec.package.interval`. Interval applied to the `OciRepository`, to check against new package version

### specPatch

**Template(map), optional:**

A patch applied to the `spec` section of all Flux `HelmRelease` resource.
It allows you to set any parameters that are not exposed by KuboCD.

### packageRedirects

For future extension

### imageRedirects

For future extension

---

## onFailureStrategy

### name

### values



