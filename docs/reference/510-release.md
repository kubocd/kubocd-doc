# The Release Kubernetes Resource

## Release

### apiVersion

**String, Required**

Always `kubocd.kubotal.io/v1alpha1`.

### kind

**String, Required**

Always `Release`.

### metadata

**Map, Required**

Standard Kubernetes metadata (name, namespace, labels, etc.).

### spec

**Release.spec, Required**

See [Release.spec](#releasespec) below.

---

## Release.spec

### description

**String, Default: {Package.description}**

A short description of the deployment.

### package

**Release.spec.package, Required**

Defines the `Package` OCI image to deploy. See [Release.spec.package](#releasespecpackage).

### contexts

**List(CrossNamespaceReference)**

Additional contexts to merge with the defaults.
If `namespace` is omitted, `Release.metadata.namespace` is used.

### protected

**Bool, Default: {Package.protected}**

If true, prevents deletion of the Release (requires KuboCD webhook).

### parameters

**Template(Map), Optional**

Deployment parameters matching `Package.parameters.schema`.
> Note: Only the `.Context` root element is available for templating in this section.

### targetNamespace

**String, Default: {Release.metadata.namespace}**

The namespace where the application workload is deployed.

### createNamespace

**Bool, Default: false**

If true, creates `targetNamespace` if it does not exist.

### roles

**List(string), Default: []**

Appended to the [`Package.roles`](./500-package.md/#roles) list.

### dependencies

**List(string), Default: []**

Appended to the [`Package.dependencies`](./500-package.md/#dependencies) list.

### skipDefaultContext

**Bool, Default: false**

If true, ignores global and namespace-level default contexts. Only explicit contexts in `spec.contexts` are used.

### modulesOverrides

**List(Release.spec.moduleOverride), Optional**

Overrides specific settings for individual modules. See [Release.spec.moduleOverride](#releasespecmoduleoverride).

### debug

**Release.spec.debug, Default: false**

See [Release.spec.debug](#releasespecdebug).

---

## Release.spec.moduleOverride

### module

**String, Required**

The name of the module to modify.

### strategy

**String, Optional**

Overrides the failure handling strategy. See [Deployment Failure](../user-guide/220-deployment-failure.md).

### timeout

**Template(duration), Optional**

Overrides the Helm deployment timeout.

### interval

**Template(duration), Optional**

Overrides the Helm reconciliation interval.

### specPatch

**Map, Optional**

A patch applied to the `spec` of the generated `HelmRelease` for this module.

---

## Release.spec.package

### repository

**String, Required**

OCI repository URL (without `oci://` or tag).

### tag

**String, Required**

Image tag.

### forwarded properties

The following Flux `OCIRepository` attributes can be overridden here:

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

### dumpContext

**Bool, Default: false**

If true, saves the fully resolved context into the `status` field. Use sparingly as contexts can be large. (Prefer `kubocd render` for debugging).

### dumpParameters

**Bool, Default: false**

If true, saves resolved parameters into the `status` field.

---

## CrossNamespaceReference

### name

**String, Required**

Resource name.

### namespace

**String, Optional**

Resource namespace. Defaults to local namespace if omitted.
