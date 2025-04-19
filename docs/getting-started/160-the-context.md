# The Context

L'un des aspect clé de KuboCD est sa capacité de généréer les fichiers de values de déployment Helm à partir de
quelques parametres d'entrée de haut niveau, ceci grance à un méchanisme de templating.

Ce mécanisme croise un template et un modèle de données. Ce modèle de données comprend les éléments racine suivants:

- `.Parameters`: Les données fournies en paramètre dans la custom resource `Release`.
- `.Release`: L'objet release lui même.
- `.Context`: Le context du déployment

Le context est un objet YAML dont la structure est libre et qui à pour vocation de contenir des informations transverses à l'ensemble des déploiements:

Par example, notre Package `podinfo` à un paramètre `ingressClassName`, avec une valeur par défaut ('nginx'). 
Si un cluster intègre un autre type d'ingress-controller, il faudrait alors le redéfinir pour l'ensemble des applications utilisant un ingress.

Ce type d'information a donc pour vocation d'ètre défni dans un Context global au cluster.

De même, toutes les URL ingress des différentes application partage une racine commune. Il est pertinant de mutualiser cette racine.

Voici un premier example d'implémenation de cette logique.

## Context creation

Un `context` est une resource KuboCD:

???+ abstract "cluster.yaml"

    ```
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Context
    metadata:
      namespace: contexts
      name: cluster
    spec:
      description: Context global for kubodoc cluster
      protected: true
      context:
        ingress:
          className: nginx
          domain: ingress.kubodoc.local
        storageClass: 
          data: standard
          workspace: standard
    ```

Les principaux attributs en sont:

- `description`: Une courte description
- `protected`: Interdit la suppretion de cet object. Necéssite le déploiment de la fonctionnalité `webhook' de KuboCD.
- `context`: Une aborescence de valeurs qui sera injecté dans le modele de données fournis au moteur de template 
  générant les paramètres de déploiement. Cette partie
    - Doit etre du YAML valide
    - La structure est libre. Néanmoins elle doit correspondre à ce que les template des packages KuboCD s'attendent à y trouver.

Dans cet exemple, on vat trouver:

- `ingress.className`: Le type de controller ingress
- `ingress.domain`: Le suffix permettant la construction des URL ingress
- `storageClass`: Le storage class utilisé.

Le ou les contexts seront placés dans un namespace dédié. 


```
kubectl create ns contexts

namespace/contexts created
```

```
kubectl create -f cluster.yaml 

context.kubocd.kubotal.io/cluster created


```

```


kubocd pack podinfo-p02.yaml 

====================================== Packaging package 'podinfo-p02.yaml'
--- Handling module 'main':
    Fetching chart podinfo:6.7.1...
    Chart: podinfo:6.7.1
--- Packaging
    Generating index file
    Wrap all in assembly.tgz
--- push OCI image: quay.io/kubodoc/packages/podinfo:6.7.1-p02
    Successfully pushed



kubectl create -f podinfo2.yaml 

release.kubocd.kubotal.io/podinfo2 created

```



Un changement dans un context provoque le refraichissement de toutes les Release le référencant.

Reférencer un context inexistante génère une erreur, sauf pour le namespace context