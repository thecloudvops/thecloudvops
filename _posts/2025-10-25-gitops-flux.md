---
title: "GitOps en Acción: Implementando Flux CD en Kubernetes"
categories:
  - blog
tags:
  - GitOps
  - Kubernetes
  - Flux
  - DevOps
  - Automatización
---

## Introducción

GitOps es un enfoque moderno para la gestión de infraestructura y aplicaciones que utiliza Git como fuente única de verdad. En este artículo, exploraremos cómo implementar GitOps usando Flux CD en Kubernetes.

## ¿Qué es GitOps?

GitOps es una práctica que:
- Utiliza Git como fuente de verdad
- Automatiza la implementación de cambios
- Garantiza la convergencia del estado deseado
- Proporciona auditabilidad completa

## Instalación de Flux CD

```bash
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/production \
  --personal
```

## Estructura del Repositorio

```plaintext
fleet-infra/
├── clusters/
│   └── production/
│       ├── infrastructure/
│       └── apps/
├── infrastructure/
│   ├── sources/
│   └── base/
└── apps/
    ├── base/
    └── overlays/
```

## Definición de Fuentes

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: GitRepository
metadata:
  name: podinfo
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/stefanprodan/podinfo
  ref:
    branch: master
```

## Kustomization

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: podinfo
  namespace: flux-system
spec:
  interval: 10m
  path: "./kustomize"
  prune: true
  sourceRef:
    kind: GitRepository
    name: podinfo
  targetNamespace: default
```

## Helm Releases

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: redis
  namespace: flux-system
spec:
  interval: 5m
  chart:
    spec:
      chart: redis
      version: "^12.0.0"
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: flux-system
  values:
    architecture: standalone
```

## Monitorización y Alertas

```yaml
apiVersion: notification.toolkit.fluxcd.io/v1beta1
kind: Alert
metadata:
  name: slack-notification
  namespace: flux-system
spec:
  providerRef:
    name: slack
  eventSeverity: info
  eventSources:
    - kind: GitRepository
      name: '*'
    - kind: Kustomization
      name: '*'
```

## Automatización de Políticas

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImagePolicy
metadata:
  name: podinfo
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: podinfo
  policy:
    semver:
      range: '>=1.0.0'
```

## Mejores Prácticas

1. **Estructura del Repositorio**:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespaces
  - infrastructure
  - apps
```

2. **Control de Versiones**:
```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  git:
    commit:
      author:
        email: fluxcdbot@users.noreply.github.com
        name: fluxcdbot
```

## Troubleshooting

```bash
flux get all
flux logs --all-namespaces
flux reconcile source git flux-system
```

## Referencias

- [Flux CD Documentation](https://fluxcd.io/docs/)
- [GitOps Principles](https://www.weave.works/technologies/gitops/)
- [Flux GitHub Repository](https://github.com/fluxcd/flux2)
