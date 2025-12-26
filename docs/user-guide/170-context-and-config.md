# Context and Configuration

## Default Context

As seen in the previous chapter, it is often necessary to include a global cluster context in every `Release`. Rather than manually specifying this in every deployment manifest, it can be defined as a default context in the global configuration.

## Config Resource

Unlike many Kubernetes applications that use `ConfigMaps`, KuboCD uses a dedicated **Config** Custom Resource.

``` { .bash .copy }
kubectl -n kubocd get Config.kubocd.kubotal.io
```

A single instance usually exists (created during the initial Helm deployment):

``` { .bash }
NAME     AGE
conf01   5d21h
```

Inspect its content:

``` { .bash .copy }
kubectl -n kubocd get Config.kubocd.kubotal.io conf01 -o yaml
```

``` { .yaml } 
apiVersion: kubocd.kubotal.io/v1alpha1
kind: Config
metadata:
  name: conf01
  namespace: kubocd
  ...
spec:
  clusterRoles: []
  defaultContexts: []
  defaultHelmInterval: 30m0s
  defaultHelmTimeout: 3m0s
  defaultNamespaceContexts: []
  defaultOnFailureStrategy: updateOnFailure
  defaultPackageInterval: 30m0s
  imageRedirects: []
  onFailureStrategies:
  ...
```

Attributes like `defaultContexts` and `defaultPackageInterval` are managed here. These will be detailed further in the [Reference section](../reference/530-config.md).

!!! note
    Using a Custom Resource for configuration provides immediate validation (structural errors are rejected) and allows KuboCD to watch for and instantly apply changes.

### Configuring Default Contexts

Create a new configuration manifest to set the default context:

???+ abstract "conf02.yaml"

    ``` { .yaml .copy }
    ---
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Config
    metadata:
      name: conf02
      namespace: kubocd
    spec:
      defaultContexts:
        - namespace: contexts
          name: cluster
    ```

Apply it:

``` { .bash .copy }
kubectl apply -f configs/conf02.yaml 
```

> Note: Any warnings can be ignored during this step.

!!! note "Merging Configurations"
    KuboCD allows breaking configuration across multiple resources. All `Config` resources in the controller's namespace (`kubocd`) are merged to form the global configuration.
    
    Resources are processed in **alphabetic order**, with later values overriding earlier ones.

!!! tip
    Use [`kubocd dump config`](./180-kubocd-cli.md/#kubocd-dump-config) to view the final merged configuration.

Now, we can create a `Release` without explicitly specifying the context:

???+ abstract "podinfo3-ctx-def.yaml"

    ``` { .yaml .copy }
    ---
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Release
    metadata:
      name: podinfo3
      namespace: default
    spec:
      description: A first sample release of podinfo
      package:
        repository: quay.io/kubodoc/packages/podinfo
        tag: 6.7.1-p02
      parameters:
        host: podinfo3
    ```

``` { .bash .copy }
kubectl apply -f releases/podinfo3-ctx-def.yaml 
```

Verify that the context was automatically applied:

``` { .bash .copy }
kubectl get release podinfo3
```

``` { .bash }
NAME       REPOSITORY                         TAG         CONTEXTS           STATUS   READY   WAIT   PRT   AGE   DESCRIPTION
podinfo3   quay.io/kubodoc/packages/podinfo   6.7.1-p02   contexts:cluster   READY    1/1            -     31m   A first sample release of podinfo
```

And confirm the ingress domain generation:

``` { .bash .copy }
kubectl get ingress podinfo3-main
```

## Namespaced Default Contexts

KuboCD also supports defining a default context per namespace.

Update the global configuration to enable this feature:

???+ abstract "conf03.yaml"

    ``` { .yaml .copy }
    ---
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Config
    metadata:
      name: conf03
      namespace: kubocd
    spec:
      defaultNamespaceContexts: 
        - project
    ```

``` { .bash .copy }
kubectl apply -f configs/conf03.yaml 
```

With this setting, every `Release` will check its own namespace for a context named `project` and use it if found.

Create a new namespace `project03`:

``` { .bash .copy }
kubectl create ns project03
```

Create a project-specific context named `project` in that namespace:

???+ abstract "project03.yaml"

    ``` { .yaml .copy }
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Context
    metadata:
      name: project
    spec:
      description: Context For projet 3
      context:
        project:
          id: p03
        ingress:
          domain: prj03.ingress.kubodoc.local
    ```

``` { .bash .copy }
kubectl -n project03 apply -f contexts/project03.yaml 
```

Deploy a `Release` without specifying contexts:

???+ abstract "podinfo-prj03.yaml"

    ``` { .yaml .copy }
    ---
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Release
    metadata:
      name: podinfo
    spec:
      description: A release of podinfo on project03
      package:
        repository: quay.io/kubodoc/packages/podinfo
        tag: 6.7.1-p02
        interval: 30m
      parameters:
        host: podinfo
      debug:
        dumpContext: true
        dumpParameters: true
    ```

``` { .bash .copy }
kubectl -n project03 apply -f releases/podinfo-prj03.yaml
```

Verify that both default contexts (global `cluster` and local `project`) are used:

``` { .bash .copy }
kubectl -n project03 get release podinfo
```

``` { .bash }
NAME      REPOSITORY                         TAG         CONTEXTS                             STATUS   READY   WAIT   PRT   AGE   DESCRIPTION
podinfo   quay.io/kubodoc/packages/podinfo   6.7.1-p02   contexts:cluster,project03:project   READY    1/1            -     10m   A release of podinfo on project03
```

---

## Context Ordering

The order in which contexts are merged is critical, as later values override earlier ones. The precedence order is:

1. **Global Default Contexts** (in list order).
2. **Namespace Default Contexts** (if they exist).
3. **Release-Specific Contexts** (in list order).

!!! warning
    Referencing a non-existent context in `spec.contexts` results in an error. Missing namespace default contexts are ignored silently.

!!! tip
    Use [`kubocd dump context`](./180-kubocd-cli.md/#kubocd-dump-context) to inspect the resolved context for a specific namespace.

### Disabling Default Contexts

If you need to disable default context merging for a specific `Release`, use the `skipDefaultContext: true` flag in the `Release` specification.

---

## Configuration via Helm Chart

In a real setup, you can define these configurations directly during the KuboCD installation via Helm values.

Example `values1-ctrl.yaml`:

???+ abstract "values1-ctrl.yaml"

    ``` { .yaml .copy }
    extraNamespaces:
      - name: contexts
    config:
      enabled: true
      content:
        defaultContexts:
          - name: cluster
            namespace: contexts
        defaultNamespaceContexts:
          - project
    contexts:
      - name: cluster
        namespace: contexts
        protected: true
        description: Context specific to the cluster 'kubodoc'
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

Upgrade the deployment:

``` { .bash .copy }
helm -n kubocd upgrade kubocd-ctrl oci://quay.io/kubocd/charts/kubocd-ctrl:v0.3.1-snapshot --values helm-values/values1-ctrl.yaml
```

!!! warning "Adoption Error"
    If you manually created the `contexts` namespace and `cluster` context earlier, Helm will fail to adopt them. Delete them first:

    ``` { .bash .copy }
    kubectl -n contexts delete context.kubocd.kubotal.io cluster
    kubectl delete ns contexts
    ```

    Then re-run the upgrade.

Once managed by Helm, you can remove the manual config resources:

``` { .bash .copy }
kubectl -n kubocd delete config.kubocd.kubotal.io conf02
kubectl -n kubocd delete config.kubocd.kubotal.io conf03
```
