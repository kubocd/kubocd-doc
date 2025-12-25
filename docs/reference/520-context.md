# The Context Kubernetes Resource

See [The Context Resource](../user-guide/160-the-context.md) for usage examples.

## Context

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `apiVersion` | String | **Yes** | Always `kubocd.kubotal.io/v1alpha1`. |
| `kind` | String | **Yes** | Always `Context`. |
| `metadata` | Map | **Yes** | Standard Kubernetes metadata. |
| `spec` | [Context.spec](#contextspec) | **Yes** | The context specification. |

---

## Context.spec

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `description` | String | `null` | A short description. |
| `protected` | Bool | `false` | If `true`, prevents deletion (requires KuboCD webhook). |
| `context` | Map | - | **Required.** The free-form context data structure. |
