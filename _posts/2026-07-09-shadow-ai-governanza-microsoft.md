---
layout: post
title: "El nuevo Shadow IT ya no son apps: son agentes IA"
image: "/assets/images/shadow-ai-governance.png"
categories:
  - blog
tags:
  - Azure
  - IA
  - Seguridad
  - Gobernanza
  - Microsoft
  - Copilot
---

Hace diez años, el mayor dolor de cabeza de cualquier equipo de seguridad era el **Shadow IT**: empleados instalando Dropbox, usando WhatsApp para compartir documentos o conectando SaaS no autorizados a datos corporativos. Las empresas respondieron con firewalls, proxies y políticas de uso aceptable.

Problema resuelto, ¿verdad? No exactamente. Porque el Shadow IT ha mutado.

El nuevo problema no son aplicaciones no autorizadas. Son **agentes de IA autónomos** que tus empleados están instalando, configurando y ejecutando ahora mismo en sus portátiles corporativos — sin que IT lo sepa, sin que Seguridad lo haya revisado, y con acceso a datos que quizás no deberían tocar.

Bienvenido al **Shadow AI**.

## ¿Qué es el Shadow AI y por qué es diferente?

El Shadow IT clásico era pasivo: un empleado usaba una app no autorizada para almacenar o compartir información. Malo, pero limitado en alcance.

El Shadow AI es activo y autónomo. Un desarrollador instala **Claude Code** o **GitHub Copilot CLI** en su máquina. Un analista configura un agente de automatización que lee correos, accede a SharePoint y envía resúmenes. Un directivo conecta un workflow de IA a su CRM. Ninguno de estos agentes ha pasado por revisión de seguridad. Ninguno tiene una identidad gestionada. Ninguno aparece en tu inventario de activos.

Y lo más importante: estos agentes **actúan**. No solo leen datos, los procesan, los envían y toman decisiones basadas en ellos.

Las implicaciones son distintas a las del Shadow IT tradicional:

- **Fugas de datos silenciosas**: un agente que resume reuniones puede estar enviando esos resúmenes a servidores externos sin que nadie lo detecte.
- **Privilegios heredados**: el agente hereda los permisos del usuario que lo instaló. Si ese usuario tiene acceso a datos financieros, el agente también lo tiene.
- **Ausencia de auditoría**: sin inventario, no hay logs, no hay trazabilidad, no hay posibilidad de investigar incidentes.
- **Cumplimiento imposible**: GDPR, ENS, ISO 27001 exigen saber qué sistemas procesan qué datos. Con agentes no inventariados, eso es imposible.

## Microsoft ha movido ficha: Agent 365 y el ecosistema de gobernanza

La respuesta de Microsoft a este problema ha llegado en 2025-2026 con una arquitectura de gobernanza que cubre toda la superficie de riesgo del Shadow AI. Estas son las piezas clave:

### 1. Agent 365: el registro central de agentes

**Agent 365** es el nuevo plano de control unificado para agentes de IA en el ecosistema Microsoft. Su función principal es crear y mantener un **inventario completo** de todos los agentes que operan en tu organización, independientemente de dónde vengan:

- Agentes construidos en **Microsoft Copilot Studio**
- Agentes desplegados en **AWS Bedrock** o **Google Cloud**
- Agentes instalados localmente en dispositivos de empleados

Desde Agent 365, los equipos de IT y seguridad pueden ver qué hace cada agente, con qué datos interactúa y, cuando es necesario, detenerlo o eliminarlo directamente desde la consola.

### 2. Microsoft Defender for Cloud Apps: descubrimiento de IA no autorizada

Si ya usas **Defender for Cloud Apps** (antes conocido como MCAS), ahora tiene una categoría dedicada de "Generative AI" en su catálogo de aplicaciones en la nube. Esto permite:

- Descubrir qué herramientas de IA está usando tu organización (ChatGPT, Claude, Perplexity, herramientas de código...)
- Ver una **puntuación de riesgo** para cada herramienta basada en sus políticas de privacidad, cifrado y retención de datos
- Marcar herramientas como "sancionadas" (aprobadas) o "no sancionadas" (bloqueadas)
- Crear políticas automáticas que bloqueen el acceso a herramientas de alto riesgo

### 3. Microsoft Purview DSPM for AI: visibilidad sobre datos en prompts

