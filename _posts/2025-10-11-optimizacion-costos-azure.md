---
title: "Optimización de Costos en Azure: Implementando Azure Cost Management con Terraform"
date: 2025-10-10
categories:
  - blog
tags:
  - Azure
  - Terraform
  - DevOps
  - Cost Management
  - IaC
---

![Azure Cost Management Dashboard](/assets/images/posts/azure-cost/azure-cost-dashboard.png)

## Introducción

¿Te preocupa gastar de más en la nube? No eres el único. Cuando usamos servicios en la nube como Azure, es fácil perder el control de los gastos si no llevamos un seguimiento.

**Azure Cost Management** es una herramienta gratuita de Microsoft que te ayuda a ver en qué se va el dinero, poner límites y recibir alertas si te pasas. Así puedes evitar sorpresas en la factura.

En este artículo, te explico cómo automatizar el control de gastos usando **Terraform** (una herramienta para gestionar la nube con archivos de texto), para que puedas aplicar buenas prácticas de ahorro desde el principio, aunque no tengas experiencia previa.

## ¿Por qué es importante la gestión de costos?

La adopción de la nube trae consigo la flexibilidad de pago por uso, pero también puede llevar a costos inesperados si no se gestiona adecuadamente. Algunos datos importantes:
- Según Microsoft, las empresas pueden ahorrar hasta un 30% en costos de Azure mediante una gestión adecuada
- El 80% de las organizaciones exceden sus presupuestos cloud por falta de monitorización
- La automatización de políticas de costos puede reducir el gasto innecesario en un 25%

## Implementación con Terraform


### 1. Configuración del Presupuesto

Este bloque de código crea un presupuesto mensual en Azure usando Terraform. Así puedes definir un límite de gasto (por ejemplo, 1000 € al mes) y recibir alertas automáticas si te acercas a ese límite. Es útil para evitar sorpresas en la factura y mantener el control de los costos desde el principio.

```hcl
resource "azurerm_consumption_budget_subscription" "dev_budget" {
  name            = "dev-monthly-budget"
  subscription_id = data.azurerm_subscription.current.id

  amount     = 1000
  time_grain = "Monthly"

  time_period {
    start_date = "2025-10-01T00:00:00Z"
    end_date   = "2026-10-01T00:00:00Z"
  }

  notification {
    enabled   = true
    threshold = 90.0
    operator  = "GreaterThan"

    contact_emails = [
      "devops@tuempresa.com"
    ]
  }
}
```

![Presupuesto en Azure Portal](../assets/images/posts/2025-10-11/budget-config.png)


### 2. Políticas de Costos Automatizadas

Este código define una política personalizada en Azure que obliga a que todos los recursos tengan la etiqueta "CostCenter". Si alguien intenta crear un recurso sin esa etiqueta, Azure lo bloqueará automáticamente. Esto ayuda a organizar los gastos por proyecto o departamento y facilita el seguimiento de los costos.

```hcl
resource "azurerm_policy_definition" "cost_center_tag" {
  name         = "require-cost-center-tag"
  policy_type  = "Custom"
  mode         = "All"
  display_name = "Require cost center tag for resources"

  policy_rule = <<POLICY_RULE
    {
      "if": {
        "allOf": [
          {
            "field": "tags['CostCenter']",
            "exists": "false"
          }
        ]
      },
      "then": {
        "effect": "deny"
      }
    }
POLICY_RULE
}
```

## Mejores Prácticas

1. **Implementa Tags Consistentes:**
   - CostCenter
   - Environment
   - Project
   - Owner

2. **Configura Alertas Tempranas:**
   
   El siguiente bloque crea un grupo de acción en Azure para enviar alertas por correo electrónico cuando se detecten gastos elevados. Así, tu equipo recibirá notificaciones automáticas y podrá reaccionar rápidamente si se supera algún umbral de costos.
   
   ```hcl
   resource "azurerm_monitor_action_group" "cost_alert" {
     name                = "cost-alert-team"
     resource_group_name = azurerm_resource_group.monitoring.name
     short_name         = "costalert"

     email_receiver {
       name          = "devops-team"
       email_address = "devops@tuempresa.com"
     }
   }
   ```

3. **Automatiza el Apagado de Recursos:**
   * Configura horarios de apagado/encendido para recursos no productivos
   * Implementa políticas de auto-scaling basadas en uso
   * Utiliza Azure Automation para gestionar recursos inactivos

## Ejemplo Práctico: Dashboard de Costos


Este bloque crea un dashboard personalizado en el portal de Azure para visualizar y analizar los costos de tu suscripción. Así puedes ver de forma gráfica y centralizada cómo se distribuyen los gastos y tomar mejores decisiones.

```hcl
resource "azurerm_portal_dashboard" "cost_dashboard" {
  name                = "cost-management-dashboard"
  resource_group_name = azurerm_resource_group.monitoring.name
  location            = azurerm_resource_group.monitoring.location

  dashboard_properties = <<DASHBOARD
{
  "lenses": {
    "0": {
      "order": 0,
      "parts": {
        "0": {
          "position": {
            "x": 0,
            "y": 0,
            "rowSpan": 2,
            "colSpan": 3
          },
          "metadata": {
            "inputs": [],
            "type": "Extension/Microsoft_Azure_CostManagement/PartType/CostAnalysisPinPart"
          }
        }
      }
    }
  }
}
DASHBOARD
}
```

## Referencias y Recursos Adicionales

- [Documentación oficial de Azure Cost Management](https://docs.microsoft.com/en-us/azure/cost-management-billing/)
- [Terraform Azure Provider - Cost Management](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/consumption_budget_subscription)
- [Azure Well-Architected Framework - Cost Optimization](https://docs.microsoft.com/en-us/azure/architecture/framework/cost/)
- [GitHub - Ejemplos de Terraform para Cost Management](https://github.com/Azure/terraform-azurerm-examples)
