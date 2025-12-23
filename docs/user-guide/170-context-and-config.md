# Context and Configuration

## Default Context

Following the logic from the previous chapter, it becomes clear that the cluster’s global context should be included in every `Release`. 
From there, the idea of defining a default context naturally follows.

And this default context should be defined in a global configuration.

## Config resource

Unlike most Kubernetes applications that store configuration in a `ConfigMap`, KuboCD uses a dedicated Custom Resource for this purpose:

``` { .bash .copy }
kubectl -n kubocd get Config.kubocd.kubotal.io
```

Only one instance of this resource exists (it was created during deployment via the Helm chart)

``` { .bash }
NAME     AGE
conf01   5d21h
```

Let's inspect its content:

``` { .bash .copy }
kubectl -n kubocd get Config.kubocd.kubotal.io conf01 -o yaml
```

``` { .yaml } 
apiVersion: kubocd.kubotal.io/v1alpha1
kind: Config
metadata:
  annotations:
    meta.helm.sh/release-name: kubocd-ctrl
    meta.helm.sh/release-namespace: kubocd
  creationTimestamp: "2025-12-22T18:41:05Z"
  generation: 1
  labels:
    app.kubernetes.io/managed-by: Helm
  name: conf01
  namespace: kubocd
  resourceVersion: "748"
  uid: 3932d4de-09b9-48de-bb32-e1549dd62566
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
  - name: stopOnFailure
    values: {}
  - name: reinstallOnFailure
    values:
      install:
        remediation:
          retries: 10
        strategy:
          name: RemediateOnFailure
      upgrade:
        remediation:
          retries: 10
        strategy:
          name: RemediateOnFailure
  - name: updateOnFailure
    values:
      install:
        strategy:
          name: RetryOnFailure
          retryInterval: 1m0s
      upgrade:
        strategy:
          name: RetryOnFailure
          retryInterval: 1m0s
  packageRedirects: []
  specPatch: {}
```

At this stage, the configuration has been populated by the KuboCD Helm chart. Most of theses values will be described in a subsequent chapter, or in the [Reference](../reference/530-config.md) part 

!!! notes
    Using a Kubernetes resource for configuration has the following advantages:

      - Structural errors result in immediate rejection without impacting the running service.
      - The resource can be watched. KuboCD will therefore instantly apply any configuration changes.

Here is a new configuration manifest:

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

It defines a list of default contexts, which here includes only the cluster context created earlier.

Apply it:

``` { .bash .copy }
kubectl apply -f configs/conf02.yaml 
```

> Don't worry about any warning messages

!!! note

    As demonstrated here, the configuration may be built from a set of resources. All `configs.kubocd.kubotal.io` located in the  controller’s namespace (`kubocd`) 
    are merged to build a global configuration referential.

    The `configs` resources are loaded in alphebetic order, latest overriding the previous value.

