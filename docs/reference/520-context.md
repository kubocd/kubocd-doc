# The Context Kubernetes resource.

You can refer to [The context resource](../user-guide/160-the-context.md) for some example of usage.

## Release

### apiVersion

**String, required:** Always `kubocd.kubotal.io/v1alpha1`

### kind

**String, required:** Always `Context'

### metadata

**Map, required:** Refer to the Kubernetes API documentation for the fields of the metadata field.

### spec

**Context.spec, required:** See [Context.spec](#contextspec) below

---

## Context.spec

### description

**String, optional:** A short description. 

### protected

**Bool, Default: false:** If true, prevent deletion. Need the KuboCD webhook to be effective

### context

**Map, required:** The context map value itself. Refer to [The Context resource](../user-guide/160-the-context.md)

