# A first deployment with KuboCD

## Package definition

For this first deployment, we will take a simple use case: A tiny web application: [podinfo](https://github.com/stefanprodan/podinfo)

A Package is defined by a YAML manifest. Here is a first version to wrap up the `podinfo` application:

???+ abstract "podinfo-p01.yaml"

    ```{ .yaml .copy }
    apiVersion: v1alpha1
    type: Package
    name: podinfo
    tag: 6.7.1-p02
    schema:
      parameters:
        $schema: http://json-schema.org/schema#
        additionalProperties: false
        properties:
          fqdn:
            type: string
          ingressClassName:
            default: nginx
            type: string
        required:
          - fqdn
        type: object
    modules:
      - name: main
        specPatch:
          timeout: 2m
        source:
          helmRepository:
            url: https://stefanprodan.github.io/podinfo
            chart: podinfo
            version: 6.7.1
        values: |
          ingress:
            enabled: true
            className: {{ .Parameters.ingressClassName  }}
            hosts:
              - host: {{ .Parameters.fqdn }}
                paths:
                  - path: /
                    pathType: ImplementationSpecific
    ```


> A KuboCD Package is NOT a Kubernetes resources.

Here is a description of the attributes used in this sample: 

- `apiVersion`: (R) The version of this YAML format. Currently only allowed value is `v1alpha1`.
- `type`: The type of resource described. `Package` is the default and the only allowed value. It could be omitted.
- `name`: The name of the package. Will be used also in the image name.
- `tag`: (R) The tag of the OCI image and is used to record the version of the package. Technically, it could be whatever you want, but we suggest the following convention:
    - Use the version of the main module Helm chart and add a `-pXX` suffix where XX allow versioning of the packaging. (The same Helm chart can be packaged in different ways). 
- `schema.parameters`: When the package will be deployed, some parameters will be provided. This schema allow validation 
   (And documentation) of these parameters. It is a standard OpenAPI/json schema.
   If not defined, this means the Release will not accept parameters.
- `modules`: (R) A package include one or several Helm charts, each one being referenced by a `module` entry.
- `modules[X].name`: (R) Each module must have a name. As there is only one here, we call it `main`.
- `modules[X].source`: (R) Where to find the Helm chart. In this example in an Helm Repository provided by the authors of
  `podinfo`. The `source` can also be an OCI Repository, a GIT repository or a local Helm Chart.
- `values`: Defines a template that will be rendered to generate the `values.yaml` file used for deploying
  the Helm chart.
    - The templating engine used is the same as Helm's. More on this [here](TODO)
    - However, the data model is different. It includes, in particular, a root object `.Parameters` that contains values
      which will be defined during deployment, by the `Release` object.
    - Although it may look like a `yaml` snippet, it is in fact a string, to allow insertion of template directive.

(R) means Required

> Of course, there is more attributes than the ones of this sample. There wil be described later in the documentation.

---

## Package build

Maintenant, il convient de générer l'image OCI correspondante.

As stated earlier, you need an access to an OCI-compatible container registry with permissions to push images.
This is will be necessary for uploading and storing KuboCD Packages as OCI artifacts.

Currently, `quay.io` (redhat), `ghcr.io` (Github) and the [distribution registry](https://github.com/distribution/distribution) 
has been testes an known to work. Although not tested, other should also works, except Docker Hub.

You must be logged on this repo with appropriate permissions. (i.e `docker login quay.io`, or `docker login ghcr.io`, ...)

Depending of the registry, you may have a concept of 'organisation'. In this sample, we will use quay.io with a `kubodoc` organization.

To build the package, save the package file `podinfo-p01.yaml` in some place and enter a command like:

```{.bash .copy}
kubocd package podinfo-p01.yaml --ociRepoPrefix quay.io/kubodoc/packages
```

> Of course, you must adjust the value of `--ociRepoPrefix` to your environment. Note the sub-path `package` is arbitrary and could be anything you want, or empty.

The repository name will be build by concatenating the value of `--ociRepoPrefix` and the name of the package (Provided in the package manifest). And obviously, the tag will be taken from the definition.

The output should look like:

```
====================================== Packaging package 'podinfo-p01.yaml'
--- Handling module 'main':
Fetching chart podinfo:6.7.1...
Chart: podinfo:6.7.1
--- Packaging
Generating index file
Wrap all in assembly.tgz
--- push OCI image: quay.io/kubodoc/packages/podinfo:6.7.1-p01
Successfully pushed
```

Instead of providing the `--ociRepoPrefix` on each run, you may also define an environment variable:

```{.bash .copy}
export OCI_REPO_PREFIX=quay.io/kubodoc/packages
```

or:

```{.bash .copy}
export OCI_REPO_PREFIX=ghcr.io/kubodoc/packages
```

or

```{.bash .copy}
export OCI_REPO_PREFIX=localhost:5000/packages
```

Last step: ensure your image is `public` (Initial default may be `private`).

> If you want the image to stay `private`, you will have to provide authentication information on the `Release`. This will be described later in this doc.

---

## Releasing the application

To deploy our application, a `Release` custom resource is used:

???+ abstract "podinfo1.yaml"
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
    ```

The attributes used here in the 'spec' part:

- `description`: Optional. A short description of the release.
- `package.repository`: The OCI repository to fetch the package image. See the packaging step above.
- `package.tag`: The tag/version of the package. See the packaging step above.
- `package.interval`: Interval between OCI image reconciliation. In other word, the maximum duration to wait for a package modification to be taken in account.
- `parameters`: An object which comply the the `parameter.schema` defined in the Package. Just a single value in this first sample.

To deploy our application, save the above file in some place and edit it:

- Adjust the `spec.package.repository` to whatever was set on package built.
- Adjust the `spec.parameters.fqdn` to a value appropriate for your network configuration.

then, you can proceed to deployment:

```{.bash}
kubectl create -f podinfo1.yaml

release.kubocd.kubotal.io/podinfo1 created
```

If everything go well, the status of the newly created `Release` object should go to different `WAIT_XXX` steps and ends to reach `READY` state. 

```{.bash}
kubectl get releases

NAME       REPOSITORY                         TAG         CONTEXTS   STATUS   READY   WAIT   PRT   AGE     DESCRIPTION
podinfo1   quay.io/kubodoc/packages/podinfo   6.7.1-p01              READY    1/1            -     6m40s   A first sample release of podinfo
```

At this point, a `podinfo` pod should be deployed in the same namespace as the `Release` (Here: default):

```{.bash}
kubectl get pods

NAME                             READY   STATUS    RESTARTS   AGE
podinfo1-main-779b6b9fd4-zbgbx   1/1     Running   0          8h
```

---

## Under the hood (If things go wrong)

The KuboCD controller build up several Flux object to handle the deployment.

For a successful operation, here are the events attached to the `Release` object:

```{.bash}
Events:
Type    Reason                 Age   From     Message
  ----    ------                 ----  ----     -------
Normal  OCIRepositoryCreated   12s   release  Created OCIRepository "kcd-podinfo1"
Normal  HelmRepositoryCreated  10s   release  Created HelmRepository "kcd-podinfo1"
Normal  HelmReleaseCreated     8s    release  Created HelmRelease "podinfo1-main"
```

> This display is provided by [k9s](https://k9scli.io/)

All this objects are created in the same namespace as the Release one (here: default)

### The OCIRepository

This Flux custom resource is in charge of fetching the KuboCD package OCI image.

```{.bash}
kubectl get OCIRepository

NAME           URL                                      READY   STATUS                                                                                                           AGE
kcd-podinfo1   oci://quay.io/kubodoc/packages/podinfo   True    stored artifact for digest '6.7.1-p01@sha256:985e4e2f89a4b17bd5cc2936a0b305df914ae479e0b8c96e61cb22725b61cd24'   9m1s
```

If your release is stuck in the `WAIT_OCI` state, have a look on the status of this resource, or in its associated events.

Most usual reason are a wrong URL or the OCI image still in `private` mode.

If you manually delete this resource, the KuboCD controller will recreate it. This can be useful to force a relaod of a modified OCI image.

### The HelmRepository

The podinfo Helm chart has been embedded in the package. But we need to make it available for the deployment. 

For this, the KuboCD controller act as a cluster internal HelmRepository server. And a HelmRepository Flux custom resource is created to reference this server.

```
kubectl get HelmRepository

NAME           URL                                                                            AGE    READY   STATUS
kcd-podinfo1   http://kubocd-ctrl-controller-helm-repository.kubocd.svc/hr/default/podinfo1   105m   True    stored artifact: revision 'sha256:d8db03cf45ecd75064c2a2582812dc4df5cd624d0e295b24ff79569bf46a070b'
```

It is unusual to get somme error on this object.

### The HelmRelease

This Flux custom resources is in charge of the effective deployment of the Helm chart.

```{.bash}
kubectl get HelmRelease

NAME            AGE     READY   STATUS
podinfo1-main   7m59s   True    Helm install succeeded for release default/podinfo1-main.v1 with chart podinfo@6.7.1
```

There will be one `HelmRelease` per module in the Package.

If your `Release` is stuck in the `WAIT_HREL` state, the reason is in the status of this object, or in its associated events.

But, in most case, you will have to wait for a timeout on failure to expire to have some relevant information. 
Default value is 2mn, but is configurable on package definition or on Release. 

If you dig inside the object, you will find, among other informations, the effective values applied:

```{.bash}
kubectl get HelmRelease podinfo1-main -o yaml

apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  ....
spec:
  ....
  values:
    ingress:
      className: nginx
      enabled: true
      hosts:
      - host: podinfo1.ingress.kubodoc.local
        paths:
        - path: /
          pathType: ImplementationSpecific
  ...
```

---

## Release deletion

To fully uninstall the application, just delete the Release KuboCD object.

All generated flux CD resource are owned by this Release object. This means they will be deleted with the parent Release object.

And Flux will handle the application removal on the HelmRelease deletion.

---

## Setting up the ingress controller.

> WARNING: If you are using an existing cluster, an ingress controller exists probably. In such case, DON'T install this one. 
But, don't skip this section, as some new information are provided. So, just read in, without performing any action.

If you follow the local Kind cluster path, you will need to effectively apply this.

The deployment has created an `ingress` object. But it is currently ineffective, as there is no ingress controller. 

```{.bash}
kubectl get ingresses

NAME            CLASS   HOSTS                            ADDRESS   PORTS   AGE
podinfo1-main   nginx   podinfo1.ingress.kubodoc.local             80      6m33s
```

Here is a KuboCD package for the nginx ingress controller.

???+ abstract "ingress-nginx-p01.yaml"

    ```{ .yaml .copy }
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

> This first version is dedicated to the way we setup the cluster, using the kind `portMapping` and `NodePorts`

Compare to the `podinfo` package built previously, there is some addition:

- There is a `protected` attribute. This is a flag which will prevent deletion of the `Release` object. But,
 at this stage, this is not enforced, as this feature require the KuboCD webhook to be installed.
- There is a `timeout` attribute set to `4m`. This override the default value of `2m`, as the deployment may take some 
  time, due to some jobs to be executed.
- The `values` attribute is a map. As there is no templating, this part is valid YAML, so no need to consider this as 
  string. KuboCD will auto-detect the type.
- There is a `roles` list. This is related to the [dependencies management](/user-guide/dependencies).

To build this package, save the file in some place and:

```{.bash}
kubocd pack ingress-nginx-p01.yaml

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

Create the `Release' resource:

???+ abstract "ingress-nginx.yaml"

    ```
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

Some point to note:

- `spec.protected: false`: Just to demonstrate the `protected` flag added in the package can be overridden at the Release level.
- `metadata.namespace: kubocd`: As it is a 'system' component, we better create it in a namespace which will not be freely accessible (All associated Flux custom resources will also be create in the `kubocd` namespace)
- `spec.targetNamespace: ingress-nginx`: For the application itself, we don't want to deploy in the `kubocd` namespace but in a dedicated one.
- `spec.createNamespace: true:` We need to create this dedicated namespace, it it doesn't exists.

Save this file and apply it:

```{.bash}
kubectl create -f ingress-nginx.yaml
 
release.kubocd.kubotal.io/ingress-nginx created
```

Check the `Release` reach the `READY` state:

```{.bash}
kubectl -n kubocd  get release

NAME            REPOSITORY                               TAG          CONTEXTS   STATUS   READY   WAIT   PRT   AGE   DESCRIPTION
ingress-nginx   quay.io/kubodoc/packages/ingress-nginx   4.12.1-p01              READY    1/1            -     86s   The Ingress controller
```

Then, to access the `podinfo`application, you need to define corresponding DNS entry.

The simple way in our case is to user the `/etc/hosts` file:

```{.bash}
127.0.0.1 localhost podinfo1.ingress.kubodoc.local
```

> The value must strictly match the one set in the `fqdn` podinfo parameters.

Now, [http://podinfo1.ingress.kubodoc.local](http://podinfo1.ingress.kubodoc.local) should works.


## Alternate KuboCD Schema.

In this first sample, the `schema.parameters` has been defined using a standard OpenAPI schema.
But this definition turn out to be quite verbose. So a new, simplified version has been introduced for KuboCD.

Here is the a `podnfo` identical package , using he KuboCD definition:

???+ abstract "podinfo-p02.yaml"

    ```{ .yaml .copy }
    apiVersion: v1alpha1
    type: Package
    name: podinfo
    tag: 6.7.1-p01
    schema:
      parameters:
        properties:
          fqdn: { type: string, required: true }
          ingressClassName: { type: string, default: "nginx"}
    modules:
      - name: main
        source:
          helmRepository:
            url: https://stefanprodan.github.io/podinfo
            chart: podinfo
            version: 6.7.1
        values: |
          ingress:
            enabled: true
            className: {{ .Parameters.ingressClassName  }}
            hosts:
              - host: {{ .Parameters.fqdn }}
                paths:
                  - path: /
                    pathType: ImplementationSpecific
    
    ```

A bit shorter isn't it.

Cette définition n'est pas directement compatible avec le standard. Mais elle est facilement convertible vers celui-ci.
C'est d'ailleurs ce que fait KuboCD en interne.

A noter que c'est le token `$schema:` qui permet de distinguer un format standard d'un format KuboCD

Cette definition spécifique de schema fait l'objet d'un [chapitre dédié](../user-guide/schemas.md).


