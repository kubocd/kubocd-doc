# Under the Hood (If Things Go Wrong)

Behind the scenes, KuboCD creates several FluxCD resources to manage the deployment.

You can inspect these resources to debug problems or understand the internals.

Check the events bound to the `Release` which was previously created:

``` { .bash .copy }
kubectl describe release podinfo1
```

``` { .bash }
......
Events:
Type    Reason                 Age   From     Message
----    ------                 ----  ----     -------
Normal  OCIRepositoryCreated   12s   release  Created OCIRepository "kcd-podinfo1"
Normal  HelmRepositoryCreated  10s   release  Created HelmRepository "kcd-podinfo1"
Normal  HelmReleaseCreated     8s    release  Created HelmRelease "podinfo1-main"
```

All of these resources are created in the same namespace as the `Release` object (e.g. `default`).

---

## The OCIRepository

This Flux resource pulls the KuboCD package image:

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
    - Image still private (set to public or provide authentication)

You can manually delete this resource to trigger a refresh:

``` { .bash .copy }
kubectl delete ocirepository kcd-podinfo1
```

KuboCD will recreate it.

This can be useful to force a reload of a modified OCI image, without waiting for the sync period.

---

## The HelmRepository

As the `podinfo` Helm the chart is embedded in the package, it must be served to Flux via an internal Helm repository.

KuboCD creates a `HelmRepository` resource pointing to its internal server:

``` { .bash .copy }
kubectl get HelmRepository
```

``` { .bash }
NAME           URL                                                                            AGE    READY   STATUS
kcd-podinfo1   http://kubocd-ctrl-controller-helm-repository.kubocd.svc/hr/default/podinfo1   105m   True    stored artifact: revision 'sha256:d8db03cf45ecd75064c2a2582812dc4df5cd624d0e295b24ff79569bf46a070b'
```

This step rarely causes errors unless the internal controller is unreachable.

--- 

## The HelmRelease

This Flux resource handles the actual Helm chart deployment.

``` { .bash .copy }
kubectl get HelmRelease
```

``` { .bash }
NAME            AGE     READY   STATUS
podinfo1-main   7m59s   True    Helm install succeeded for release default/podinfo1-main.v1 with chart podinfo@6.7.1
```

!!! note
    There will be one HelmRelease per module in the package.

If the release is stuck in `WAIT_HREL`, inspect this resource:

``` { .bash .copy }
kubectl describe helmrelease podinfo1-main
```

``` { .bash }
.....
Events:
  Type    Reason            Age    From             Message
  ----    ------            ----   ----             -------
  Normal  HelmChartCreated  5m42s  helm-controller  Created HelmChart/default/default-podinfo1-main with SourceRef 'HelmRepository/default/kcd-podinfo1'
  Normal  InstallSucceeded  5m39s  helm-controller  Helm install succeeded for release default/podinfo1-main.v1 with chart podinfo@6.7.1
```

One of the point to check if the generated `values` for the Helm chart deployment.

!!! note
    You will need to unstall the [yq command](https://github.com/mikefarah/yq){:target="_blank"}.

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
    If deployment fails, you may need to wait for the Helm timeout to expire (default: 2 minutes) to have failure reason. 
    You can configure this value in the `Package` or in the `Release`.


