
# Advanced Features: Redis Sample

This chapter demonstrates advanced KuboCD features using a Redis deployment example.

???+ abstract "redis-p01.yaml"

    ``` { .yaml .copy }
    apiVersion: v1alpha1
    name: redis
    tag: 20.6.1-p01
    description: Redis and a front UI
    schema:
      parameters:
        properties:
          redis:
            properties:
              password: { type: string, default: redis123 }
              replicaCount: { type: integer, default: 1, description: "The number of replicas"}
          ui:
            required: true
            properties:
              enabled: { type: boolean, default: true }
              host: { type: string, required: true }
      context:
        properties:
          ingress:
            required: true
            properties:
              className: { type: string, default: "nginx"}
              domain: { type: string, required: true }
    modules:
      - name: noname
        source:
          oci:
            repository: registry-1.docker.io/bitnamicharts/redis
            tag: 20.6.1
        values: |
          image:
            tag: latest
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
            path: ./k8s/helm-chart/redis-commander
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
          - noname
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

---

## Multiple Modules

This package integrates two modules:
1. **Redis Server**: The core database.
2. **Redis Commander**: A web-based management UI.

Both modules share a single data model, allowing easy variable sharing. Consequently, `schema.parameters` and `schema.context` are global to the package.

When deployed, Flux creates one `OCIRepository`, one `HelmRepository`, and two `HelmReleases`.

Note that the main module is named `noname` to simplify resource naming.

---

## Mixed Source Types

- **Redis**: Published as an OCI image. The `source` section references the OCI registry.
    > Note: The Bitnami Redis Docker image version in the chart is outdated, so we override the tag to `latest`.
- **Redis Commander**: Only provides a chart via GitHub. The `source` section references the Git repository.

---

## Conditional Deployment

The `ui` module includes an `enabled` attribute. This allows its deployment to be toggleable via a `Release` parameter (`.Parameters.ui.enabled`).

---

## Dependency Management

While Kubernetes applications often rely on retry loops to handle dependencies, KuboCD offers a structured dependency system.

### Module Dependencies

Dependencies between modules within the same package:

```yaml
modules:
  - name: noname
    ...
  - name: ui
    ...
    dependsOn:
      - noname
```

The `ui` module waits for the `noname` module (Redis) to become ready. This translates to a `dependsOn` rule in the underlying Flux `HelmRelease`.

### Release Dependencies and Roles

For dependencies between different Releases, KuboCD uses **Roles**.

- A `Release` can **fulfill** one or more roles.
- A `Release` can **depend on** one or more roles.

This abstraction provides flexibility. For example, an application needing an ingress controller depends on the `ingress` role, regardless of the implementation (NGINX, Traefik, etc.).

Roles and dependencies can be defined at:
- **Package Level**: Static definitions (like in the example above).
- **Release Level**: Dynamic definitions via `spec.roles` and `spec.dependencies`.

During rendering, definitions from both levels are merged.

### Cluster Roles

If a role is fulfilled by an external system (e.g., a pre-installed Ingress Controller on a cloud provider), it is defined as a **ClusterRole**.

ClusterRoles are configured in the global [`Config` resource](./170-context-and-config.md) to inform KuboCD that a dependency is externally satisfied.

---

## Deployment

### Packaging

Build the package:

``` { .bash .copy }
kubocd pack packages/redis-p01.yaml 
```

### Full Deployment

Deploy both Redis and the UI:

???+ abstract "redis1-basic.yaml"

    ``` { .yaml .copy }
    ---
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Release
    metadata:
      name: redis1
      namespace: default
    spec:
      package:
        repository: quay.io/kubodoc/packages/redis
        tag: 20.6.1-p01
      parameters:
        ui:
          host: redis1
    ```

``` { .bash .copy }
kubectl apply -f releases/redis1-basic.yaml 
```

Check status (Redis deployment may take time):

``` { .bash .copy }
kubectl get release redis1
```

``` { .bash }
NAME     REPOSITORY                       TAG          CONTEXTS           STATUS   READY   WAIT   PRT   AGE     DESCRIPTION
redis1   quay.io/kubodoc/packages/redis   20.6.1-p01   contexts:cluster   READY    2/2            -     3m19s   Redis and a front UI
```

Note `READY 2/2`, indicating two active modules.

Access the UI at:
[http://redis1.ingress.kubodoc.local](http://redis1.ingress.kubodoc.local) (Requires DNS/hosts configuration).

### Redis-Only Deployment

Deploy only the Redis server:

???+ abstract "redis2-basic.yaml"

    ``` { .yaml .copy }
    ---
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Release
    metadata:
      name: redis2
      namespace: default
    spec:
      package:
        repository: quay.io/kubodoc/packages/redis
        tag: 20.6.1-p01
      parameters:
        ui:
          enabled: false
          host: dummy
    ```

> Note: `host: dummy` is required to satisfy the schema validation, even though the UI is disabled.

``` { .bash .copy }
kubectl apply -f releases/redis2-basic.yaml 
```

Verify that only the Redis pod is created for `redis2`.

!!! tip
    To avoid the `host: dummy` workaround, you can refine the schema using the normalized syntax to make parameters conditional. [See sample implementation](https://github.com/kubocd/kubocd-doc/blob/main/samples/packages/redis-p09.yaml){:target="_blank"}.
