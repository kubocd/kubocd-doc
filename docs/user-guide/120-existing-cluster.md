# Installing KuboCD on an existing cluster.

If you have an existing cluster, you can use it to test KuboCD

!!! tip
    If you don't have one, you can [use a Kind cluster](./110-kind.md) on your local workstation

---

## Prerequisites

Ensure the following tools are installed on your workstation:

- [Docker](https://www.docker.com/){:target="_blank"}.
- [kubectl](https://kubernetes.io/docs/tasks/tools/){:target="_blank"}.
- [Helm](https://helm.sh/){:target="_blank"}.
- [Flux CLI](https://fluxcd.io/flux/installation/#install-the-flux-cli){:target="_blank"}.

Make sure:

- Docker is running
- You have an active internet connection
- You have full admin rights on the target cluster.

You also need an access to an OCI-compatible container registry with permissions to push images.
This is will be necessary for uploading and storing KuboCD Packages as OCI artifacts.

---

## Install Flux

### Install the Flux CLI

If not already installed, follow the [Flux CLI installation guide](https://fluxcd.io/flux/installation/#install-the-flux-cli){:target="_blank"}..

### Deploy Flux (Basic Mode)

If Flux is not already installed on your cluster, we’ll begin with a basic installation (no Git repository linked for now):



```{ .bash .copy }
flux install
```

``` { .bash  }
✚ generating manifests
✔ manifests build completed
► installing components in flux-system namespace
...
✔ notification-controller: deployment ready
✔ source-controller: deployment ready
✔ install finished
```

Verify deployment:

```{ .bash .copy }
kubectl -n flux-system get pods
```

```{ .bash  }
NAME                                       READY   STATUS    RESTARTS   AGE
helm-controller-b6767d66-q27gd             1/1     Running   0          14m
kustomize-controller-5b56686fbc-hpkhl      1/1     Running   0          14m
notification-controller-58ffd586f7-bbvwv   1/1     Running   0          14m
source-controller-6ff87cb475-hnmxv         1/1     Running   0          14m
```

!!! tip
    **💡 Want a minimal install?**  
    You can limit Flux to the required components for KuboCD:  
    `flux install --components source-controller,helm-controller`

---

## Install KuboCD

Deploy the KuboCD controller using Helm:

```{ .bash .copy }
helm -n kubocd install kubocd-ctrl --create-namespace oci://quay.io/kubocd/charts/kubocd-ctrl:v0.2.0
```

---

## Enabling Webhook-Based Features

Some advanced features in **KuboCD** such as **Release protection** rely on a Kubernetes **validating webhook.**

To enable these features, you need to deploy a webhook component alongside the controller. This webhook requires 
[**cert-manager**](https://cert-manager.io/){:target="_blank"}. to be installed in your cluster to handle TLS certificate provisioning.

If you already have `cert-manager` installed, you can deploy the webhook with the following command:

```{ .bash .copy }
helm -n kubocd install kubocd-wh oci://quay.io/kubocd/charts/kubocd-wh:v0.2.0
```

!!! note
    Don’t have `cert-manager` yet? No problem !
    <br>We’ll show you how to package and install it with KuboCD in a later section.

---

## Install the KuboCD CLI

Download the KuboCD CLI from the [GitHub releases page](https://github.com/kubocd/kubocd/releases/tag/v0.2.0){:target="_blank"}..
and rename it to `kubocd`. Then make it executable and move it to your path:

```{ .bash .copy }
mv kubocd_*_* kubocd
chmod +x kubocd
sudo mv kubocd /usr/local/bin/
```

Verify the installation:

```{ .bash .copy }
kubocd version
```

You can now move to [your first deployment with KuboCD](130-a-first-deployment.md)
