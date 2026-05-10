---
layout: post
title: "CAF Landing Zone: Cómo definir y asignar políticas de Azure con Terraform"
image: "/assets/images/caf-landing-zone-policies.png"
categories:
  - blog
tags:
  - Azure
  - Landing Zone
  - Terraform
  - Azure Policy
  - CAF
  - Gobernanza
---

Si alguna vez has intentado explicar a alguien cómo funciona la gobernanza en Azure, probablemente hayas dibujado un árbol en una pizarra. Un árbol con cajas que representan grupos, suscripciones y recursos, y flechas que muestran cómo las políticas "bajan" de arriba a abajo aplicándose a todo lo que hay debajo.

Ese árbol tiene un nombre: la jerarquía de **Management Groups** del **Cloud Adoption Framework (CAF)** de Microsoft. Y las políticas que fluyen por él son el mecanismo más potente que tienes para garantizar que tu infraestructura en Azure cumple las reglas desde el minuto cero, sin depender de que cada equipo lo haga manualmente.

En este artículo vamos a ver exactamente cómo funciona esa jerarquía y cómo definir y asignar políticas con **Terraform** de forma estructurada y reproducible.

## La jerarquía CAF: qué es cada caja y para qué sirve

El CAF define una estructura de Management Groups de referencia que cualquier organización puede adoptar como punto de partida. No es obligatorio seguirla al pie de la letra, pero parte de una lógica muy sólida:

```
Tenant Root Group
└── Intermediate Root MG  (nombre de tu organización)
    ├── Platform
    │   ├── Identity       → suscripción con AAD DS, ADCS
    │   ├── Management     → Log Analytics, Defender, Automation
    │   └── Connectivity   → Hub de red, Firewall, VPN/ExpressRoute
    ├── Landing Zones
    │   ├── Corp           → workloads conectados a la red corporativa
    │   └── Online         → workloads públicos, sin peering a on-prem
    ├── Sandbox            → entornos de prueba sin restricciones de prod
    └── Decommissioned     → suscripciones en proceso de baja
```

**¿Por qué esta estructura importa para las políticas?**

Porque Azure Policy hereda hacia abajo. Una política asignada en el MG raíz de tu organización se aplica automáticamente a **todas** las suscripciones y recursos que cuelguen de él. Esto te permite definir reglas una sola vez y tener la certeza de que se aplican en todos sitios, sin excepciones.

### Qué va en cada nivel

| Management Group | Políticas típicas |
|---|---|
| **Intermediate Root** | Logs de diagnóstico obligatorios, etiquetas mínimas requeridas, regiones permitidas |
| **Platform** | Defender for Cloud activado, workspace de Log Analytics vinculado |
| **Landing Zones** | Restricciones de SKU, límites de coste, auditoría de configuración |
| **Corp** | Peering de red obligatorio, DNS privado, NSG requerido |
| **Online** | Permitir IPs públicas, WAF obligatorio en Application Gateway |
| **Sandbox** | Relajar restricciones de prod, límite de presupuesto bajo |

## Terraform: estructura de módulos para gestionar políticas

Antes de escribir código, la decisión más importante es **cómo organizar los ficheros**. Una estructura que funciona bien en proyectos reales:

```
infra/
├── management_groups/
│   ├── main.tf          → definición de la jerarquía de MGs
│   └── variables.tf
├── policies/
│   ├── definitions/
│   │   ├── require_tags.tf          → definición de políticas custom
│   │   ├── allowed_locations.tf
│   │   └── require_diagnostic_logs.tf
│   ├── initiatives/
│   │   └── security_baseline.tf    → agrupación de políticas en iniciativas
│   └── assignments/
│       ├── root_mg.tf               → asignaciones por nivel de MG
│       ├── platform_mg.tf
│       ├── landing_zones_mg.tf
│       └── corp_mg.tf
└── providers.tf
```

Esta separación entre **definiciones**, **iniciativas** y **asignaciones** es clave. Te permite reutilizar la misma definición de política en múltiples niveles con distintos parámetros.

## Paso 1: Crear la jerarquía de Management Groups

