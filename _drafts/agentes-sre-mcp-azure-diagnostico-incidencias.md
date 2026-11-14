---
layout: post
title: "De scripts reactivos a Agentes SRE: Cómo usar Model Context Protocol (MCP) para diagnosticar incidencias en Azure"
date: 2026-11-20
categories:
  - blog
tags:
  - Azure
  - SRE
  - DevOps
  - MCP
  - IA
  - Observabilidad
  - Python
image: "/assets/images/mcp-sre-azure-cover.png"
excerpt: "Aprende a construir un agente de SRE utilizando Model Context Protocol (MCP) en Python para correlacionar automáticamente alertas de Azure Monitor, consultas KQL y despliegues recientes de CI/CD."
---

![Agente SRE con MCP en Azure](/assets/images/mcp-sre-azure-cover.png)

Son las 3:00 AM y suena PagerDuty. Una alerta crítica de Azure Monitor te despierta: **Tasa de errores 5XX superior al 15% en el cluster de microservicios**.

Cualquier ingeniero de DevOps o SRE conoce el ritual que empieza en ese instante:
1. Abrir Azure Portal y verificar el estado del servicio afectado (Container Apps, AKS o App Service).
2. Ir a Log Analytics Workspace y pelearse con una consulta KQL a contrarreloj para aislar la excepción.
3. Abrir GitHub o Azure DevOps para comprobar si alguien desplegó un cambio en los últimos 30 minutos.
4. Revisar métricas de CPU, memoria y saturación de base de datos.
5. Redactar el primer mensaje de estado para el canal de incidentes en Slack o Teams.

En ese proceso se pierden entre 15 y 25 minutos de **triage puramente mecánico** antes de empezar a mitigar el problema real.

¿Y si tuviéramos un **Agente SRE** al que la alerta invoque en el segundo cero, que ejecute esas mismas comprobaciones en paralelo y te entregue un diagnóstico estructurado con las causas más probables antes de que abras el portátil?

Gracias al **Model Context Protocol (MCP)**, esto ya no es ciencia ficción ni requiere complejas integraciones ad-hoc. Vamos a ver cómo construirlo paso a paso.

---

## El problema de las alertas tradicionales vs. Agentes con MCP

Hasta ahora, la automatización de incidentes se limitaba a webhooks estáticos: Azure Monitor enviaba un JSON a un webhook que posteaba en Slack *"Alerta disparada en WebApp-Prod"*. Pero ese mensaje no tiene contexto: no sabe qué versión se desplegó ni qué excepción exacta se está repitiendo en los logs.

Con **MCP (Model Context Protocol)**, un modelo de lenguaje (como Claude o GPT) puede consultar herramientas externas mediante un protocolo estandarizado y seguro. El agente no "adivina": tiene a su disposición herramientas para interactuar con la API de Azure y recopilar telemetría real.

```
       [Alerta de Azure Monitor / EventGrid]
                        │
                        ▼
            [Agente de Triage SRE (LLM)]
                        │
            (Model Context Protocol - MCP)
                        │
       ┌────────────────┴────────────────┐
       ▼                                 ▼
[Servidor MCP Azure SRE]       [Herramientas de Contexto]
  ├── Logs Query (KQL)           ├── GitHub API (Últimos deploys)
  ├── Resource Metrics           └── StatusPage / Dependencias
  └── Activity Log
```

---

## Arquitectura del Servidor MCP para Azure SRE

Vamos a crear un servidor MCP en Python utilizando el SDK de **FastMCP** y las librerías oficiales del SDK de Azure (`azure-monitor-query` y `azure-identity`).

El servidor expondrá 3 herramientas clave:
1. `consultar_logs_kql`: Ejecuta consultas KQL seguras sobre nuestro Log Analytics Workspace.
2. `verificar_despliegues_recientes`: Consulta la API de GitHub para identificar si hubo despliegues en la última hora.
3. `obtener_metricas_recurso`: Revisa métricas de CPU, memoria y requests para detectar anomalías.

### Estructura del proyecto

```bash
mcp-azure-sre-agent/
├── server.py             # Servidor MCP y registro de tools
├── azure_client.py       # Wrapper de autenticación y consultas Azure
├── github_client.py      # Conexión con GitHub Actions API
└── requirements.txt
```

