# The Context

L'un des aspect clé de KuboCD est sa capacité de générer les fichiers de values de déployment Helm à partir de
quelques paramètres d'entrée de haut niveau, ceci grâce à un mechanism de templating.

Ce mécanisme croise un template et un modèle de données. 

Notre premier example utilise uniquement l'élément `.Parametres` de notre modèle de données:

???+ abstract "podinfo-p01.yaml"

    ```yaml
    apiVersion: v1alpha1
    type: Package
    name: podinfo
    .....
    modules:
      - name: main
        .....
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

 En fait, ce modèle de données comprend les éléments racine suivants:


- `.Parameters`: Les données fournies en paramètre dans la custom resource `Release`.
- `.Release`: L'objet release lui même.
- `.Context`: Le context du déployment

Le context est un objet YAML dont la structure est libre et qui à pour vocation de contenir des informations transverses à l'ensemble des déploiements:

Par example, notre Package `podinfo` à un paramètre `ingressClassName`, avec une valeur par défaut ('nginx'). 
Si un cluster intègre un autre type d'ingress-controller, il faudrait alors le redéfinir pour l'ensemble des `Release` utilisant un ingress.

Ce type d'information a donc pour vocation d'ètre défni dans un Context global au cluster.

De même, toutes les URL ingress des différentes application partage une racine commune. Il est pertinant de mutualiser cette racine.

Voici un premier example implementation de cette logique.

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
- `protected`: Interdit la suppression de cet object. Nécessite le deployment de la fonctionnalité `webhook' de KuboCD.
- `context`: Une aborescence de valeurs qui sera injecté dans le model de données fournis au moteur de template 
  générant les paramètres de déploiement. Cette partie 
    - Doit etre du YAML valide
    - La structure est libre. Néanmoins elle doit correspondre à ce que les template des packages KuboCD s'attendent à y trouver.

Dans cet exemple, le context comprend les éléménts suivants:

- `ingress.className`: Le type de controller ingress
- `ingress.domain`: Le suffix permettant la construction des URL ingress
- `storageClass`: Deux types de `StorageClass` Kubernetes, pour deux profils d'applications, pour les application 'statefull'. 
  Dans le cas de notre cluster `kind`, il n'y a qu'un seul choix: `standard`

Le ou les contexts seront placés dans un namespace dédié. 

```
kubectl create ns contexts

namespace/contexts created
```

```
kubectl create -f cluster.yaml 

context.kubocd.kubotal.io/cluster created
```

!!! notes
    Le context étant un object partagé entre toutes les applications, sa structure doit être conçue avec soin et documentée.


## Package adaptation

Notre premier packaging de podinfo ne prenait pas en compte ce concept de context. En voici une version adaptée:

???+ abstract "podinfo-p02.yaml"

    ```
    apiVersion: v1alpha1
    type: Package
    name: podinfo
    tag: 6.7.1-p02
    schema:
      parameters:
        $schema: http://json-schema.org/schema#
        type: object
        additionalProperties: false
        properties:
          host: { type: string }
        required:
          - host
      context:
        $schema: http://json-schema.org/schema#
        additionalProperties: true
        type: object
        properties:
          ingress:
            type: object
            additionalProperties: true
            properties:
              className: { type: string }
              domain: { type: string }
            required:
              - domain
              - className
        required:
          - ingress
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
            className: {{ .Context.ingress.className  }}
            hosts:
              - host: {{ .Parameters.host }}.{{ .Context.ingress.domain }}
                paths:
                  - path: /
                    pathType: ImplementationSpecific
    ```

On notera les points suivants:

- Le `tag` a été modifié, permettant ainsi de générer une nouvelle version.
- Le paramètre `fqdn` a été remplacé par `host` car il ne définie maintenant que le hostname, sans le domaine.
- La section `modules[X].values` a été modifiée afin de prendre en compte le `context`.
- Une section `schema.context` a été ajouté, permettant de specifier et valider ce que notre package s'attend à trouver dans le context.

Cette nouvelle version doit etres packagée:

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

```

## Deployment

Voici le manifest de déployment correspondant:

???+ abstract "podinfo-ctx.yaml"
    ```
    ---
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Release
    metadata:
      name: podinfo2
      namespace: default
    spec:
      description: A first sample release of podinfo
      package:
        repository: quay.io/kubodoc/packages/podinfo
        tag: 6.7.1-p02
        interval: 30m
      parameters:
        host: podinfo2
      contexts:
        - namespace: contexts
          name: cluster
    ```

On notera les points suivants:

- Le paramètre `fqdn` a été remplacé par `host`
- Une section `spec.contexts` a été ajoutée. Il s'agit d'une liste de contexts, qui seront agrégés pour constituer
  un object unique, intégré dans le modèle.

!!! warning
    Reférencer un context inexistant génère une erreur


Après avoir ajusté `spec.repository`, le nouveau déploiement peut ètre effectué

```
kubectl create -f podinfo-ctx.yaml 

release.kubocd.kubotal.io/podinfo2 created
```

Si tout se passe correctement, la nouvelle `Release` doit atteindre l'état `READY` et la liste des context doit etre affichée:

```
kubectl get releases podinfo2

