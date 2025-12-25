# Under the Hood (KuboCD Components)

Behind the scenes, KuboCD creates several Flux resources to manage deployments. You can inspect these resources to debug issues or understand the internal workings.

Check the events for the `Release` created earlier:

``` { .bash .copy }
kubectl describe release podinfo1
```

Events are generated for each Flux resource created:

``` { .bash }
......
Events:
  Type    Reason                 Age   From     Message
  ----    ------                 ----  ----     -------
  Normal  OCIRepositoryCreated   49s   release  Created OCIRepository "kcd-podinfo1"
  Normal  HelmRepositoryCreated  45s   release  Created HelmRepository "kcd-podinfo1"
  Normal  HelmReleaseCreated     44s   release  Created HelmRelease "podinfo1-main"
```

All resources are created in the same namespace as the `Release` object (in this case, `default`).

---

## The OCIRepository

This Flux resource is responsible for pulling the KuboCD package image:

``` { .bash .copy }
kubectl get OCIRepository
```

``` { .bash }
NAME           URL                                      READY   STATUS                                                                                                           AGE
kcd-podinfo1   oci://quay.io/kubodoc/packages/podinfo   True    stored artifact for digest '6.7.1-p01@sha256:985e4e2f89a4b17bd5cc2936a0b305df914ae479e0b8c96e61cb22725b61cd24'   9m1s
```

!!! tip
    If the release is stuck in the `WAIT_OCI` state, check this resource and its events. Common issues include:
    
    - Incorrect URL
    - Image is private (needs to be public or have authentication configured)

To force a refresh (e.g., to load a modified OCI image before the sync period expires), you can manually delete this resource:

``` { .bash .copy }
kubectl delete ocirepository kcd-podinfo1
```

KuboCD will automatically recreate it.

### Naming

The `OCIRepository` is a namespaced resource located in the same namespace as the `Release`.
Its name is derived from the `Release` name, prefixed with `kcd-`.

---

## The HelmRepository

Since the `podinfo` Helm chart is embedded in the package, it must be exposed to Flux via an internal Helm repository. KuboCD creates a `HelmRepository` resource pointing to its internal server:

``` { .bash .copy }
kubectl get HelmRepository
```

``` { .bash }
NAME           URL                                                                            AGE    READY   STATUS
kcd-podinfo1   http://kubocd-ctrl-controller-helm-repository.kubocd.svc/hr/default/podinfo1   105m   True    stored artifact: revision 'sha256:d8db03cf45ecd75064c2a2582812dc4df5cd624d0e295b24ff79569bf46a070b'
```

This step rarely causes errors unless the internal controller is unreachable.

### Naming

The `HelmRepository` is located in the same namespace as the `Release`.
Its name matches the `Release` name, prefixed with `kcd-`.

---

## The HelmRelease

This Flux resource manages the actual Helm chart deployment.

``` { .bash .copy }
kubectl get HelmRelease
```

``` { .bash }
NAME                AGE     READY   STATUS
podinfo1-main   7m29s   True    Helm install succeeded for release default/kcd-podinfo1-main.v1 with chart podinfo@6.7.1
```

!!! note
    One `HelmRelease` is created per module in the package.

If the release is stuck in `WAIT_HREL`, inspect this resource:

``` { .bash .copy }
kubectl describe helmrelease podinfo1-main
```

``` { .bash }
.....
Events:
  Type    Reason            Age    From             Message
  ----    ------            ----   ----             -------
  Normal  HelmChartCreated  4m38s  helm-controller  Created HelmChart/default/default-podinfo1-main with SourceRef 'HelmRepository/default/kcd-podinfo1'
  Normal  InstallSucceeded  4m37s  helm-controller  Helm install succeeded for release default/podinfo1-main.v1 with chart podinfo@6.7.1
```

A key troubleshooting step is to verify the generated `values` passed to the Helm chart:

!!! note
    This requires the [yq](https://github.com/mikefarah/yq){:target="_blank"} tool.

``` { .bash .copy }
kubectl get HelmRelease podinfo1-main -o yaml | yq '.spec.values'
```

``` { .yaml }
ingress:
  className: nginx
  enabled: true
  hosts:
    - host: podinfo1.ingress.kubodoc.local
      paths:
        - path: /
          pathType: ImplementationSpecific
```

!!! note
    If a deployment fails, you may need to wait for the Helm timeout (default: 3 minutes) to see the failure reason. This timeout can be configured in the `Package` or the `Release`.

!!! tip
    You can also check the generated `values` using the CLI:
    
    ```{ .bash .copy }
    kubocd render releases/podinfo1-basic.yaml
    ```
    
    See the [KuboCD CLI section](./180-kubocd-cli.md/#kubocd-render) for details.

### Naming

The `HelmRelease` resides in the same namespace as the `Release`.

Its name usually follows the pattern: `<Release name>-<ModuleName>`.

The corresponding Helm deployment (stored as a Kubernetes Secret) also follows this naming convention:

```
kubectl get secrets
NAME                                  TYPE                 DATA   AGE
sh.helm.release.v1.podinfo1-main.v1   helm.sh/release.v1   1      66m
```

Since many Helm charts use the release name as a base for resource naming, this can lead to redundant suffixes (e.g., `podinfo1-main-...`).

Example pod name:
```bash
NAME                             READY   STATUS    RESTARTS   AGE
podinfo1-main-779b6b9fd4-zbgbx   1/1     Running   0          8h
```

To avoid this, KuboCD supports a special module name: **`noname`**.
When a module is named `noname`, the corresponding `HelmRelease` (and derived resources) uses only the `Release` name.

> Note: Since module names must be unique, a package can contain only one `noname` module.

The next chapter demonstrates this feature.
