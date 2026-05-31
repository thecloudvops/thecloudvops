---
layout: post
title: "Compliance Dashboard en Azure: construye tu propio Security Hub con Azure Policy y Workbooks"
image: "/assets/images/compliance-dashboard-cover.png"
categories:
  - blog
tags:
  - Azure
  - Azure Policy
  - Landing Zone
  - Terraform
  - Gobernanza
  - Log Analytics
  - Azure Workbooks
  - CAF
---

En las últimas semanas hemos construido, paso a paso, la capa de gobernanza de una Landing Zone real: primero definiendo y asignando **políticas con Terraform** a lo largo de la jerarquía de Management Groups del CAF, y después aprendiendo a gestionar las **excepciones a esas políticas como código**, con trazabilidad, fecha de caducidad y control a través de Pull Requests.

Todo ese trabajo —políticas, iniciativas, asignaciones, excepciones— genera una cantidad enorme de datos de compliance. El problema es que, por defecto, esos datos viven dispersos en cada suscripción. No hay una vista única que diga: ¿cuántos recursos están fuera de regla hoy? ¿En qué suscripción está el mayor problema? ¿Estamos mejorando o empeorando?

Este artículo es el cierre de esa trilogía: vamos a construir el **panel de control centralizado** que da visibilidad sobre todo lo que hemos definido. Algo similar a lo que AWS Security Hub ofrece de serie, pero construido sobre nuestras propias políticas, con **Azure Policy**, **Log Analytics** y **Azure Workbooks**.

El objetivo al final del artículo: un dashboard en tiempo real con el estado de compliance de todas tus suscripciones, desglosado por política, por Management Group y por severidad, con alertas proactivas cuando algo cae por debajo del umbral.

## Las piezas del puzzle

Antes de escribir código, entendamos qué papel juega cada componente:

| Componente | Rol |
|---|---|
| **Azure Policy** | Genera los datos de compliance (qué recursos cumplen, cuáles no) |
| **Azure Monitor / Diagnostic Settings** | Exporta los eventos de compliance a Log Analytics |
| **Log Analytics Workspace** | Almacén centralizado de todos los datos de compliance |
| **Azure Workbooks** | Capa de visualización: dashboards interactivos sobre los datos de Log Analytics |
| **Terraform** | Gestiona toda la infraestructura como código y reproducible |

La clave es el flujo: **Azure Policy → Log Analytics → Workbooks**. Una vez establecido, el dashboard se actualiza automáticamente.

## Paso 1: Log Analytics Workspace centralizado

Todo el compliance data necesita ir a un único workspace. En una Landing Zone real, este workspace vive en la suscripción de Management:

```hcl
# management/log_analytics.tf

resource "azurerm_log_analytics_workspace" "central" {
  name                = "law-platform-management"
  location            = var.location
  resource_group_name = azurerm_resource_group.management.name
  sku                 = "PerGB2018"
  retention_in_days   = 90

  tags = {
    environment = "platform"
    owner       = "equipo-plataforma"
    cost-center = "platform-engineering"
  }
}
```

## Paso 2: Exportar compliance data de Azure Policy a Log Analytics

Azure Policy puede exportar sus datos de compliance directamente a Log Analytics mediante **Diagnostic Settings** a nivel de suscripción:

```hcl
# management/policy_diagnostics.tf

# Exportar compliance de cada suscripción al workspace central
resource "azurerm_monitor_diagnostic_setting" "policy_compliance" {
  for_each = var.subscription_ids

  name               = "policy-compliance-to-law"
  target_resource_id = "/subscriptions/${each.value}"
  log_analytics_workspace_id = azurerm_log_analytics_workspace.central.id

  enabled_log {
    category = "Policy"
  }

  metric {
    category = "AllMetrics"
    enabled  = false
  }
}
```

Y en `variables.tf`, el mapa de suscripciones:

```hcl
variable "subscription_ids" {
  description = "Mapa de suscripciones a monitorizar"
  type        = map(string)
  default     = {
    "corp-prod"    = "00000000-0000-0000-0000-000000000001"
    "corp-preprod" = "00000000-0000-0000-0000-000000000002"
    "online-prod"  = "00000000-0000-0000-0000-000000000003"
    "platform"     = "00000000-0000-0000-0000-000000000004"
  }
}
```

## Paso 3: Consultar los datos con KQL

Una vez que los datos fluyen a Log Analytics, puedes consultarlos con **Kusto Query Language (KQL)**. Aquí las queries base que alimentarán el dashboard:

### Estado de compliance por suscripción

```kusto
AzureActivity
| where CategoryValue == "Policy"
| where OperationNameValue contains "policyStates"
| extend PolicyName = tostring(Properties.policyDefinitionName)
| extend ComplianceState = tostring(Properties.isCompliant)
| extend SubscriptionId = tostring(SubscriptionId)
| summarize
    Total = count(),
    Compliant = countif(ComplianceState == "true"),
    NonCompliant = countif(ComplianceState == "false")
    by SubscriptionId, PolicyName
| extend ComplianceRate = round(100.0 * Compliant / Total, 1)
| sort by ComplianceRate asc
```

### Top 10 políticas con más incumplimientos

```kusto
PolicyStates
| where TimeGenerated > ago(24h)
| where ComplianceState == "NonCompliant"
| summarize NonCompliantResources = count() by PolicyDefinitionName, PolicyAssignmentScope
| top 10 by NonCompliantResources desc
| project PolicyDefinitionName, PolicyAssignmentScope, NonCompliantResources
```

### Evolución del compliance en el tiempo

```kusto
PolicyStates
| where TimeGenerated > ago(30d)
| summarize
    Compliant = countif(ComplianceState == "Compliant"),
    NonCompliant = countif(ComplianceState == "NonCompliant")
    by bin(TimeGenerated, 1d)
| extend ComplianceRate = round(100.0 * Compliant / (Compliant + NonCompliant), 1)
| project TimeGenerated, ComplianceRate
| render timechart
```

## Paso 4: Azure Workbook como dashboard centralizado

Un **Azure Workbook** es un dashboard interactivo que vive dentro de Azure Monitor y puede leer directamente de Log Analytics. La ventaja sobre Power BI: no hay gateway que mantener, está integrado nativamente y se puede compartir con permisos de Azure RBAC.

Puedes desplegar el Workbook como código con Terraform usando un template ARM embebido:

```hcl
# management/compliance_workbook.tf

resource "azurerm_application_insights_workbook" "compliance_dashboard" {
  name                = "compliance-dashboard"
  resource_group_name = azurerm_resource_group.management.name
  location            = var.location
  display_name        = "Compliance Dashboard - Landing Zone"

  data_json = jsonencode({
    version = "Notebook/1.0"
    items = [
      {
        type = 1  # Text
        content = {
          json = "## 🛡️ Compliance Dashboard\nEstado de cumplimiento de Azure Policy en todas las suscripciones"
        }
      },
      {
        type = 3  # Query
        content = {
          version        = "KqlItem/1.0"
          query          = <<-KQL
            PolicyStates
            | where TimeGenerated > ago(1d)
            | summarize
                Total = count(),
                Compliant = countif(ComplianceState == "Compliant"),
                NonCompliant = countif(ComplianceState == "NonCompliant")
                by SubscriptionId
            | extend ComplianceRate = round(100.0 * Compliant / Total, 1)
          KQL
          queryType      = 0
          resourceType   = "microsoft.operationalinsights/workspaces"
          visualization  = "table"
          gridSettings = {
            formatters = [
              {
                columnMatch = "ComplianceRate"
                formatter   = 18  # Threshold coloring
                thresholdsOptions = {
                  steps = [
                    { value = 0, color = "red" }
                    { value = 80, color = "yellow" }
                    { value = 95, color = "green" }
                  ]
                }
              }
            ]
          }
        }
      }
    ]
  })

  tags = {
    environment = "platform"
    owner       = "equipo-plataforma"
  }
}
```

