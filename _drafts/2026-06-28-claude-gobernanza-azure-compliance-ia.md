---
layout: post
title: "Claude como asistente de gobernanza: analiza el compliance de tu Landing Zone con IA"
image: "/assets/images/ai-governance-cover.png"
categories:
  - blog
tags:
  - Azure
  - IA
  - Claude
  - MCP
  - Azure Policy
  - Gobernanza
  - Landing Zone
---

En el artículo anterior montamos el servidor MCP que conecta Claude a nuestra infraestructura de Azure. Ahora vamos a ver para qué sirve realmente esa conexión en el día a día de un equipo de plataforma.

El caso de uso más inmediato y potente: usar Claude para **analizar el estado de compliance de tu Landing Zone en lenguaje natural**. Sin escribir queries KQL. Sin navegar por el portal. Sin esperar a que alguien prepare un informe. Una pregunta directa y una respuesta contextualizada.

## El problema que resuelve

Con la trilogía de gobernanza que publicamos las semanas anteriores tienes:
- Políticas definidas y asignadas en los Management Groups correctos
- Excepciones gestionadas como código con fecha de caducidad
- Un dashboard de Workbooks que muestra el compliance en tiempo real

Pero incluso con todo eso, hay un gap: **interpretar los datos**. Un dashboard te dice que hay 47 recursos non-compliant en Corp-Prod. No te dice cuáles son los más críticos, qué política están incumpliendo concretamente, ni qué acción deberías tomar primero.

Claude con MCP puede hacer esa interpretación por ti.

## Ampliando el servidor MCP con herramientas de análisis

Partiendo del servidor que construimos la semana pasada, añadimos dos herramientas nuevas orientadas a análisis de compliance:

```python
# tools/compliance.py

import subprocess
import json
from mcp.types import Tool, TextContent


def get_compliance_tools() -> list[Tool]:
    return [
        Tool(
            name="compliance_summary",
            description="Genera un resumen ejecutivo del estado de compliance de todas las suscripciones de una organización, agrupado por Management Group y política",
            inputSchema={
                "type": "object",
                "properties": {
                    "management_group_id": {
                        "type": "string",
                        "description": "ID del Management Group raíz de la organización"
                    },
                    "top_n": {
                        "type": "integer",
                        "description": "Número de políticas con más incumplimientos a mostrar (default: 10)"
                    }
                },
                "required": ["management_group_id"]
            }
        ),
        Tool(
            name="policy_exemptions_expiring",
            description="Lista las excepciones de política que caducan en los próximos N días",
            inputSchema={
                "type": "object",
                "properties": {
                    "days_ahead": {
                        "type": "integer",
                        "description": "Número de días a mirar hacia adelante (default: 30)"
                    },
                    "scope": {
                        "type": "string",
                        "description": "Scope a consultar (management group o subscription)"
                    }
                },
                "required": ["scope"]
            }
        ),
        Tool(
            name="resource_compliance_detail",
            description="Obtiene el detalle de compliance de un recurso específico: qué políticas incumple y cuál es el motivo exacto",
            inputSchema={
                "type": "object",
                "properties": {
                    "resource_id": {
                        "type": "string",
                        "description": "Resource ID completo del recurso a analizar"
                    }
                },
                "required": ["resource_id"]
            }
        )
    ]


async def compliance_summary(management_group_id: str, top_n: int = 10) -> list[TextContent]:
    """Agrega el estado de compliance por política en todo el MG."""

    cmd = [
        "az", "policy", "state", "summarize",
        "--management-group", management_group_id,
        "--output", "json"
    ]

    result = subprocess.run(cmd, capture_output=True, text=True)
    if result.returncode != 0:
        return [TextContent(type="text", text=f"Error: {result.stderr}")]

    data = json.loads(result.stdout)

    # Extraer y ordenar por número de recursos non-compliant
    policy_results = data.get("policyAssignments", [])
    non_compliant = []

    for pa in policy_results:
        results = pa.get("results", {})
        nc_count = results.get("nonCompliantResources", 0)
        if nc_count > 0:
            non_compliant.append({
                "policy": pa.get("policyAssignmentId", "").split("/")[-1],
                "nonCompliantResources": nc_count,
                "nonCompliantPolicies": results.get("nonCompliantPolicies", 0)
            })

    non_compliant.sort(key=lambda x: x["nonCompliantResources"], reverse=True)
    top = non_compliant[:top_n]

    total_nc = sum(p["nonCompliantResources"] for p in non_compliant)
    summary = f"## Compliance Summary - {management_group_id}\n\n"
    summary += f"**Total non-compliant resources:** {total_nc}\n"
    summary += f"**Policies with violations:** {len(non_compliant)}\n\n"
    summary += f"### Top {top_n} policies by violations:\n\n"

    for i, p in enumerate(top, 1):
        summary += f"{i}. **{p['policy']}** — {p['nonCompliantResources']} resources\n"

    return [TextContent(type="text", text=summary)]


async def policy_exemptions_expiring(scope: str, days_ahead: int = 30) -> list[TextContent]:
    """Lista excepciones próximas a caducar."""

    cmd = [
        "az", "policy", "exemption", "list",
        "--scope", scope,
        "--output", "json",
        "--query", "[].{name:name, expires:expiresOn, category:exemptionCategory, display:displayName}"
    ]

    result = subprocess.run(cmd, capture_output=True, text=True)
    if result.returncode != 0:
        return [TextContent(type="text", text=f"Error: {result.stderr}")]

    from datetime import datetime, timezone, timedelta
    data = json.loads(result.stdout)
    cutoff = datetime.now(timezone.utc) + timedelta(days=days_ahead)

    expiring = []
    for ex in data:
        if ex.get("expires"):
            try:
                exp_date = datetime.fromisoformat(ex["expires"].replace("Z", "+00:00"))
                days_left = (exp_date - datetime.now(timezone.utc)).days
                if days_left <= days_ahead:
                    expiring.append({**ex, "daysLeft": days_left})
            except Exception:
                pass

    expiring.sort(key=lambda x: x["daysLeft"])

    if not expiring:
        return [TextContent(type="text", text=f"✅ No exemptions expiring in the next {days_ahead} days.")]

    output = f"⚠️ **{len(expiring)} exemptions expiring in the next {days_ahead} days:**\n\n"
    for ex in expiring:
        status = "🔴" if ex["daysLeft"] <= 7 else "🟡"
        output += f"{status} **{ex['name']}** — {ex['daysLeft']} days left ({ex.get('category', 'N/A')})\n"
        output += f"   {ex.get('display', '')}\n\n"

    return [TextContent(type="text", text=output)]
```

