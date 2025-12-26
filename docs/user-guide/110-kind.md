# Installing KuboCD on a Local Kind Cluster

This guide details the process of setting up a local Kubernetes cluster using **Kind (Kubernetes in Docker)**, followed by the installation of **FluxCD** and **KuboCD**.

!!! tip
    **Existing Cluster?**  
    If you already have a Kubernetes cluster, you can skip the cluster creation steps and proceed directly to [Installation on an Existing Cluster](120-existing-cluster.md).

---

## Prerequisites

Ensure the following tools are installed on your workstation:

- [Docker](https://www.docker.com/){:target="_blank"}
- [kubectl](https://kubernetes.io/docs/tasks/tools/){:target="_blank"}
- [Helm](https://helm.sh/){:target="_blank"}
- [Kind](https://kind.sigs.k8s.io/){:target="_blank"}
- [Flux CLI](https://fluxcd.io/flux/installation/#install-the-flux-cli){:target="_blank"}

Please ensure that:

- Docker is running.
- You have an active internet connection.
- Ports **80** and **443** are available on your local machine.

You will also need access to an OCI-compatible container registry with push permissions. This is required for storing KuboCD Packages as OCI artifacts.

---

## Create a Kind Cluster

First, create a configuration file that defines the necessary port mappings for the ingress controller:

```{ .bash .copy }
cat >/tmp/kubodoc-config.yaml <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: kubodoc
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30080
    hostPort: 80
    protocol: TCP
  - containerPort: 30443
    hostPort: 443
    protocol: TCP
EOF
```

!!! note
    The `extraPortMappings` configuration exposes services, such as the ingress controller, directly to your local machine.

Next, create the cluster using the configuration file:

```{ .bash .copy }
kind create cluster --config /tmp/kubodoc-config.yaml
```

This command provisions a single-node cluster that functions as both the control plane and a worker node.

**Example Output:**

```bash
Creating cluster "kubodoc" ...
 ✓ Ensuring node image (kindest/node:v1.32.2)
 ✓ Preparing nodes
 ✓ Writing configuration
 ✓ Starting control-plane
 ✓ Installing CNI
 ✓ Installing StorageClass
Set kubectl context to "kind-kubodoc"
You can now use your cluster with:

kubectl cluster-info --context kind-kubodoc
```

Verify that the cluster is operational:

```{ .bash .copy }
kubectl get pods -A
```

```text
NAMESPACE            NAME                                            READY   STATUS    RESTARTS   AGE
kube-system          coredns-668d6bf9bc-nwzqj                        1/1     Running   0          52s
kube-system          coredns-668d6bf9bc-xgv9f                        1/1     Running   0          52s
kube-system          etcd-kubodoc-control-plane                      1/1     Running   0          59s
kube-system          kindnet-xwfp8                                   1/1     Running   0          52s
kube-system          kube-apiserver-kubodoc-control-plane            1/1     Running   0          59s
kube-system          kube-controller-manager-kubodoc-control-plane   1/1     Running   0          58s
kube-system          kube-proxy-6hv6w                                1/1     Running   0          52s
kube-system          kube-scheduler-kubodoc-control-plane            1/1     Running   0          59s
local-path-storage   local-path-provisioner-7dc846544d-k8bhb         1/1     Running   0          52s
```

---

## Install Flux

### Install the Flux CLI

If not already installed, please refer to the [Flux CLI installation guide](https://fluxcd.io/flux/installation/#install-the-flux-cli){:target="_blank"}.

### Deploy Flux (Standalone Mode)

Perform a standalone installation of Flux (without linking a Git repository initially):

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

Deploy KuboCD using Helm:

```{ .bash .copy }
helm -n kubocd install kubocd-ctrl --create-namespace oci://quay.io/kubocd/charts/kubocd-ctrl --version v0.3.0
```

---

## Webhook-Based Features

Certain advanced **KuboCD** features, such as **Release protection**, depend on a Kubernetes **validating webhook**.

Enabling this webhook requires [**cert-manager**](https://cert-manager.io/){:target="_blank"} for TLS certificate management. We will cover how to package and install cert-manager using KuboCD in a later section. Once cert-manager is available, you can proceed to deploy the KuboCD webhook.

---

## Install the KuboCD CLI

Download the KuboCD CLI from the [GitHub releases page](https://github.com/kubocd/kubocd/releases/tag/v0.3.0){:target="_blank"}.

Rename the binary to `kubocd`, make it executable, and move it to a directory in your system `$PATH`:

```{ .bash .copy }
# Replace <binary-name> with the actual downloaded filename
mv <binary-name> kubocd
chmod +x kubocd
sudo mv kubocd /usr/local/bin/
```

Verify the installation:

```{ .bash .copy }
kubocd version
```

You are now ready for [your first deployment with KuboCD](130-a-first-deployment.md).
