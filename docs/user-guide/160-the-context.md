# The Context

One of the key features of KuboCD is its ability to generate Helm deployment values files from a small set of high-level input parameters, using a templating mechanism.

This mechanism combines a template with a data model.

Our first example uses only the `.Parameters` element of the data model:

???+ abstract "podinfo-p01.yaml"

    ``` { .yaml }
    apiVersion: v1alpha1
    type: Package
    name: podinfo
    ...
    modules:
      - name: main
        ...
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

Actually, the data model includes the following top-level elements:

- `.Parameters`: The parameters provided in the `Release` custom resource.
- `.Release`: The release object itself.
- `.Context`: The deployment context.

The context is a YAML object with a flexible structure, designed to hold shared configuration data relevant to all deployments.

For example, the `podinfo` package includes a parameter `ingressClassName` with a default value (`nginx`). If a cluster uses a different ingress controller, this value would need to be overridden for all relevant `Release` objects.

This type of shared configuration is best defined in a global cluster-level context.

Similarly, if all application ingress URLs share a common root domain, that too should be centralized.

Here's an initial example of how this logic can be implemented.

---

## Context Creation

A `Context` is a KuboCD resource:

???+ abstract "cluster.yaml"

    ``` { .yaml .copy }
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Context
    metadata:
      namespace: contexts
      name: cluster
    spec:
      description: Global context for the kubodoc cluster
      protected: true
      context:
        ingress:
          className: nginx
          domain: ingress.kubodoc.local
        storageClass: 
          data: standard
          workspace: standard
        certificateIssuer:
          public: cluster-self
          internal: cluster-self
    ```

Key attributes:

- `description`: A short description.
- `protected`: Prevents deletion of this object. Requires KuboCD's webhook feature.
- `context`: A tree of values that is injected into the data model for the templating of the `values` section. This section:
    - Must be valid YAML.
    - Has a flexible structure, but should align with what the `Package` templates expect.

In this example, the context includes:

- `ingress.className`: The ingress controller type.
- `ingress.domain`: The suffix used for building ingress URLs.
- `storageClass`: Two Kubernetes `StorageClass` definitions for different application profiles. For our `kind` based 
  cluster, there is only one available option: `standard`.
- `certificateIssuer`: Two certificate issuers, one to use internally and one intended for endpoints exposed to external 
  world. As configuring a CA (Certificate Authority) is out of the scope of this documentation, we will set-up only a 
  self-signed CA. This will be performed later with the chapter on [cert-manager deployment](./210-cert-manager.md) 

Cluster-wide contexts should be placed in a dedicated namespace:

``` {.bash .copy }
kubectl create ns contexts
```

``` {.bash .copy }
kubectl apply -f contexts/cluster.yaml
```

!!! note
    Since the context is shared among most of all applications, its structure must be carefully designed and well documented.

---

## Package modification

Our initial `podinfo` package did not account for the context concept. Here is an updated version:

???+ abstract "podinfo-p02.yaml"

    ``` { .yaml .copy }
    apiVersion: v1alpha1
    type: Package
    name: podinfo
    tag: 6.7.1-p02
    schema:
      parameters:
        $schema: http://json-schema.org/schema#
        type: object
        additionalProperties: false
        properties:
          host: { type: string }
        required:
          - host
      context:
        $schema: http://json-schema.org/schema#
        additionalProperties: true
        type: object
        properties:
          ingress:
            type: object
            additionalProperties: true
            properties:
              className: { type: string }
              domain: { type: string }
            required:
              - domain
              - className
        required:
          - ingress
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
            className: {{ .Context.ingress.className  }}
            hosts:
              - host: {{ .Parameters.host }}.{{ .Context.ingress.domain }}
                paths:
                  - path: /
                    pathType: ImplementationSpecific
    ```

Key points:

- The `tag` was updated to generate a new version.
- The `fqdn` parameter was replaced with `host` to represent only the hostname (excluding the domain).
- The `modules[X].values` section now uses the context.
- A `schema.context` section has been added to define and validate the expected context structure.

This new version must be packaged:

``` {.bash .copy }
kubocd pack packages/podinfo-p02.yaml
```

``` {.bash  }
====================================== Packaging package 'podinfo-p02.yaml'
--- Handling module 'main':
    Fetching chart podinfo:6.7.1...
    Chart: podinfo:6.7.1
