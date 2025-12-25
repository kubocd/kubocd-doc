# Setting Up the Ingress Controller

!!! warning
    **Existing Clusters:** If you are using an existing cluster, an ingress controller is likely already installed. **Do not install another one.**
    However, we recommend reading this section as it introduces several new features.

If you are following the local `kind` cluster guide, you must install an ingress controller to support the `Ingress` resources.

Check the current `Ingress` object created by the `podinfo` deployment:

``` { .base .copy }
kubectl get ingresses
```

``` { .bash }
NAME            CLASS   HOSTS                            ADDRESS   PORTS   AGE
podinfo1-main   nginx   podinfo1.ingress.kubodoc.local             80      6m33s
```

The ingress is currently inactive because no controller is running.

---

## Build the Ingress-Nginx Package

Below is a sample package definition for the `ingress-nginx` controller.

!!! warning
    This configuration is specific to our local Kind cluster setup, leveraging `portMapping` and `NodePorts`.

???+ abstract "ingress-nginx-p01.yaml"

    ``` { .yaml .copy }
    apiVersion: v1alpha1
    name: ingress-nginx
    tag: 4.12.1-p01
    protected: true
    modules:
      - name: noname
        timeout: 4m
        source:
          helmRepository:
            url: https://kubernetes.github.io/ingress-nginx
            chart: ingress-nginx
            version: 4.12.1
        values:
          controller:
            extraArgs:
              enable-ssl-passthrough: true
            service:
              type: NodePort
              nodePorts:
                http: "30080"
                https: "30443"
    roles:
      - ingress
    ```

**Key Differences from the `podinfo` Package:**

- **`protected: true`**: Prevents accidental deletion of the release. (Requires the KuboCD webhook to be enforced).
- **`modules[0].name`**: The value `noname` prevents appending the module name to the Helm release and derived objects.
- **`modules[0].timeout: 4m`**: Overrides the default timeout (`3m`) to accommodate the Helm chart deployment time.
- **`modules[0].values`**: Pure YAML is used here instead of a templated string since no dynamic parameters are needed.
- **`roles`**: Assigns the package to the `ingress` role. This is used for dependency management (discussed later).

Build the package:

``` { .bash .copy }
kubocd pack packages/ingress-nginx-p01.yaml
```

``` { .bash }
====================================== Packaging package 'ingress-nginx.yaml'
--- Handling module 'noname':
Fetching chart ingress-nginx:4.12.1...
Chart: ingress-nginx:4.12.1
--- Packaging
Generating index file
Wrap all in assembly.tgz
--- push OCI image: quay.io/kubodoc/packages/ingress-nginx:4.12.1-p01
Successfully pushed
```

---

## Deploy the Package

Define the `Release` resource:

???+ abstract "ingress-nginx.yaml"

    ``` { .yaml .copy }
    ---
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Release
    metadata:
      name: ingress-nginx
      namespace: kubocd
    spec:
      description: The Ingress controller
      protected: false
      package:
        repository: quay.io/kubodoc/packages/ingress-nginx
        tag: 4.12.1-p01
      targetNamespace: ingress-nginx
      createNamespace: true
    ```

**Key Features:**

- **`metadata.namespace: kubocd`**: Deployed in a restricted system namespace.
- **`spec.protected: false`**: Demonstrates overriding the package-level `protected` flag at the release level.
- **`spec.targetNamespace: ingress-nginx`**: Installs the controller in its own dedicated namespace.
- **`spec.createNamespace: true`**: Automatically creates the target namespace if missing.
- **`spec.package.interval`**: Defaults to `30m`.

Apply the release:

``` { .bash .copy }
kubectl apply -f releases/ingress-nginx.yaml
```

Check the status:

``` { .bash .copy }
kubectl -n kubocd get release
```

``` { .bash }
NAME            REPOSITORY                               TAG          CONTEXTS   STATUS   READY   WAIT   PRT   AGE   DESCRIPTION
ingress-nginx   quay.io/kubodoc/packages/ingress-nginx   4.12.1-p01              READY    1/1            -     86s   The Ingress controller
```

Verify the pod:

``` { .bash .copy }
kubectl -n ingress-nginx get pods
```

``` { .bash }
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-658fbb96d5-hctqp   1/1     Running   0          4m12s
```

> **Note:** If the pod name still includes `noname` (e.g., `ingress-nginx-noname-controller-...`), you may be using a KuboCD version older than `v0.2.3`.

---

## Configure DNS

To access `podinfo`, you need a DNS entry matching the `fqdn` parameter.

The simplest method is to add an entry to your `/etc/hosts` file:

``` { .text .copy }
127.0.0.1 localhost podinfo1.ingress.kubodoc.local
```

Ensure the FQDN matches **exactly** what was defined in the `Release` parameters.

You should now be able to access the `podinfo` web server:

👉 [http://podinfo1.ingress.kubodoc.local](http://podinfo1.ingress.kubodoc.local){:target="_blank"}
