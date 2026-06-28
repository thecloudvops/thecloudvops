---
layout: post
title: "MCP (Model Context Protocol): conecta Claude a tu infraestructura de Azure"
image: "/assets/images/mcp-azure-cover.png"
categories:
  - blog
tags:
  - Azure
  - IA
  - Claude
  - MCP
  - DevOps
  - Terraform
  - Automatización
---

> 💡 **Serie: Gobernanza de Agentes de IA en Azure**
> Este artículo es la primera entrega de una serie de 3 partes dedicada a conectar, analizar y automatizar la gobernanza de tu infraestructura en Azure utilizando Inteligencia Artificial:
> 
> 1. **Parte 1 (este artículo):** [MCP (Model Context Protocol): conecta Claude a tu infraestructura de Azure](/blog/mcp-claude-azure-infraestructura/)
> 2. **Parte 2:** [Claude como asistente de gobernanza: analiza el compliance de tu Landing Zone con IA](/blog/claude-gobernanza-azure-compliance-ia/)
> 3. **Parte 3:** Agentes IA en tu pipeline de Terraform: de la detección de drift al Pull Request automático *(Disponible el 5 de julio)*

Durante los últimos meses hemos construido una Landing Zone gobernada como código: políticas, excepciones, RBAC y detección de drift. Todo versionado en Git, revisado en Pull Requests, reproducible y auditable.

Ahora viene la siguiente capa: **darle a esa infraestructura un copiloto de IA que la entienda y pueda actuar sobre ella**.

No estamos hablando de un chatbot que te explica qué es un Management Group. Estamos hablando de un agente que puede ejecutar `terraform plan`, consultar el estado de compliance de Azure Policy, revisar el historial de Git de tu infraestructura y darte una respuesta basada en el contexto real de tu entorno. En lenguaje natural.

Eso es lo que hace posible el **Model Context Protocol (MCP)**.

## Qué es MCP y por qué importa

MCP es un protocolo abierto diseñado por Anthropic que define cómo un modelo de lenguaje (como Claude) puede conectarse de forma estandarizada a herramientas externas: APIs, bases de datos, sistemas de ficheros, CLIs.

Antes de MCP, integrar un LLM con tus herramientas requería código custom para cada integración. Con MCP, defines un **servidor MCP** que expone un conjunto de "herramientas" (tools), y cualquier cliente compatible (Claude Desktop, Claude API, o tu propio agente) puede usar esas herramientas de forma nativa.

La analogía con el ecosistema de datos es perfecta: MCP es para los LLMs lo que ODBC fue para las bases de datos. Un estándar que permite a cualquier modelo conectarse a cualquier herramienta sin reimplementar la integración cada vez.

```
Claude (cliente MCP)
    ↓ MCP protocol
Servidor MCP (tu código)
    ↓ llama a
Herramientas: Azure CLI, Terraform, GitHub API, Log Analytics...
```

## Arquitectura: un servidor MCP para tu Landing Zone

Vamos a construir un servidor MCP en Python que expone las herramientas más útiles para gestionar una Landing Zone de Azure:

```
mcp-azure-landing-zone/
├── server.py              → servidor MCP principal
├── tools/
│   ├── policy.py          → herramientas de Azure Policy
│   ├── terraform.py       → herramientas de Terraform
│   ├── resources.py       → consultas de recursos Azure
│   └── compliance.py      → estado de compliance
├── requirements.txt
└── config.yaml
```

## Paso 1: Instalar el SDK de MCP

```bash
pip install mcp anthropic azure-cli-core azure-mgmt-policyinsights
```

## Paso 2: Crear el servidor MCP

