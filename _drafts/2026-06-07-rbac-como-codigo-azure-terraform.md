---
layout: post
title: "RBAC como código en Azure Landing Zones con Terraform"
image: "/assets/images/rbac-code-cover.png"
categories:
  - blog
tags:
  - Azure
  - RBAC
  - Landing Zone
  - Terraform
  - Gobernanza
  - Identidad
  - CAF
---

Si has leído los artículos anteriores de esta serie, ya tienes las políticas de Azure gestionadas como código. Pero hay una segunda dimensión igual de importante que suele quedarse fuera del control de Terraform: **quién puede hacer qué** en cada suscripción.

El RBAC (Role-Based Access Control) de Azure es el mecanismo que responde a esa pregunta. Y como las políticas, si no lo gestionas como código, con el tiempo se convierte en un laberinto: `Owner` asignados a mano en el portal, roles custom creados por distintos equipos sin coordinación, y nadie que sepa a ciencia cierta por qué cierto Service Principal tiene acceso a producción.

En este artículo vamos a ver cómo llevar el RBAC a Terraform de forma estructurada, siguiendo el mismo patrón de módulos que hemos usado para políticas y excepciones.

## Por qué el RBAC manual se rompe siempre

El problema con gestionar RBAC desde el portal es siempre el mismo: **la intención se pierde**. Una asignación de rol en el portal dice quién tiene acceso, pero no dice por qué, quién lo aprobó, ni cuándo debería revisarse.

Con Terraform, cada asignación de rol es un fichero que:
- Documenta el motivo en comentarios
- Pasa por un Pull Request para su aprobación
- Queda registrado en Git con el autor del cambio
- Puede eliminarse fácilmente cuando ya no sea necesario

El offboarding de personal es el mayor punto ciego: las asignaciones de quien se fue siguen activas meses después. Con código, eliminar un acceso es tan sencillo como abrir un PR.

## Estructura de módulos para RBAC

Siguiendo la arquitectura de módulos de los artículos anteriores:

```
infra/
├── management_groups/
├── policies/
└── rbac/                          ← nuevo
    ├── custom_roles/
    │   ├── platform_engineer.tf   → roles custom de la organización
    │   └── readonly_auditor.tf
    ├── assignments/
    │   ├── platform_mg.tf         → asignaciones por MG
    │   ├── corp_mg.tf
    │   └── service_principals.tf  → identidades de automatización
    └── variables.tf
```

## Paso 1: Definir roles custom

Azure tiene más de 100 roles built-in, pero las organizaciones siempre necesitan algunos propios. Aquí el rol **Platform Engineer**: puede gestionar infraestructura de plataforma sin acceso a los datos de las aplicaciones:

```hcl
# rbac/custom_roles/platform_engineer.tf

resource "azurerm_role_definition" "platform_engineer" {
  name        = "Platform Engineer"
  scope       = azurerm_management_group.root.id
  description = "Gestión de infraestructura de plataforma sin acceso a datos de aplicación."

  permissions {
    actions = [
      "Microsoft.Management/managementGroups/read",
      "Microsoft.Management/managementGroups/write",
      "Microsoft.Network/virtualNetworks/*",
      "Microsoft.Network/networkSecurityGroups/*",
      "Microsoft.Network/firewalls/*",
      "Microsoft.Authorization/policyDefinitions/*",
      "Microsoft.Authorization/policyAssignments/*",
      "Microsoft.OperationalInsights/workspaces/*",
      "Microsoft.Insights/diagnosticSettings/*",
    ]

    not_actions = [
      # Nunca puede ver secretos ni datos de aplicación
      "Microsoft.Storage/storageAccounts/listKeys/action",
      "Microsoft.KeyVault/vaults/secrets/read",
      "Microsoft.Sql/servers/databases/read",
    ]
  }

  assignable_scopes = [
    azurerm_management_group.root.id,
    azurerm_management_group.platform.id,
  ]
}
```

Y el rol **Read-Only Auditor** para equipos de seguridad y cumplimiento:

```hcl
# rbac/custom_roles/readonly_auditor.tf

resource "azurerm_role_definition" "readonly_auditor" {
  name        = "Read-Only Auditor"
  scope       = azurerm_management_group.root.id
  description = "Acceso de solo lectura a toda la organización para auditorías. Sin acceso a secretos."

  permissions {
    actions = [
      "*/read",
      "Microsoft.Authorization/*/read",
      "Microsoft.PolicyInsights/*/read",
    ]

    not_actions = [
      "Microsoft.Storage/storageAccounts/listKeys/action",
      "Microsoft.KeyVault/vaults/secrets/read",
      "Microsoft.KeyVault/vaults/keys/read",
    ]
  }

  assignable_scopes = [
    azurerm_management_group.root.id,
  ]
}
```

