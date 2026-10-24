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
| `parents` | List([CrossNamespaceReference](./510-release.md#crossnamespacereference)) | `[]` | Contexts merged before this one, in list order. If `namespace` is omitted, the namespace of the `Context` is used. See [Parents](#parents). |
| `context` | Map | `null` | The free-form context data structure. When it is omitted, the `Context` adds no values of its own. |

---

## Parents

A `Context` with `parents` merges the values of its parents, in list order, then its own `context` on top: a later
parent overrides an earlier one, and the `Context` overrides its parents. A parent brings the values it got from its
own parents. The result is in `status.context`, which a `Release` reads, and `status.parents` lists the parents, as the
`PARENTS` column of `kubectl get contexts` shows.

The `Context` goes to `ERROR`, with the message in a `Warning` event of the `Context`, when:

- A parent does not exist: `parent 'apps:base' not found`. The `Context` is reconciled again when the parent is
  created.
- A parent cannot be read: `could not get parent 'apps:base': ...`. The controller tries again later.
- A parent is in `ERROR`: `parent 'apps:base' is in error`. The controller tries again later.
- The parents form a cycle. The message gives the path that leads back to the `Context`, such as
  `parents form a cycle: apps:a -> apps:b -> apps:a`. The `Context` stays in `ERROR` until a change of a `Context` of
  the cycle breaks it.

A `Release` that uses a `Context` in `ERROR` goes to `ERROR` too, with `context 'apps:a' is in error`.
