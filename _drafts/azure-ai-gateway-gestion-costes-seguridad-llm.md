---
layout: post
title: "Azure AI Gateway: Cómo centralizar, auditar y controlar costes de LLMs en la empresa"
date: 2026-11-27
categories:
  - blog
tags:
  - Azure
  - IA
  - FinOps
  - APIM
  - Seguridad
  - Arquitectura
  - OpenAI
image: "/assets/images/azure-ai-gateway-cover.png"
excerpt: "Aprende a diseñar e implementar un patrón de AI Gateway con Azure API Management: balanceo multi-región para evitar errores 429, límites de cuota por tokens (TPM), caché semántica y gobernanza de costes."
---

![Azure AI Gateway](/assets/images/azure-ai-gateway-cover.png)

En los últimos dos años, casi todas las organizaciones han pasado de *"vamos a experimentar con IA"* a *"tenemos 14 equipos consumiendo modelos en producción"*. 

Sin embargo, cuando la adopción de Inteligencia Artificial se deja crecer de forma orgánica, los departamentos de Cloud y DevOps se encuentran rápidamente con una pesadilla operativa bautizada como **Shadow AI**:

- **Descontrol de costes y nula atribución:** Llega una factura de miles de euros por consumo de tokens en Azure OpenAI y nadie sabe qué equipo o microservicio es el responsable.
- **Límites de cuota (Errores HTTP 429):** Cada región de Azure impone límites de Tokens por Minuto (TPM) y Requests por Minuto (RPM). Un batch de un equipo bloquea a los demás.
- **Falta de seguridad y fuga de datos:** Claves de API compartidas en ficheros `.env`, llamadas directas a modelos externos sin filtrado de datos personales (PII) y cero auditoría de prompts.

Para solucionar este caos existe un patrón arquitectónico fundamental: el **AI Gateway**. 

En este artículo vamos a ver cómo diseñar e implementar una pasarela centralizada de IA empresarial utilizando **Azure API Management (APIM)**, con balanceo multi-región, control de cuotas por tokens y observabilidad FinOps.

---

## Qué es un AI Gateway y qué problemas resuelve

Un **AI Gateway** es una capa intermedia (un reverse proxy inteligente) situada entre tus aplicaciones internas y los proveedores de modelos fundacionales (Azure OpenAI, Anthropic, Mistral o modelos self-hosted).

En lugar de que cada aplicación hable directamente con una instancia de OpenAI, todas las peticiones pasan obligatoriamente por el Gateway:

```
[Aplicaciones Internas / Microservicios]
                   │
                   │ (Autenticación via Entra ID / API Key)
                   ▼
┌────────────────────────────────────────────────────────┐
│           AZURE API MANAGEMENT (AI GATEWAY)            │
│  ├── 1. Control de cuotas por tokens (TPM/RPM)         │
│  ├── 2. Caché Semántica (Azure Cache for Redis)        │
│  ├── 3. Circuit Breaker & Balanceo Multi-Región        │
│  └── 4. Filtro de PII y Seguridad (Content Safety)    │
└────────────────────────────────────────────────────────┘
         │                   │                   │
         ▼                   ▼                   ▼
[Azure OpenAI           [Azure OpenAI       [Azure OpenAI
 Sweden Central]        West Europe]         East US 2]
```

### Ventajas clave:
1. **Alta disponibilidad real (Zero 429s):** Si la región de *Sweden Central* devuelve un error `429 Too Many Requests`, el Gateway redirige la petición en milisegundos a *West Europe* o *East US 2* de forma transparente para el cliente.
2. **FinOps y Atribución:** Cada departamento o proyecto tiene su propia *Subscription Key* o App Registration. Puedes limitar el presupuesto mensual en tokens y saber al céntimo quién gasta qué.
3. **Caché Semántica:** Si dos usuarios hacen preguntas idénticas o semánticamente similares, el Gateway devuelve la respuesta desde una caché rápida (Redis) sin consumir tokens del modelo.

---

## Implementación: Configurando APIM como AI Gateway

Azure API Management incluye políticas nativas diseñadas específicamente para Azure OpenAI y modelos de lenguaje.

### 1. Balanceo de carga y Circuit Breaker con Terraform

Primero, definimos los backends de Azure OpenAI en distintas regiones y un backend pool en APIM mediante Terraform:

```hcl
# Backends de Azure OpenAI en distintas regiones
resource "azurerm_api_management_backend" "openai_sweden" {
  name                = "openai-sweden"
  resource_group_name = var.resource_group_name
  api_management_name = var.apim_name
  protocol            = "http"
  url                 = "https://aoai-sweden.openai.azure.com/openai"

  circuit_breaker {
    rules {
      failure_condition {
        count                = 3
        error_reasons        = ["Server errors", "Timeout"]
        interval             = "PT1M"
        status_code_ranges {
          min = 429
          max = 503
        }
      }
      trip_duration = "PT1M"
    }
  }
}

resource "azurerm_api_management_backend" "openai_westeurope" {
  name                = "openai-westeurope"
  resource_group_name = var.resource_group_name
  api_management_name = var.apim_name
  protocol            = "http"
  url                 = "https://aoai-westeurope.openai.azure.com/openai"

  circuit_breaker {
    rules {
      failure_condition {
        count         = 3
        interval      = "PT1M"
        status_code_ranges {
          min = 429
          max = 503
        }
      }
      trip_duration = "PT1M"
    }
  }
}

# Pool de balanceo con prioridad y circuit breaker
resource "azurerm_api_management_backend" "openai_pool" {
  name                = "openai-backend-pool"
  resource_group_name = var.resource_group_name
  api_management_name = var.apim_name
  protocol            = "http"
  type                = "Pool"

  pool {
    services {
      id       = azurerm_api_management_backend.openai_sweden.id
      priority = 1
      weight   = 70
    }
    services {
      id       = azurerm_api_management_backend.openai_westeurope.id
      priority = 2
      weight   = 30
    }
  }
}
```

