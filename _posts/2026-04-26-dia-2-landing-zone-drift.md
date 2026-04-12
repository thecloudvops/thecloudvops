---
title: "El \"Día 2\" de una Landing Zone: Cómo gestionar el mantenimiento y el drift"
image: "/assets/images/day-2-landing-zone.png"
categories:
  - blog
tags:
  - Terraform
  - Operaciones
  - Arquitectura Cloud
---

Cualquier consultoría tecnológica puede venderte el despliegue de una Landing Zone impecable. El "Día 1" (el diseño inicial y la ejecución del primer *Terraform Apply*) es siempre bonito, emocionante y suele acabar sin grandes sobresaltos. 

Sin embargo, el verdadero reto para todo equipo CloudOps comienza la mañana siguiente. Bienvenidos al **"Día 2"**.

## ¿A qué nos enfrentamos el Día 2?

La infraestructura Cloud no es estática. Crece, evoluciona e, inevitablemente, empeora (la amada entropía). Los dos grandes monstruos a combatir son:

1. El *Drift* (la deriva de configuración).
2. El mantenimiento y actualización de módulos.

## 1. Detectar y mitigar el Drift

El Configuration Drift ocurre cuando la realidad (lo que hay desplegado en Azure o AWS) difiere del código de Terraform (lo que dice el *State* o tu repositorio). A menudo sucede porque alguien entró de madrugada al portal a arreglar algo de emergencia "a mano" o por integraciones de terceros.

**Estrategias para sobrevivir:**
- **Cron Jobs de Detección:** Configura tus pipelines de CI/CD para ejecutar un `terraform plan` periódicamente (por ejemplo, todas las noches). Si este *plan* arroja cambios (`Plan: 1 to add, 1 to change...`), envía automáticamente una notificación a Slack/Teams informando del *drift*.
- **Cultura de inmutabilidad:** El drift persistente indica un problema cultural. Si está permitido tocar el portal manualmente, el IaC pierde su propósito. La regla debe ser: los accesos de escritura directo desde el portal a producción se restringen, todo pasa por *Pull Requests*.
- **ClickOps de Emergencia:** Si hay un *hotfix* de madrugada, el equipo debe tener como tarea N°1 del día siguiente reconciliar ese cambio manual, ya sea importándolo al state o metiendo el código en el repositorio.

## 2. Renovación: Actualizaciones de Módulos (AVM, etc.)

Cuando dependes de módulos públicos o comunitarios como los Azure Verified Modules (AVM) o los módulos de AWS de Anton Babenko, verás que evolucionan cada pocas semanas. Nuevos *features*, corrección de bugs, y también... *breaking changes*.

**Cómo mantener la paz:**
- Pinea (fija) SIEMPRE la versión del módulo (`version = "2.1.0"`). Nunca uses branch de `main` u omites la versión.
- Crea entornos pre-productivos o "sandboxes" exactos donde puedas probar los saltos de "Major Version" antes de arriesgarte en producción. 
- Utiliza la herramienta del ecosistema como `Dependabot` o `Renovate` para que los pipelines te abran *Pull Requests* automatizadas siempre que salga una actualización menor, liberándote de revisar los *Releases* en GitHub todas las mañanas.

### Conclusión

Desplegar infraestructura es una habilidad. Mantener un despliegue de la Landing Zone limpio, sin drift y al mismo ritmo que las actualizaciones de los proveedores Cloud es un **arte** que diferencia a un buen administrador Cloud de un verdadero Arquitecto CloudOps.