NAME       REPOSITORY                         TAG         CONTEXTS           STATUS   READY   WAIT   PRT   AGE   DESCRIPTION
podinfo2   quay.io/kubodoc/packages/podinfo   6.7.1-p02   contexts:cluster   READY    1/1            -     17m   A first sample release of podinfo
```

## Aggregation de contexts

### Example 1

Le context perçu par une application peut résulter de l'agrégation de plusieurs object contexts.

Par exemple, un 'context projet' peut etre crée pour partagé des variables commune à l'ensemble des applications composant ce projet. Celui-ci sera mergé avec le context global du cluster.

Dans l'exemple suivant, un namespace et un context seront crées par projet déployé. 

Création du namespace:

```
kubectl create namespace project01

namespace/project01 created
```

Et du context:

???+ abstract "project01.yaml"

    ```
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Context
    metadata:
      name: project01
    spec:
      description: Context For projet 1
      context:
        project:
          id: p01
          subdomain: prj01
    
    ```

On notera que le `namespace` n'est pas défini dans le manifest. Il sera défini en ligne de commande:

```
kubectl -n project01 create -f project01.yaml 

context.kubocd.kubotal.io/project01 created
```

On peu lister l'ensemble des contexts définis

```
kubectl get --all-namespaces contexts.kubocd.kubotal.io

NAMESPACE   NAME        DESCRIPTION                          PARENTS   STATUS   AGE
contexts    cluster     Context global for kubodoc cluster             READY    2d2h
project01   project01   Context For projet 1                           READY    2m35s
```

Ce premier exemple implique de modifier le package afin de prendre en compte notre nouvelle variable:

???+ abstract "podinfo-p03.yaml"
    
    ```
    apiVersion: v1alpha1
    type: Package
    name: podinfo
    tag: 6.7.1-p03
    schema:
      parameters:
        $schema: http://json-schema.org/schema#
        type: object
        additionalProperties: false
        properties:
          host: { type: string }
        required:
          - host
      context:
        $schema: http://json-schema.org/schema#
        additionalProperties: true
        type: object
        properties:
          ingress:
            type: object
            additionalProperties: true
            properties:
              className: { type: string }
              domain: { type: string }
            required:
              - domain
              - className
          project:
            type: object
            additionalProperties: true
            properties:
              subdomain: { type: string }
            required:
              - subdomain
        required:
          - ingress
          - project
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
            className: {{ .Context.ingress.className  }}
            hosts:
              - host: {{ .Parameters.host }}.{{ .Context.project.subdomain }}.{{ .Context.ingress.domain }}
                paths:
                  - path: /
                    pathType: ImplementationSpecific
    ```

On n'oubliera pas de la packager:

```
kubocd pack podinfo-p03.yaml 
```

Une nouvelle `Release` sera crée pour le déploiement:


???+ abstract "podinfo-prj01.yaml"

    ```
    ---
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Release
    metadata:
      name: podinfo
    spec:
      description: A release of podinfo on project01
      package:
        repository: quay.io/kubodoc/packages/podinfo
        tag: 6.7.1-p03
        interval: 30m
      parameters:
        host: podinfo
      contexts:
        - namespace: contexts
          name: cluster
        - name: project01
      debug:
        dumpContext: true
        dumpParameters: true
    ```

A noter:

- `metadata.namespace` n'est pas défini. Il le sera en ligne de commande.
- `metadata.name` est simplement `podinfo`. Il n'est pas prévu de déployer plus d'une instance de `podinfo` par namespace.
- `spec.context` contient maintenant deux entrées. La seconde référence notre 'context project'. Le `namespace` n'y est pas défini. Il sera donc celui de la `Release`.
- Une nouvelle section `spec.debug` est introduite. Elle comporte deux flags qui vont demande à KuboCD de placer dans le `Status` de l'objet `Release` la vision perçue du context et des paramètres.

Le déploiement:

```
kubectl -n project01 create -f podinfo-prj01.yaml 

release.kubocd.kubotal.io/podinfo3 created
```

Si l'on examine l'objet résultant, on y trouve bien les deux contexts:

```
kubectl -n project01 get releases podinfo
NAME      REPOSITORY                         TAG         CONTEXTS                               STATUS   READY   WAIT   PRT   AGE     DESCRIPTION
podinfo   quay.io/kubodoc/packages/podinfo   6.7.1-p03   contexts:cluster,project01:project01   READY    1/1            -     8m31s   A release of podinfo on project01
```

On peut vérifier que l'objet `ìngress` associé a bien la bonne valeur;

```
kubectl get --all-namespaces ingress
NAMESPACE   NAME            CLASS   HOSTS                                  ADDRESS        PORTS   AGE
default     podinfo1-main   nginx   podinfo1.ingress.kubodoc.local         10.96.218.98   80      2d20h
default     podinfo2-main   nginx   podinfo2.ingress.kubodoc.local         10.96.218.98   80      71m
project01   podinfo-main    nginx   podinfo.prj01.ingress.kubodoc.local    10.96.218.98   80      13m
```

Maintenant, examinons plus en détail le `Status` de la `Release`:

```
kubectl -n project01 get release podinfo -o yaml

