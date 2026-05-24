---
layout: post
title: "Policy Exceptions as Code en Azure Landing Zones con Terraform"
image: "/assets/images/policy-exceptions-cover.png"
categories:
  - blog
tags:
  - Azure
  - Landing Zone
  - Terraform
  - Azure Policy
  - Gobernanza
  - CAF
---

Si tienes una Landing Zone bien construida, tarde o temprano llegarás al mismo momento que todos: alguien en tu organización necesita una excepción a una política. Un proyecto legacy que no puede cumplir las restricciones de red. Un equipo de I+D que necesita desplegar en una región no permitida. Un workload que requiere un SKU que está bloqueado.

El error habitual es gestionar esa excepción haciendo clic en el portal de Azure. Se aprueba, se olvida, y seis meses después nadie sabe por qué ese recurso está fuera de la gobernanza corporativa. El estado de compliance en Azure Policy se convierte en un misterio.

En este artículo vamos a ver cómo gestionar las excepciones de políticas como código, con Terraform, de forma que sean trazables, auditables y, sobre todo, temporales.

## Por qué las excepciones son inevitables

Antes de hablar de cómo gestionarlas, vale la pena entender por qué existen. Una Landing Zone bien diseñada aplica políticas estrictas por defecto. Pero la realidad operativa siempre genera casos que se escapan del modelo:

- **Proyectos legacy:** Sistemas existentes que no pueden adaptarse a las nuevas políticas de red o identidad sin un trabajo de refactorización largo.
- **Entornos de investigación:** Equipos que necesitan libertad para experimentar con tecnologías fuera del catálogo aprobado.
- **Migraciones en curso:** Durante una migración, hay un período de transición donde el recurso no puede cumplir la política hasta que el proceso termina.
- **Requisitos de terceros:** Soluciones SaaS o de partners que imponen restricciones técnicas incompatibles con tus políticas.

La solución no es quitar la política. La solución es **gestionar la excepción de forma controlada**.

## Cómo funcionan las exemptions en Azure Policy

Azure Policy tiene un mecanismo nativo para esto: las **exemptions**. Una exemption es un objeto que indica a Azure Policy que ignore (exima) un scope determinado de una asignación de política concreta.

Las exemptions tienen dos categorías:

| Categoría | Cuándo usarla |
|---|---|
| **Waiver** | El recurso no cumple la política y se acepta el riesgo de forma explícita. |
| **Mitigated** | El recurso no cumple la política técnicamente, pero hay un control compensatorio equivalente. |

La diferencia es importante porque queda registrada en el compliance report de Azure Policy. Un `Waiver` dice "sabemos que no cumple y lo aceptamos". Un `Mitigated` dice "no cumple el control técnico pero sí el objetivo de seguridad por otro camino".

## La estructura de ficheros para gestionar excepciones

Siguiendo la estructura de módulos del artículo anterior, añadimos una carpeta `exemptions`:

```
infra/
├── management_groups/
├── policies/
│   ├── definitions/
│   ├── initiatives/
│   ├── assignments/
│   └── exemptions/              ← nuevo
│       ├── mg_exemptions.tf     → excepciones a nivel de MG
│       ├── sub_exemptions.tf    → excepciones a nivel de suscripción
│       └── variables.tf
└── providers.tf
```

Separar las excepciones en su propia carpeta tiene una ventaja clave: un `grep` o una búsqueda en el repositorio te da de inmediato una lista completa de todas las excepciones activas. No hay que buscar en el portal.

## Exemptions a nivel de Management Group

Cuando una excepción aplica a todas las suscripciones bajo un MG concreto, la gestionamos con `azurerm_management_group_policy_exemption`:

```hcl
# policies/exemptions/mg_exemptions.tf

resource "azurerm_management_group_policy_exemption" "sandbox_location_waiver" {
  name                 = "sandbox-location-waiver"
  management_group_id  = azurerm_management_group.sandbox.id
  policy_assignment_id = azurerm_management_group_policy_assignment.security_baseline.id
  exemption_category   = "Waiver"

  display_name = "[SANDBOX] Exención de política de regiones"
  description  = <<-EOT
    Motivo: El MG Sandbox necesita acceso a todas las regiones para pruebas de concepto.
    Aprobado por: Comité de Arquitectura Cloud
    Ticket: ARCH-1234
    Revisión: Trimestral
  EOT

  expires_on = "2026-09-01T00:00:00Z"

  metadata = jsonencode({
    requestedBy  = "equipo-plataforma"
    approvedBy   = "comite-arquitectura"
    ticket       = "ARCH-1234"
    reviewDate   = "2026-08-01"
  })
}
```

El campo `metadata` es especialmente útil: aunque es libre (un JSON genérico), usarlo con una estructura consistente te permite construir dashboards o scripts que extraigan todas las excepciones y sus metadatos directamente desde la API de Azure.

## Exemptions a nivel de Suscripción

Cuando la excepción es más acotada —afecta solo a una suscripción concreta— usamos `azurerm_subscription_policy_exemption`. Esto es mucho más preciso y menos arriesgado que eximir todo un MG:

