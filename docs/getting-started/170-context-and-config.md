# Context and configuration

## Default context

Si l'on suit la logique du chapitre précédent, il devient évident que le context global du cluster devra être inclue dans toutes les `Release`. 
A partir de là, la possibilité de définit un context par défaut devient évidente.

A la différence de la plupart des applications Kubernetes qui sauvegarde leur configuration dans une `ConfigMap`, KuboCD utilise une Custom Resource dédié à cet usage:

```
kubectl -n kubocd get Config.kubocd.kubotal.io
```

Une seule instance de cette resource existe (Elle a été crée lors du déploiement par la chart Helm)

```
NAME     AGE
conf01   5d21h
```

Examinons son contenu:

```
kubectl -n kubocd get Config.kubocd.kubotal.io conf01 -o yaml
```

```
apiVersion: kubocd.kubotal.io/v1alpha1
kind: Config
metadata:
  annotations:
    meta.helm.sh/release-name: kubocd-ctrl
    meta.helm.sh/release-namespace: kubocd
  creationTimestamp: "2025-04-16T12:29:27Z"
  generation: 1
  labels:
    app.kubernetes.io/managed-by: Helm
  name: conf01
  namespace: kubocd
  resourceVersion: "5191"
  uid: 84ff9a8a-83ee-48e7-984d-c95a6a665d5b
spec:
  clusterRoles: []
  defaultContexts: []
  imageRedirects: []
  packageRedirects: []
```

A ce stade, cette configuration est vide

!!! notes
    Utiliser une resource Kubernetes pour la configuration présente les avantages suivants:

      - Les erreurs de structure entrainent un rejet immédiat, sans impacter l'état du service actif.
      - Une ressource peut etre 'watchée'. KuboCD prend donc en compte immédiatement les modifications de configuration.

Le manifest suivant est crée et appliqué:

???+ abstract "conf01-b.yaml"

    ``` { .yaml .copy }
    ---
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Config
    metadata:
      name: conf01
      namespace: kubocd
    spec:
      defaultContexts:
        - namespace: contexts
          name: cluster
    ```

Il défini une liste de contexts par défaut, qui comprend comme unique élémént le context du cluster crée au chapitre précédent.

``` { .bash .coopy }
kubectl apply -f conf01-b.yaml 
```

> Ne pas s'inquiéter d'un éventuel warning'

!!! note 
    Pour être pris en compte par KuboCD, l'objet `Config` doit ce trouver dans le namespace du controller KuboCD (Soit `kubocd`). Par contre, le nom est indifferent. 


Nous pouvons maintenant créer une nouvelle `Release` de notre application `podinfo`, avec la version utilisant le context, mais sans préciser ce dernier:

???+ abstract "podinfo3-ctx-def.yaml"

    ``` { .yaml .copy }
    ---
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Release
    metadata:
      name: podinfo3
      namespace: default
    spec:
      description: A first sample release of podinfo
      package:
        repository: quay.io/kubodoc/packages/podinfo
        tag: 6.7.1-p02
        interval: 30m
      parameters:
        host: podinfo3
    ```

``` { .bash .copy }
kubectl apply -f podinfo3-ctx-def.yaml 
```

On peut vérifier que la `Release` prend bien en compte le context:

``` { .bash .copy }
kubectl get release podinfo3
```

``` { .bash }
NAME       REPOSITORY                         TAG         CONTEXTS           STATUS   READY   WAIT   PRT   AGE   DESCRIPTION
podinfo3   quay.io/kubodoc/packages/podinfo   6.7.1-p02   contexts:cluster   READY    1/1            -     31m   A first sample release of podinfo
```

Ce qui se traduit par un `domain` correct sur l'ingress:

``` { .bash .copy }
kubectl get ingress podinfo3-main
```

``` { .bash }
NAME            CLASS   HOSTS                            ADDRESS        PORTS   AGE
podinfo3-main   nginx   podinfo3.ingress.kubodoc.local   10.96.218.98   80      84s
```

---

## Namespaced default context

Kubocd fournis aussi la possibilité de définir un context par defaut pour chaque namespace:

Modifions à nouveau notre ressource de configuration:

???+ abstract "conf01-c.yaml"

    ``` { .yaml .copy }
    ---
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Config
    metadata:
      name: conf01
      namespace: kubocd
    spec:
      defaultContexts:
        - namespace: contexts
          name: cluster
      defaultNamespaceContext: project
    ```


``` { .bash .copy }
kubectl apply -f conf01-c.yaml 
```

Grace à cette configuration, toutes les `Release` vont rechercher un context nommé `project` dans le namespace ou elles 
sont déployées. Et le prendre en compte si il existe.

Un nouveau namespace `project03` est crée:

``` { .bash .copy }
kubectl create ns project03
```

Et un context spécifique y est crée, mais sous le nom générique `project`:

???+ abstract "project03.yaml"

    ```
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Context
    metadata:
      name: project
    spec:
      description: Context For projet 3
      context:
        project:
          id: p03
        ingress:
          domain: prj03.ingress.kubodoc.local
    ```


``` { .bash .copy }
kubectl -n project03 apply -f project03.yaml 
```

Nous pouvons maintenant créer une nouvelle `Release`, sans précision de context utilisé:

???+ abstract "podinfo-prj03.yaml"

    ```
    ---
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Release
    metadata:
      name: podinfo
    spec:
      description: A release of podinfo on project03
      package:
        repository: quay.io/kubodoc/packages/podinfo
        tag: 6.7.1-p02
        interval: 30m
      parameters:
        host: podinfo
      debug:
        dumpContext: true
        dumpParameters: true
    ```