## Conversaciones reales con tu infraestructura

Con estas herramientas integradas en el servidor MCP, estas son conversaciones reales que puedes mantener con Claude:

---

**Tú:** ¿Cuál es el estado de compliance general de la Landing Zone?

**Claude:** He consultado el estado de compliance del Management Group `mg-contoso`. Aquí el resumen:

- **Total de recursos non-compliant:** 143
- **Políticas con violaciones:** 8

Las 3 políticas con más incumplimientos son:
1. `require-mandatory-tags` — 89 recursos sin las etiquetas obligatorias
2. `allowed-azure-regions` — 31 recursos en regiones no permitidas
3. `require-diagnostic-logs` — 23 recursos sin logs de diagnóstico

La mayor concentración está en la suscripción `corp-prod`. ¿Quieres que profundice en alguna de estas políticas?

---

**Tú:** ¿Hay excepciones que caduquen pronto?

**Claude:** He revisado las excepciones activas. Hay 2 que caducan en los próximos 30 días:

🔴 **legacy-app-network-exemption** — caduca en 6 días (Mitigated)
"Excepción temporal de red para App Contabilidad. Migración en curso."

🟡 **sandbox-location-waiver** — caduca en 18 días (Waiver)
"MG Sandbox necesita acceso a todas las regiones para PoCs."

La primera es urgente: si caduca sin renovación o sin completar la migración, la App de Contabilidad quedará non-compliant automáticamente. ¿Quieres que revise cuál es el estado actual de la migración?

---

**Tú:** ¿Qué recursos en Corp-Prod no tienen la etiqueta `owner`?

**Claude:** He consultado los recursos de la suscripción Corp-Prod filtrando por la política `require-mandatory-tags`. Hay 34 recursos sin la etiqueta `owner`. Los tipos más frecuentes son:

- 18 Virtual Machines
- 9 Storage Accounts
- 7 Key Vaults

¿Quieres que genere un script de Azure CLI para añadir la etiqueta a todos ellos con un valor por defecto para revisión posterior?

---

## Por qué esto es diferente a un dashboard

Un dashboard de Workbooks te muestra los datos. Claude los **interpreta**. La diferencia es sutil pero enorme en la práctica:

- El dashboard te dice "89 recursos non-compliant con require-mandatory-tags"
- Claude te dice "la mayoría son VMs en Corp-Prod sin etiqueta `owner`, que probablemente son recursos legacy de antes de implantar la política, y te sugiere el comando para corregirlo"

El modelo tiene contexto de todo lo que le has explicado antes: qué políticas tienes, cómo funciona tu jerarquía de MGs, cuáles son las excepciones activas. Eso convierte cada respuesta en algo accionable, no solo informativo.

## Conclusión

Conectar Claude a tu stack de gobernanza de Azure no es un experimento de laboratorio: es una herramienta práctica que el equipo de plataforma puede usar desde el primer día para reducir el tiempo que pasa navegando portales y buscando en dashboards.

En el próximo artículo cerramos la serie con el caso más avanzado: **un agente que detecta drift de forma autónoma, propone el fix en Terraform y abre un Pull Request** para que el equipo lo revise y apruebe.
