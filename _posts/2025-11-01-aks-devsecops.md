---
title: "DevSecOps en AKS: Pipeline Seguro para Aplicaciones en Azure Kubernetes Service"
categories:
  - blog
tags:
  - Azure
  - Kubernetes
  - DevSecOps
  - AKS
  - Security
---

## Introducción

La implementación de DevSecOps en Azure Kubernetes Service (AKS) es fundamental para garantizar la seguridad desde el desarrollo hasta la producción. Este artículo explora cómo crear un pipeline de DevSecOps completo en AKS.

## Configuración de AKS Seguro

```hcl
resource "azurerm_kubernetes_cluster" "secure_aks" {
  name                = "secure-aks"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  dns_prefix          = "secure-aks"

  default_node_pool {
    name       = "default"
    node_count = 3
    vm_size    = "Standard_DS2_v2"
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin    = "azure"
    network_policy    = "calico"
    load_balancer_sku = "standard"
  }

  api_server_access_profile {
    authorized_ip_ranges = ["YOUR_IP_RANGE"]
  }
}
```

## Pipeline de Azure DevOps

```yaml
trigger:
- main

stages:
- stage: Security
  jobs:
  - job: SecurityScans
    steps:
    - task: Container-security@0
      inputs:
        imageName: 'your-app:$(Build.BuildId)'
        severityThreshold: 'CRITICAL'
    
    - task: SonarQube@4
      inputs:
        scannerMode: 'CLI'
        configMode: 'manual'
        cliProjectKey: 'your-project'
        cliProjectName: 'Your Project'

- stage: Deploy
  jobs:
  - deployment: AKS
    environment: production
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@0
            inputs:
              action: deploy
              manifests: |
                $(System.DefaultWorkingDirectory)/k8s/*.yml
```

## Políticas de Azure

```hcl
resource "azurerm_policy_definition" "aks_policy" {
  name         = "aks-security-baseline"
  policy_type  = "Custom"
  mode         = "All"
  display_name = "AKS Security Baseline"

  policy_rule = <<POLICY_RULE
{
  "if": {
    "allOf": [
      {
        "field": "type",
        "equals": "Microsoft.ContainerService/managedClusters"
      }
    ]
  },
  "then": {
    "effect": "audit",
    "details": {
      "type": "Microsoft.Security/complianceResults"
    }
  }
}
POLICY_RULE
}
```

## Monitorización de Seguridad

```yaml
apiVersion: securitycenter.azure.com/v1alpha1
kind: AzureSecurityCenter
metadata:
  name: security-monitoring
spec:
  scope: subscription
  email: "security@yourdomain.com"
  phone: "+1234567890"
  alertNotifications: "on"
  alertsToAdmins: "on"
```

## Integración con Azure Key Vault

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kvname
spec:
  provider: azure
  parameters:
    usePodIdentity: "true"
    keyvaultName: "your-key-vault"
    objects: |
      array:
        - |
          objectName: secret1
          objectType: secret
          objectVersion: ""
```

## Network Security

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

## Container Security

```dockerfile
FROM mcr.microsoft.com/azuredocs/aks-helloworld:v1

# Ejecutar como usuario no root
USER 1000

# Hacer el sistema de archivos de solo lectura
VOLUME ["/tmp"]
```

## Referencias

- [Azure Kubernetes Service Security](https://docs.microsoft.com/en-us/azure/aks/concepts-security)
- [Azure DevOps Security Best Practices](https://docs.microsoft.com/en-us/azure/devops/security/)
- [Azure Security Center for Containers](https://docs.microsoft.com/en-us/azure/security-center/container-security)
