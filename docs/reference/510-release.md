# The Release Kubernetes resources


## Release

### apiVersion

**String, required:** Always `kubocd.kubotal.io/v1alpha1`

### kind

**String, required:** Always `Release'

### metadata

**Map, required:** Refer to the Kubernetes API documentation for the fields of the metadata field.

### spec

**Release.spec, required:** See [Release.spec](./#releasespec) below

---

## Release.spec

### description

**String, default: {Package.description}:** A short description. 

### package

**Release.spec.package, required:** Where to fetch the `Package` OCI image. Description in [Release.spec.package](./#releasespecpackage)

### contexts

**List(CrossNamespaceReference):** A list of supplementary contexts merged with the default ones. 
If the `namespace` of the context is not defined, it will take the `Release.metadata.namespace`.

### protected

**Bool, Default: {Package.protected}:** If true, prevent deletion. Need the KuboCD webhook to be effective

### parameters

**Template(Map), optional:** The deployments parameters. Must comply to the `Package.parameters.schema`. 
It is a template, thus allowing variable substitution. But, only the root elements `.Context` is present in the data model. 

### specPatchByModule

**Map(Map), optional:** A patch applied to the `spec` section of the Flux `HelmRelease` resource of the specified module.
It allows you to set any parameters that are not exposed by KuboCD at the `Release` level

You will find an example [here](https://github.com/kubocd/kubocd-doc/blob/main/samples/releases/redis3-basic.yaml) 
(Modification of the `helmRelease` timeout for the `redis` module.)

### targetNamespace

**String, default: {Release.metadata.namespace}:** The namespace to deploy the application into

### createNamespace

**Bool, default: false:** Allow `targetNamespace` creation if it does not exists.

### roles

**List(string), default: []:** This list is appended to the [`Package.roles`](./500-package.md/#roles) list

### dependencies

**List(string), default: []:** This list is appended to the [`Package.dependencies`](./500-package.md/#dependencies) list

### skipDefaultContext

**Bool, default: false:** If set, this `Release` will ignore the global default contexts as well as those defined at the namespace level.
Only the contexts explicitly listed in the `Release` resource will be taken into account.

### debug

**Release.spec.debug, optional:** Refer to [Release.spec.debug](./#releasespecdebug) below



---

## Release.spec.package

### repository

**String, required:** Part of OCI url oci://<repository>:<tag>

### tag

**String, required:** Part of OCI url oci://<repository>:<tag>

### Other attributes

When a `Release` resource is created, a Flux `OCIRepository` resource is also created to reference the Package's OCI image.
This object supports many configuration attributes. These attributes can be overridden at the Release level and will be passed through as-is.

Refer to the [Flux documentation for OCIRepositorySpec](https://fluxcd.io/flux/components/source/api/v1beta2/#source.toolkit.fluxcd.io/v1beta2.OCIRepositorySpec){:target="_blank"}
for full details.

Here is the list of supported forwarded attributes:

- interval
- timeout
- secretRef
- certSecretRef
- proxySecretRef
- provider
- verify 
- serviceAccountName
- insecure
- suspend

---

## Release.spec.debug

### dumpContext

**Bool, default: false:** If set, the computed context is saved in the `status` field. This can be useful for debugging. 
Anyway, the context may become quite large. Use this debug mode sparingly. You may prefer to use the [’render`](../user-guide/180-kubocd-cli.md/#kubocd-render) CLI command.

### dumpParameters

**Bool, default: false:** If set, the computed parameters are saved in the `status` field. Useful in case `Release.spec.parameters` is a template.

---

## CrossNamespaceReference

### name

**String, required:**

### namespace

**String, optional:**
