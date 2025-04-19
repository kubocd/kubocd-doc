# Alternate KuboCD Schema Format

In our first example, we defined `schema.parameters` using a standard OpenAPI/JSON schema. While fully supported, it can be quite verbose.

KuboCD also supports a more concise schema syntax, specifically designed for this use case.

Here’s an identical version of the `podinfo` package using the simplified KuboCD schema format:

???+ abstract "podinfo-p02.yaml"
    ```yaml
    apiVersion: v1alpha1
    type: Package
    name: podinfo
    tag: 6.7.1-p01
    schema:
      parameters:
        properties:
          fqdn: { type: string, required: true }
          ingressClassName: { type: string, default: "nginx" }
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
            className: {{ .Parameters.ingressClassName }}
            hosts:
              - host: {{ .Parameters.fqdn }}
                paths:
                  - path: /
                    pathType: ImplementationSpecific
    ```

As you can see, the format is significantly more compact.

!!! info
    This schema format is **not** standard OpenAPI, but KuboCD internally converts it into a compliant schema.  
    It is a shorthand designed for convenience.

The presence or absence of the `$schema:` key is what KuboCD uses to distinguish between standard and KuboCD schema formats.

This simplified schema is explained further in the [Schema Reference](schemas.md).