```hcl
# policies/exemptions/sub_exemptions.tf

resource "azurerm_subscription_policy_exemption" "legacy_app_network_exemption" {
  name                 = "legacy-app-network-exemption"
  subscription_id      = "/subscriptions/00000000-0000-0000-0000-000000000000"
  policy_assignment_id = azurerm_management_group_policy_assignment.security_baseline.id
  exemption_category   = "Mitigated"

  display_name = "[LEGACY] Exención de política de red para App Contabilidad"
  description  = <<-EOT
    Motivo: La aplicación de contabilidad legacy no puede conectarse via Private Endpoint
    hasta que se complete la migración (Q3 2026). Control compensatorio: acceso restringido
    por NSG con reglas estrictas de IP de origen.
    Aprobado por: CISO
    Ticket: SEC-5521
    Revisión: Mensual hasta fin de migración
  EOT

  expires_on = "2026-10-01T00:00:00Z"

  metadata = jsonencode({
    requestedBy         = "equipo-contabilidad"
    approvedBy          = "CISO"
    ticket              = "SEC-5521"
    mitigatingControl   = "NSG-Contabilidad-Strict con whitelist de IPs corporativas"
    migrationEndDate    = "2026-09-30"
  })
}
```

Fíjate en dos cosas importantes aquí:
1. La categoría es `Mitigated`, no `Waiver`, porque hay un control compensatorio (el NSG estricto).
2. La descripción explica el control compensatorio con detalle. Eso es lo que diferencia una excepción gestionada de una excepción improvisada.

## Variables para evitar hardcodear IDs de suscripción

Un patrón limpio es centralizar los IDs de suscripción en variables para que el código sea reutilizable entre entornos:

```hcl
# policies/exemptions/variables.tf

variable "exemptions" {
  description = "Mapa de excepciones a nivel de suscripción"
  type = map(object({
    subscription_id      = string
    policy_assignment_id = string
    exemption_category   = string
    display_name         = string
    description          = string
    expires_on           = string
    ticket               = string
    approved_by          = string
  }))
  default = {}
}
```

Y en el `tfvars` correspondiente:

```hcl
# environments/prod/exemptions.tfvars

exemptions = {
  "legacy-app-network" = {
    subscription_id      = "00000000-0000-0000-0000-000000000000"
    policy_assignment_id = "/providers/Microsoft.Management/managementGroups/mg-landing-zones/providers/Microsoft.Authorization/policyAssignments/security-baseline"
    exemption_category   = "Mitigated"
    display_name         = "[LEGACY] Exención de política de red para App Contabilidad"
    description          = "Migración en curso. Control compensatorio: NSG-Contabilidad-Strict."
    expires_on           = "2026-10-01T00:00:00Z"
    ticket               = "SEC-5521"
    approved_by          = "CISO"
  }
}
```

Con este patrón, añadir una nueva excepción es solo añadir una entrada al `tfvars` y hacer un PR. El equipo de plataforma revisa el PR, lo aprueba y aplica. **La excepción tiene la misma trazabilidad que cualquier otro cambio de infraestructura.**

## El flujo de aprobación: Pull Requests como proceso de gobernanza

El mayor beneficio de gestionar excepciones como código no es técnico: es organizativo. Un Pull Request para añadir una exemption fuerza automáticamente:

1. **Documentación:** El autor tiene que describir el motivo, el control compensatorio y la fecha de caducidad.
2. **Revisión:** El equipo de seguridad o arquitectura puede revisar y aprobar antes de que se aplique.
3. **Registro histórico:** Git guarda quién aprobó, cuándo y con qué justificación.
4. **Reversibilidad:** Revocar la excepción es un PR que elimina la entrada. También queda registrado.

Este flujo convierte el proceso de gobernanza en algo que el equipo de plataforma **puede auditar** en lugar de intentar reconstruir a partir de logs del portal.

## Detectar excepciones caducadas antes de que sean un problema

Las excepciones con `expires_on` caducan automáticamente en Azure, pero eso no significa que el recurso vuelva a cumplir la política: simplemente deja de estar exento y puede quedar en estado de incumplimiento sin que nadie lo note.

Una buena práctica es añadir un script de comprobación en tu pipeline de CI que avise cuando una excepción esté próxima a caducar:

```bash
#!/bin/bash
# scripts/check_expiring_exemptions.sh

WARNING_DAYS=30
TODAY=$(date -u +%Y-%m-%dT%H:%M:%SZ)

echo "=== Checking expiring policy exemptions ==="

az policy exemption list --query "[].{name:name, expires:expiresOn, scope:id}" -o json | \
  jq --arg today "$TODAY" --argjson days "$WARNING_DAYS" '
    .[] | select(.expires != null) |
    . + {daysLeft: ((.expires | fromdateiso8601) - ($today | fromdateiso8601)) / 86400 | floor} |
    select(.daysLeft <= $days)
  ' | jq -r '"⚠️  EXPIRING SOON: \(.name) — \(.daysLeft) days left (\(.expires))"'
```

Integrado en un pipeline de GitHub Actions que se ejecute semanalmente, este script envía avisos antes de que una excepción caduque sin que nadie esté preparado.

## Conclusión

Gestionar excepciones de política en Azure no tiene por qué ser un agujero negro de gobernanza. Con Terraform y el mecanismo de exemptions de Azure Policy tienes todas las herramientas para hacerlo bien:

- **Temporalidad:** Fecha de caducidad obligatoria en cada excepción.
- **Trazabilidad:** Quién la pidió, quién la aprobó, qué ticket la justifica.
- **Control compensatorio documentado:** La diferencia entre `Waiver` y `Mitigated` no es cosmética.
- **Proceso reproducible:** Un PR que cualquiera puede seguir y auditar.

Una Landing Zone sin excepciones gestionadas no es una Landing Zone sin excepciones: es una Landing Zone con excepciones que nadie sabe que existen. Y eso es mucho más peligroso.
