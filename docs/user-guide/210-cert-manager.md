# Cert-manager setup

!!! warning
    If you're using an existing cluster, there's likely already a cert-manager installed. **Do not install another one**.  
    However, you should still read this section as several new features are described.

If you're following the local `kind` cluster setup, you can safely proceed with this installation.

## Cert and Trust manager

To enable secure communication within the cluster and with external services, we will deploy both cert-manager and trust-manager.

- cert-manager will handle the automated issuance and renewal of TLS certificates
- while trust-manager will manage the distribution of trusted CA certificates across the cluster.

After deploying cert-manager and trust-manager, the next steps involve setting up a self-signed Certificate Authority 
(CA) and creating trust bundles:

- Create a Self-Signed Certificate Authority (CA):

    We will create a private, self-signed CA within the cluster.
    cert-manager will issue and maintain a self-signed certificate that acts as the root CA, 
    which will be used to sign internal service certificates.

- Create a Trust Bundle:
  
    Once the self-signed CA is available, we will create a trust bundle using trust-manager.
    A trust bundle collects and distributes trusted CA certificates (including our self-signed CA) to applications 
    across the cluster, ensuring that services can trust certificates issued by this CA.

## The KuboCD package

All of these components will be grouped into a single package, composed of three modules:

- `main`, which handles the deployment of the cert-manager Helm chart
- `trust`, which handles the deployment of the trust-manager Helm chart
- `issuer`, which handles the deployment of an ad hoc Helm chart that creates the self-signed CA and the Trust Bundle

???+ abstract "cert-manager-p01.yaml"

    ``` { .yaml .copy }
    ---
    apiVersion: v1alpha1
    name: redis
    tag: 20.6.1-p01
    description: Redis and a front UI
    schema:
      parameters:
        $schema: http://json-schema.org/schema#
        type: object
        additionalProperties: false
        description: Redis stack
        required:
          - ui
        properties:
          redis:
            type: object
            additionalProperties: false
            properties:
              password:
                default: redis123
                type: string
              replicaCount:
                default: 1
                description: The number of replicas
                type: integer
            required: []
          ui:
            type: object
            additionalProperties: false
            properties:
              enabled:
                default: true
                type: boolean
              host:
                type: string
            anyOf:
              - properties:
                  enabled: { const: true }
                required:
                  - host
              - properties:
                  enabled: { const: false }
                required: []
      context:
        properties:
          ingress:
            required: true
            properties:
              className: { type: string, default: "nginx"}
              domain: { type: string, required: true }
    modules:
      - name: redis
        timeout: 4m
        source:
          oci:
            repository: registry-1.docker.io/bitnamicharts/redis
            tag: 20.6.1
        values: |
          fullnameOverride: {{ .Release.metadata.name }}-main
          global:
            redis:
              password: {{ .Parameters.redis.password }}
            security:
              allowInsecureImages: true
          master:
            persistence:
              enabled: false
          replica:
            persistence:
              enabled: false
            replicaCount: {{ .Parameters.redis.replicaCount }}
      - name: ui
        enabled: "{{ .Parameters.ui.enabled }}"
        source:
          git:
            url: https://github.com/joeferner/redis-commander.git
            path: ././k8s/helm-chart/redis-commander
            branch: master
        values: |
          fullnameOverride: {{ .Release.metadata.name }}-ui
          redis:
            # host is <fullnameOverride>-<moduleName>-master
            host: {{ printf "%s-master" .Release.metadata.name }}
            password: {{ .Parameters.redis.password }}
          ingress:
            enabled: true
            className: {{ .Context.ingress.className }}
            hosts:
              - host: {{ .Parameters.ui.host }}.{{.Context.ingress.domain}}
                paths:
                  - "/"
        dependsOn:
          - redis
    roles: |
      - redis
      {{ if .Parameters.ui.enabled }}
      - redis-ui
      {{ end }}
    dependencies: |
      {{ if .Parameters.ui.enabled }}
      - ingress
      {{ end }}
    ```


!!! Notes
    More detailed explanations about the usage, configuration, and associated Helm charts of cert-manager and trust-manager are out of scope for this manual.
    Please refer to their official documentation for further information.

### Local Helm Chart

In addition to demonstrating KuboCD’s ability to deploy a relatively complex application stack, this example also 
introduces a new type of source for Helm charts.

