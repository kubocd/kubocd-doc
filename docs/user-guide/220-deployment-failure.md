# Deployment failure

This chapter is about how to handle failure on KuboCD `release` deployment.

As stated in [Under the hood](140-under-the-hood.md) chapter, for each module of a deployed package, KuboCD generate a FluxCD `HelmRelease` resource.

Then, the FluxCD `helm-controller` will handle the deployment, using its embedded Helm instance.

The FluxCd `helmRelease` resource provide several configuration parameters to handle behavior in case of deployment failure:

- `HelmRelease.spec.timeout`<br>This is the time to wait for any individual Kubernetes operation (like Jobs for hooks) during the performance of a Helm action.

- `HelmRelease.spec.install`<br>Holds the configuration for Helm install actions for this HelmRelease.

- `HelmRelease.spec.upgrade`<br>Holds the configuration for Helm upgrade actions for this HelmRelease.

A more complete description can be found [here for install](https://fluxcd.io/flux/components/helm/helmreleases/#install-configuration){:target="_blank"} 
and [here for upgrade](https://fluxcd.io/flux/components/helm/helmreleases/#upgrade-configuration){:target="_blank"}.

Here is a small sample of a configuration which, in case of deployment failure will try to uninstall and re-install the release. This up to 10 times. 

```yaml
kind: HelmRelease
metadata:
  .....
spec:
  ....
  install:
    strategy:
      name: RemediateOnFailure
    remediation:
      retries: 10
  upgrade:
    strategy:
      name: RemediateOnFailure
    remediation:
      retries: 10
```

To apply these configuration through KuboCD, one can use the `module.specPath` attribute on the `Package` or the `moduleOverrides[.].specPatch` on the `Release` object. But managing this for each release 
will quickly be cumbersome.

KuboCD provide a mechanism to simplify this configuration, while preserving flexibility. The global configuration (see [Context and configuration](./170-context-and-config.md) ) can host a set of
predefined configuration values set, called `onFailureStrategy`

Here is a sample configuration resource: 

```yaml
---
apiVersion: kubocd.kubotal.io/v1alpha1
kind: Config
metadata:
  ....
spec:
  .....
  defaultHelmTimeout: 3m0s
  defaultOnFailureStrategy: updateOnFailure
  onFailureStrategies:
    - name: stopOnFailure
      strategy: {}

    - name: reinstallOnFailure
      strategy:
        install:
          strategy:
            name: RemediateOnFailure
          remediation:
            retries: 10
        upgrade:
          strategy:
            name: RemediateOnFailure
          remediation:
            retries: 10

    - name: updateOnFailure
      strategy:
        install:
          strategy:
            name: RetryOnFailure
            retryInterval: 1m0s
        upgrade:
          strategy:
            name: RetryOnFailure
            retryInterval: 1m0s
```

Three strategies are defined here:

- `stopOnFailure`: Don't attempt any action?
- `reinstallOnFailure`: Try to uninstall and re-install the release. This up to 10 times.
- `updateOnFailure`: After 1 minute, try to update the release. This until success.

> For a complete explanation of the resulting behavior, please refer to the FluxCD documentation mentioned above. 

!!! notes

    These sample values are the initial dataset defined by the HelmChart. You can modify them, or create new strategies during Helm deployment.

There is also an `defaultOnFailureStrategy` which set the one used by default for all deployment. And a `defaultHelmTimeout` which will set the value of `HelmRelease.spec.timeout` by default.

These values can be overridden at the package level for each module with the `package.module[X].onFailureStrategy.s` and `package.module[X].timeout` attribute.

And they can be overridden at the release level, by using the `moduleOverrides` attribute, like the following sample:

```
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
  moduleOverrides:
    module: main
    onFailureStrategy: reinstallOnFailure
    timeout: 2m
```