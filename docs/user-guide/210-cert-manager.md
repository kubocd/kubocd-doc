# Cert-Manager Setup

!!! warning
    **Existing Clusters:** If cert-manager is already installed, **do not install another one**. However, read this section to understand the concepts.

If you are using the local `kind` cluster, continue with this installation.

## Components

We will deploy:

- **cert-manager**: Automates issuance and renewal of TLS certificates.
- **trust-manager**: Distributes trusted CA certificates across the cluster.

After deployment, we will configure:

1. **Self-Signed CA**: A private Certificate Authority within the cluster.
2. **Trust Bundle**: A collection of trusted CAs distributed by trust-manager to ensure services trust the self-signed CA.

## The KuboCD Package

We group all components into a single package with three modules:

- **`noname`**: Deploys `cert-manager`. (Using `noname` prevents suffixing resource names).
- **`trust`**: Deploys `trust-manager`.
- **`issuer`**: An ad hoc Helm chart that configures the self-signed CA and Trust Bundle.

???+ abstract "cert-manager-p01.yaml"

    ``` { .yaml .copy }
    apiVersion: v1alpha1
    name: cert-manager
    tag: 1.17.1-p01
    protected: true
    schema:
      parameters:
        properties:
          trust:
            properties:
              enabled: { type: boolean, default: false }
              bundle:
                properties:
                  name: { type: string, default: certs-bundle }
                  target:
                    properties:
                      configMap:
                        properties:
                          enabled: { type: boolean, default: false }
                          key: { type: string, default: "root-certs.pem" }
                      secret:
                        properties:
                          enabled: { type: boolean, default: false }
                          key: { type: string, default: "ca.crt" }
          issuers:
            properties:
              enabled: { type: boolean, default: true }
              caClusterIssuers:
                items:
                  properties:
                    name: { type: string, required: true }
                    ca_crt: { type: string, required: true }
                    ca_key: { type: string, required: true }
              selfSignedClusterIssuers:
                items:
                  type: object
                  properties:
                    name: { type: string, required: true}
    modules:
      - name: noname
        source:
          helmRepository:
            url: https://charts.jetstack.io
            chart: cert-manager
            version: v1.17.1
        values: |
          installCRDs: true
          enableCertificateOwnerRef: true
      - name: trust
        enabled: "{{ .Parameters.trust.enabled }}"
        source:
          helmRepository:
            url: https://charts.jetstack.io
            chart: trust-manager
            version: v0.16.0
        values: |
          app:
            trust:
              namespace: {{ .Release.spec.targetNamespace }}
          secretTargets:
            enabled: {{ and .Parameters.trust.enabled .Parameters.trust.bundle.target.secret.enabled }}
            authorizedSecrets:
              - {{ .Parameters.trust.bundle.name }}
        dependsOn:
          - noname
      - name: issuers
        enabled: "{{ .Parameters.issuers.enabled }}"
        source:
          local:
            path: ../charts/cert-issuers/0.1.0
        values: |
          {{ toYaml .Parameters.issuers }}
          bundle:
            name: {{ .Parameters.trust.bundle.name }}
            target:
              configMap:
                enabled: {{ .Parameters.trust.bundle.target.configMap.enabled }}
                key: {{ .Parameters.trust.bundle.target.configMap.key }}
              secret:
                enabled: {{ .Parameters.trust.bundle.target.secret.enabled }}
                key: {{ .Parameters.trust.bundle.target.secret.key }}
        dependsOn: |
          - noname
          {{ if and .Parameters.trust.enabled }}
          - trust
          {{ end }}
    roles:
      - certManager
    ```

!!! Note "Scope"
    Detailed configuration of cert-manager itself is outside the scope of this manual. Refer to the official documentation.

### Local Helm Chart Source

This example introduces a **local source** for Helm charts. The `cert-issuers` chart is stored alongside the manifest and referenced via `modules[3].source.local.path`. During packaging, this local chart is embedded into the OCI image.

## Deployment

Build the package:

``` { .bash .copy } 
kubocd pack packages/cert-manager-p01.yaml
```

Create the Release:

???+ abstract "cert-manager.yaml"

    ``` { .yaml .copy } 
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

The `Release` resides in the `kubocd` namespace, but it installs the application into `cert-manager` (via `targetNamespace`).

Apply the manifest:

``` { .bash .copy }
kubectl apply -f releases/cert-manager.yaml
```

Check the status (wait for `READY 3/3`):

``` { .bash .copy }
kubectl -n kubocd get releases cert-manager
```

Verify pods:

``` { .bash .copy }
kubectl -n cert-manager get pods
```

The `trust-manager` controller distributes the trust bundle (as both Secret and ConfigMap) to all namespaces. Verify distribution using:

``` { .bash .copy }
kubectl get --all-namespaces secrets | grep certs-bundle
```

---

## Secure Podinfo

Now we use this infrastructure to secure the `podinfo` application.

Update the package to support TLS:

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
      - name: noname
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

Pack the update:

``` { .bash .copy }
kubocd pack packages/podinfo-p04.yaml
```

Deploy the secured release:

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
      parameters:
        tls: true
        host: podinfo4
    ```

``` { .bash .copy }
kubectl apply -f releases/podinfo4-tls.yaml
```

Verify the ingress now serves on port 443:

``` { .bash .copy }
kubectl get ingresses podinfo4
```

Access the secured URL:
👉 [https://podinfo4.ingress.kubodoc.local](https://podinfo4.ingress.kubodoc.local){:target="_blank"}

> Note: You will encounter browser security warnings because we are using a self-signed certificate.
