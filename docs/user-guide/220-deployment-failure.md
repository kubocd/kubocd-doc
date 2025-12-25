# Handling Deployment Failures

This chapter explains how to handle failures during KuboCD `Release` deployments.

As described in [Under the hood](140-under-the-hood.md), KuboCD generates a FluxCD `HelmRelease` resource for each module in a package. The Flux `helm-controller` then executes the deployment using its embedded Helm instance.

The Flux `HelmRelease` resource provides several parameters to configure behavior when failures occur:

- **`HelmRelease.spec.timeout`**: The wait time for individual Kubernetes operations (e.g., Hooks) during a Helm action.
- **`HelmRelease.spec.install`**: Configuration for Helm install actions.
- **`HelmRelease.spec.upgrade`**: Configuration for Helm upgrade actions.

For full details, refer to the Flux documentation for [install](https://fluxcd.io/flux/components/helm/helmreleases/#install-configuration){:target="_blank"} and [upgrade](https://fluxcd.io/flux/components/helm/helmreleases/#upgrade-configuration){:target="_blank"}.

## Failure Strategies

A common requirement is to automatically retry or remediate failed deployments. For example, a configuration that attempts to uninstall and re-install a failed release up to 10 times:

```yaml
spec:
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

To avoid manually defining these verbose configurations for every release, KuboCD allows you to define named strategies in the global configuration (`Config`).

### Configuring Global Strategies

Sample `Config` resource:

```yaml
apiVersion: kubocd.kubotal.io/v1alpha1
kind: Config
metadata:
  name: global-config
spec:
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

**Defined Strategies:**

- **`stopOnFailure`**: Do nothing on failure (default Flux behavior).
- **`reinstallOnFailure`**: Uninstall and re-install up to 10 times.
- **`updateOnFailure`**: Retry the update every minute until success.

**Default Settings:**

- **`defaultOnFailureStrategy`**: Sets the strategy used by default for all deployments (`updateOnFailure` in this example).
- **`defaultHelmTimeout`**: Sets the default `HelmRelease.spec.timeout` (3 minutes).

### Applying Strategies

Strategies can be applied or overridden at multiple levels:

1. **Global Default**: As defined in `defaultOnFailureStrategy`.
2. **Package Level**: Override per module using `package.module[X].onFailureStrategy` and `timeout`.
3. **Release Level**: Override per module using `moduleOverrides`.

**Example: Overriding at Release Level**

```yaml
apiVersion: kubocd.kubotal.io/v1alpha1
kind: Release
metadata:
  name: podinfo1
  namespace: default
spec:
  package:
    repository: quay.io/kubodoc/packages/podinfo
    tag: 6.7.1-p01
  moduleOverrides:
    main:
      onFailureStrategy: reinstallOnFailure
      timeout: 2m
```

In this example, the `main` module will use the `reinstallOnFailure` strategy and a custom timeout of 2 minutes.