As mentioned earlier, the deployment of the self-signed CA and the Trust Bundle is handled by a small ad hoc Helm chart.
This chart is stored alongside the manifest and is referenced via `modules[3].source.local.path`, where the path is 
relative to the location of the manifest.

Like the other modules, this chart will be embedded into the OCI image during packaging.


## Deployment

As usual, a package must be built:

``` { .bash .copy } 
kubocd pack packages/cert-manager-p01.yaml
```

``` { .bash }
====================================== Packaging package 'packages/cert-manager-p01.yaml'
--- Handling module 'main':
    Fetching chart cert-manager:v1.17.1...
    Chart: cert-manager:v1.17.1
--- Handling module 'trust':
    Fetching chart trust-manager:v0.16.0...
    Chart: trust-manager:v0.16.0
--- Handling module 'issuers':
    Fetching chart from '/Users/sa/dev/d1/git/kubocd-doc/samples/charts/cert-issuers/0.1.0'
    Chart: cert-issuers:0.1.0
--- Packaging
    Generating index file
    Wrap all in assembly.tgz
--- push OCI image: quay.io/kubodoc/packages/cert-manager:1.17.1-p01
    Successfully pushed
```

And deployed by creating a `Release` resource:

???+ abstract "cert-manager-p01.yaml"

    ``` { .bash .copy } 
    ---
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Release
    metadata:
      name: cert-manager
      namespace: kubocd
    spec:
      description: The certificate manager
      package:
        repository: quay.io/kubodoc/packages/cert-manager
        tag: 1.17.1-p01
        interval: 30m
      parameters:
        issuers:
          selfSignedClusterIssuers:
            - name: cluster-self
        trust:
          enabled: true
          bundle:
            target:
              configMap:
                enabled: true
              secret:
                enabled: true
      targetNamespace: cert-manager
      createNamespace: true 
    ```

Like the ingress controller, the `Release` resource is deployed in the system namespace `kubocd`, while the deployment 
itself takes place in the `cert-manager` namespace, which is created automatically.

Apply the `Release` manifest:

``` { .bash .copy }
kubectl apply -f releases/cert-manager.yaml
```

We can check the `Release` state:

``` { .bash .copy }
kubectl -n kubocd get releases cert-manager
```

``` { .bash}
NAME           REPOSITORY                              TAG          CONTEXTS           STATUS   READY   WAIT   PRT   AGE    DESCRIPTION
cert-manager   quay.io/kubodoc/packages/cert-manager   1.17.1-p01   contexts:cluster   READY    3/3            X     3m5s   The certificate manager
```

> It may take a couple of minutes to reach the `READY 3/3` state.

We can also check cert-manager and trust-manager pods are up and running:


``` { .bash .copy }
kubectl -n cert-manager get pods
```

``` { .bash  }
NAME                                           READY   STATUS    RESTARTS   AGE
cert-manager-main-6c8f6bf7bf-zwftl             1/1     Running   0          12m
cert-manager-main-cainjector-ff8b5d695-hdv7z   1/1     Running   0          12m
cert-manager-main-webhook-7d776cf7b8-6b5gh     1/1     Running   0          12m
trust-manager-54d8c969b5-xjjbz                 1/1     Running   0          10m
```

As mentioned earlier, the role of trust-manager is to distribute, across the various namespaces of the cluster, the 
certificates needed to validate the CAs used by cert-manager.

This certificate can be provided either as a Secret and/or a ConfigMap. In our example, both forms are enabled 
(`.spec.parameters.trust.bundle.target.configMap.enabled: true` and `.spec.parameters.trust.bundle.target.secret.enabled: true` in the Release object).

The name of this object is a parameter, with its default value defined in the `Package` parameter schema as `certs-bundle`:
(`schema.parameters.properties.trust.properties.bundle.properties.name.default: certs-bundle`).

> Using both a Secret and a ConfigMap allows compatibility with different workloads.
> The Secret provides better security for sensitive data, while the ConfigMap can be used by applications that don't require strict secret handling.

So, let's display this:

``` { .bash .copy }
kubectl get --all-namespaces secrets | grep certs-bundle
```

