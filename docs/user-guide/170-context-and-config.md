# Context and Configuration

## Default Context

Following the logic from the previous chapter, it becomes clear that the cluster’s global context should be included in every `Release`. 
From there, the idea of defining a default context naturally follows.

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

``` { .bash }
apiVersion: kubocd.kubotal.io/v1alpha1
kind: Config
metadata:
  annotations:
    meta.helm.sh/release-name: kubocd-ctrl
    meta.helm.sh/release-namespace: kubocd
  creationTimestamp: "2025-04-16T12:29:27Z"
  generation: 1
  labels:
    app.kubernetes.io/managed-by: Helm
  name: conf01
  namespace: kubocd
  resourceVersion: "5191"
  uid: 84ff9a8a-83ee-48e7-984d-c95a6a665d5b
spec:
  clusterRoles: []
  defaultContexts: []
  defaultNamespaceContexts: []
  imageRedirects: []
  packageRedirects: []
```

At this stage, the configuration is empty

!!! notes
    Using a Kubernetes resource for configuration has the following advantages:

      - Structural errors result in immediate rejection without impacting the running service.
      - The resource can be watched. KuboCD will therefore instantly apply any configuration changes.

Here is a new configuration manifest:

???+ abstract "conf01-b.yaml"

    ``` { .yaml .copy }
    ---
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Config
    metadata:
      name: conf01
      namespace: kubocd
    spec:
      defaultContexts:
        - namespace: contexts
          name: cluster
    ```

It defines a list of default contexts, which here includes only the cluster context created earlier.

Apply it:

``` { .bash .copy }
kubectl apply -f configs/conf01-b.yaml 
```

> Don't worry about any warning messages

!!! note 
    For KuboCD to recognize the `Config` object, it must reside in the controller’s namespace (`kubocd`). However, its name does not matter.


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
        interval: 30m
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

We modify our configuration resource again:

???+ abstract "conf01-c.yaml"

    ``` { .yaml .copy }
    ---
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Config
    metadata:
      name: conf01
      namespace: kubocd
    spec:
      defaultContexts:
        - namespace: contexts
          name: cluster
      defaultNamespaceContexts: 
        - project
    ```


``` { .bash .copy }
kubectl apply -f configs/conf01-c.yaml 
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
    config:
      defaultContexts:
        - name: cluster
          namespace: contexts
      defaultNamespaceContext: 
        - project
    extraNamespaces:
      - name: contexts
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
    ```

- - The `config` section is injected directly into the `Config` resource’s `spec`.
- - The `extraNamespaces` list creates additional namespaces via the Helm chart.
- - The `contexts` section defines context objects created automatically by the Helm chart.

To upgrade the KuboCD deployment:

``` { .bash .copy }
helm -n kubocd upgrade kubocd-ctrl oci://quay.io/kubocd/charts/kubocd-ctrl:v0.2.1 --values helm-values/values1-ctrl.yaml
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