```hcl
# management_groups/main.tf

# MG raíz de la organización (hijo directo del Tenant Root Group)
resource "azurerm_management_group" "root" {
  name         = "mg-contoso"
  display_name = "Contoso"
}

# Platform
resource "azurerm_management_group" "platform" {
  name                       = "mg-platform"
  display_name               = "Platform"
  parent_management_group_id = azurerm_management_group.root.id
}

resource "azurerm_management_group" "identity" {
  name                       = "mg-identity"
  display_name               = "Identity"
  parent_management_group_id = azurerm_management_group.platform.id
}

resource "azurerm_management_group" "management" {
  name                       = "mg-management"
  display_name               = "Management"
  parent_management_group_id = azurerm_management_group.platform.id
}

resource "azurerm_management_group" "connectivity" {
  name                       = "mg-connectivity"
  display_name               = "Connectivity"
  parent_management_group_id = azurerm_management_group.platform.id
}

# Landing Zones
resource "azurerm_management_group" "landing_zones" {
  name                       = "mg-landing-zones"
  display_name               = "Landing Zones"
  parent_management_group_id = azurerm_management_group.root.id
}

resource "azurerm_management_group" "corp" {
  name                       = "mg-corp"
  display_name               = "Corp"
  parent_management_group_id = azurerm_management_group.landing_zones.id
}

resource "azurerm_management_group" "online" {
  name                       = "mg-online"
  display_name               = "Online"
  parent_management_group_id = azurerm_management_group.landing_zones.id
}

# Sandbox y Decommissioned
resource "azurerm_management_group" "sandbox" {
  name                       = "mg-sandbox"
  display_name               = "Sandbox"
  parent_management_group_id = azurerm_management_group.root.id
}

resource "azurerm_management_group" "decommissioned" {
  name                       = "mg-decommissioned"
  display_name               = "Decommissioned"
  parent_management_group_id = azurerm_management_group.root.id
}
```

## Paso 2: Definir políticas custom

Azure tiene cientos de políticas built-in, pero casi siempre necesitas alguna propia. Aquí una política que obliga a que todos los recursos tengan las etiquetas `environment` y `owner`:

```hcl
# policies/definitions/require_tags.tf

resource "azurerm_policy_definition" "require_tags" {
  name         = "require-mandatory-tags"
  policy_type  = "Custom"
  mode         = "Indexed"
  display_name = "Requerir etiquetas obligatorias en recursos"

  metadata = jsonencode({
    category = "Tags"
    version  = "1.0.0"
  })

  parameters = jsonencode({
    tagNames = {
      type = "Array"
      metadata = {
        displayName = "Nombres de etiquetas requeridas"
        description = "Lista de etiquetas que deben existir en el recurso"
      }
      defaultValue = ["environment", "owner", "cost-center"]
    }
  })

  policy_rule = jsonencode({
    if = {
      allOf = [
        {
          field  = "tags['environment']"
          exists = "false"
        }
      ]
    }
    then = {
      effect = "Deny"
    }
  })
}
```

Y aquí una política que restringe las regiones donde se pueden crear recursos:

```hcl
# policies/definitions/allowed_locations.tf

resource "azurerm_policy_definition" "allowed_locations" {
  name         = "allowed-azure-regions"
  policy_type  = "Custom"
  mode         = "Indexed"
  display_name = "Regiones de Azure permitidas"

  metadata = jsonencode({
    category = "General"
    version  = "1.0.0"
  })

  parameters = jsonencode({
    allowedLocations = {
      type = "Array"
      metadata = {
        displayName     = "Regiones permitidas"
        description     = "Lista de regiones donde se pueden desplegar recursos"
        strongType      = "location"
      }
    }
  })

  policy_rule = jsonencode({
    if = {
      not = {
        field  = "location"
        in     = "[parameters('allowedLocations')]"
      }
    }
    then = {
      effect = "Deny"
    }
  })
}
```

## Paso 3: Agrupar políticas en una iniciativa (Policy Set)

Las iniciativas permiten agrupar varias políticas relacionadas y asignarlas de una sola vez. Aquí un baseline de seguridad para todas las Landing Zones:

```hcl
# policies/initiatives/security_baseline.tf

resource "azurerm_policy_set_definition" "security_baseline" {
  name         = "security-baseline-landingzones"
  policy_type  = "Custom"
  display_name = "Security Baseline - Landing Zones"

  metadata = jsonencode({
    category = "Security"
    version  = "1.0.0"
  })

  parameters = jsonencode({
    allowedLocations = {
      type = "Array"
      metadata = {
        displayName = "Regiones permitidas"
      }
      defaultValue = ["westeurope", "northeurope"]
    }
  })

  # Política de regiones
  policy_definition_reference {
    policy_definition_id = azurerm_policy_definition.allowed_locations.id
    parameter_values = jsonencode({
      allowedLocations = {
        value = "[parameters('allowedLocations')]"
      }
    })
  }

  # Política de etiquetas
  policy_definition_reference {
    policy_definition_id = azurerm_policy_definition.require_tags.id
  }

  # Política built-in: Defender for Cloud activado
  policy_definition_reference {
    policy_definition_id = "/providers/Microsoft.Authorization/policyDefinitions/a9b99dd8-06c5-4317-8629-9d86a3c6e7d6"
  }
}
```