``` { .bash  }
cert-manager         certs-bundle                                 Opaque               3      11m
contexts             certs-bundle                                 Opaque               3      11m
default              certs-bundle                                 Opaque               3      11m
flux-system          certs-bundle                                 Opaque               3      11m
ingress-nginx        certs-bundle                                 Opaque               3      11m
kube-node-lease      certs-bundle                                 Opaque               3      11m
kube-public          certs-bundle                                 Opaque               3      11m
kube-system          certs-bundle                                 Opaque               3      11m
kubocd               certs-bundle                                 Opaque               3      11m
local-path-storage   certs-bundle                                 Opaque               3      11m
project01            certs-bundle                                 Opaque               3      11m
project02            certs-bundle                                 Opaque               3      11m
project03            certs-bundle                                 Opaque               3      11m
```

``` { .bash .copy }
kubectl get --all-namespaces configMap | grep certs-bundle
```

``` { .bash  }
cert-manager         certs-bundle                                           3      62m
contexts             certs-bundle                                           3      62m
default              certs-bundle                                           3      62m
flux-system          certs-bundle                                           3      62m
ingress-nginx        certs-bundle                                           3      62m
kube-node-lease      certs-bundle                                           3      62m
kube-public          certs-bundle                                           3      62m
kube-system          certs-bundle                                           3      62m
kubocd               certs-bundle                                           3      62m
local-path-storage   certs-bundle                                           3      62m
project01            certs-bundle                                           3      62m
project02            certs-bundle                                           3      62m
project03            certs-bundle                                           3      62m
```

Of course, the `trust-manager` controller watches new namespace creation to populate them with these resources.


## Secure podinfo

Now it’s time to use this deployment to secure our `podinfo` application.

The following is an improved version of the package, which optionally adds TLS encryption to the ingress controller:

???+ abstract "podinfo-p04.yaml"

    ``` { .yaml .copy }
    apiVersion: v1alpha1
    type: Package
    name: podinfo
    tag: 6.7.1-p04
    schema:
      parameters:
        properties:
          host: { type: string, required: true }
          tls: { type: boolean, default: false }
      context:
        properties:
          ingress:
            required: true
            properties:
              className: { type: string, required: true }
              domain: { type: string, required: true }
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
            className: {{ .Context.ingress.className  }}
            {{- if .Parameters.tls }}
            annotations:
              cert-manager.io/cluster-issuer: {{ required ".Context.certificateIssuer.public must be defined if tls: true" .Context.certificateIssuer.public }}
            {{- end }}
            hosts:
              - host: {{ .Parameters.host }}.{{ .Context.ingress.domain }}
                paths:
                  - path: /
                    pathType: ImplementationSpecific
            {{- if .Parameters.tls }}
            tls:
              - secretName: {{ .Release.metadata.name }}-tls
                hosts:
                  - {{  .Parameters.host }}.{{ .Context.ingress.domain }}
            {{- end }}
    ```

As usual, this need to be packaged:

``` { .bash .copy }
kubocd pack packages/podinfo-p04.yaml
```

``` { .bash  }
====================================== Packaging package 'packages/podinfo-p04.yaml'
--- Handling module 'main':
    Fetching chart podinfo:6.7.1...
    Chart: podinfo:6.7.1
--- Packaging
    Generating index file
    Wrap all in assembly.tgz
--- push OCI image: quay.io/kubodoc/packages/podinfo:6.7.1-p04
    Successfully pushed
```

Here is the new version of the related `Release` resource:

???+ abstract "podinfo4-tls.yaml"

    ``` { .yaml .copy }
    ---
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Release
    metadata:
      name: podinfo4
      namespace: default
    spec:
      description: A secured release of podinfo
      package:
        repository: quay.io/kubodoc/packages/podinfo
        tag: 6.7.1-p04
        interval: 30m
      parameters:
        tls: true
        host: podinfo4
    ```

Deploy it:

``` { .bash .copy }
kubectl apply -f releases/podinfo4-tls.yaml
```

And configure the DNS entry (Here if you use `/etc/hosts`)

``` { .bash .copy }
127.0.0.1 localhost host.docker.internal podinfo1.ingress.kubodoc.local podinfo2.ingress.kubodoc.local podinfo4.ingress.kubodoc.local
```

You should now be able to access the secures 'podinfo` web server:

👉 [http://podinfo4.ingress.kubodoc.local](http://podinfo4.ingress.kubodoc.local){:target="_blank"}.