## Paso 5: Alertas automáticas cuando el compliance cae

El dashboard es útil para revisiones periódicas, pero también quieres que el equipo sea notificado proactivamente cuando el compliance de alguna suscripción cae por debajo de un umbral:

```hcl
# management/compliance_alert.tf

resource "azurerm_monitor_scheduled_query_rules_alert_v2" "low_compliance" {
  name                = "alert-compliance-below-threshold"
  resource_group_name = azurerm_resource_group.management.name
  location            = var.location

  evaluation_frequency = "PT1H"
  window_duration      = "PT1H"
  scopes               = [azurerm_log_analytics_workspace.central.id]
  severity             = 2  # Warning

  criteria {
    query = <<-KQL
      PolicyStates
      | where TimeGenerated > ago(1h)
      | summarize
          Total = count(),
          NonCompliant = countif(ComplianceState == "NonCompliant")
          by SubscriptionId
      | extend ComplianceRate = round(100.0 * (Total - NonCompliant) / Total, 1)
      | where ComplianceRate < 90
    KQL

    time_aggregation_method = "Count"
    threshold               = 0
    operator                = "GreaterThan"

    failing_periods {
      minimum_failing_periods_to_trigger_alert = 1
      number_of_evaluation_periods             = 1
    }
  }

  action {
    action_groups = [azurerm_monitor_action_group.platform_team.id]
  }

  display_name = "Compliance por debajo del 90% en alguna suscripción"
}

resource "azurerm_monitor_action_group" "platform_team" {
  name                = "ag-platform-team"
  resource_group_name = azurerm_resource_group.management.name
  short_name          = "platform"

  email_receiver {
    name          = "Platform Team"
    email_address = "platform-team@contoso.com"
  }

  # También puedes añadir un webhook a Teams o Slack
  webhook_receiver {
    name        = "Teams Channel"
    service_uri = var.teams_webhook_url
  }
}
```

## El resultado: tu propio Security Hub en Azure

Con estas piezas ensambladas tienes:

- **Vista centralizada** de todas las suscripciones sin entrar en cada una
- **Evolución temporal** del compliance: ¿estamos mejorando o empeorando?
- **Top de incumplimientos**: qué políticas tienen más recursos fuera de regla
- **Alertas proactivas**: el equipo se entera antes de que lo escale alguien de negocio
- **Todo como código**: reproducible, versionado y auditable en Git

La diferencia con AWS Security Hub es que aquí tú controlas exactamente qué ves y cómo se presenta. No dependes de los findings predefinidos de AWS: tus políticas son tuyas, tu dashboard es tuyo.

## Conclusión: la trilogía de gobernanza completa

Con este artículo cerramos un ciclo que empezamos hace dos semanas:

1. **[CAF Landing Zone: Cómo definir y asignar políticas de Azure con Terraform](/blog/caf-landing-zone-policies-terraform)** — construir la estructura de Management Groups y gestionar políticas como código
2. **[Policy Exceptions as Code en Azure Landing Zones con Terraform](/blog/policy-exceptions-as-code-azure-terraform)** — controlar las excepciones con trazabilidad, caducidad y proceso de aprobación
3. **Este artículo** — el panel de control que da visibilidad sobre todo lo anterior en tiempo real

Azure Policy genera datos de compliance muy ricos, pero por defecto viven dispersos en cada suscripción. La combinación de **Diagnostic Settings → Log Analytics → Workbooks + Alertas** convierte esos datos en inteligencia centralizada y accionable.

El equipo de plataforma deja de reaccionar a problemas de compliance cuando alguien los descubre por accidente, y empieza a detectarlos antes de que tengan impacto. Eso, en esencia, es lo que hace un Security Hub: no más sorpresas en la auditoría.

En las próximas semanas continuamos con la siguiente dimensión de gobernanza: **el RBAC como código** y la **detección de drift** para garantizar que nadie pueda saltarse el proceso sin que el equipo lo sepa.
