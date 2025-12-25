# The Release Kubernetes Resource

## Release

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `apiVersion` | String | **Yes** | Always `kubocd.kubotal.io/v1alpha1`. |
| `kind` | String | **Yes** | Always `Release`. |
| `metadata` | Map | **Yes** | Standard Kubernetes metadata (name, namespace, labels, etc.). |
| `spec` | [Release.spec](#releasespec) | **Yes** | The release specification. |

---

## Release.spec

| Field                | Type | Default | Description |
|:---------------------| :--- | :--- | :--- |
| `description`        | String | `{Package.description}` | A short description of the deployment. |
| `package`            | [Release.spec.package](#releasespecpackage) | - | **Required.** Defines the `Package` OCI image to deploy. |
| `contexts`           | List([CrossNamespaceReference](#crossnamespacereference)) | `[]` | Additional contexts to merge with the defaults. If `namespace` is omitted, `Release.metadata.namespace` is used. |
| `protected`          | Bool | `{Package.protected}` | If `true`, prevents deletion of the Release (requires KuboCD webhook). |
| `parameters`         | Template(Map) | `null` | Deployment parameters matching `Package.parameters.schema`. <br>_Note: Only the `.Context` root element is available for templating in this section._ |
| `targetNamespace`    | String | `{Release.metadata.namespace}` | The namespace where the application workload is deployed. |
| `createNamespace`    | Bool | `false` | If `true`, creates `targetNamespace` if it does not exist. |
| `roles`              | List(String) | `[]` | Appended to the [`Package.roles`](./500-package.md/#roles) list. |
| `dependencies`       | List(String) | `[]` | Appended to the [`Package.dependencies`](./500-package.md/#dependencies) list. |
| `skipDefaultContext` | Bool | `false` | If `true`, ignores global and namespace-level default contexts. Only explicit contexts in `spec.contexts` are used. |
| `modulesOverrides`   | List([Release.spec.moduleOverride](#releasespecmoduleoverride)) | `[]` | Overrides specific settings for individual modules. |
| `debug`              | [Release.spec.debug](#releasespecdebug) | `false` | Debugging options. |

---

## Release.spec.moduleOverride

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `module` | String | - | **Required.** The name of the module to modify. |
| `strategy` | String | `null` | Overrides the failure handling strategy. See [Deployment Failure](../user-guide/220-deployment-failure.md). |
| `timeout` | Template(Duration) | `null` | Overrides the Helm deployment timeout. |
| `interval` | Template(Duration) | `null` | Overrides the Helm reconciliation interval. |
| `specPatch` | Map | `null` | A patch applied to the `spec` of the generated `HelmRelease` for this module. |

---

## Release.spec.package

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `repository` | String | - | **Required.** OCI repository URL (without `oci://` or tag). |
| `tag` | String | - | **Required.** Image tag. |

### Forwarded Properties

The following Flux `OCIRepository` attributes can be overridden in this section and will be passed through to the generated resource:

- `interval`
- `timeout`
- `secretRef`
- `certSecretRef`
- `proxySecretRef`
- `provider`
- `verify`
- `serviceAccountName`
- `insecure`
- `suspend`

See [Flux OCIRepositorySpec](https://fluxcd.io/flux/components/source/api/v1/#source.toolkit.fluxcd.io/v1.OCIRepositorySpec){:target="_blank"} for details.

---

## Release.spec.debug

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `dumpContext` | Bool | `false` | If `true`, saves the fully resolved context into the `status` field. Use sparingly as contexts can be large. (Prefer `kubocd render` for debugging). |
| `dumpParameters` | Bool | `false` | If `true`, saves resolved parameters into the `status` field. |

---

## CrossNamespaceReference

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `name` | String | **Yes** | Resource name. |
| `namespace` | String | No | Resource namespace. Defaults to local namespace if omitted. |