--- Packaging
    Generating index file
    Wrap all in assembly.tgz
--- push OCI image: quay.io/kubodoc/packages/podinfo:6.7.1-p02
    Successfully pushed
```

---

## Deployment

Here is the corresponding `Release` manifest:

???+ abstract "podinfo2-ctx.yaml"

    ``` { .yaml .copy }
    ---
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Release
    metadata:
      name: podinfo2
      namespace: default
    spec:
      description: A first sample release of podinfo
      package:
        repository: quay.io/kubodoc/packages/podinfo
        tag: 6.7.1-p02
        interval: 30m
      parameters:
        host: podinfo2
      contexts:
        - namespace: contexts
          name: cluster
    ```

Key points:

- The `fqdn` parameter was replaced with `host`.
- A new `spec.contexts` section lists the contexts to merge into a single object passed to the template engine.

!!! warning
    Referencing a non-existent context results in an error.

Once `spec.repository` is set according to your repository, apply the deployment:

``` {.bash .copy }
kubectl apply -f releases/podinfo2-ctx.yaml
```

Check that the new `Release` reaches the `READY` state:

``` { .bash .copy }
kubectl get releases podinfo2
```

``` { .bash }
NAME       REPOSITORY                         TAG         CONTEXTS           STATUS   READY   WAIT   PRT   AGE   DESCRIPTION
podinfo2   quay.io/kubodoc/packages/podinfo   6.7.1-p02   contexts:cluster   READY    1/1            -     17m   A first sample release of podinfo
```

And check the corresponding `ìngress` has been configured properly:

``` { .bash .copy }
kubectl get ingresses podinfo2-main
```

``` { .bash }
NAME            CLASS   HOSTS                            ADDRESS      PORTS   AGE
podinfo2-main   nginx   podinfo2.ingress.kubodoc.local   10.96.59.9   80      120m
```

!!! notes
    If you want to test access through this ingress, don't forget to update your `/etc/host` or your DNS.


---

## Context Aggregation

An application's effective context may result from the aggregation of multiple context objects.

For instance, a project-level context can be created to share variables across all applications within a project. This will be merged with the global cluster context.

In the following examples, each deployed project has its own namespace and context.

### Example 1: Context merge

Create the namespace:

``` {.bash .copy }
kubectl create namespace project01
```

Then create the project context:

???+ abstract "project01.yaml"

    ``` { .yaml .copy }
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Context
    metadata:
      name: project01
    spec:
      description: Context for project 1
      context:
        project:
          id: p01
          subdomain: prj01
    ```

Note that the `namespace` is not specified in the manifest. It will be set via the command line:

``` { .bash .copy }
kubectl -n project01 apply -f contexts/project01.yaml
```

List all defined contexts:

``` { .bash .copy }
kubectl get --all-namespaces contexts.kubocd.kubotal.io
```

``` { .text }
NAMESPACE   NAME        DESCRIPTION                          PARENTS   STATUS   AGE
contexts    cluster     Global context for the kubodoc cluster          READY    2d2h
project01   project01   Context for project 1                           READY    2m35s
```

This example requires modifying the package to include the new variable `project.subdomain` in the `values` template and in the `schema.context` section:

???+ abstract "podinfo-p03.yaml"

    ``` { .yaml }
    apiVersion: v1alpha1
    type: Package
    name: podinfo
    tag: 6.7.1-p03
    schema:
      parameters:
        $schema: http://json-schema.org/schema#
        type: object
        additionalProperties: false
        properties:
          host: { type: string }
        required:
          - host
      context:
        $schema: http://json-schema.org/schema#
        additionalProperties: true
        type: object
        properties:
          ingress:
            type: object
            additionalProperties: true
            properties:
              className: { type: string }
              domain: { type: string }
            required:
              - domain
              - className
          project:
            type: object
            additionalProperties: true
            properties:
              subdomain: { type: string }
            required:
              - subdomain
        required:
          - ingress
          - project
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
            className: {{ .Context.ingress.className  }}
            hosts:
              - host: {{ .Parameters.host }}.{{ .Context.project.subdomain }}.{{ .Context.ingress.domain }}
                paths:
                  - path: /
                    pathType: ImplementationSpecific
    ```

Package it:

``` {.bash .copy }
kubocd pack packages/podinfo-p03.yaml
```

Create a new `Release` for deployment:

???+ abstract "podinfo-prj01.yaml"

    ``` { .yaml .copy }
    ---
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Release
    metadata:
      name: podinfo
    spec:
      description: A release of podinfo on project01
      package:
        repository: quay.io/kubodoc/packages/podinfo
        tag: 6.7.1-p03
        interval: 30m
      parameters:
        host: podinfo
      contexts:
        - namespace: contexts
          name: cluster
        - name: project01
      debug:
        dumpContext: true
        dumpParameters: true
    ```

Notes:

- `metadata.namespace` is not defined; it will be set via command line.
- `metadata.name` is simply `podinfo`, assuming only one instance per namespace.
- `spec.contexts` includes now two entries, with the second referencing the project context. As the namespace is not defined, it will be set to the `Release` one.
- A `debug` section is added to include the resulting `context` and `parameters` in the `Release` status.

Deploy the release:

``` {.bash .copy }
kubectl -n project01 apply -f releases/podinfo-prj01.yaml
```

Verify both contexts are listed:

``` {.bash .copy }
kubectl -n project01 get releases podinfo
```

``` {.bash  }
NAME      REPOSITORY                         TAG         CONTEXTS                               STATUS   READY   WAIT   PRT   AGE     DESCRIPTION
podinfo   quay.io/kubodoc/packages/podinfo   6.7.1-p03   contexts:cluster,project01:project01   READY    1/1            -     8m31s   A release of podinfo on project01
```

Check the resulting ingress object:

``` {.bash .copy }
kubectl get --all-namespaces ingress
```

``` {.bash  }
NAMESPACE   NAME            CLASS   HOSTS                                 ADDRESS        PORTS   AGE
default     podinfo1-main   nginx   podinfo1.ingress.kubodoc.local        10.96.207.51   80      4h46m
default     podinfo2-main   nginx   podinfo2.ingress.kubodoc.local        10.96.207.51   80      4h6m
project01   podinfo-main    nginx   podinfo.prj01.ingress.kubodoc.local   10.96.207.51   80      4h
```

Inspect the `Release` status:

``` {.bash .copy }
kubectl -n project01 get release podinfo -o yaml
```

``` {.bash  }
apiVersion: kubocd.kubotal.io/v1alpha1
kind: Release
metadata:
  ....