## Paso 2: Asignar roles en los Management Groups

Una vez definidos los roles, las asignaciones siguen el mismo principio que las políticas: asignar al nivel más alto donde tiene sentido para todos los hijos:

```hcl
# rbac/assignments/platform_mg.tf

# Equipo de plataforma → Platform Engineer en el MG de Platform
resource "azurerm_role_assignment" "platform_team_platform_mg" {
  scope              = azurerm_management_group.platform.id
  role_definition_id = azurerm_role_definition.platform_engineer.role_definition_resource_id
  principal_id       = var.platform_team_group_object_id
}

# Auditores → solo lectura en toda la organización
resource "azurerm_role_assignment" "auditors_root_mg" {
  scope              = azurerm_management_group.root.id
  role_definition_id = azurerm_role_definition.readonly_auditor.role_definition_resource_id
  principal_id       = var.audit_team_group_object_id
}

# Equipos de aplicaciones → Contributor solo en su suscripción
resource "azurerm_role_assignment" "corp_app_teams" {
  for_each = var.corp_subscriptions

  scope                = "/subscriptions/${each.value.subscription_id}"
  role_definition_name = "Contributor"
  principal_id         = each.value.team_group_object_id
}
```

## Paso 3: Controlar los Service Principals de CI/CD

Una parte que suele estar completamente fuera de control: los Service Principals de los pipelines. Con Terraform puedes inventariarlos y controlar exactamente qué permisos tienen:

```hcl
# rbac/assignments/service_principals.tf

# SP del pipeline de Terraform (mínimos permisos necesarios)
resource "azurerm_role_assignment" "terraform_pipeline_contributor" {
  scope                = azurerm_management_group.landing_zones.id
  role_definition_name = "Contributor"
  principal_id         = var.terraform_pipeline_sp_object_id
}

# Permisos adicionales para gestionar políticas
resource "azurerm_role_assignment" "terraform_pipeline_policy" {
  scope                = azurerm_management_group.landing_zones.id
  role_definition_name = "Resource Policy Contributor"
  principal_id         = var.terraform_pipeline_sp_object_id
}
```

## Paso 4: Variables tipadas para el inventario de identidades

Para evitar hardcodear Object IDs, un fichero de variables bien tipado:

```hcl
# rbac/variables.tf

variable "platform_team_group_object_id" {
  description = "Object ID del grupo Azure AD del equipo de Plataforma"
  type        = string
}

variable "audit_team_group_object_id" {
  description = "Object ID del grupo Azure AD del equipo de Auditoría"
  type        = string
}

variable "terraform_pipeline_sp_object_id" {
  description = "Object ID del Service Principal de los pipelines de Terraform"
  type        = string
}

variable "corp_subscriptions" {
  description = "Mapa de suscripciones Corp con su equipo asociado"
  type = map(object({
    subscription_id      = string
    team_group_object_id = string
  }))
}
```

Y en el `tfvars` de producción:

```hcl
# environments/prod/rbac.tfvars

platform_team_group_object_id   = "aaaaaaaa-0000-0000-0000-000000000001"
audit_team_group_object_id      = "bbbbbbbb-0000-0000-0000-000000000002"
terraform_pipeline_sp_object_id = "cccccccc-0000-0000-0000-000000000003"

corp_subscriptions = {
  "equipo-contabilidad" = {
    subscription_id      = "00000000-0000-0000-0000-000000000010"
    team_group_object_id = "dddddddd-0000-0000-0000-000000000004"
  }
  "equipo-rrhh" = {
    subscription_id      = "00000000-0000-0000-0000-000000000011"
    team_group_object_id = "eeeeeeee-0000-0000-0000-000000000005"
  }
}
```

## El ciclo de vida de una asignación de rol

Con este modelo el proceso es siempre el mismo:

1. **Solicitud:** El equipo abre un PR añadiendo la entrada en `tfvars`
2. **Revisión:** El equipo de plataforma valida el scope mínimo necesario
3. **Aprobación y merge:** La pipeline aplica la asignación automáticamente
4. **Offboarding:** Se abre un PR eliminando la entrada — queda en el historial de Git como evidencia

## Conclusión

RBAC como código no es solo comodidad: es **seguridad operativa**. Cuando sabes exactamente qué accesos existen, quién los aprobó y cuándo, dejas de tener sorpresas en las auditorías.

Combinado con las políticas y las excepciones de los artículos anteriores, el RBAC como código completa la trilogía de gobernanza: **qué está permitido (políticas), quién puede hacerlo (RBAC) y cuándo se hacen excepciones (exemptions)**. Una Landing Zone que controla las tres dimensiones es una Landing Zone madura.