---

### 2. Políticas de APIM: Control de Cuotas por Tokens (TPM)

En el fichero de políticas XML de la API (`policies.xml`), aplicamos las directivas nativas para proteger el servicio y registrar la telemetría:

```xml
<policies>
    <inbound>
        <base />
        <!-- 1. Enrutar al pool balanceado con circuit breaker -->
        <set-backend-service backend-id="openai-backend-pool" />
        
        <!-- 2. Autenticación con Managed Identity hacia Azure OpenAI (sin claves compartidas) -->
        <authentication-managed-identity resource="https://cognitiveservices.azure.com" />
        
        <!-- 3. Limitar cuota por Tokens por Minuto (TPM) según la suscripción del equipo -->
        <azure-openai-token-limit 
            counter-key="@(context.Subscription.Id)" 
            tokens-per-minute="100000" 
            estimate-prompt-tokens="true" 
            tokens-consumed-header-name="x-consumed-tokens" 
            remaining-tokens-header-name="x-remaining-tokens" />
    </inbound>

    <backend>
        <!-- Retry automático inteligente si el backend primario falla con 429 -->
        <retry condition="@(context.Response.StatusCode == 429)" count="3" interval="1" max-interval="5" delta="1" first-fast-retry="true">
            <forward-request buffer-request-body="true" />
        </retry>
    </backend>

    <outbound>
        <base />
        <!-- 4. Emitir métricas de consumo de tokens a Application Insights para FinOps -->
        <azure-openai-emit-token-metric
            namespace="AzureOpenAIGateway">
            <dimension name="SubscriptionId" value="@(context.Subscription.Id)" />
            <dimension name="ClientApp" value="@(context.Request.Headers.GetValueOrDefault("x-client-app", "anonymous"))" />
            <dimension name="DeploymentId" value="@(context.Request.MatchedParameters["deployment-id"])" />
        </azure-openai-emit-token-metric>
    </outbound>

    <on-error>
        <base />
    </on-error>
</policies>
```

---

## FinOps en acción: Atribución de costes con KQL

Gracias a la política `azure-openai-emit-token-metric`, todos los tokens consumidos (prompt tokens + completion tokens) viajan directamente a nuestro **Log Analytics Workspace**.

Con una simple consulta en Kusto (KQL), el equipo de FinOps y DevOps puede ver en tiempo real el reparto del gasto por departamento:

```kql
// Consumo de tokens agrupado por aplicación y suscripción en los últimos 7 días
customMetrics
| where name == "TotalTokens"
| extend ClientApp = tostring(customDimensions["ClientApp"]),
         Model = tostring(customDimensions["DeploymentId"]),
         Tokens = value
| summarize 
    TotalTokens = sum(Tokens),
    PromptTokens = sum(toint(customDimensions["PromptTokens"])),
    CompletionTokens = sum(toint(customDimensions["CompletionTokens"]))
    by ClientApp, Model
| extend CosteAproxUSD = round((TotalTokens / 1000000.0) * 5.0, 2) // Estimación según precio por 1M tokens
| order by TotalTokens desc
```

Esto permite generar dashboards interactivos en Azure Workbooks donde cada Product Owner ve el gasto de su aplicación en tiempo real, eliminando las sorpresas en la factura mensual.

---

## Seguridad y Gobernanza de Datos

El AI Gateway también actúa como un cortafuegos para cumplir normativas como la **EU AI Act** o regulaciones de protección de datos:

1. **Eliminación de API Keys en clientes:** Las aplicaciones consumidoras solo se autentican contra APIM usando tokens JWT de Azure Entra ID o una clave de suscripción interna. Las credenciales de Azure OpenAI quedan aisladas en el Gateway mediante **Managed Identities**.
2. **Inspección de PII (Personally Identifiable Information):** Puedes integrar una política que invoque **Azure AI Content Safety** para redactar números de tarjetas de crédito, DNI o datos médicos antes de que el texto llegue al modelo fundacional.
3. **Auditoría completa:** Cada petición queda registrada con timestamp, identidad del llamante, tamaño de la petición y latencia, permitiendo trazar cualquier anomalía.

---

## Conclusión

Desplegar modelos de IA sin una pasarela de gobernanza es el equivalente a exponer bases de datos directamente a Internet sin firewall. 

Implementar un **AI Gateway con Azure API Management** te permite:
- Evitar caídas y errores 429 gracias al balanceo multi-región.
- Proteger el presupuesto cloud con cuotas estrictas de tokens.
- Auditar y securizar el consumo de IA bajo estándares corporativos.

Es la pieza que transforma un experimento desordenado en una plataforma de IA robusta, escalable y lista para producción.

¿Cómo gestionáis actualmente las llamadas a modelos de IA en vuestra infraestructura? ¿Habéis sufrido los temidos errores 429? ¡Contádmelo en los comentarios!

