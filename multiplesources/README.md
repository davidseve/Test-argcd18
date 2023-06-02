# Multiple Sources for an Application

Multiple Sources is a feature to generate and combine resources from different sources in a single Application.


## Documentation
[Official documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/)

## Prerequisites:

- **Red Hat Openshift 4.13** with admin rights.
- Install **OpenShift GitOps 1.8** operator.

## Demo
For this demo, we have a Helm chart that is in a chart repository:
https://redhat-developer.github.io/redhat-helm-charts

And we have a different git repository with values, to override the previous chart:
[values.yaml](chart-values/values.yaml)

```
build:
  enabled: false

deploy:
  route:
    tls:
      enabled: true

image:
  name: quay.io/redhatworkshops/gitops-helm-quarkus
```

This is the application for the demo:[multisoureces-applications.yaml](argo-cd/multisoureces-applications.yaml)
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: multiple-sources
  namespace: openshift-gitops
spec:
  destination:
    namespace: openshift-gitops
    server: 'https://kubernetes.default.svc'
  project: default
  sources:
    - chart: quarkus
      repoURL: https://redhat-developer.github.io/redhat-helm-charts
      targetRevision: 0.0.3
      helm:
        valueFiles:
        - $values/multiplesources/chart-values/values.yaml
    - ref: values
      repoURL: 'https://github.com/davidseve/Test-argcd18.git'
      targetRevision: HEAD
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
```

To do the demo execute this command:
```
oc apply -f argo-cd/multisoureces-applications.yaml
```

You can see that the application that is created is using our values.yaml

## Delete:
```
oc delete -f argo-cd/multisoureces-applications.yaml
```