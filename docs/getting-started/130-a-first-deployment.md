
# A First Deployment with KuboCD

## Package Definition

For this initial deployment, we’ll use a simple and illustrative example: a tiny web application called [**podinfo**](https://github.com/stefanprodan/podinfo).

A **Package** in KuboCD is defined using a YAML manifest. Below is an example that wraps the `podinfo` application:

???+ abstract "podinfo-p01.yaml"
    ```yaml
    apiVersion: v1alpha1
    type: Package
    name: podinfo
    tag: 6.7.1-p02
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
        specPatch:
          timeout: 2m
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

!!! note
    A KuboCD Package is **not** a native Kubernetes resource.

Description of the Sample Package Attributes:

- **`apiVersion`** (Required): Defines the version of the KuboCD Package format. The only supported value currently is `v1alpha1`.
- **`type`**: Specifies the resource type. It must be `Package`, which is also the default and can be omitted.
- **`name`**: The name of the package. This will also be used in the OCI image name.
- **`tag`** (Required): Specifies the version tag of the OCI image. While technically flexible, it is recommended to follow this convention:
    - Use the Helm chart version as a base, followed by `-pXX` where `XX` denotes the packaging revision (e.g. different configurations for the same chart).
- **`schema.parameters`**: Defines input parameters for the package, using a standard OpenAPI/JSON Schema. This enables validation and documentation of parameters at deployment time.
    - If not defined, the release will not accept parameters.
- **`modules`** (Required): A package contains one or more Helm charts, each represented as a module.
    - **`modules[X].name`** (Required): A unique name for the module. In this example, there's only one module, called `main`.
    - **`modules[X].source`** (Required): Defines where to find the Helm chart. In this example, it's in a Helm repository, but it could also come from an OCI registry, Git repository, or local chart.
    - **`values`**: This is a template rendered into a `values.yaml` for Helm. 
        - The templating engine is the same as Helm’s.
        - The data model, however, differs. It includes a `.Parameters` object containing the values provided during deployment (via the `Release` object).
        - Though it appears as YAML, it is actually a string, allowing full templating flexibility.

!!! note "Required Fields"
    Any attribute marked with **(Required)** must be specified for the package to be valid.

!!! tip
    More attributes and advanced features will be introduced later in the documentation.

---

## Package Build

Now that the package definition is complete, it’s time to generate the corresponding OCI image.

As mentioned earlier, KuboCD uses an OCI-compatible container registry to store and distribute packages. You'll need access to one with permission to push images.

✅ Supported registries:
- `quay.io` (Red Hat)
- `ghcr.io` (GitHub)
- [distribution registry](https://github.com/distribution/distribution)

⚠️ **Note**: Docker Hub is **not supported** at the moment.

Make sure you're authenticated with the registry, e.g.:

```bash
docker login quay.io
```

Depending on the registry, the image may need to be pushed under an organization or namespace. For this example, we'll use `quay.io/kubodoc`.

To build and push the package image:

```bash
kubocd package podinfo-p01.yaml --ociRepoPrefix quay.io/kubodoc/packages
```

!!! note
    Adjust `--ociRepoPrefix` to your own registry setup. The `packages` suffix is arbitrary — you can use any subpath or omit it.

The resulting repository and tag are determined from the package manifest:

- Repository = `--ociRepoPrefix` + package `name`
- Tag = package `tag`

Expected Output:

```text
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

You can also set the repository prefix globally via an environment variable:

```bash
export OCI_REPO_PREFIX=quay.io/kubodoc/packages
```

Or for other registries:

```bash
export OCI_REPO_PREFIX=ghcr.io/kubodoc/packages
export OCI_REPO_PREFIX=localhost:5000/packages
```

!!! warning
    By default, pushed images may be private. To make them accessible for deployment, ensure the image is set to **public**.

!!! note
    If you prefer to keep the image private, you will need to provide authentication credentials in the `Release` configuration. This is explained later in the documentation.

---

## Releasing the Application

To deploy the application, define a KuboCD `Release` custom resource:

???+ abstract "podinfo1.yaml"
    ```yaml
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

Explanation of Key Attributes:

- **`description`**: (Optional) A short description of this release.
- **`package.repository`**: The OCI image repository that contains the package. This should match the registry used during package build.
- **`package.tag`**: The image tag, which should match the one defined in the package manifest.
- **`package.interval`**: Specifies how frequently KuboCD checks the registry for updates to the image.
- **`parameters`**: The values required by the package schema. In this example, only a single parameter (`fqdn`) is needed.

Deploying the Application:

1. Adjust the repository and parameters (if needed) to match your environment.
2. Apply the Release:

```bash
kubectl create -f podinfo1.yaml

release.kubocd.kubotal.io/podinfo1 created
```

Once deployed, monitor the status:

```bash
kubectl get releases

NAME       REPOSITORY                         TAG         CONTEXTS   STATUS   READY   WAIT   PRT   AGE     DESCRIPTION
podinfo1   quay.io/kubodoc/packages/podinfo   6.7.1-p01              READY    1/1            -     6m40s   A first sample release of podinfo
```

You can also verify the pod:

```bash
kubectl get pods

NAME                             READY   STATUS    RESTARTS   AGE
podinfo1-main-779b6b9fd4-zbgbx   1/1     Running   0          8h
```