``` { .bash .copy }
kubectl -n project03 apply -f podinfo-prj03.yaml
```

Et on peut valider que les deux contexts par défaut sont bien pris en compte.

``` { .bash .copy }
kubectl -n project03 get release podinfo
```

``` { .bash }
NAME      REPOSITORY                         TAG         CONTEXTS                             STATUS   READY   WAIT   PRT   AGE   DESCRIPTION
podinfo   quay.io/kubodoc/packages/podinfo   6.7.1-p02   contexts:cluster,project03:project   READY    1/1            -     10m   A release of podinfo on project03
```

On vérifie que la valeur de l'ingress est bien correcte:


``` { .bash .copy }
kubectl -n project03 get ingress
```

``` { .bash }
NAME           CLASS   HOSTS                                 ADDRESS        PORTS   AGE
podinfo-main   nginx   podinfo.prj03.ingress.kubodoc.local   10.96.218.98   80      11m
```

---

## Context ordering

Comme il été évoqué au chapitre précédent, l'ordre sans lequel sont aggrègés les différents context pour une application 
peut avoir sont importance.

Cet ordre est le suivant

- Les contexts par défault dans l'ordre de la liste présent dans la configuration
- Le context par défault lié au namespace, si il est présent
- Les contexts définis dans la `Release`, dans l'ordre de la liste.

Un context inexistant génère une erreur, sauf pour le context par défaut du namespace.

Si vous avez effectué l'ensemble des opérations décrites dans cette documentation, vous devriez arriver à un résultat tel que celui-ci:

``` { .bash .copy }
kubectl get --all-namespaces releases
```

``` { .bash  }
NAMESPACE   NAME            REPOSITORY                               TAG          CONTEXTS                                                STATUS   READY   WAIT   PRT   AGE     DESCRIPTION
default     podinfo1        quay.io/kubodoc/packages/podinfo         6.7.1-p01    contexts:cluster                                        READY    1/1            -     3d20h   A first sample release of podinfo
default     podinfo2        quay.io/kubodoc/packages/podinfo         6.7.1-p02    contexts:cluster,contexts:cluster                       READY    1/1            -     25h     A first sample release of podinfo
default     podinfo3        quay.io/kubodoc/packages/podinfo         6.7.1-p02    contexts:cluster                                        READY    1/1            -     27m     A first sample release of podinfo
kubocd      ingress-nginx   quay.io/kubodoc/packages/ingress-nginx   4.12.1-p01   contexts:cluster                                        READY    1/1            -     3d23h   The Ingress controller
project01   podinfo         quay.io/kubodoc/packages/podinfo         6.7.1-p03    contexts:cluster,contexts:cluster,project01:project01   READY    1/1            -     24h     A release of podinfo on project01
project02   podinfo         quay.io/kubodoc/packages/podinfo         6.7.1-p02    contexts:cluster,contexts:cluster,project02:project02   READY    1/1            -     24h     A release of podinfo on project02
project03   podinfo         quay.io/kubodoc/packages/podinfo         6.7.1-p02    contexts:cluster,project03:project                      READY    1/1            -     5m55s   A release of podinfo on project03
```

On trouve des cas ou des contexts sont défini deux fois pour une `Release`. Car c'est le context par défaut, mais il a 
été aussi défini explicitement dans la `Release'. Ceci ne prète pas à conséquences.

!!! tip
    Si pour une quelconque raison, vous ne souhaitez pas utiliser ce mécanisme de context par défault pour une 
    `Release` particulière, il existe un flag `skipDefaultContext: true` dans les attributs possibles de la `Release`. 

## KuboCD Helm chart

Il est possible d'intégrer ces valeurs de configuration à l'installation de KuboCD, en les fournissant comme `values` 
lors du déploiement de celle-ci.

Par example, en créant le fichier suivant:

???+ abstract "values1-ctrl.yaml"

    ```
    config:
      defaultContexts:
        - name: cluster
          namespace: contexts
      defaultNamespaceContext: project
    extraNamespaces:
      - name: contexts
    contexts:
      - name: cluster
        namespace: contexts
        protected: true
        description: Context specific to the cluster 'kubodoc'
        context:
          ingress:
            className: nginx
            domain: ingress.kubodoc.local
          storageClass:
            data: standard
            workspace: standard
    ```

- La section `config` sera directement ajouté à la section `spec` de la resource de configuration.
- La liste `extraNamespaces` permet de fournir une liste de namespace qui seront crées par la Chart helm.
- La section `contexts` permet de définir une liste de contexts qui seront crées par la Chart Helm.  

On peut donc upgrader le déploiement de KuboCD:

``` { .bash .copy }
helm -n kubocd upgrade kubocd-ctrl oci://quay.io/kubocd/charts/kubocd-ctrl:v0.2.0 --values values1-ctrl.yaml
```
!!! warning

    Si vous avez effectué l'ensemble des opérations décrites dans ce chapitre, une erreur sera générée. En effet, Helm 
    refuse de gérer un object qui n'a pas été crée par lui. 

Dans ce cas, il faut donc supprimer le namespace `contexts` et les object associés:

``` { .bash .copy }
kubectl -n contexts delete context.kubocd.kubotal.io cluster
```

``` { .bash .copy }
kubectl delete ns contexts
```

et ensuite effectuer la commande `Helm upgrade` précédente.

Si vous examinez l'état des `Releases` pendant cette opération, vous les verrez passer en `ERROR` puis repasser en état 
`READY` lorsque le context sera recréé. Par contre, les applications elle même (Les pods et ingress `podinfo...`) 
ne seront pas affectées

