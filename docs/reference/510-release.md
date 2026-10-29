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
| `protected`          | Bool | `{Package.protected}` | If `true`, prevents deletion of the Release (requires KuboCD webhook). When it is not set, the deletion is refused until the controller has written the protection of the package in the status. See [Protection](#protection). |
| `parameters`         | Template(Map) | `null` | Deployment parameters matching `Package.parameters.schema`. <br>_Note: Only the `.Release`, `.Context` and `.ImageRedirects` root elements are available for templating in this section. See [The data model](./550-template-functions.md#the-data-model)._ |
| `targetNamespace`    | String | `{Release.metadata.namespace}` | The namespace where the application workload is deployed. |
| `createNamespace`    | Bool | `false` | If `true`, creates `targetNamespace` if it does not exist. |
| `roles`              | List(String) | `[]` | Appended to the [`Package.roles`](./500-package.md/#roles) list. |
| `dependencies`       | List(String) | `[]` | Appended to the [`Package.dependencies`](./500-package.md/#dependencies) list. |
| `skipDefaultContext` | Bool | `false` | If `true`, ignores global and namespace-level default contexts. Only explicit contexts in `spec.contexts` are used. |
| `modulesOverrides`   | List([Release.spec.moduleOverride](#releasespecmoduleoverride)) | `[]` | Overrides specific settings for individual modules. |
| `debug`              | [Release.spec.debug](#releasespecdebug) | `false` | Debugging options. |

---

## Protection

With the webhook of the `kubocd-wh` chart, the deletion of a `Release` follows `spec.protected` when it is set, and
the protection of its package otherwise:

- `spec.protected: true`: the deletion is refused with `release <name> is protected`.
- `spec.protected: false`: the deletion is accepted.
- `spec.protected` not set: the controller computes the protection from `Package.protected` once it has loaded the
  package, and writes it in the status of the `Release`. `status.printProtected`, the `PRT` column of
  `kubectl get releases`, then shows `X` for a protected `Release`, whose deletion is refused as above, and `-`
  otherwise. While the column is empty, the deletion is refused too:

    ```
    Error from server (Forbidden): admission webhook "vrelease-v1alpha1.kb.io" denied the request: release podinfo1 may be protected by its package, and the controller has not computed its protection yet: set spec.protected to false to delete it anyway
    ```

Without the webhook, nothing refuses the deletion of a `Release`.

A `Release` that has never loaded a package, for instance one that stays in `WAIT_OCI` from its creation, keeps an
empty `PRT` column. Setting `spec.protected: false` on it is an update, which the webhook does not check. The deletion
is then accepted, and the controller completes it without the package.

A namespace stays `Terminating` while the webhook refuses the deletion of one of its `Release` resources, protected or
with an empty `PRT` column. The `NamespaceDeletionContentFailure` condition of the namespace gives the message of the
webhook. Setting `spec.protected: false` on each of these `Release` resources lifts their refusal. A `Context` with
`protected: true` holds the namespace in the same way.

When `spec.protected` is not set, the webhook reads the protection that the controller wrote last. After a change of
the package, or once `spec.protected` is removed, the deletion follows the previous protection until the controller has
loaded the package again and written the new protection in the status.

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

The following Flux `OCIRepository` attributes can be set in this section. They are passed to the generated resource
subject to the source policy below and any matching [packageRedirect](./530-config.md#packageredirect):

- `interval`, `5m` by default, set by the API server. A value of `0s` takes the
  [`defaultPackageInterval`](./530-config.md#configspec) of the `Config` resources.
- `timeout`
- `secretRef`
- `certSecretRef`
- `proxySecretRef`
- `provider`
- `verify`
- `serviceAccountName`
- `insecure`
- `suspend`

A Release with `insecure: true` and no matching `packageRedirect` is refused with
`Release.spec.package.insecure is not allowed without a matching packageRedirect`. The controller schedules no retry
on this refusal. Use TLS, configure a matching redirect, or set `--allowReleaseInsecureSource=true` in the controller
and `kubocd render` to allow direct HTTP sources. With the `kubocd-ctrl` chart, set the flag in
`controller.additionalArgs`. A matching redirect uses its own `insecure` setting, regardless of the Release's value.

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