spec:
  ....
status:
  context:
    ingress:
      className: nginx
      domain: ingress.kubodoc.local
    project:
      id: p01
      subdomain: prj01
    storageClass:
      data: standard
      workspace: standard
  ....      
  parameters:
    host: podinfo2
  ....
```

The merged context includes values from both the cluster and project contexts.

!!! warning

    In real-world scenarios, the context may become quite large. Use this debug mode sparingly.


### Example 2: Context override

In this second example, the objective remains the same (adding a subdomain to the ingress), but we use the initial version of the `Package`, which does not handle `project.subdomain` context value.

Create a dedicated namespace:

``` {.bash .copy }
kubectl create ns project02
```

Create a project context in that namespace:

???+ abstract "project02.yaml"

    ``` { .yaml .copy }
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Context
    metadata:
      name: project02
    spec:
      description: Context for project 2
      context:
        project:
          id: p02
        ingress:
          domain: prj02.ingress.kubodoc.local
    ```

``` {.bash .copy }
kubectl -n project02 apply -f contexts/project02.yaml
```

Note that the same `spec.context.ingress.domain` path exists in both the project and cluster contexts. 
When contexts are merged in a `Release`, later contexts in the list override earlier ones. Thus, the project’s value takes precedence.

Create and deploy a new `Release` object:

???+ abstract "podinfo-prj02.yaml"

    ``` { .yaml .copy }
    ---
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Release
    metadata:
      name: podinfo
    spec:
      description: A release of podinfo on project02
      package:
        repository: quay.io/kubodoc/packages/podinfo
        tag: 6.7.1-p02
        interval: 30m
      parameters:
        host: podinfo
      contexts:
        - namespace: contexts
          name: cluster
        - name: project02
      debug:
        dumpContext: true
        dumpParameters: true
    ```

``` {.bash .copy }
kubectl -n project02 apply -f releases/podinfo-prj02.yaml
```

Check the resulting context in the `Release` object:

``` {.bash .copy }
kubectl -n project02 get release podinfo -o yaml
```

``` {.bash }
apiVersion: kubocd.kubotal.io/v1alpha1
kind: Release
metadata:
    ....
