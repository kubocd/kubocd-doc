# Setting Up the Ingress Controller

!!! warning
    If you're using an existing cluster, there's likely already an ingress controller installed. **Do not install another one**.  
    However, you should still read this section as several new features are described.

If you're following the local `kind` cluster setup, you’ll need to install an ingress controller to make use of the deployed `Ingress` object.

Check the current `Ingress` object created by the `podinfo` deployment:

```bash
kubectl get ingresses

NAME            CLASS   HOSTS                            ADDRESS   PORTS   AGE
podinfo1-main   nginx   podinfo1.ingress.kubodoc.local             80      6m33s
```

At this point, the ingress is inactive since no ingress controller is installed.

---

## Build the ingress-nginx package:

Here is a sample package definition for deploying the `ingress-nginx` controller:

!!! warning
    This first version is dedicated to the way we setup the cluster, using the kind portMapping and NodePorts


???+ abstract "ingress-nginx-p01.yaml"

    ``` { .yaml .copy }
    apiVersion: v1alpha1
    name: ingress-nginx
    tag: 4.12.1-p01
    protected: true
    modules:
      - name: main
        specPatch:
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

New key points compared to the `podinfo` Package:

- `protected: true`: Prevents accidental deletion of the release. (Currently not enforced unless KuboCD webhook is installed.)
- `timeout: 4m`: Overrides the default deployment timeout (`2m`) because this Helm chart may take some time to deploy.
- `values`: This section is in proper YAML format (no '|': not a templated string), since it does not include any templating.
- `roles`: Assigns the package to the `ingress` role. This is used for [dependency management](dependencies.md) between releases.

Build the package:

``` { .bash .copy }
kubocd pack ingress-nginx-p01.yaml
```

``` { .bash }
====================================== Packaging package 'ingress-nginx.yaml'
--- Handling module 'main':
Fetching chart ingress-nginx:4.12.1...
Chart: ingress-nginx:4.12.1
--- Packaging
Generating index file
Wrap all in assembly.tgz
--- push OCI image: quay.io/kubodoc/packages/ingress-nginx:4.12.1-p01
Successfully pushed
```

---

## Deploy the package

Then define the `Release` resource:

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
        interval: 30m
      targetNamespace: ingress-nginx
      createNamespace: true
    ```

Key points:

- `metadata.namespace: kubocd`: As it is a system components, deploy in a restricted namespace.
- `spec.protected: false`: Just to demonstrates that the package-level `protected` flag can be overridden at the release level.
- `spec.targetNamespace: ingress-nginx`: Installs the ingress controller in its own namespace.
- `spec.createNamespace: true`: Automatically creates the target namespace if it doesn't exist.

Apply the release:

``` { .bash .copy }
kubectl apply -f ingress-nginx.yaml
```

Check the release status:

``` { .bash .copy }
kubectl -n kubocd get release
```

``` { .bash }
NAME            REPOSITORY                               TAG          CONTEXTS   STATUS   READY   WAIT   PRT   AGE   DESCRIPTION
ingress-nginx   quay.io/kubodoc/packages/ingress-nginx   4.12.1-p01              READY    1/1            -     86s   The Ingress controller
```

---

## Configure the DNS entry

To access the `podinfo` application, you'll need to define a DNS entry matching the `fqdn` parameter.

The simplest way in our case is to use the `/etc/hosts` file:

``` { .text .copy }
127.0.0.1 localhost podinfo1.ingress.kubodoc.local
```

Make sure the hostname matches **exactly** what was provided in the Release parameters.

You should now be able to access the 'podinfo` web server:

👉 [http://podinfo1.ingress.kubodoc.local](http://podinfo1.ingress.kubodoc.local)