Quizás la pieza más potente del ecosistema. **Data Security Posture Management for AI** dentro de Microsoft Purview permite a los administradores ver, en tiempo real, qué tipo de datos están fluyendo a través de los prompts de los usuarios y las respuestas de los agentes.

¿Un empleado está pegando contratos en ChatGPT? ¿Un agente está procesando datos de clientes? Purview lo detecta, lo registra y puede aplicar políticas DLP para bloquearlo antes de que llegue al servicio externo.

### 4. Intune: bloqueo en el endpoint

Cuando un agente no autorizado es identificado (por ejemplo, un asistente de código local que hace llamadas a APIs externas no aprobadas), **Microsoft Intune** permite propagar políticas de bloqueo directamente a los dispositivos gestionados. El agente queda desactivado en el endpoint, sin intervención manual en cada máquina.

### 5. Microsoft Entra: control sobre el tráfico de agentes

Las nuevas capacidades de **Microsoft Entra** extienden los controles de red a las conexiones que generan los agentes. Esto significa que puedes inspeccionar el tráfico que produce un agente, bloquear conexiones a destinos no aprobados y detectar transferencias de ficheros sospechosas — todo dentro del mismo flujo de trabajo que ya usas para gestionar identidades humanas.

## El modelo de cuatro pasos para gobernar el Shadow AI

Microsoft recomienda una aproximación en fases que es directamente implementable en cualquier organización:

**Fase 1 — Descubrir**
Antes de bloquear nada, necesitas saber qué hay. Activa Defender for Cloud Apps y el DSPM de Purview para generar un inventario de herramientas de IA en uso. Incluyendo las que nadie sabía que existían.

**Fase 2 — Evaluar y sancionar**
Con el inventario en mano, clasifica cada herramienta. ¿Qué riesgo tiene? ¿Qué datos puede tocar? ¿Tiene certificaciones de seguridad? Las herramientas de bajo riesgo se aprueban formalmente; las de alto riesgo se bloquean.

**Fase 3 — Proteger los datos**
Implementa políticas DLP en Microsoft Purview para evitar que datos clasificados (contratos, datos personales, información financiera) lleguen a servicios de IA no aprobados. Esto funciona en tiempo real, incluso dentro del navegador con Microsoft Edge for Business.

**Fase 4 — Auditar y mantener**
El Shadow AI es dinámico: cada semana aparecen nuevas herramientas. Establece revisiones periódicas del inventario y usa las capacidades de auditoría de Purview para mantener trazabilidad completa de qué agente accedió a qué dato y cuándo.

## ¿Y los agentes locales? El reto de los desarrolladores

El caso más complicado es el de los **agentes locales en máquinas de desarrolladores**: herramientas como Claude Code, Continue, o scripts personalizados que llaman a APIs de LLMs directamente desde el portátil.

Estos agentes no pasan por el proxy corporativo, no generan logs en los sistemas de seguridad y son funcionalmente invisibles para los controles tradicionales.

La respuesta de Microsoft en este frente está en el **telemetría de endpoints**: Defender for Endpoint puede detectar aplicaciones que realizan llamadas a endpoints de inferencia conocidos (OpenAI, Anthropic, etc.) y reportarlas a Agent 365, aunque el agente no esté desplegado en Azure. Desde ahí, Intune puede tomar acción.

No es perfecto todavía — la cobertura de endpoints de inferencia de terceros sigue evolucionando — pero es el primer paso hacia la visibilidad real en este espacio.

## Conclusión: gobernar la IA como lo que es — infraestructura

El Shadow AI no va a desaparecer. Cada semana hay nuevas herramientas más potentes, más fáciles de instalar y más difíciles de detectar. Prohibir el uso de IA en la empresa no es una opción realista; solo empuja el problema más a las sombras.

La respuesta correcta es la misma que se aplicó al Shadow IT hace una década: **visibilidad, clasificación y gobernanza**. Saber qué hay, evaluar el riesgo, aprobar lo que sea seguro, bloquear lo que no y auditar todo.

La diferencia es que ahora las herramientas están diseñadas específicamente para este problema. Microsoft ha construido un ecosistema —Agent 365, Purview, Defender, Intune, Entra— que trata a los agentes de IA como lo que son: **identidades con privilegios que necesitan ser gestionadas**, no aplicaciones que se instalan y se olvidan.

Si gestionas infraestructura en un entorno enterprise, este es el momento de incluir la gobernanza de IA en tu modelo de seguridad. El Shadow AI ya es un problema real. Las herramientas para abordarlo también lo son.