spec:
    ....
status:
  context:
    ingress:
      className: nginx
      domain: prj02.ingress.kubodoc.local
    project:
      id: p02
    storageClass:
      data: standard
      workspace: standard
  ....
```

Ensure the correct ingress host is used:

``` {.bash .copy }
kubectl get --all-namespaces ingress
```

``` {.bash }
NAMESPACE   NAME            CLASS   HOSTS                                 ADDRESS        PORTS   AGE
default     podinfo1-main   nginx   podinfo1.ingress.kubodoc.local        10.96.218.98   80      2d20h
default     podinfo2-main   nginx   podinfo2.ingress.kubodoc.local        10.96.218.98   80      110m
project01   podinfo-main    nginx   podinfo.prj01.ingress.kubodoc.local   10.96.218.98   80      26m
project02   podinfo-main    nginx   podinfo.prj02.ingress.kubodoc.local   10.96.218.98   80      2m52s
```

---

## Context Change

Any change to a context is automatically applied to all associated `Release` objects. However, only the deployments that are actually affected will be updated.

!!! notes

    Technically, KuboCD patches the corresponding Flux `helmRelease` objects, which triggers a `helm upgrade`. This should only update the resources that are truly impacted.

For example, modify the context for `project01`:

``` {.bash .copy }
kubectl -n project01 patch context.kubocd.kubotal.io project01 --type='json' -p='[{"op": "replace", "path": "/spec/context/project/subdomain", "value": "project01" }]'
```

Observe that the ingress is quickly updated accordingly:

``` { .bash .copy }
kubectl get --all-namespaces ingress
```

``` { .bash }
NAMESPACE   NAME            CLASS   HOSTS                                     ADDRESS        PORTS   AGE
default     podinfo1-main   nginx   podinfo1.ingress.kubodoc.local            10.96.218.98   80      3d3h
default     podinfo2-main   nginx   podinfo2.ingress.kubodoc.local            10.96.218.98   80      8h
project01   podinfo-main    nginx   podinfo.project01.ingress.kubodoc.local   10.96.218.98   80      7h13m
project02   podinfo-main    nginx   podinfo.prj02.ingress.kubodoc.local       10.96.218.98   80      6h49m
```

To restore the original value:

``` {.bash .copy }
kubectl -n project01 patch context.kubocd.kubotal.io project01 --type='json' -p='[{"op": "replace", "path": "/spec/context/project/subdomain", "value": "prj01" }]'
```

