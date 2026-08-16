# The KuboCD Package Resource

A `Package` is an OCI-compliant container image that bundles an application descriptor and one or more Helm charts. As the standardized unit of deployment, it encapsulates everything needed to install and configure an application.

See [A first deployment](../user-guide/a-first-deployment.md) for an introductory example.

The OCI image is built from a YAML manifest described below.

## Templating

Some attributes support templating to render dynamic values. These are denoted as:

- **Template(string)**: Renders a string.
- **Template(bool)**: Renders a string convertible to a boolean (e.g., `true`, `false`, `1`, `0`, `t`, `f`).
- **Template(list(string))**: Renders a list of strings.
- **Template(map)**: Renders a map/object.
- **Template(duration)**: Renders a string convertible to a duration (e.g., `5m`, `1h`).

---

## Package

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `apiVersion` | String | **Yes** | The API version. Currently `v1alpha1`. |
| `type` | String | No | The resource type. Must be `Package` if specified. |
| `name` | String | **Yes** | The package name. Used in the OCI repository name. Must be a valid DNS Subdomain name. |
| `tag` | String | **Yes** | The specific tag for the OCI image. |
| `description` | Template(String) | No | A short description of the package. Default description for the `Release`. |
| `usage` | Map(String, Template(String)) | No | Usage information (e.g., access URLs) displayed after deployment. Each key names one entry, so a package can publish several, and each value is templated. Rendered into `Release.status.usage`. |
| `protected` | Bool | No (Default: `false`) | If `true`, prevents accidental deletion. Default for the `Release`. |
| `schema.parameters` | Map | No | Schema to validate `Release.parameters`. Supports standard JSON/OpenAPI or [KuboCD simplified schema](../user-guide/alternate-schema-format.md). |
| `schema.context` | Map | No | Schema to validate the injected `Context`. Supports standard JSON/OpenAPI or [KuboCD simplified schema](../user-guide/alternate-schema-format.md). |
| `modules` | List([Module](#packagemodule)) | **Yes** | The list of modules embedded in this package. |
| `roles` | Template(List(String)) | No | The roles this application fulfills. See [Release Dependencies](../user-guide/redis.md/#release-dependencies-and-roles). |
| `dependencies` | Template(List(String)) | No | The roles this application depends on. See [Release Dependencies](../user-guide/redis.md/#release-dependencies-and-roles). |
| `outputs` | Template(List([Output](#packageoutputs))) | No | The connections this package publishes once deployed. Experimental. |
| `inputs` | Template(List([Input](#packageinputs))) | No | The connections this package consumes. Experimental. |

---

## Package.module

A `Module` defines a Helm Chart deployment.

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `name` | String | - | **Required.** Unique name for the module. Suffix for the generated `HelmRelease`. Use `noname` for no suffix. |
| `source` | Map | - | **Required.** Defines where to fetch the chart. Must contain exactly one of: [helmRepository](#packagemodulesourcehelmrepository), [git](#packagemodulesourcegit), [oci](#packagemodulesourceoci), or [local](#packagemodulesourcelocal). |
| `values` | Template(Map) | `null` | Template for generating the Helm `values.yaml`. |
| `enabled` | Template(Bool) | `true` | Conditional deployment. If `false`, the module is skipped. |
| `targetNamespace` | Template(String) | `{{ .Release.spec.targetNamespace }}` | The namespace where the module is deployed. |
| `onFailureStrategy` | Template(String) | `null` | Strategy for failure handling. See [Deployment Failure](../user-guide/deployment-failure.md). |
| `timeout` | Template(Duration) | `{config.defaultHelmTimeout}` | Sets `spec.timeout` in the generated `HelmRelease`. |
| `interval` | Template(Duration) | `{config.defaultHelmInterval}` | Sets `spec.interval` in the generated `HelmRelease` (reconciliation frequency). |
| `dependsOn` | Template(List(String)) | `null` | List of other module names this module depends on. |
| `specPatch` | Template(Map) | `null` | Arbitrary patch applied to the `spec` of the generated Flux `HelmRelease`. |

---

## Package.module.source.helmRepository

Fetches a Chart from an HTTP(S) Helm Repository.

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `url` | String | **Yes** | The repository URL. |
| `chart` | String | **Yes** | The chart name. |
| `version` | String | **Yes** | The chart version. |

---

## Package.module.source.git

Fetches a Chart from a Git Repository.

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `url` | String | **Yes** | The repository URL. |
| `branch` | String | Yes (if tag omitted) | Git branch to fetch. |
| `tag` | String | Yes (if branch omitted) | Git tag to fetch. |
| `path` | String | **Yes** | Relative path to the chart directory (containing `Chart.yaml`). |
| `extraFilePrefixes` | List(String) | No | Include extra files/folders starting with these prefixes (relative to chart path). |

---

## Package.module.source.oci

Fetches a Chart stored as an OCI artifact.

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `repository` | String | **Yes** | The repository URL (without `oci://` prefix or tag). |
| `tag` | String | **Yes** | The image tag. |
| `insecure` | Bool | No (Default: `false`) | If `true`, allows unencrypted (HTTP) connections. |

---

## Package.module.source.local

Fetches a Chart from the local filesystem (relative to the Package manifest).

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `path` | String | **Yes** | Relative path to the chart directory. |

---

## Package.outputs

!!! experimental "Experimental"

    Outputs, inputs and the `connectionRef` parameter type belong to the
    [Connection subsystem](connection.md), which is experimental. Their field names and
    behaviour may change in a future release.

Each entry produces one `Connection` (or `ClusterConnection`) when the Release is deployed, so
other Releases can consume what this one exposes. The whole stanza is a template, which is how a
package publishes one connection per item of a list parameter.

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `name` | String | No (Default: `connectionType`) | Identifies the output inside the package, and must be unique across them. Reported on the produced connection as `spec.outputName`. |
| `connectionType` | String | **Yes** | The `ConnectionType` or `ClusterConnectionType` the produced connection complies with. |
| `values` | Map | No | What the connection carries. Must validate against the schema of its type. |
| `kind` | String | No (Default: `Connection`) | `Connection` or `ClusterConnection`. Any other value fails the build. |
| `connectionName` | String | No | Name of the produced connection. Defaults to `kcd-<release>-<output>`, which two Releases of the same package would collide on when they mean to publish the same thing. |
| `labels` | Map | No | Labels set on the produced connection, for operators selecting connections rather than naming them. |
| `displayName` | String | No (Default: `name`) | A human oriented name. |
| `priority` | Int | No (Default: `100`) | Ranks this connection against others of the same type. The highest wins. |
| `description` | String | No | A short description. |

```yaml
outputs: |
  {{- range .Parameters.databases }}
  - name: "{{ .name }}"
    connectionType: database-server
    values:
      host: "{{ $.Release.metadata.name }}-rw.{{ $.Release.spec.targetNamespace }}.svc.cluster.local"
      port: 5432
      dbName: "{{ .name }}"
  {{- end }}
```

---

## Package.inputs

Each entry declares one connection the package needs. The resolved values are exposed to the
module templates under `.Inputs.<alias>`.

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `alias` | String | **Yes** | The key the input is exposed under in `.Inputs`. |
| `connectionType` | String | **Yes** | The type the chosen connection must comply with. |
| `kind` | String | No | Restricts the lookup to `Connection` or `ClusterConnection`. Absent means both. |
| `optional` | Bool | No (Default: `false`) | The Release deploys even with no connection matching. |
| `allowMultiple` | Bool | No (Default: `false`) | Accept several matching connections instead of electing one. |
| `connectionTypeLookup.namespace` | String | No | Namespace to resolve the connection type in. |
| `namedConnection.name` / `.namespace` | String | No | Wire this input to one connection by name. |
| `release.name` / `.namespace` / `.output` | String | No | Wire this input to a named output of a given Release. |

When several connections match and `allowMultiple` is false, the one with the highest `priority`
wins, ties broken by ascending name.

---

## The connectionRef parameter type

A `connectionRef` types a parameter (or a context variable) that carries the **name** of a
connection. It replaces the `inputs` stanza for the common case: the input is generated from the
declaration, and the resolved connection is substituted in place, so templates read
`.Parameters.<path>` and never `.Inputs`.

```yaml
schema:
  parameters:
    properties:
      db: { type: connectionRef, connectionType: database-server, required: true }
```

| Property | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `type` | String | **Yes** | Always `connectionRef`. |
| `connectionType` | String | **Yes** | The type the referenced connection must comply with. |
| `kind` | String | No | `Connection` or `ClusterConnection`. Absent means both, which is ambiguous only when the same name is carried by one of each. |
| `required` | Bool | No | Standard schema semantics. |
| `default` | Template(String) | No | Must be a template string rendered against the Context. A literal is rejected, because it would hardcode a wiring target inside the package. |
| `title`, `description` | String | No | Standard schema annotations. |

Two positions forbid a `default`, and both fail the build rather than passing silently:

- inside `schema.context`, where the name always comes from the Context,
- inside array `items`, which the defaulter never descends into.

An all-digit property name is also rejected on the path leading to a `connectionRef`, because the
substitution machinery tells map keys from array indices by their digit-only shape.