## Paso 4: Asignar las políticas en los Management Groups correctos

Aquí está la parte donde muchos proyectos fallan: asignar demasiado arriba (todo se bloquea) o demasiado abajo (se pierde la herencia). La regla de oro: **asigna al nivel más alto donde la política tenga sentido para todos los hijos**.

```hcl
# policies/assignments/root_mg.tf

# El baseline de seguridad se aplica a TODAS las Landing Zones
resource "azurerm_management_group_policy_assignment" "security_baseline" {
  name                 = "security-baseline"
  management_group_id  = azurerm_management_group.landing_zones.id
  policy_definition_id = azurerm_policy_set_definition.security_baseline.id
  display_name         = "Security Baseline - Landing Zones"
  location             = "westeurope"

  # Identidad necesaria para políticas con efecto DeployIfNotExists o Modify
  identity {
    type = "SystemAssigned"
  }

  parameters = jsonencode({
    allowedLocations = {
      value = ["westeurope", "northeurope"]
    }
  })
}

# policies/assignments/corp_mg.tf

# En Corp, además, exigimos que todas las VMs estén en una VNet aprobada
resource "azurerm_management_group_policy_assignment" "corp_network" {
  name                 = "corp-require-vnet"
  management_group_id  = azurerm_management_group.corp.id
  policy_definition_id = "/providers/Microsoft.Authorization/policyDefinitions/88c0b9da-ce96-4b03-9635-f29a937e2900"
  display_name         = "Corp: VMs deben estar en VNet aprobada"
  location             = "westeurope"

  identity {
    type = "SystemAssigned"
  }
}

# policies/assignments/sandbox_mg.tf

# En Sandbox relajamos las restricciones de región
resource "azurerm_management_group_policy_assignment" "sandbox_locations" {
  name                 = "sandbox-locations"
  management_group_id  = azurerm_management_group.sandbox.id
  policy_definition_id = azurerm_policy_definition.allowed_locations.id
  display_name         = "Sandbox: Regiones permitidas (ampliado)"
  location             = "westeurope"

  parameters = jsonencode({
    allowedLocations = {
      value = ["westeurope", "northeurope", "eastus", "eastus2"]
    }
  })
}
```

## Paso 5: Dar permisos a la identidad de la política (remediación)

Cuando una política tiene efecto `DeployIfNotExists` o `Modify` —es decir, que no solo audita sino que actúa—, la identidad asignada necesita permisos de escritura en el scope. Terraform puede gestionarlo:

```hcl
# Dar permisos a la identidad de la asignación para que pueda remediar
resource "azurerm_role_assignment" "security_baseline_remediation" {
  scope                = azurerm_management_group.landing_zones.id
  role_definition_name = "Contributor"
  principal_id         = azurerm_management_group_policy_assignment.security_baseline.identity[0].principal_id
}
```

## El ciclo completo en la práctica

Una vez desplegada la estructura, el flujo de trabajo para añadir una nueva política es siempre el mismo:

1. **Definir** la política en `policies/definitions/` (o identificar la built-in que necesitas)
2. **Incluirla** en la iniciativa correspondiente en `policies/initiatives/` si aplica
3. **Asignarla** en el MG correcto en `policies/assignments/`
4. **Ejecutar** `terraform plan` para revisar el impacto
5. **Aplicar** con `terraform apply`
6. Si la política tiene remediación, lanzar una **tarea de remediación** desde el portal o con Azure CLI:

```bash
az policy remediation create \
  --name "remediate-security-baseline" \
  --policy-assignment "/providers/Microsoft.Management/managementGroups/mg-landing-zones/providers/Microsoft.Authorization/policyAssignments/security-baseline" \
  --resource-discovery-mode ReEvaluateCompliance
```

## Conclusión

La jerarquía CAF no es solo un diagrama bonito para presentaciones. Es la infraestructura sobre la que construyes toda la gobernanza de tu organización en Azure. Cuando la combinas con Terraform para gestionar las políticas como código, consigues algo muy valioso: **gobernanza reproducible, revisable y auditable**.

Cada política vive en un fichero con su historial de cambios en Git. Cada asignación es explícita y documentada. Y cuando alguien pregunta "¿por qué no puedo desplegar en esta región?", la respuesta está en un `azurerm_policy_definition` que puedes leer, entender y modificar si es necesario.

Eso, en definitiva, es lo que separa una Landing Zone gestionada de un entorno Azure que fue creciendo sin control.
