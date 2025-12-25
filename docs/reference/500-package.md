# The KuboCD Package Resource

A `Package` is an OCI-compliant container image that bundles an application descriptor and one or more Helm charts. As the standardized unit of deployment, it encapsulates everything needed to install and configure an application.

See [A first deployment](../user-guide/130-a-first-deployment.md) for an introductory example.

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

### *apiVersion*

**String, Required**

The API version. Currently, the only supported value is `v1alpha1`.

### *type*

**String, Optional**

The resource type. Must be `Package` if specified. Can be omitted.

### *name*

**String, Required**

The package name. Used in the OCI repository name. Must be a valid DNS Subdomain name ([RFC 1123](https://tools.ietf.org/html/rfc1123){:target="_blank"}).

### *tag*

**String, Required**

The specific tag for the OCI image.

### description

**Template(string), Optional**

A short description of the package. Acts as the default description for the corresponding `Release`.

### usage

**Template(string), Optional**

Intended to provide usage information (e.g., access URLs, credentials) after deployment. Can be displayed by UI tools.

### protected

**Bool, Default: false**

If true, prevents accidental deletion. Acts as the default for the corresponding `Release`.

### schema.parameters

**Map, Optional**

A schema to validate the `Parameters` provided in the `Release`. Supports:
- Standard JSON/OpenAPI schema.
- [KuboCD simplified schema](../user-guide/190-alternate-schema-format.md).

If omitted, the `Release` accepts no parameters.

### schema.context

**Map, Optional**

A schema to validate the injected `Context`. Supports:
- Standard JSON/OpenAPI schema.
- [KuboCD simplified schema](../user-guide/190-alternate-schema-format.md).

If omitted, no context validation is performed.

### modules

**List([Modules](#packagemodule)), Required**

The list of modules embedded in this package.

### roles

**Template(List(string)), Optional**

The roles this application fulfills. See [Release Dependencies and Roles](../user-guide/200-redis.md/#release-dependencies-and-roles).

### dependencies

**Template(List(string)), Optional**

The roles this application depends on. See [Release Dependencies and Roles](../user-guide/200-redis.md/#release-dependencies-and-roles).

---

## Package.module

A `Module` defines a Helm Chart deployment.

### name

**String, Required**

Unique name for the module within the package. Used as a suffix for the generated `HelmRelease` name. Must be a valid DNS Subdomain name.

- **Special Value**: `noname`. If used, the Flux `HelmRelease` name matches the KuboCD `Release` name exactly (no suffix).

### source

**Map, Required**

Defines where to fetch the Helm chart. Must contain exactly one of:

- [helmRepository](#packagemodulesourcehelmrepository) 
- [git](#packagemodulesourcegit)
- [oci](#packagemodulesourceoci)
- [local](#packagemodulesourcelocal)

### values

**Template(map), Optional**

A template for generating the Helm `values.yaml`. See [A first deployment](../user-guide/130-a-first-deployment.md).

### enabled

**Template(bool), Default: true**

Conditional deployment. If `false`, the module is skipped entirely.

### targetNamespace

**Template(string), Default: {{ .Release.spec.targetNamespace }}**

The namespace where the module is deployed. Can be overridden at the `Release` level.

### onFailureStrategy

**Template(string), Optional**

Defines the strategy to use in case of failure. Strategies are defined in the global KuboCD configuration. See [Deployment Failure](../user-guide/220-deployment-failure.md).

### timeout

**Template(duration), Default: {config.defaultHelmTimeout}**

Sets `spec.timeout` in the generated `HelmRelease`.

### interval

**Template(duration), Default: {config.defaultHelmInterval}**

Sets `spec.interval` in the generated `HelmRelease` (reconciliation frequency).

### dependsOn

**Template(list(string)), Optional**

A list of other module names within the same package that this module depends on.

### specPatch

**Template(map), Optional**

Arbitrary patch applied to the `spec` of the generated Flux `HelmRelease`. Useful for setting Flux properties not directly exposed by KuboCD.

---

## Package.module.source.helmRepository

Fetches a Chart from an HTTP(S) Helm Repository.

### url

**String, Required**

The repository URL.

### chart

**String, Required**

The chart name.

### version

**String, Required**

The chart version.

---

## Package.module.source.git

Fetches a Chart from a Git Repository.

### url

**String, Required**

The repository URL.

### branch

**String, Required if tag is omitted**

Git branch to fetch.

### tag

**String, Required if branch is omitted**

Git tag to fetch.

### path

**String, Required**

Relative path to the chart directory (containing `Chart.yaml`).

### extraFilePrefixes

**List(string), Optional**

KuboCD normally filters files to include only standard Chart content. This attribute allows including extra files/folders starting with the specified prefixes (relative to the chart path, leading `/` required).

Example:
```yaml
extraFilePrefixes:
  - /ci
  - /files
```

---

## Package.module.source.oci

Fetches a Chart stored as an OCI artifact.

### repository

**String, Required**

The repository URL (without `oci://` prefix or tag).

### tag

**String, Required**

The image tag.

### insecure

**Bool, Default: false**

If true, allows unencrypted (HTTP) connections.

---

## Package.module.source.local

Fetches a Chart from the local filesystem (relative to the Package manifest).

### path

**String, Required**

Relative path to the chart directory.
