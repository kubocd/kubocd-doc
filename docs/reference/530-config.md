# The Config Kubernetes resource.

## Config

### apiVersion

**String, required:** Always `kubocd.kubotal.io/v1alpha1`

### kind

**String, required:** Always `Config'

### metadata

**Map, required:** Refer to the Kubernetes API documentation for the fields of the metadata field.

### spec

**Config.spec, required:** See [Config.spec](./#configspec) below

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

### packageRedirects

For future extension

### imageRedirects

For future extension
