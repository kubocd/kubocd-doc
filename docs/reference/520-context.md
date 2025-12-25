# The Context Kubernetes Resource

See [The Context Resource](../user-guide/160-the-context.md) for usage examples.

## Context

### apiVersion

**String, Required:** Always `kubocd.kubotal.io/v1alpha1`.

### kind

**String, Required:** Always `Context`.

### metadata

**Map, Required:** Standard Kubernetes metadata.

### spec

**Context.spec, Required:** See [Context.spec](#contextspec) below.

---

## Context.spec

### description

**String, Optional:** A short description.

### protected

**Bool, Default: false:** If true, prevents deletion (requires KuboCD webhook).

### context

**Map, Required:** The free-form context data structure.
