# The Context

One of KuboCD's key features is its ability to generate Helm value files from a concise set of high-level input parameters using a templating mechanism.

This mechanism combines a template with a data model.

Our previous example used only the `.Parameters` element of the data model:

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

However, the full data model includes the following top-level elements:

- **`.Parameters`**: The parameters provided in the `Release` custom resource.
- **`.Release`**: The release object itself.
- **`.Context`**: The deployment context.

The **Context** is a flexible YAML object designed to hold shared configuration data relevant across multiple deployments.

For example, the `podinfo` package includes an `ingressClassName` parameter with a default value of `nginx`. If a cluster uses a different ingress controller, this value would need to be overridden for every `Release`. Ideally, widespread configuration like this should be defined once in a global cluster-level context.

Similarly, if all application ingress URLs share a common root domain, that domain should also be centralized.

Here is how to implement this logic.

---

## Creating a Context

A `Context` is a KuboCD Custom Resource:

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

**Key Attributes:**

- **`description`**: A short description of the context.
- **`protected`**: Prevents accidental deletion (requires the KuboCD webhook).
- **`context`**: A tree of values injected into the data model during `values` templating.
    - Must be valid YAML.
    - Has a flexible structure but should align with what `Package` templates expect.

In this example, the context includes:

- **`ingress.className`**: The ingress controller type.
- **`ingress.domain`**: The suffix for ingress URLs.
- **`storageClass`**: Definitions for `data` and `workspace` storage profiles (set to `standard` for Kind).
- **`certificateIssuer`**: Identifiers for public and internal certificate issuers. (We will configure a self-signed CA in the [cert-manager section](./210-cert-manager.md)).

Cluster-wide contexts should be placed in a dedicated namespace:

``` {.bash .copy }
kubectl create ns contexts
```

``` {.bash .copy }
kubectl apply -f contexts/cluster.yaml
```

!!! note
    Since the context is often shared across many applications, its structure should be carefully designed and well-documented.

---

## Modifying the Package

Our initial `podinfo` package did not utilize the context. Here is an updated version that does:

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

**Key Changes:**

- **`tag`**: updated to `6.7.1-p02`.
- **`host` parameter**: Replaces `fqdn`. Now represents only the hostname (excluding the domain).
- **`modules[X].values`**: Updated to use `.Context` variables.
- **`schema.context`**: Added to define and validate the expected context structure.

Pack and push this new version:

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

## Deployment with Context

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
      parameters:
        host: podinfo2
      contexts:
        - namespace: contexts
          name: cluster
    ```

**Key Features:**

- **Interval Removed**: `spec.package.interval` is omitted and will default to global settings (30m).
- **Parameter Update**: `fqdn` is replaced by `host`.
- **`contexts` Section**: Lists contexts to merge into the template data model.

!!! warning
    Referencing a non-existent context will result in an error.

Apply the deployment:

``` {.bash .copy }
kubectl apply -f releases/podinfo2-ctx.yaml
```

Check that the new `Release` is `READY`:

``` { .bash .copy }
kubectl get releases podinfo2
```

``` { .bash }
NAME       REPOSITORY                         TAG         CONTEXTS           STATUS   READY   WAIT   PRT   AGE   DESCRIPTION
podinfo2   quay.io/kubodoc/packages/podinfo   6.7.1-p02   contexts:cluster   READY    1/1            -     17m   A first sample release of podinfo
```

Verify that the ingress has been configured correctly:

``` { .bash .copy }
kubectl get ingresses podinfo2-main
```

``` { .bash }
NAME            CLASS   HOSTS                            ADDRESS      PORTS   AGE
podinfo2-main   nginx   podinfo2.ingress.kubodoc.local   10.96.59.9   80      120m
```

!!! note
    Remember to update your `/etc/hosts` or DNS if you wish to access this ingress.

---

## Context Aggregation

An application's effective context is often the result of aggregating multiple context objects.

For instance, you might create a project-level context to share variables across all applications in a specific project, which is then merged with the global cluster context.

### Example 1: merging Contexts

Create a project namespace:

``` {.bash .copy }
kubectl create namespace project01
```

Create the project context:

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

Apply it to the namespace:

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

We need to update the package to use the new `project.subdomain` variable:

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

Pack the updated package:

``` {.bash .copy }
kubocd pack packages/podinfo-p03.yaml
```

Create a new `Release`:

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

**Notes:**

- **`metadata.namespace`**: Not defined in the file; will be set via CLI.
- **`metadata.name`**: Simply `podinfo`.
- **`spec.contexts`**: Now includes two entries. The second one references the project context (defaults to the release namespace).
- **`debug`**: Added to dump the resulting context and parameters into the `Release` status for inspection.

Deploy the release:

``` {.bash .copy }
kubectl -n project01 apply -f releases/podinfo-prj01.yaml
```

Verify that both contexts are active:

``` {.bash .copy }
kubectl -n project01 get releases podinfo
```

``` {.bash  }
NAME      REPOSITORY                         TAG         CONTEXTS                               STATUS   READY   WAIT   PRT   AGE     DESCRIPTION
podinfo   quay.io/kubodoc/packages/podinfo   6.7.1-p03   contexts:cluster,project01:project01   READY    1/1            -     8m31s   A release of podinfo on project01
```

Check the resulting ingress:

``` {.bash .copy }
kubectl get --all-namespaces ingress
```

``` {.bash  }
NAMESPACE   NAME            CLASS   HOSTS                                 ADDRESS        PORTS   AGE
default     podinfo1-main   nginx   podinfo1.ingress.kubodoc.local        10.96.207.51   80      4h46m
default     podinfo2-main   nginx   podinfo2.ingress.kubodoc.local        10.96.207.51   80      4h6m
project01   podinfo-main    nginx   podinfo.prj01.ingress.kubodoc.local   10.96.207.51   80      4h
```

Inspect the `Release` status to see the merged context:

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

!!! warning
    In production, contexts can become large. Use `debug` mode sparingly.

### Example 2: Context Overrides

In this example, we keep the objective (adding a subdomain) but use the `podinfo-p02` package (which does not use `project.subdomain`). Instead, we will override `ingress.domain`.

Create a dedicated namespace:

``` {.bash .copy }
kubectl create ns project02
```

Create a project context:

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

Notice that `ingress.domain` exists in both the cluster and project contexts. When merging, **later contexts in the list override earlier ones**.

Create and deploy the release:

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

Check the `Release` context:

``` {.bash .copy }
kubectl -n project02 get release podinfo -o yaml
```

``` {.bash }
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
```

Verify the ingress host:

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

## Modifying Contexts

Any change to a context resource is automatically propagated to all associated `Release` objects. However, actual updates (Helm upgrades) only occur for deployments that are genuinely affected by the change.

For example, modify the `project01` context:

``` {.bash .copy }
kubectl -n project01 patch context.kubocd.kubotal.io project01 --type='json' -p='[{"op": "replace", "path": "/spec/context/project/subdomain", "value": "project01" }]'
```

Observe the ingress update:

``` { .bash .copy }
kubectl get --all-namespaces ingress
```

The host `podinfo.project01.ingress.kubodoc.local` should appear shortly.

To restore the original value:

``` {.bash .copy }
kubectl -n project01 patch context.kubocd.kubotal.io project01 --type='json' -p='[{"op": "replace", "path": "/spec/context/project/subdomain", "value": "prj01" }]'
```
