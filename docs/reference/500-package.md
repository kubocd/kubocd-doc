# The KuboCD Package

A `Package` is an OCI-compliant container image that bundles an application descriptor along with one or more Helm charts. 
It serves as the standardized unit of deployment, encapsulating everything needed to describe and install an application.

Refer to [A first deployment](../user-guide/130-a-first-deployment.md) for an introductory example.

The OCI image is built from a manifest, whose structure is described below.

## Templating

Some attributes can accept a template, rendering the final value. There are noted as the following:

- template(string): A template which must render a string.
- template(bool): A template that must render a string convertible to a boolean value. (1, t, T, TRUE, true, True, 0, f, F, FALSE, false, False)
- template(list(string)): A template which must render a list of string
- template(map): A template which must render a map/object. 
- template(duration): A template that must render a string convertible to a duration value.

---

## Package

### *apiVersion*

**String, required**

The manifest version. Only allowed value is currently `v1alpha1`

### *type*

**String, optional**

The manifest type. Only allowed value is `Package`. May be omitted.

### *name*

**String, required**

The name of the package. Will be used in the OCI repository name. Must be a valid DNS Subdomain name ([RFC 1123](https://tools.ietf.org/html/rfc1123){:target="_blank"}).

### *tag*

**String, required**

The tag for the OCI image.

### description

**Template(string), optional**

A short description of the package. Will act as default for its `Release` counterpart. 

### usage

**Template(string), optional**

A template aimed to be rendered on deployment. Intended to provide user with usage information (Access link, configuration, ....).

To be exploited by some UI tool.

### protected

**Bool, Default: false**

Prevent deletion. Will act as default for its `Release` counterpart

### schema.parameters

**Map, Optional**

Allow validation or the `Parameters` defined in the `Release` object. It could be a JSON/OpenAPI 
schema or a [KuboCD specific format](../user-guide/190-alternate-schema-format.md).

If none is provided, this means the `Release` will not accept any parameters.

### schema.context

**Map, Optional** 

Allow validation or the `Context` of the deployment. It could be a JSON/OpenAPI
schema or a [KuboCD specific format](../user-guide/190-alternate-schema-format.md).

If none is provided, there will be no control on provided `Context`

### modules

**List([Modules](#packagemodule)), Required:**

The list of [modules](#packagemodule) included in this package.

### roles

**Template(List(string)), optional**

The roles this package aims to fulfill. See [Release Dependencies and Roles](../user-guide/200-redis.md/#release-dependencies-and-roles)

### dependencies

**Template(List(string)), optional**

The roles this package depends on. See [Release Dependencies and Roles](../user-guide/200-redis.md/#release-dependencies-and-roles)

---

## Package.module 

A `Module` embed an Helm Chart

### name

**String, required** 

The module name. Must be unique for a package. Used as postfix for the name of the generated HelmRelease, 
so it must be a valid DNS Subdomain name ([RFC 1123](https://tools.ietf.org/html/rfc1123)) 

There is a specific value `noname`, which make the FluxCD `HelmRelease` name identical to the KuboCD `Release` name.

### source

**Map, required:**

Provide the location from which to fetch the Helm chart. Must contain of the following sub-elements:

- [helmRepository](#packagemodulesourcehelmrepository) 
- [git](#packagemodulesourcegit)
- [oci](#packagemodulesourceoci)
- [local](#packagemodulesourcelocal)

### values

**Template(map), Optional**

The template rendering the 'values file' used to deploy the Helm Chart. Refer to
[A first deployment](../user-guide/130-a-first-deployment.md) for an simple first example.

### enabled

**Template(bool), default: true**

When set to false, the module is not deployed at all. See []() for an example.

### targetNamespace

**Template(string), default: {{ .Release.spec.targetNamespace }}**

The namespace to deploy the application into.

May be overridden at the release level

### onFailureStrategy

**Template(string), optional**

Define the strategy to use in case of Helm deployment failure. Strategies are defined in the KuboCD global configuration. See the [dedicated chapter](../user-guide/220-deployment-failure.md) 

May be overridden at the release level

### timeout

**Template(duration), default: {config.defaultHelmTimeout}**

Will be set as `spec.timeout` of the generated HelmRelease, thus providing timeout on Helm deployment.

May be overridden at the release level

### interval

**Template(duration), default: {config.defaultHelmInterval}**

Will be set as `spec.interval` of the generated HelmRelease, interval at which to reconcile the Helm release.

May be overridden at the release level

### dependsOn

**Template(list(string)), optional** 

A list of other modules of this package we depends on.

### specPatch

**Template(map), optional:**

A patch applied to the `spec` section of the Flux `HelmRelease` resource. 
It allows you to set any parameters that are not exposed by KuboCD.

---

## Package.module.source.helmRepository

Use this section to fetch an Helm Chart stored on an Helm Repository.

### url

**String, required**

The URL of the Helm repository

### chart

**String, required** 

The Chart name

### version

**String, required**

The Chart version

---

## Package.module.source.git

Use this section to fetch an Helm Chart stored on an Git Repository.

### url

**String, required**

The URL of the Git repository

### branch

**String, required if tag is not defined**

The branch to fetch

### tag

**String, required if branch is not defined**

The tag to fetch

### path

**String, required**

The folder inside the Git repository of the Chart (Where is `Chart.yaml` is located)

### extraFilePrefixes

**String, optional**

KuboCD packaging filter files to only include chart content, as defined [here](https://helm.sh/docs/v3/topics/charts){:target="_blank"}.

Some helm charts require extra files to be included. This attribute defines a list of prefixes for files which should also be included. This starting from the `Path` referenced above, with a leading '/'.

As an example, here is a extract from a package definition for [argo-workflows community Helm chart](https://github.com/argoproj/argo-helm/tree/main/charts/argo-workflows){:target="_blank"}.

```
apiVersion: v1alpha1
name: argo-workflows
....
modules:
  - name: noname
    source:
      git:
        url: https://github.com/argoproj/argo-helm
        tag: argo-workflows-0.46.2
        path: /charts/argo-workflows
        extraFilePrefixes:
          - /ci
          - /files
```

---

## Package.module.source.oci

Use this section to fetch an Helm Chart stored as an OCI image.

### repository

**String, required**

The repository URL without `oci://` and `tag`.

### tag

**String, required**

The image tag.

### insecure

**Bool, default: false**

Repository access is in clear text (non encrypted) 

---

## Package.module.source.local

Use this section to fetch an Helm Chart stored on your workstation, alongside the `Package` manifest.


### path

**String, required**

The local folder of the Chart (Where is `Chart.yaml` is located). Relative to the `Package`
manifest location
