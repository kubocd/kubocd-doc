# Installing KuboCD on an Existing Cluster

If you already manage a Kubernetes cluster, you can use it to test KuboCD.

!!! tip
    **No cluster?**
    If you do not have a cluster available, you can [install a Kind cluster on your local workstation](./110-kind.md).

---

## Prerequisites

Ensure the following tools are installed on your workstation:

- [Docker](https://www.docker.com/){:target="_blank"}
- [kubectl](https://kubernetes.io/docs/tasks/tools/){:target="_blank"}
- [Helm](https://helm.sh/){:target="_blank"}
- [Flux CLI](https://fluxcd.io/flux/installation/#install-the-flux-cli){:target="_blank"}

Please ensure that:

- Docker is running.
- You have an active internet connection.
- You have full administrative privileges on the target cluster.

You will also need access to an OCI-compatible container registry with push permissions. This is required for storing KuboCD Packages as OCI artifacts.

---

## Install Flux

### Install the Flux CLI

If not already installed, please refer to the [Flux CLI installation guide](https://fluxcd.io/flux/installation/#install-the-flux-cli){:target="_blank"}.

### Deploy Flux (Standalone Mode)

If Flux is not installed on your cluster, perform a standalone installation (without linking a Git repository initially):

```{ .bash .copy }
flux install
```

```text
✚ generating manifests
✔ manifests build completed
► installing components in flux-system namespace
...
✔ notification-controller: deployment ready
✔ source-controller: deployment ready
✔ install finished
```

Verify the deployment:

```{ .bash .copy }
kubectl -n flux-system get pods
```

```text
NAME                                       READY   STATUS    RESTARTS   AGE
helm-controller-b6767d66-q27gd             1/1     Running   0          14m
kustomize-controller-5b56686fbc-hpkhl      1/1     Running   0          14m
notification-controller-58ffd586f7-bbvwv   1/1     Running   0          14m
source-controller-6ff87cb475-hnmxv         1/1     Running   0          14m
```

!!! tip
    **Minimal Installation**  
    For a lighter setup, you can install only the components required by KuboCD:  
    `flux install --components source-controller,helm-controller`

---

## Install KuboCD

Deploy the KuboCD controller using Helm:

```{ .bash .copy }
helm -n kubocd install kubocd-ctrl --create-namespace oci://quay.io/kubocd/charts/kubocd-ctrl --version v0.3.0
```

---

## Enable Webhook-Based Features

Certain advanced **KuboCD** features, such as **Release protection**, depend on a Kubernetes **validating webhook**.

To enable these features, you must deploy the webhook component alongside the controller. This webhook requires [**cert-manager**](https://cert-manager.io/){:target="_blank"} to handle TLS certificate provisioning.

If `cert-manager` is already installed on your cluster, you can deploy the webhook using:

```{ .bash .copy }
helm -n kubocd install kubocd-wh oci://quay.io/kubocd/charts/kubocd-wh --version v0.3.0
```

!!! note
    **No cert-manager?**
    If you don't have `cert-manager` installed, don't worry. We will cover how to package and install it using KuboCD in a later section.

---

## Install the KuboCD CLI

Download the KuboCD CLI from the [GitHub releases page](https://github.com/kubocd/kubocd/releases/tag/v0.3.0){:target="_blank"}.

Rename the binary to `kubocd`, make it executable, and move it to a directory in your system `$PATH`:

```{ .bash .copy }
# Replace <binary-name> with the downloaded filename
mv <binary-name> kubocd
chmod +x kubocd
sudo mv kubocd /usr/local/bin/
```

Verify the installation:

```{ .bash .copy }
kubocd version
```

You are now ready for [your first deployment with KuboCD](130-a-first-deployment.md).
