# The Connection Kubernetes Resources

!!! experimental "Experimental"

    The Connection subsystem is experimental. Its resources, field names and behaviour may change
    in a future release without following the usual deprecation path. A cluster that declares no
    connection at all is unaffected by it.

A Connection carries the coordinates of a service, so a Release can consume what another Release
publishes without hardcoding a host, a port or a secret name. Four resources take part:

| Resource | Scope | Short name | Role |
| :--- | :--- | :--- | :--- |
| `Contract` | Namespaced | `cnxtype` | Names a kind of service and validates what a connection of that kind carries. |
| `ClusterContract` | Cluster | `ccnxtype` | Same, shared by every namespace. |
| `Connection` | Namespaced | `cnx` | One concrete service, described by the values its type allows. |
| `ClusterConnection` | Cluster | `ccnx` | Same, visible from every namespace. |

A connection is **managed** when a Release produced it from one of its `outputs`, in which case
`status.parent` names the owning Release. It is **standalone** when a user declared it directly.

---

## Contract

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `apiVersion` | String | **Yes** | Always `kubocd.kubotal.io/v1alpha1`. |
| `kind` | String | **Yes** | Always `Contract`. |
| `metadata` | Map | **Yes** | Standard Kubernetes metadata. The name is what connections reference. |
| `spec` | [Contract.spec](#contractspec) | **Yes** | The type specification. |

### Contract.spec

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `schema` | Map | `null` | Validates the `values` of every connection of this type. Either a JSON/OpenAPI schema or the [KuboCD simplified schema format](../user-guide/alternate-schema-format.md). With no schema, values are accepted as they come. |
| `description` | String | `null` | A short description. |

### Contract.status

| Field | Type | Description |
| :--- | :--- | :--- |
| `phase` | String | `READY` or `ERROR`. |
| `message` | String | The reason behind an `ERROR` phase. |

---

## ClusterContract

Identical to [Contract](#contract) in both spec and status, with `kind` set to
`ClusterContract` and a cluster scope. Declaring the types a platform offers at cluster level
is the usual setup, so every namespace names the same thing the same way.

---

## Connection

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `apiVersion` | String | **Yes** | Always `kubocd.kubotal.io/v1alpha1`. |
| `kind` | String | **Yes** | Always `Connection`. |
| `metadata` | Map | **Yes** | Standard Kubernetes metadata. |
| `spec` | [Connection.spec](#connectionspec) | **Yes** | The connection specification. |

### Connection.spec

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `contract` | String | - | **Required.** The name of the `Contract` or `ClusterContract` this connection complies with. See [Type resolution](#type-resolution). |
| `values` | Map | `null` | What the connection carries. Must validate against the schema of its type. |
| `priority` | Int | `100` | Ranks candidates when several connections of the same type could satisfy one input. The **highest** value wins, ties broken by ascending name. |
| `description` | String | `null` | A short description. |
| `outputName` | String | `null` | The `outputs` entry that produced this connection. Set by the Release controller on managed connections, not meant to be written by hand. |

### Connection.status

| Field | Type | Description |
| :--- | :--- | :--- |
| `phase` | String | `READY` or `ERROR`. |
| `message` | String | The reason behind an `ERROR` phase. |
| `parent` | String | The Release owning this connection, empty when standalone. |
| `contractKind` | String | `Contract` or `ClusterContract`, whichever answered the resolution. |
| `contractDisplay` | String | The type name for display: bare for a `Contract`, bracketed as `[name]` for a `ClusterContract`, suffixed with `?` when no type answered. |
| `contractGeneration` | Int | The `metadata.generation` of the type this connection was last validated against. |

---

## ClusterConnection

Same fields as [Connection](#connection), with a cluster scope and two differences:

| Field | Type | Description |
| :--- | :--- | :--- |
| `spec.parentRelease` | Map | `{name, namespace}` of the owning Release on a managed cluster connection. A namespaced `Connection` carries that link as an owner reference instead, a cluster-scoped object cannot. |

Its status carries `phase`, `message`, `parent` and `contractGeneration`, but neither
`contractKind` nor `contractDisplay`. The `Cnx type` print column therefore reads
`spec.contract` directly.

---

## Type resolution

The controller resolves `spec.contract` by name, in this order:

1. a `Contract` of that name **in the namespace of the connection**,
2. otherwise a `ClusterContract` of that name.

A namespaced type shadows a cluster one of the same name, which is how a namespace overrides the
schema the platform ships.

When neither exists, the outcome depends on the connection itself:

- with **no** `values`, the connection is `READY`. Nothing was claimed, so nothing can be wrong.
- with `values`, the connection is `ERROR` with `Contract or ClusterContract '<name>' missing`,
  because the values it carries could not be checked against anything.

This is what keeps the subsystem inert on a cluster that declares no type at all.