---

## Paso 1: Implementación del Servidor MCP

En nuestro fichero `requirements.txt` necesitamos:

```text
mcp[cli]>=1.0.0
fastmcp>=0.4.0
azure-identity>=1.15.0
azure-monitor-query>=1.2.0
requests>=2.31.0
```

Veamos la implementación de `server.py`:

```python
"""
Servidor MCP para triage de incidentes SRE en Azure.
Proporciona herramientas de consulta de logs KQL, métricas y despliegues CI/CD.
"""
import os
from datetime import timedelta
from fastmcp import FastMCP
from azure.identity import DefaultAzureCredential
from azure.monitor.query import LogsQueryClient, LogsQueryStatus
import requests

# Inicialización del servidor FastMCP
mcp = FastMCP("Azure-SRE-Triage-Server")

# Credenciales y cliente de Azure
credential = DefaultAzureCredential()
logs_client = LogsQueryClient(credential)

WORKSPACE_ID = os.getenv("AZURE_LOG_ANALYTICS_WORKSPACE_ID")
GITHUB_TOKEN = os.getenv("GITHUB_TOKEN")
GITHUB_REPO = os.getenv("GITHUB_REPOSITORY")  # ej: "mi-org/backend-api"

@mcp.tool()
def consultar_logs_kql(consulta_kql: str, minutos_atras: int = 30) -> str:
    """
    Ejecuta una consulta KQL de solo lectura sobre el workspace de Log Analytics.
    Útil para buscar excepciones, errores HTTP 5xx o stack traces recientes.
    """
    try:
        timespan = timedelta(minutes=minutos_atras)
        response = logs_client.query_workspace(
            workspace_id=WORKSPACE_ID,
            query=consulta_kql,
            timespan=timespan
        )
        
        if response.status == LogsQueryStatus.SUCCESS:
            data = response.tables[0]
            if not data.rows:
                return "No se encontraron resultados para la consulta en el periodo indicado."
            
            # Formatear salida en formato texto legible para el LLM
            columns = [col.name for col in data.columns]
            results = []
            for row in data.rows[:15]:  # Limitamos a 15 filas para no saturar contexto
                row_dict = {col: str(val) for col, val in zip(columns, row)}
                results.append(row_dict)
            return str(results)
        else:
            return f"Error en la consulta KQL: {response.error}"
    except Exception as e:
        return f"Excepción ejecutando KQL: {str(e)}"

@mcp.tool()
def verificar_despliegues_recientes(limite: int = 5) -> str:
    """
    Obtiene los últimos despliegues y commits ejecutados en GitHub Actions.
    Permite correlacionar el incidente con un cambio reciente de código o infraestructura.
    """
    if not GITHUB_TOKEN or not GITHUB_REPO:
        return "Configuración de GitHub no disponible en las variables de entorno."
    
    url = f"https://api.github.com/repos/{GITHUB_REPO}/actions/runs?per_page={limite}&status=success"
    headers = {
        "Authorization": f"Bearer {GITHUB_TOKEN}",
        "Accept": "application/vnd.github+json"
    }
    
    resp = requests.get(url, headers=headers)
    if resp.status_code != 200:
        return f"Error consultando GitHub API: {resp.status_code}"
    
    runs = resp.json().get("workflow_runs", [])
    deploys = []
    for r in runs:
        deploys.append({
            "workflow": r.get("name"),
            "commit_msg": r.get("head_commit", {}).get("message"),
            "autor": r.get("head_commit", {}).get("author", {}).get("name"),
            "fecha": r.get("updated_at"),
            "html_url": r.get("html_url")
        })
    return str(deploys)

if __name__ == "__main__":
    mcp.run()
```

---

## Paso 2: El bucle de razonamiento del Agente SRE

El valor real no está solo en las herramientas, sino en las **instrucciones del sistema** que guían al agente para actuar como un ingeniero de fiabilidad sénior:

```text
ERES: Un Agente de Triage SRE especializado en infraestructuras Azure.
OBJETIVO: Investigar alertas de incidentes, correlacionar telemetría y generar un reporte de causa raíz probable.

REGLAS DE ACTUACIÓN:
1. Empieza consultando los logs de error en Log Analytics usando KQL estructurado (busca AppExceptions, HTTP 500, timeouts).
2. Comprueba inmediatamente los despliegues recientes de GitHub Actions para ver si coincide con la hora de inicio de la alerta.
3. Cruza la telemetría: ¿El error empezó exactamente tras el commit? ¿Qué microservicio o componente falló primero?
4. Genera un informe conciso con:
   - Resumen del impacto (blast radius).
   - Commit o cambio sospechoso (si aplica).
   - Stack trace / mensaje de error principal extraído de los logs.
   - 2 recomendaciones inmediatas de mitigación (ej: rollback, restart, escalado).
```

---

## Paso 3: Caso real en acción

Imaginemos que a las 03:02 AM salta una alerta de Azure Container Apps:
`Alerta: HTTP 500 rate > 20% en servicio "checkout-api"`

El agente entra en acción y ejecuta las herramientas de MCP:

1. **Llamada 1:** `consultar_logs_kql(query="ContainerAppConsoleLogs_CL | where TimeGenerated > ago(30m) | where Log_s has 'ERROR' or Log_s has 'Exception' | summarize count() by Log_s | top 5 by count_")`
   * *Resultado:* `Npgsql.NpgsqlException: Connection refused to redis-prod.internal.cloud.net:6379`
2. **Llamada 2:** `verificar_despliegues_recientes()`
   * *Resultado:* `Commit hace 22 minutos: "refactor: migrar variables de entorno de caché a Key Vault secret por @dev_user"`

### El reporte generado automáticamente por el Agente:

> ### 🚨 Informe de Triage Preliminar - Incidente Checkout-API
> - **Severidad:** Sev-1 (Impacto en usuarios activos)
> - **Causa Raíz Probable:** Fallo de conexión a la base de datos / caché Redis.
> - **Evidencia en Logs:** `NpgsqlException: Connection refused to redis-prod.internal.cloud.net:6379` (1.420 repeticiones en los últimos 20 min).
> - **Correlación de Cambios:** Despliegue completado a las 02:45 AM (commit `b8a92f1`: *"refactor: migrar variables de entorno de caché a Key Vault secret"*). Es muy probable que el secreto apunte a un host incorrecto o la identidad gestionada no tenga permisos `Key Vault Secrets User`.
> - **Acción de Mitigación Recomendada:** Ejecutar rollback del workflow `deploy-checkout` al release previo `v2.4.1` mientras se revisan los permisos en Azure Key Vault.

Cuando el ingeniero de guardia mira el móvil a las 03:04 AM, no tiene que empezar de cero: **la investigación ya está hecha**.

---

## Seguridad y Gobernanza: El principio de mínimo privilegio

Al dar acceso a un modelo de IA a nuestra infraestructura, la seguridad no es negociable:

1. **Acceso de solo lectura (Read-Only):**
   La identidad del servidor MCP (Managed Identity o Service Principal) solo debe tener el rol `Monitoring Reader` y `Log Analytics Reader` en Azure. **Nunca otorgues permisos de Contributor o Owner a un agente de triage.**
2. **Guardrails para consultas KQL:**
   Asegúrate de que la función MCP no acepte comandos que modifiquen datos o tablas (`.drop`, `.alter`).
3. **Humano en el bucle (Human-in-the-Loop):**
   El agente **sugiere el rollback**, pero la ejecución del comando o el merge del revert en Git siempre debe requerir confirmación humana mediante un botón interactivo en Slack o una aprobación de GitHub Environment.

---

## Conclusiones

La observabilidad tradicional nos dice *que* algo está roto; los Agentes SRE impulsados por MCP nos explican **por qué se rompió y qué cambio lo provocó**.

Construir este tipo de herramientas no requiere meses de desarrollo: con unas pocas funciones bien diseñadas en Python y un protocolo abierto como MCP, convertimos un proceso estresante de madrugada en un flujo de trabajo ágil, estructurado y gobernado.

¿Utilizas ya algún tipo de automatización con IA en tus guardias o prefieres seguir investigando a mano con KQL? ¡Te leo en comentarios!

