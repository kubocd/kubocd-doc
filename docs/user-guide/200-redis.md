
# Redis: a realistic example.


We will use this sample to demonstrate some more advanced feature of KuboCD.


```
kubocd pack redis-p01.yaml 

====================================== Packaging package 'redis-p01.yaml'
--- Handling module 'redis':
    Pulling image 'registry-1.docker.io/bitnamicharts/redis:20.6.1'
    Chart: redis:20.6.1
--- Handling module 'commander':
    Cloning git repository 'https://github.com/joeferner/redis-commander.git'
    Chart: redis-commander:0.6.0
--- Packaging
    Generating index file
    Wrap all in assembly.tgz
--- push OCI image: quay.io/kubodoc/packages/redis:20.6.1-p01
    Successfully pushed
```

