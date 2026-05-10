---
title: "Landing Zone Híbrida: Gestiona tu infraestructura on-premises y cloud desde un único lugar con Azure Arc"
image: "/assets/images/hybrid-landing-zone.png"
categories:
  - blog
tags:
  - Azure
  - Landing Zone
  - Azure Arc
  - Terraform
  - Híbrido
---

Cuando hablamos de Landing Zones, tendemos a imaginar un entorno 100% en la nube: suscripciones perfectamente organizadas, políticas aplicadas, redes bien segmentadas. El problema es que la realidad de la mayoría de empresas es bastante más complicada.

Servidores físicos en el centro de datos. Máquinas virtuales en VMware que nadie se ha atrevido a migrar. Algún workload en AWS porque "así lo dejó el equipo anterior". Y en medio de todo eso, tú, intentando aplicar gobernanza de forma coherente.

Para eso existe la **Landing Zone Híbrida** y, concretamente, **Azure Arc**.

## ¿Qué diferencia una Landing Zone Híbrida de una estándar?

Una Landing Zone convencional define el "suelo firme" sobre el que corren tus cargas de trabajo en Azure: seguridad, redes, identidades, políticas. Una Landing Zone Híbrida hace exactamente lo mismo, pero **extiende ese perímetro más allá de Azure**. Incorpora tus recursos on-premises, otros clouds y ubicaciones remotas al mismo modelo de gobernanza.

El objetivo es concreto: que tu equipo de operaciones tenga **una sola consola, un solo modelo de políticas y una sola fuente de verdad**, independientemente de dónde esté corriendo cada carga de trabajo.

## Azure Arc: el puente entre mundos

Azure Arc es el servicio que hace posible esto. Funciona instalando un agente ligero (`azcmagent`) en tus máquinas —servidores físicos Linux o Windows, clusters de Kubernetes, o instancias en AWS o GCP. Una vez instalado, ese recurso **aparece en Azure como si fuera nativo**, con su propio Resource ID, gestionable con las mismas herramientas de siempre.

La clave: Arc no mueve tus cargas de trabajo a Azure. Las registra en el plano de control de Azure sin moverlas físicamente. La carga sigue corriendo donde estaba, pero tú la ves y la gestionas desde Azure Portal, Azure CLI o tus pipelines de Terraform.

## Arquitectura de referencia

Una Landing Zone Híbrida bien diseñada añade estos componentes a tu diseño habitual:

**1. Suscripción de Management centralizada**
Aquí viven los recursos comunes: Log Analytics Workspace, Microsoft Defender for Cloud, Azure Policy y el hub de red. Exactamente igual que en una LZ cloud-native, sin cambios.

**2. Resource Group de onboarding Arc**
Cada servidor o cluster externo que incorporas crea un recurso de tipo `Microsoft.HybridCompute/machines` o `Microsoft.Kubernetes/connectedClusters`. Desde aquí, las políticas actúan sobre ellos igual que sobre recursos nativos de Azure.

**3. Políticas unificadas**
Puedes asignar iniciativas de política que se apliquen **tanto a tus VMs de Azure como a tus servidores físicos del datacenter**. Por ejemplo: "todos los servidores deben tener el agente de Defender instalado" o "todos los clusters deben tener Flux configurado para GitOps".

## Onboarding de un servidor con Arc

En la práctica, el onboarding se hace con scripts de Azure CLI. Aquí el flujo básico para un servidor Linux:

```bash
# 1. Descargar e instalar el agente de Azure Arc
wget https://aka.ms/azcmagent -O ~/install_linux_azcmagent.sh
bash ~/install_linux_azcmagent.sh

# 2. Conectar el servidor a tu suscripción
azcmagent connect \
  --resource-group "rg-arc-hybrid" \
  --tenant-id "<tu-tenant-id>" \
  --location "westeurope" \
  --subscription-id "<tu-subscription-id>"
```

Una vez conectado, desde Terraform puedes aplicar políticas sobre esos recursos con el mismo módulo que usas para el resto de tu Landing Zone:

```hcl
resource "azurerm_policy_assignment" "arc_defender" {
  name                 = "arc-defender-servers"
  scope                = azurerm_resource_group.arc.id
  policy_definition_id = "/providers/Microsoft.Authorization/policyDefinitions/69bf4abd-ca1e-4cf6-8b5a-762d42e61d4f"
  display_name         = "Instalar Defender en servidores Arc"
}
```

## ¿Cuándo tiene sentido adoptar este modelo?

No todo el mundo necesita una Landing Zone Híbrida. Pero si te reconoces en alguno de estos escenarios, es el momento de planteártela:

- Tienes **regulaciones** que te obligan a mantener ciertos datos on-premises (banca, sanidad, administración pública).
- Tienes cargas de trabajo en el **edge** —fábricas, tiendas, hospitales— que no pueden depender de conectividad estable.
- Estás en **mitad de una migración** y necesitas gobernar recursos legacy y cloud al mismo tiempo.
- Tienes presencia en **múltiples clouds** y quieres un modelo de seguridad coherente sin herramientas distintas para cada uno.

## Conclusión

Una Landing Zone Híbrida con Azure Arc no es simplemente "instalar un agente en unos servidores". Es un cambio de modelo operativo: pasar de gestionar infraestructura por ubicación física a gestionarla por política y propósito.

Tus servidores del datacenter, tus VMs en Azure y tus clusters en AWS pasan a estar bajo el mismo paraguas de gobernanza. El resultado es menos caos operativo, más visibilidad y un equipo que puede aplicar las mismas reglas de seguridad en todos lados, sin excepciones.