apiVersion: kubocd.kubotal.io/v1alpha1
kind: Release
metadata:
  ....
spec:
  ....
status:
  context:
    ingress:
      className: nginx
      domain: ingress.kubodoc.local
    project:
      id: p01
      subdomain: prj01
    storageClass:
      data: standard
      workspace: standard
  ....
  parameters:
    host: podinfo2
  ....
```

On trouve bien dans le `Status` un context. Et on peut constater que celui-ci intègre bien les valeurs mergées du context du cluster et du context project.

!!! Warning

    Dans la vrai vie, le context peut devenir assez volumineux. Il convient donc d'utiliser ce mode de debugging avec parcimonie.

### Example 2

Dans ce second exemple, l'objectif reste identique (Ajouter un subdomain à l'ingress). Mais, nous utiiserons la version initiale du package, qui ne prend en compte que le context du cluster.

Un namspace dédié est crée:

```
kubectl create ns project02
namespace/project02 created
```

Et un context est créé dans ce namespace:

???+ abstract "project02.yaml"

    ```
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Context
    metadata:
      name: project02
    spec:
      description: Context For projet 2
      context:
        project:
          id: p02
        ingress:
          domain: prj02.ingress.kubodoc.local
    ```

```
kubectl -n project02 create -f project02.yaml 

context.kubocd.kubotal.io/project02 created
```

On y trouve le même path `spec.context.ingress.domain` que dans le context du cluster. 
Lorsque les deux context seront mergé, ils le seront dans l'ordre définie dans l'object `Release`. 
Le context projet étant listé après le context du cluster, sa valeur sera prioritaire. Et cette valeur inclue notre `subdomain`.  


???+ abstract "podinfo-prj02.yaml"

    ```
    ---
    apiVersion: kubocd.kubotal.io/v1alpha1
    kind: Release
    metadata:
      name: podinfo
    spec:
      description: A release of podinfo on project02
      package:
        repository: quay.io/kubodoc/packages/podinfo
        tag: 6.7.1-p02
        interval: 30m
      parameters:
        host: podinfo
      contexts:
        - namespace: contexts
          name: cluster
        - name: project02
      debug:
        dumpContext: true
        dumpParameters: true
    ```

```
kubectl -n project02 create -f podinfo-prj02.yaml 

release.kubocd.kubotal.io/podinfo created
```

On peut examiner le context résultant dans l'objet `Release` déployé:

```
kubectl -n project02 get release podinfo -o yaml
 
apiVersion: kubocd.kubotal.io/v1alpha1
kind: Release
metadata:
    ....
spec:
    ....
status:
  context:
    ingress:
      className: nginx
      domain: prj02.ingress.kubodoc.local
    project:
      id: p02
    storageClass:
      data: standard
      workspace: standard
  ....
```

Et vérifier que l'on a bien la bonne valeur pour notre `ìngress`:

```
kubectl get --all-namespaces ingress
NAMESPACE   NAME            CLASS   HOSTS                                 ADDRESS        PORTS   AGE
default     podinfo1-main   nginx   podinfo1.ingress.kubodoc.local        10.96.218.98   80      2d20h
default     podinfo2-main   nginx   podinfo2.ingress.kubodoc.local        10.96.218.98   80      110m
project01   podinfo-main    nginx   podinfo.prj01.ingress.kubodoc.local   10.96.218.98   80      26m
project02   podinfo-main    nginx   podinfo.prj02.ingress.kubodoc.local   10.96.218.98   80      2m52s
```

## Context change.

Un changement dans un context est automatiquement pris en compte par toutes les `Releases` associées. 
Mais, seules les applications effectivement impactées seront redéployées.

!!! info

    Techiquement, KuboCD vas patcher les object Flux `helmRelease`, ce qui vas entrainer un `helm upgrade` qui, normalement, ne touche que les ressources réellement impacté. 

Par example, modifions le context du project01:

```
kubectl -n project01 patch context.kubocd.kubotal.io project01 --type='json' -p='[{"op": "replace", "path": "/spec/context/project/subdomain", "value": "project01" }]'

context.kubocd.kubotal.io/project01 patched
```

On peut constater que la modification est rapidement prise en compte:

```
kubectl get --all-namespaces ingress

NAMESPACE   NAME            CLASS   HOSTS                                     ADDRESS        PORTS   AGE
default     podinfo1-main   nginx   podinfo1.ingress.kubodoc.local            10.96.218.98   80      3d3h
default     podinfo2-main   nginx   podinfo2.ingress.kubodoc.local            10.96.218.98   80      8h
project01   podinfo-main    nginx   podinfo.project01.ingress.kubodoc.local   10.96.218.98   80      7h13m
project02   podinfo-main    nginx   podinfo.prj02.ingress.kubodoc.local       10.96.218.98   80      6h49m
```

Pour remettre la valeur initiale:

```
kubectl -n project01 patch context.kubocd.kubotal.io project01 --type='json' -p='[{"op": "replace", "path": "/spec/context/project/subdomain", "value": "prj01" }]'

context.kubocd.kubotal.io/project01 patched
```

