# Installing KuboCD on a Local Kind Cluster

This section walks you through setting up a local Kubernetes cluster using Docker and **Kind**,
then installing **FluxCD** and **KuboCD** on top of it.

!!! tip

    **Already have a Kubernetes cluster?**  
    You can skip the cluster creation and follow the instructions in [Installation on an existing cluster](120-existing-cluster.md).

---

## Prerequisites

Ensure the following tools are installed on your workstation:

- [Docker](https://www.docker.com/){:target="_blank"}.
- [kubectl](https://kubernetes.io/docs/tasks/tools/){:target="_blank"}.
- [Helm](https://helm.sh/){:target="_blank"}.
- [Kind](https://kind.sigs.k8s.io/){:target="_blank"}.
- [Flux CLI](https://fluxcd.io/flux/installation/#install-the-flux-cli){:target="_blank"}.

Make sure:

- Docker is running
- You have an active internet connection
- Ports **80** and **443** are available on your local machine

You also need an access to an OCI-compatible container registry with permissions to push images.
This is will be necessary for uploading and storing KuboCD Packages as OCI artifacts.

---

## Create the Kind Cluster

Create a configuration file with ingress-compatible port mappings:

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
    The `extraPortMappings` allow direct access to services like the ingress controller from your local machine.

Then create the cluster:

```{ .bash .copy }
kind create cluster --config /tmp/kubodoc-config.yaml
```

This will create a single-node cluster acting as both control plane and worker node.

Example output:

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

Verify everything is running:

```{ .bash .copy }
kubectl get pods -A
```

```{ .bash }
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

If not already installed, follow the [Flux CLI installation guide](https://fluxcd.io/flux/installation/#install-the-flux-cli){:target="_blank"}..

### Deploy Flux (Basic Mode)

We’ll begin with a basic installation of Flux (no Git repository linked for now):

```{ .bash .copy }
flux install
```

``` { .bash }
✚ generating manifests
✔ manifests build completed
► installing components in flux-system namespace
...
✔ notification-controller: deployment ready
✔ source-controller: deployment ready
✔ install finished
```

Verify deployment:

``` { .bash .copy }
kubectl -n flux-system get pods
```

``` { .bash }
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

Deploy KuboCD using Helm:

```{ .bash .copy }
helm -n kubocd install kubocd-ctrl --create-namespace oci://quay.io/kubocd/charts/kubocd-ctrl --version v0.2.3
```
---

## About Webhook-Based Features

Some advanced features in **KuboCD** such as **Release protection** rely on a Kubernetes **validating webhook.**

To enable these features, you need to deploy a webhook component alongside the controller. 

But, this webhook requires [**cert-manager**](https://cert-manager.io/){:target="_blank"}. to be installed in your cluster to handle TLS certificate provisioning. 
We’ll show you how to package and install it with KuboCD in a later section. After that, you will be able to deploy the KuboCD webhook. 

---

## Install the KuboCD CLI

Download the KuboCD CLI from the [GitHub releases page](https://github.com/kubocd/kubocd/releases/tag/v0.2.3){:target="_blank"}
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