```python
# server.py

import asyncio
import subprocess
import json
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

app = Server("azure-landing-zone")

@app.list_tools()
async def list_tools() -> list[Tool]:
    """Expone las herramientas disponibles al cliente MCP."""
    return [
        Tool(
            name="az_policy_state",
            description="Consulta el estado de compliance de Azure Policy para una suscripción o Management Group",
            inputSchema={
                "type": "object",
                "properties": {
                    "scope": {
                        "type": "string",
                        "description": "Scope a consultar: subscription ID o management group ID"
                    },
                    "filter": {
                        "type": "string",
                        "description": "Filtro opcional, por ejemplo: 'complianceState eq NonCompliant'"
                    }
                },
                "required": ["scope"]
            }
        ),
        Tool(
            name="terraform_plan",
            description="Ejecuta terraform plan en el directorio de infraestructura y devuelve el resumen de cambios",
            inputSchema={
                "type": "object",
                "properties": {
                    "working_dir": {
                        "type": "string",
                        "description": "Directorio donde ejecutar terraform plan"
                    },
                    "var_file": {
                        "type": "string",
                        "description": "Fichero de variables tfvars a usar (opcional)"
                    }
                },
                "required": ["working_dir"]
            }
        ),
        Tool(
            name="az_resource_list",
            description="Lista recursos de Azure en un scope dado, con filtros opcionales por tipo o etiqueta",
            inputSchema={
                "type": "object",
                "properties": {
                    "subscription_id": {"type": "string"},
                    "resource_type": {
                        "type": "string",
                        "description": "Tipo de recurso, por ejemplo: Microsoft.Compute/virtualMachines"
                    },
                    "tag_filter": {
                        "type": "string",
                        "description": "Filtro por etiqueta, por ejemplo: environment=prod"
                    }
                },
                "required": ["subscription_id"]
            }
        ),
        Tool(
            name="az_mg_hierarchy",
            description="Obtiene la jerarquía de Management Groups de la organización",
            inputSchema={
                "type": "object",
                "properties": {},
                "required": []
            }
        )
    ]

@app.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    """Ejecuta la herramienta solicitada y devuelve el resultado."""

    if name == "az_policy_state":
        return await _az_policy_state(arguments)
    elif name == "terraform_plan":
        return await _terraform_plan(arguments)
    elif name == "az_resource_list":
        return await _az_resource_list(arguments)
    elif name == "az_mg_hierarchy":
        return await _az_mg_hierarchy(arguments)
    else:
        return [TextContent(type="text", text=f"Tool '{name}' not found")]


async def _az_policy_state(args: dict) -> list[TextContent]:
    scope = args["scope"]
    filter_expr = args.get("filter", "complianceState eq 'NonCompliant'")

    cmd = [
        "az", "policy", "state", "list",
        "--scope", scope,
        "--filter", filter_expr,
        "--query", "[].{resource:resourceId, policy:policyDefinitionName, state:complianceState}",
        "--output", "json"
    ]

    result = subprocess.run(cmd, capture_output=True, text=True)

    if result.returncode != 0:
        return [TextContent(type="text", text=f"Error: {result.stderr}")]

    data = json.loads(result.stdout)
    summary = f"Found {len(data)} non-compliant resources\n\n"
    summary += json.dumps(data[:20], indent=2)  # Limitar a 20 resultados

    return [TextContent(type="text", text=summary)]


async def _terraform_plan(args: dict) -> list[TextContent]:
    working_dir = args["working_dir"]
    var_file = args.get("var_file", "")

    cmd = ["terraform", "plan", "-no-color", "-compact-warnings"]
    if var_file:
        cmd += [f"-var-file={var_file}"]

    result = subprocess.run(cmd, capture_output=True, text=True, cwd=working_dir)

    # Extraer solo el resumen del plan
    output = result.stdout + result.stderr
    lines = output.split('\n')
    relevant = [l for l in lines if any(
        keyword in l for keyword in ['Plan:', 'will be', 'must be', 'No changes', '#']
    )]

    return [TextContent(type="text", text='\n'.join(relevant[:50]))]


async def _az_resource_list(args: dict) -> list[TextContent]:
    sub_id = args["subscription_id"]
    resource_type = args.get("resource_type", "")
    tag_filter = args.get("tag_filter", "")

    cmd = ["az", "resource", "list",
           "--subscription", sub_id,
           "--output", "json",
           "--query", "[].{name:name, type:type, location:location, tags:tags}"]

    if resource_type:
        cmd += ["--resource-type", resource_type]
    if tag_filter:
        cmd += ["--tag", tag_filter]

    result = subprocess.run(cmd, capture_output=True, text=True)

    if result.returncode != 0:
        return [TextContent(type="text", text=f"Error: {result.stderr}")]

    data = json.loads(result.stdout)
    return [TextContent(type="text", text=f"Found {len(data)} resources\n\n{json.dumps(data[:15], indent=2)}")]


async def _az_mg_hierarchy(args: dict) -> list[TextContent]:
    cmd = ["az", "account", "management-group", "list", "--output", "json",
           "--query", "[].{name:name, displayName:displayName, parent:details.parent.id}"]

    result = subprocess.run(cmd, capture_output=True, text=True)

    if result.returncode != 0:
        return [TextContent(type="text", text=f"Error: {result.stderr}")]

    return [TextContent(type="text", text=result.stdout)]


async def main():
    async with stdio_server() as (read_stream, write_stream):
        await app.run(read_stream, write_stream, app.create_initialization_options())

if __name__ == "__main__":
    asyncio.run(main())
```

## Paso 3: Configurar Claude Desktop para usar el servidor

Una vez que tienes el servidor, configurarlo en Claude Desktop es inmediato. Edita `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "azure-landing-zone": {
      "command": "python",
      "args": ["/ruta/a/tu/mcp-azure-landing-zone/server.py"],
      "env": {
        "AZURE_SUBSCRIPTION_ID": "tu-subscription-id",
        "AZURE_TENANT_ID": "tu-tenant-id"
      }
    }
  }
}
```

Reinicia Claude Desktop. Verás el icono de herramientas activado. Desde ese momento puedes escribir en lenguaje natural:

> *"¿Cuántos recursos no cumplen las políticas en la suscripción de Corp-Prod?"*

Y Claude ejecutará `az_policy_state` automáticamente, interpretará los resultados y te dará una respuesta estructurada con el resumen de incumplimientos.

## Lo que puedes hacer a partir de aquí

Con el servidor MCP conectado, las posibilidades son inmediatas:

- **"¿Hay drift en la infraestructura?"** → Claude ejecuta `terraform plan` y te explica qué ha cambiado
- **"¿Qué VMs en producción no tienen la etiqueta owner?"** → Consulta `az_resource_list` con filtros
- **"¿Cómo está el compliance general de la Landing Zone?"** → Agrega datos de policy state por suscripción
- **"Genera un informe de los recursos fuera de regla esta semana"** → Combina múltiples herramientas en una sola consulta

## Conclusión

MCP no es una feature más de Claude: es la pieza que convierte un modelo de lenguaje en un **agente que puede trabajar con tu infraestructura real**. Y lo hace de forma estandarizada, sin necesidad de un middleware propietario ni de integrar webhooks manualmente.

En los próximos artículos vamos a ver cómo usar este servidor MCP para analizar el compliance de la Landing Zone en lenguaje natural y, después, cómo construir un agente que detecte drift y proponga fixes automáticamente en forma de Pull Request.

La IA que opera sobre infraestructura real, con contexto real, ya no es ciencia ficción. Es lo que puedes tener funcionando esta tarde.