!!! tips

    KuboCD provide a tool to display the resulting global configuration: [`kubocd dump config`](./180-kubocd-cli.md/#kubocd-dump-config) 
    

We can now create a new `Release` of the `podinfo` application using the context-enabled version, without explicitly specifying the context:

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

We can verify that the `Release` does pick up the context:

``` { .bash .copy }
kubectl get release podinfo3
```

``` { .bash }
NAME       REPOSITORY                         TAG         CONTEXTS           STATUS   READY   WAIT   PRT   AGE   DESCRIPTION
podinfo3   quay.io/kubodoc/packages/podinfo   6.7.1-p02   contexts:cluster   READY    1/1            -     31m   A first sample release of podinfo
```

Which results in a correct `domain` in the ingress:

``` { .bash .copy }
kubectl get ingress podinfo3-main
```

``` { .bash }
NAME            CLASS   HOSTS                            ADDRESS        PORTS   AGE
podinfo3-main   nginx   podinfo3.ingress.kubodoc.local   10.96.218.98   80      84s
```

---

## Namespaced Default Context

KuboCD also supports defining a default context per namespace.

We modify our configuration referential again, by adding a new resource 

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

Thanks to this setup, every `Release` will attempt to load a context named `project` from its own namespace,
and use it if it exists.

Create a new namespace `project03`:

``` { .bash .copy }
kubectl create ns project03
```

Create A project-specific context there, using the generic name `project`:

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

We now create a new `Release` without explicitly specifying which context to use:

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

We can confirm both default contexts are applied:

``` { .bash .copy }
kubectl -n project03 get release podinfo
```

``` { .bash }
NAME      REPOSITORY                         TAG         CONTEXTS                             STATUS   READY   WAIT   PRT   AGE   DESCRIPTION
podinfo   quay.io/kubodoc/packages/podinfo   6.7.1-p02   contexts:cluster,project03:project   READY    1/1            -     10m   A release of podinfo on project03
```

And verify that the ingress domain is correctly resolved:


``` { .bash .copy }
kubectl -n project03 get ingress
```

``` { .bash }
NAME           CLASS   HOSTS                                 ADDRESS        PORTS   AGE
podinfo-main   nginx   podinfo.prj03.ingress.kubodoc.local   10.96.218.98   80      11m
```

---

## Context Ordering

As mentioned earlier, the order in which contexts are merged for a `Release` can be important.

The order is:

- Global default context, in the order of the list defined in the configuration.
- Namespace default context, if it exists.
- Context defined in the `Release`object, in list order

Last ones will take precedence.

!!! warning 
    Referencing a non-existent context results in an error, except for the namespace-level default context.


!!! tips

    KuboCD provide a tool to display the resulting context, depending of the namespace: [`kubocd dump context`](./180-kubocd-cli.md/#kubocd-dump-context) 

If you've followed the full guide, your setup should look something like this:

``` { .bash .copy }
kubectl get --all-namespaces releases
```

``` { .bash  }
NAMESPACE   NAME            REPOSITORY                               TAG          CONTEXTS                                                STATUS   READY   WAIT   PRT   AGE     DESCRIPTION
default     podinfo1        quay.io/kubodoc/packages/podinfo         6.7.1-p01    contexts:cluster                                        READY    1/1            -     3d20h   A first sample release of podinfo
default     podinfo2        quay.io/kubodoc/packages/podinfo         6.7.1-p02    contexts:cluster,contexts:cluster                       READY    1/1            -     25h     A first sample release of podinfo
default     podinfo3        quay.io/kubodoc/packages/podinfo         6.7.1-p02    contexts:cluster                                        READY    1/1            -     27m     A first sample release of podinfo
kubocd      ingress-nginx   quay.io/kubodoc/packages/ingress-nginx   4.12.1-p01   contexts:cluster                                        READY    1/1            -     3d23h   The Ingress controller
project01   podinfo         quay.io/kubodoc/packages/podinfo         6.7.1-p03    contexts:cluster,contexts:cluster,project01:project01   READY    1/1            -     24h     A release of podinfo on project01
project02   podinfo         quay.io/kubodoc/packages/podinfo         6.7.1-p02    contexts:cluster,contexts:cluster,project02:project02   READY    1/1            -     24h     A release of podinfo on project02
project03   podinfo         quay.io/kubodoc/packages/podinfo         6.7.1-p02    contexts:cluster,project03:project                      READY    1/1            -     5m55s   A release of podinfo on project03
```

You might see cases where the same context is listed twice in a `Release`, both as a default and explicitly.
This has no adverse effects.

!!! tip
    If for any reason you want to disable default context merging for a specific `Release`,
    you can use the `skipDefaultContext: true` flag in the `Release` specification.

## KuboCD Helm Chart

These configuration values can be integrated during the KuboCD installation by passing them as Helm `values`.


For example, by creating the following file:

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

- The `extraNamespaces` list creates additional namespaces via the Helm chart.
- The `config` section is combined with the Helm chart default values to create the `conf01` Config resource.
- The `contexts` section defines context objects created automatically by the Helm chart.

To upgrade the KuboCD deployment:

``` { .bash .copy }
helm -n kubocd upgrade kubocd-ctrl oci://quay.io/kubocd/charts/kubocd-ctrl:v0.2.3 --values helm-values/values1-ctrl.yaml
```
!!! warning

    If you've followed the steps in this chapter, an error will be raised — Helm refuses to manage objects it did not originally create. 
         
    In that case, delete the `contexts` namespace and its associated objects first:
    
    ``` { .bash .copy }
    kubectl -n contexts delete context.kubocd.kubotal.io cluster
    ```
    
    ``` { .bash .copy }
    kubectl delete ns contexts
    ```
    
    Then re-run the `helm upgrade` command.

While performing this operation, `Releases` may temporarily enter the `ERROR` state before returning to `READY` once the context is recreated.
However, the applications themselves (pods and ingress for `podinfo`, etc.) will remain unaffected.

If you've followed the steps in this chapter, as the helm Chart has now included our specificities 
(`defaultContexts` and `defaultNamespaceContext`) in `conf01`, we can remove `conf01` and `conf02`, which are duplicate.

``` { .bash .copy }
kubectl -n kubocd delete config.kubocd.kubotal.io conf02
```

``` { .bash .copy }
kubectl -n kubocd delete config.kubocd.kubotal.io conf03
```
