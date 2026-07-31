---
layout: post
title: "Hablamos de Azure en la comunidad: Landing Zones sin caos con Terraform"
date: 2026-07-31
categories: [Azure, Comunidad, Terraform, Gobernanza]
tags: [azure, landing-zones, terraform, gobernanza, msh, comunidad, iac]
image: "/assets/images/msh-charla-azure-sin-caos.png"
excerpt: "La semana pasada tuve el placer de dar una charla junto a Gabriel Naranjo Lituma en el Microsoft Speakers Hub en Español sobre cómo construir Landing Zones en Azure con Terraform. Te cuento cómo fue y qué vimos."
---

La semana pasada tuve el placer de participar como speaker en una sesión organizada por el **Microsoft Speakers Hub en Español (MSH)**, la comunidad de referencia para la comunidad técnica hispanohablante de Microsoft. Y el tema no podía ser más cercano a lo que trabajamos aquí en The Cloud Vops: **Azure sin caos**.

Junto a [**Gabriel Naranjo Lituma**](https://es.linkedin.com/in/gnl), estuvimos desgranando durante más de una hora cómo se construyen **Landing Zones en Azure** con Terraform, por qué la gobernanza es lo primero que se debe diseñar —no lo último— y cómo evitar que tu nube acabe pareciendo el cajón de los cables.

👉 Puedes ver el post oficial de la sesión en LinkedIn aquí: [**Azure sin caos: Construyendo Landing Zones — Microsoft Speakers Hub en Español**](https://www.linkedin.com/posts/msft-speakers-hub-en-espanol_mshespaaehol-azure-comunidadtecnica-activity-7487885685811712004-P5iA)

---

## ¿De qué hablamos?

El eje central de la sesión fue demostrar que la gobernanza en Azure no es burocracia ni fricción. Es exactamente lo contrario: es la infraestructura invisible que permite a los equipos moverse rápido de forma segura y sostenible.

Estas fueron las tres grandes ideas que vertebraron la charla:

### 1. La Landing Zone no es un producto, es una decisión de arquitectura

Muchos equipos buscan "instalar" una Landing Zone como si fuera un software. El problema es que no existe esa caja mágica. Una Landing Zone es el resultado de una serie de decisiones de diseño: ¿cómo organizo mis suscripciones? ¿Qué políticas son no negociables? ¿Qué equipos tienen autonomía y sobre qué recursos?

La jerarquía de **Management Groups** del Cloud Adoption Framework (CAF) es el punto de partida, no el destino. Adaptarla a tu organización es el trabajo real.

```
Tenant Root Group
└── Organización (Intermediate Root)
    ├── Platform
    │   ├── Identity
    │   ├── Management
    │   └── Connectivity
    ├── Landing Zones
    │   ├── Corp        → workloads conectados a red corporativa
    │   └── Online      → workloads públicos
    ├── Sandbox         → entornos de prueba
    └── Decommissioned
```

### 2. La gobernanza va de código, no de clics

El portal de Azure es una trampa para aprender, pero un desastre para gobernar. Cuando configuras tus políticas, RBAC y estructuras de red haciendo clics, estás creando **snowflakes** de infraestructura: entornos irrepetibles que nadie sabe reproducir ni auditar.

Con Terraform, en cambio, cada política, cada asignación de rol y cada regla de red vive en un fichero `.tf`, en un repositorio, con su historial de cambios y su proceso de revisión. Cuando alguien pregunta "¿por qué no puedo desplegar en esta región?", la respuesta está en el código, no en la cabeza de alguien.

### 3. Las excepciones también son código

Este fue uno de los puntos que más resonó con el público. Toda organización tiene excepciones: el proyecto legacy que no puede cumplir la política de red, el entorno de pruebas que necesita una región especial. El error clásico es aprobarlas con clics en el portal, sin dejar rastro.

`azurerm_subscription_policy_exemption` te permite documentar cada excepción con su justificación, su ticket de aprobación y —lo más importante— su **fecha de caducidad**. La excepción existe hasta cuando tiene sentido, y punto.

---

## El repositorio de demostración

Durante la sesión usamos código real. Si quieres seguir los pasos en tu propio entorno de Azure, puedes clonar el repositorio de demostración que acompaña a este y otros posts del blog:

👉 [**GitHub — MSH-Azure-sin-caos**](https://github.com/AZAlberto/MSH-Azure-sin-caos)

Encontrarás ejemplos de:
- Definición de la jerarquía de Management Groups con Terraform
- Políticas custom (etiquetado obligatorio, regiones permitidas)
- Iniciativas (Policy Sets) que agrupan políticas relacionadas
- Asignaciones a nivel de Management Group con identidades gestionadas para remediación
- Gestión de excepciones con caducidad automática

---

## Por qué me importa la comunidad

Estas sesiones tienen un valor que va más allá de la charla en sí. La comunidad técnica hispanohablante de Azure es vibrante y está creciendo, pero aún hay una brecha enorme entre lo que se documenta en inglés y los recursos prácticos disponibles en español.

El **Microsoft Speakers Hub en Español** es una de las iniciativas que están cerrando esa brecha. Si quieres seguir su actividad, puedes hacerlo en su [perfil de LinkedIn](https://es.linkedin.com/company/msft-speakers-hub-en-espanol).

Y si quieres profundizar en los temas que vimos, en este blog tienes una serie completa de artículos técnicos sobre cada pieza de la Landing Zone:

- 📖 [**CAF Landing Zone: cómo definir y asignar políticas con Terraform**](/2026/05/17/caf-landing-zone-policies-terraform)
- 📖 [**Gestión de excepciones de política como código**](/2026/05/24/policy-exceptions-as-code-azure-terraform)
- 📖 [**RBAC como código en Azure con Terraform**](/2026/06/07/rbac-como-codigo-azure-terraform)
- 📖 [**Detección de drift con Terraform y GitHub Actions**](/2026/06/14/drift-detection-azure-terraform-github-actions)

---

Gracias a Gabriel por hacer la sesión mucho más dinámica, y gracias a todos los que os conectasteis en directo y dejasteis vuestras preguntas. Ese feedback es el que da sentido a seguir compartiendo.

Si tienes alguna pregunta sobre lo que vimos en la charla, déjala en los comentarios o escríbeme directamente. ¡Nos leemos!
