
# Advanced features: Redis sample


We will use another example to demonstrate some of KuboCD’s more advanced features.


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

We will now explore the new features used in this package.

---

## Integrate Multiple Modules

This package integrates two modules: the Redis server itself, and [Commander](https://github.com/joeferner/redis-commander), a Redis UI front end.

There is a single data model used by both modules, which allows variables to be shared easily.

Consequently, the `schema.parameters` and `schema.context` are also global.

Regarding the related Flux objects, there will be one `OCIRepository`, one `HelmRepository`, and two `HelmReleases`.

Note also the first, main module is named `noname`, to have a clean name of all related resources.

---

## Configure Different Source Types

Redis publishes its Helm chart as an OCI image. Therefore, in the source section of the first module, the chart is 
referenced in this form.

Redis Commander, on the other hand, provides its chart only through its GitHub repository. Thus, it is referenced
in the source section of the second module accordingly.

---

## Control Module Deployment

The `ui` module includes a `disabled` attribute, which can be templated. This allows the deployment of this module to be 
conditionally triggered based on a choice made during deployment, using a parameter passed to the `Release` resource.

---

## Manage Dependencies

Usually, in the Kubernetes ecosystem, it is possible to ignore dependencies between applications, as they are typically 
well-designed and operate with retry logic until they eventually reach a stable state.

However, this approach has its limitations. To address them, KuboCD introduces a structured dependency system between 
deployments.

KuboCD manages dependencies:

- Between modules within the same package
- Between different packages

These two types of dependencies rely on different mechanisms.

### Module Dependencies

From the sample above:

```
apiVersion: v1alpha1
name: redis
......
modules:
  - name: noname
    ........
  - name: ui
    ..........
    dependsOn:
      - noname
```

The `dependsOn` attribute lists other module names that the current module depends on. These modules must belong to the same `Package`

In this example, it ensures that the deployment of the `ui` module will wait until the deployment of the `noname` module (redis itself) is complete.

> This mechanism results in adding the `dependsOn` attribute to the Flux `HelmRelease` resource corresponding to the `ui` module.

### Release Dependencies and Roles

For cross-Release dependencies, KuboCD introduces an abstraction: the concept of roles.

A `Release` can fulfill one (or more) roles, and similarly, it can depend on one (or more) roles.

This abstraction offers much greater flexibility. For example, an application that exposes a web service outside the 
cluster depends on the presence of an ingress controller. It would therefore depend on a role named `ingress`, 
regardless of whether the controller is implemented by NGINX, Traefik, Kong, or another solution.

Roles can be defined at the `Package` level, as in this example, or directly at the `Release` level, using the `spec.roles` attribute.

Similarly, dependencies can be defined at either the `Package` level or the `Release` level, using the `spec.dependencies` attribute.

> Note:
    During rendering, both roles and dependencies defined at the Package and Release levels are concatenated.
    This ensures that the final dependency graph combines all relevant definitions from both scopes.

### Cluster roles

The deployment of a package is conditioned on the availability of the roles it depends on, meaning that at least one 
application fulfilling each required role must be successfully deployed.

In some cases, a role may already be fulfilled by an application that was not deployed by KuboCD.
For example, if you're using a cluster that already includes an Ingress controller and a Load Balancer, you need 
to inform KuboCD that these roles are already satisfied externally.

Such roles are called `ClusterRoles`, and their list is defined in the global [KuboCD configuration resource: `Config`](./170-context-and-config.md).

---

## Deployment

### Packaging

This new application must be packaged:


``` { .bash .copy }
kubocd pack packages/redis-p01.yaml 
```

```
====================================== Packaging package 'packages/redis-p01.yaml'
--- Handling module 'noname':
    Pulling image 'registry-1.docker.io/bitnamicharts/redis:20.6.1'
    Chart: redis:20.6.1
--- Handling module 'ui':
    Cloning git repository 'https://github.com/joeferner/redis-commander.git'
    Chart: redis-commander:0.6.0
--- Packaging
    Generating index file
    Wrap all in assembly.tgz
--- push OCI image: quay.io/kubodoc/packages/redis:20.6.1-p01
    Successfully pushed
```

### Full deployment

A first deployment, including the front end:

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
        interval: 30m
      parameters:
        ui:
          host: redis1
    ```


``` { .bash .copy }
kubectl apply -f releases/redis1-basic.yaml 
```

You can now check the status of the deployment.
Note that this may take some time, as the Redis deployment can be quite slow.

``` { .bash .copy }
kubectl get release redis1
```

``` { .bash }
NAME     REPOSITORY                       TAG          CONTEXTS           STATUS   READY   WAIT   PRT   AGE     DESCRIPTION
redis1   quay.io/kubodoc/packages/redis   20.6.1-p01   contexts:cluster   READY    2/2            -     3m19s   Redis and a front UI
```

Notice the 2/2 under the READY column, which corresponds to the two modules.

You can also inspect the status of the corresponding Flux `HelmRelease` resources:

``` { .bash .copy }
kubectl get helmReleases 
```

``` { .bash }
NAME            AGE     READY   STATUS
........
redis1-redis    2m31s   True    Helm install succeeded for release default/redis1-redis.v1 with chart redis@20.6.1
redis1-ui       2m31s   True    Helm install succeeded for release default/redis1-ui.v1 with chart redis-commander@0.6.0
.......
```

And related `pods`:


``` { .bash .copy }
kubectl get pods
```

``` { .bash }
NAME                                                     READY   STATUS    RESTARTS   AGE
.......
redis1-main-master-0                                     1/1     Running   0          12m
redis1-main-replicas-0                                   1/1     Running   0          12m
redis1-ui-855cb8d656-rlqqh                               1/1     Running   0          11m
.......
```

And related ingress resource:

``` { .bash .copy }
kubectl get ingresses redis1-ui
```

``` { .bash }
NAME        CLASS   HOSTS                          ADDRESS      PORTS   AGE
redis1-ui   nginx   redis1.ingress.kubodoc.local   10.96.59.9   80      3m40s
```

### Redis only deployment

In this other deployment, only the Redis server is deployed.

> To satisfy the `schema.Parameters`, a dummy value must be provided for `ui.host`.

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
        interval: 30m
      parameters:
        ui:
          enabled: false
          host: dummy
    ```

Then:

``` { .bash .copy }
kubectl apply -f releases/redis2-basic.yaml 
```


``` { .bash .copy }
kubectl get helmReleases 
```

``` { .bash }
.......
redis1-redis    10m     True    Helm install succeeded for release default/redis1-redis.v1 with chart redis@20.6.1
redis1-ui       10m     True    Helm install succeeded for release default/redis1-ui.v1 with chart redis-commander@0.6.0
redis2-redis    2m45s   True    Helm install succeeded for release default/redis2-redis.v1 with chart redis@20.6.1
.......
```

``` { .bash .copy }
kubectl get pods
```

``` { .bash }
NAME                                                     READY   STATUS    RESTARTS   AGE
.......
redis1-main-master-0                                     1/1     Running   0          12m
redis1-main-replicas-0                                   1/1     Running   0          12m
redis1-ui-855cb8d656-rlqqh                               1/1     Running   0          11m
redis2-main-master-0                                     1/1     Running   0          4m51s
redis2-main-replicas-0                                   1/1     Running   0          4m51s
.......
```

!!! tip
    It is possible to define a more precise schema, without the `host: dummy` constraint, by using the normalized syntax.
    You can find an implementation [here](https://github.com/kubocd/kubocd-doc/blob/main/samples/packages/redis-p09.yaml).

