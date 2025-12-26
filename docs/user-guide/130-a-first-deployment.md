
# A First Deployment with KuboCD

## Package Definition

For this initial deployment, we will use a simple example: a lightweight web application called [**podinfo**](https://github.com/stefanprodan/podinfo){:target="_blank"}.

A **Package** in KuboCD is defined using a YAML manifest. Below is an example that wraps the `podinfo` application:

???+ abstract "podinfo-p01.yaml"

    ``` { .yaml .copy }
    apiVersion: v1alpha1
    type: Package
    name: podinfo
    tag: 6.7.1-p01
    schema:
      parameters:
        $schema: http://json-schema.org/schema#
        additionalProperties: false
        properties:
          fqdn:
            type: string
          ingressClassName:
            default: nginx
            type: string
        required:
          - fqdn
        type: object
    modules:
      - name: main
        source:
          helmRepository:
            url: https://stefanprodan.github.io/podinfo
            chart: podinfo
            version: 6.7.1
        values: |
          ingress:
            enabled: true
            className: {{ .Parameters.ingressClassName }}
            hosts:
              - host: {{ .Parameters.fqdn }}
                paths:
                  - path: /
                    pathType: ImplementationSpecific
    ```

> A KuboCD Package is **NOT** a native Kubernetes resource.

!!! tip
    You can find most samples used in this documentation in the [samples directory](https://github.com/kubocd/kubocd-doc/tree/main/samples){:target="_blank"}.

### Attribute Description

- **`apiVersion`** (Required): Defines the version of the KuboCD Package format. Currently, the only supported value is `v1alpha1`.
- **`type`**: Specifies the resource type. Must be `Package` (default).
- **`name`**: The package name. This is used as the OCI image name.
- **`tag`** (Required): The version tag of the OCI image. We recommend the following convention:
    - Use the Helm chart version of the main module as a base.
    - Append `-pXX`, where `XX` denotes the packaging revision (e.g., configuration changes for the same chart version).
- **`schema.parameters`**: Defines input parameters for the package using standard OpenAPI/JSON Schema. This enables validation and documentation at deployment time.
    - If omitted, the release will not accept parameters.
- **`modules`** (Required): A package contains one or more Helm charts, each defined as a module.
    - **`modules[X].name`** (Required): A unique name for the module. Here, we use `main`.
    - **`modules[X].source`** (Required): The source of the Helm chart. This example uses a Helm repository, but OCI registries, Git repositories, or local charts are also supported.
    - **`values`**: A template rendered into `values.yaml` for Helm.
        - Uses the same templating engine as Helm.
        - The data model includes a `.Parameters` object containing values provided during deployment (via the `Release` object).
        - Although strictly a string, it is written as YAML block to allow full templating flexibility.

!!! note "Required Fields"
    Attributes marked **(Required)** are mandatory.

!!! tip
    More attributes and advanced features will be introduced later in the documentation.

---

## Package Build

Once the package definition is complete, the next step is to generate the OCI image.

KuboCD uses an OCI-compatible container registry to store and distribute packages. You must have push access to a registry.

**Tested registries:**

- `quay.io` (Red Hat)
- `ghcr.io` (GitHub)
- [distribution registry](https://github.com/distribution/distribution){:target="_blank"}

Other OCI-compliant registries should also work. 

> **Docker Hub is currently not supported.**

Ensure you are authenticated with your registry:

``` { .bash .copy }
docker login quay.io
```

or

``` { .bash .copy }
docker login ghcr.io
```

!!! tip "Authentication"

    If interactive authentication is not possible (e.g., in CI/CD pipelines), KuboCD allows providing credentials via environment variables: `KCD_OCI_{REGISTRY}_USER` and `KCD_OCI_{REGISTRY}_SECRET`.
    
    Replace `{REGISTRY}` with an identifier for the target registry, constructed as follows:
    - Replace characters `.`, `:`, `/`, and `-` with `_`
    - Convert to uppercase

    For example, for `quay.io`, use `KCD_OCI_QUAY_IO_USER` and `KCD_OCI_QUAY_IO_SECRET`.
    
    *Note: The older `KCD_OCI_USER` and `KCD_OCI_SECRET` variables are deprecated.*

    **Tip:** Run the `package` command with `--logLevel debug` to see the expected environment variable names.

Depending on the registry, the image may be pushed under an organization or namespace. Here, we use `quay.io/kubodoc`.

To build and push the package image:

``` { .bash .copy }
kubocd package packages/podinfo-p01.yaml --ociRepoPrefix quay.io/kubodoc/packages
```

!!! note
    Adjust `--ociRepoPrefix` to match your registry setup.<br>The `packages` suffix is optional.

The resulting repository and tag are derived from the package manifest:

- Repository = `--ociRepoPrefix` + package `name`
- Tag = package `tag`

**Expected Output:**

``` bash
====================================== Packaging package 'podinfo-p01.yaml'
--- Handling module 'main':
Fetching chart podinfo:6.7.1...
Chart: podinfo:6.7.1
--- Packaging
Generating index file
Wrap all in assembly.tgz
--- push OCI image: quay.io/kubodoc/packages/podinfo:6.7.1-p01
Successfully pushed
```

You can set the repository prefix globally via an environment variable to avoid repeating the flag:

```bash { .bash .copy }
export OCI_REPO_PREFIX=quay.io/kubodoc/packages
```

Then simply run:

``` { .bash .copy }
kubocd package packages/podinfo-p01.yaml
```

!!! warning "Image Visibility"
    By default, pushed images may be private. To allow deployment without extra configuration, ensure the image is **public**.
    <br>
    If you keep the image private, you must provide authentication credentials in the `Release` configuration (explained later).

---

## Releasing the Application

To deploy the application, define a KuboCD `Release` custom resource:

???+ abstract "podinfo1-basic.yaml"

    ``` { .yaml .copy }
    ---
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Release
    metadata:
      name: podinfo1
      namespace: default
    spec:
      description: A first sample release of podinfo
      package:
        repository: quay.io/kubodoc/packages/podinfo
        tag: 6.7.1-p01
        interval: 30m
      parameters:
        fqdn: podinfo1.ingress.kubodoc.local
    ```

### Attribute Description

- **`spec.description`**: (Optional) A short description.
- **`spec.package.repository`**: The OCI image repository containing the package.
- **`spec.package.tag`**: The image tag, matching the package manifest.
- **`spec.package.interval`**: (Optional) How frequently KuboCD checks for updates (e.g., `30m`).
- **`spec.parameters`**: The values required by the package schema. Here, only `fqdn` is needed.

### deploy

1. Adjust the repository and parameters manifest if needed.
2. Apply the Release:

``` { .bash .copy }
kubectl apply -f releases/podinfo1-basic.yaml
```

Monitor the status:

```{ .bash .copy }
kubectl get releases
```

```bash
NAME       REPOSITORY                         TAG         CONTEXTS   STATUS   READY   WAIT   PRT   AGE     DESCRIPTION
podinfo1   quay.io/kubodoc/packages/podinfo   6.7.1-p01              READY    1/1            -     6m40s   A first sample release of podinfo
```

Verify the pod is running:

```{ .bash .copy }
kubectl get pods
```

```bash
NAME                             READY   STATUS    RESTARTS   AGE
podinfo1-main-779b6b9fd4-zbgbx   1/1     Running   0          8h
```

## Preflight Check

Before deploying, it is good practice to test the `Release` using the `render` CLI command:

```{ .bash .copy }
kubocd render releases/podinfo1-basic.yaml
```

See the [KuboCD CLI section](./180-kubocd-cli.md/#kubocd-render) for details.
