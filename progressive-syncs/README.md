# Progressive Syncs

Progressive Syncs is a feature to enable applications in AppSets to sync in a declarative order.

## Documentation
[Official documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Progressive-Syncs/)

## Prerequisites:

- **Red Hat Openshift 4.13** with admin rights.
- Install **OpenShift GitOps 1.8** operator.

Because **Progressive Syncs** is a tech preview feature for **OpenShift GitOps 1.8**. To enable it, ArgoCD instanced have to be configured with this argument:
```
  applicationSet:
    extraCommandArgs:
      - '--enable-progressive-syncs'
```

## Demo

We are also going to use the feature [Multiple Sources](../multiplesources/README.md)

For this demo, we have a Helm chart that is in a chart repository:
https://redhat-developer.github.io/redhat-helm-charts

ArgoCD will deploy this chart progressively in the three environments that we have configured.

- Values file for **dev** environment:[values.yaml](chart-values/dev/values.yaml)
```
deploy:
  route:
    tls:
      enabled: false
```

- Values file for **qa** environment:[values.yaml](chart-values/qa/values.yaml)
```
deploy:
  route:
    tls:
      enabled: true
```

- Values file for **prod** environment:[values.yaml](chart-values/prod/values.yaml)
```
deploy:
  route:
    tls:
      enabled: true
  replicas: 4
```


This is the application for the demo:[progressive-syncs-applications.yaml](argo-cd/progressive-syncs-applications.yaml)

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: progressive-syncs
  namespace: openshift-gitops
spec:
  generators:
  - list:
      elements:
      - env: dev
      - env: qa
      - env: prod
  strategy:
    type: RollingSync
    rollingSync:
      steps:
        - matchExpressions:
            - key: envLabel
              operator: In
              values:
                - dev
        - matchExpressions:
            - key: envLabel
              operator: In
              values:
                - qa
        - matchExpressions:
            - key: envLabel
              operator: In
              values:
                - prod
  goTemplate: true
  template:
    metadata:
      name: '{{.env}}-progressive-syncs'
      labels:
        envLabel: '{{.env}}'
    spec:
      destination:
        namespace: openshift-gitops
        server: 'https://kubernetes.default.svc'
      project: default
      sources:
        - chart: quarkus
          repoURL: https://redhat-developer.github.io/redhat-helm-charts
          targetRevision: 0.0.2
          helm:
            valueFiles:
            - $values/progressive-syncs/chart-values/{{.env}}/values.yaml
            - $values/progressive-syncs/chart-values/values.yaml
        - ref: values
          repoURL: 'https://github.com/davidseve/Test-argcd18.git'
          targetRevision: HEAD
```

To do the demo execute this command:
```
oc apply -f argo-cd/progressive-syncs-applications.yaml
```

You can see that the application that is created is using our values.yaml for each environment.
There is also an order in the application creations:
1. dev
2. qa
3. prod

If there is an error in one environment the next environments are not synchronized.

To simulate a new release the `targetRevision` can be changed to `0.0.3`. The new release will be deployed progressively.
```
oc apply -f argo-cd/progressive-syncs-applications.yaml
```

## Delete:
```
oc delete -f argo-cd/progressive-syncs-applications.yaml
```