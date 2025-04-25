
# Redis: a realistic example.


We will use this sample to demonstrate some more advanced feature of KuboCD.


???+ abstract "redis-p01.yaml"

    ``` { .yaml .copy }

    ```


``` { .bash .copy }
kubocd pack packages/redis-p01.yaml 
```

```
====================================== Packaging package 'packages/redis-p01.yaml'
--- Handling module 'redis':
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

``` { .bash .copy }
kubectl get release redis1
```

``` { .bash }
NAME     REPOSITORY                       TAG          CONTEXTS           STATUS   READY   WAIT   PRT   AGE     DESCRIPTION
redis1   quay.io/kubodoc/packages/redis   20.6.1-p01   contexts:cluster   READY    2/2            -     3m19s   Redis and a front UI
```


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


``` { .bash .copy }
kubectl get ingresses redis1-ui
```

``` { .bash }
NAME        CLASS   HOSTS                          ADDRESS      PORTS   AGE
redis1-ui   nginx   redis1.ingress.kubodoc.local   10.96.59.9   80      3m40s
```

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
