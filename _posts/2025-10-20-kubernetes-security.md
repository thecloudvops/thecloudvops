---
title: "Seguridad en Kubernetes: Mejores Prácticas y Herramientas"
categories:
  - blog
tags:
  - Kubernetes
  - Security
  - DevSecOps
  - Container
  - Cloud Native
---

## Introducción

La seguridad en Kubernetes es fundamental para proteger aplicaciones cloud-native. En este artículo, exploraremos las mejores prácticas de seguridad y las herramientas más efectivas para proteger clusters de Kubernetes.

## Network Policies

Las Network Policies son cruciales para controlar el tráfico entre pods:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

## RBAC Configuration

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

## Pod Security Policies

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  seLinux:
    rule: RunAsAny
  runAsUser:
    rule: MustRunAsNonRoot
  fsGroup:
    rule: RunAsAny
  volumes:
  - 'configMap'
  - 'emptyDir'
  - 'persistentVolumeClaim'
```

## Herramientas de Seguridad

1. **Trivy para Escaneo de Contenedores**:
```bash
trivy image nginx:latest
```

2. **Falco para Detección de Amenazas**:
```yaml
- rule: Terminal Shell in Container
  desc: A shell was opened in a container
  condition: container and proc.name = bash
  output: Shell opened in container (user=%user.name container=%container.id)
  priority: WARNING
```

## Monitorización de Seguridad

1. **Prometheus y Grafana**:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: security-metrics
spec:
  endpoints:
  - port: metrics
  selector:
    matchLabels:
      app: security-monitor
```

## Auditoría y Compliance

1. **Kube-bench para CIS Benchmarks**:
```bash
kube-bench run --targets master
```

2. **Audit Logging**:
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse
```

## Gestión de Secretos

1. **Sealed Secrets**:
```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: mysecret
spec:
  encryptedData:
    secret.key: AgBy3i4OJSWK+PiTySYZZA==
```

## Pipeline de Seguridad CI/CD

```yaml
security-scan:
  stage: test
  script:
    - trivy image $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - kubesec scan deployment.yaml
  only:
    - master
```

## Referencias

- [Kubernetes Security Best Practices](https://kubernetes.io/docs/concepts/security/)
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
- [CNCF Security Projects](https://landscape.cncf.io/category=security-compliance)
